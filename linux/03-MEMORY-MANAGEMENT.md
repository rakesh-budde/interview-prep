# Section 3: Memory Management

This section covers how Linux manages physical and virtual memory — address space layout, paging,
allocators, the page cache, swap, NUMA, and the OOM killer — the material behind nearly every "why is
memory usage high" or "why did the OOM killer fire" production incident.

## Subtopic Index
- [Physical vs Virtual Memory](#physical-vs-virtual-memory)
- [Virtual Address Space Layout (text, data, heap, stack, mmap region)](#virtual-address-space-layout-text-data-heap-stack-mmap-region)
- [Paging and Page Tables](#paging-and-page-tables)
- [Multi-level Page Tables and TLB](#multi-level-page-tables-and-tlb)
- [Page Fault Handling (minor/major faults)](#page-fault-handling-minormajor-faults)
- [Demand Paging](#demand-paging)
- [Copy-on-Write Pages](#copy-on-write-pages)
- [Buddy Allocator](#buddy-allocator)
- [Slab/Slub/Slob Allocators](#slabslubslob-allocators)
- [Kernel vs User Memory](#kernel-vs-user-memory)
- [mmap() Internals](#mmap-internals)
- [brk() vs mmap() for heap growth](#brk-vs-mmap-for-heap-growth)
- [Memory Overcommit and OOM Killer](#memory-overcommit-and-oom-killer)
- [OOM Score and oom_score_adj](#oom-score-and-oom_score_adj)
- [Swap Space and Swappiness](#swap-space-and-swappiness)
- [Huge Pages (Transparent Huge Pages)](#huge-pages-transparent-huge-pages)
- [Page Cache and Buffer Cache](#page-cache-and-buffer-cache)
- [Dirty Page Writeback](#dirty-page-writeback)
- [NUMA Architecture](#numa-architecture)
- [Memory Cgroups and Limits](#memory-cgroups-and-limits)

---

## Physical vs Virtual Memory

Physical memory is the actual DRAM installed in the machine, addressed by physical frame numbers that
the memory controller understands directly. Virtual memory is an abstraction the kernel presents to
every process: each process believes it has its own private, contiguous, enormous address space
(2^48 or 2^57 bytes of addressable range on modern x86_64, depending on paging mode), when in reality
that address space is a sparse set of mappings translated on every single memory access into physical
frames by the MMU (Memory Management Unit) consulting per-process page tables. This indirection is
what enables several properties taken for granted today: isolation (process A cannot address process
B's physical memory because A's page tables simply have no translation for it — there's no bounds
check to bypass, the address literally doesn't resolve to anything in A's context), overcommitment
(the kernel can promise more virtual address space than physical RAM actually exists, backing pages
with real frames lazily only when touched), and relocation transparency (a process's code/data can be
placed at different physical addresses on every run — including deliberately randomized via ASLR —
without the process's own instructions needing to change, since they only ever reference virtual
addresses). The translation from virtual to physical happens via hardware page tables, cached in the
TLB for performance, and every memory access a CPU core issues passes through this translation
invisibly; the kernel's job is to construct and maintain the page tables so translations point at
correct, appropriately-protected physical frames, and to handle it via a page fault whenever a virtual
address has no valid translation yet or the access violates the entry's permission bits.

### Key commands
```
cat /proc/meminfo              # system-wide physical memory breakdown
free -h                         # human-readable physical + swap summary
cat /proc/<pid>/maps             # this process's virtual address space layout
pmap -x <pid>                    # per-mapping resident/dirty memory for a process
```

## Virtual Address Space Layout (text, data, heap, stack, mmap region)

A typical Linux process's virtual address space is organized into distinct regions, each backed by a
Virtual Memory Area (`vm_area_struct`, "VMA") describing its permissions and backing source. At the
low end sits the text segment (the executable's compiled machine code, mapped read-only and
executable directly from the binary file on disk, shared read-only across every process running that
same binary). Just above it, the data segment holds initialized global/static variables (mapped
read-write, copy-on-write from the binary's data section) and the BSS segment holds zero-initialized
globals (backed by anonymous zero-fill-on-demand pages, not any file content). Above BSS, the heap
grows upward via `brk()`/`sbrk()` (or increasingly via `mmap()` for larger allocations, discussed
below) as the program calls `malloc()` and its underlying allocator requests more memory from the
kernel. A large middle region is reserved for the memory-mapped segment, where shared libraries
(`.so` files, mapped similarly to the executable's own text/data), anonymous `mmap()` allocations, and
explicit file mappings live; because ASLR randomizes this region's base address, shared library
addresses differ between runs and even between processes, which is a deliberate security mitigation
against attacks relying on predictable code addresses. Near the top of user-space address range, the
stack grows *downward* from a high address, holding function call frames, local variables, and return
addresses, with a guard region below it that triggers a fault (rather than silently corrupting the
heap) if the stack grows too large. Beyond the top of user-space entirely lies the kernel's own
portion of the address space — on x86_64 with a typical split, the upper half of the full 64-bit range
is reserved for kernel mappings, shared identically across every process's page tables (with
protection bits ensuring userspace code cannot read or execute it), which is exactly the region that
Meltdown-style attacks exploited before KPTI (Kernel Page Table Isolation) largely unmapped kernel
addresses from user-mode page tables entirely as a mitigation.

```
High addr  ┌───────────────────────┐
           │   Kernel space         │  (shared across all processes, protected from user-mode access)
           ├───────────────────────┤
           │   Stack (grows down)   │  ← function frames, locals, return addrs
           │        ...              │
           │   mmap region           │  ← shared libs, anonymous mmap, file mappings (ASLR-randomized)
           │        ...              │
           │   Heap (grows up)       │  ← malloc()'d memory via brk()/mmap()
           ├───────────────────────┤
           │   BSS (zero-fill)       │  ← uninitialized globals
           │   Data (initialized)    │  ← initialized globals/statics
           │   Text (code, r-x)      │  ← compiled program code, shared read-only
Low addr   └───────────────────────┘
```

### Key commands
```
cat /proc/<pid>/maps            # every VMA: address range, perms, backing file/anon, offset
cat /proc/<pid>/smaps            # per-VMA detail: RSS, PSS, Shared/Private Clean/Dirty
pmap -X <pid>                     # extended per-mapping memory accounting, human-friendly
readelf -l <binary>                # program headers showing how segments are laid out in the ELF file
```

## Paging and Page Tables

Paging divides both virtual and physical address space into fixed-size chunks called pages (4KB by
far the most common size on x86_64, with optional larger huge pages discussed later), and a page
table is the per-process data structure the kernel maintains and the hardware MMU walks to translate a
virtual page number into a physical frame number, plus permission bits (readable, writable,
executable, user/supervisor, cacheable). Rather than a flat array (which would be enormous and mostly
empty/wasteful for a sparse address space), x86_64 uses a multi-level, radix-tree-like page table
structure: a virtual address is split into fields that index successive levels — PML4 (top level, one
per process, pointed to by the CR3 register), PDPT, PD, and PT (leaf level, actually holding the
physical frame number for a 4KB page) — with each level's entry either pointing to the next level's
table or, for huge pages, terminating early at a higher level to directly map a larger contiguous
region. On a memory access, the MMU performs a "page table walk": it reads CR3 to find the PML4 table,
indexes into it using bits from the virtual address to find the PDPT table's physical address, repeats
for PD and PT, and finally reads the physical frame number plus permission bits from the PT entry —
four sequential memory reads for a single 4KB translation, which would be prohibitively slow for every
memory access without the TLB caching recent translations (see below). Every process has its own
independent set of these tables for user-space addresses (which is what provides isolation — process
A's page tables simply have no valid entries pointing at process B's physical frames), while the
kernel-space portion of the tables is typically shared/kept synchronized across every process's tables
since kernel code and data must be reachable identically regardless of which process happens to be
running when a syscall or interrupt occurs.

### Key commands
```
cat /proc/<pid>/status | grep VmPTE     # kernel memory consumed by this process's own page tables
cat /proc/meminfo | grep -i pagetables   # system-wide page-table memory overhead
x86info / cpuid                          # inspect CPU paging mode support (PAE, 4/5-level paging)
```

## Multi-level Page Tables and TLB

The Translation Lookaside Buffer is a small, extremely fast, per-core hardware cache of recent
virtual-to-physical translations, existing specifically to avoid paying the cost of a full multi-level
page table walk on every single memory access — without it, every load/store instruction would incur
the equivalent of four dependent memory reads just to resolve the address, an unacceptable slowdown.
A TLB hit resolves a translation in effectively zero extra cycles; a TLB miss forces the CPU's
hardware page-table walker (or, on some older/RISC architectures, a software-handled trap) to perform
the full multi-level walk described above and then caches the result in the TLB for next time. Because
the TLB is small (typically tens to a few thousand entries depending on cache level), it can only hold
a limited number of hot translations, which is exactly why large, sparse, or randomly-accessed working
sets suffer "TLB thrashing" — constant misses as translations are evicted before being reused — a real
performance bottleneck for large in-memory databases or workloads with poor locality, and precisely
the problem huge pages solve by making each TLB entry cover far more memory (2MB or 1GB instead of
4KB), multiplying the effective reach of the same fixed number of TLB entries. TLB entries are tagged
per address-space in modern CPUs via PCID (Process-Context Identifiers, x86) or ASID (ARM), letting
the CPU keep multiple processes' translations resident in the TLB simultaneously without needing a
full flush on every context switch — before this feature existed (or when disabled, as historically
required by some Meltdown mitigations before more targeted fixes landed), every process context switch
had to flush the entire TLB, forcing every subsequent memory access after a switch to pay full
page-walk cost until the new process's working translations were re-populated, a substantial and
measurable overhead on context-switch-heavy workloads.

### Key commands
```
perf stat -e dTLB-load-misses,iTLB-load-misses ./program   # measure TLB miss rates for a workload
cat /proc/cpuinfo | grep pcid                                # confirm PCID hardware support
perf record -e dTLB-load-misses ./program && perf report      # attribute TLB misses to code locations
```

## Page Fault Handling (minor/major faults)

A page fault is a CPU exception raised by the MMU whenever a memory access cannot be completed by the
current page table state — either because there is no valid translation at all for that virtual
address, or because the access violates the entry's permission bits (writing to a read-only page,
executing a non-executable page). Far from always being an error, page faults are a core, expected
mechanism the kernel deliberately relies on to implement lazy memory management. A minor (soft) fault
occurs when the virtual address is legitimately mapped in the process's VMA but has no page table
entry yet or the underlying physical page is already resident in memory for another reason (e.g., it's
already in the page cache from another process's identical file mapping, or it's a copy-on-write page
still shared and just needs its reference-counted mapping established) — the kernel's fault handler
(`handle_mm_fault()` → architecture-specific fault entry → `do_fault()`/`do_wp_page()`/etc.) resolves
this quickly, often without ever touching a block device, just updating page tables and reference
counts. A major (hard) fault occurs when the needed data genuinely isn't resident in RAM at all and
must be fetched from a block device — reading in a page of a memory-mapped file not yet cached, or
reading back a swapped-out anonymous page from swap space — which involves issuing actual I/O and
blocking the faulting process (transitioning it to `D`/uninterruptible sleep) until the read completes,
making major faults orders of magnitude more expensive than minor faults and a direct, measurable
contributor to application-perceived latency. The full path for a typical fault: the CPU traps into
the kernel with the faulting address (CR2 register on x86) and an error code describing the access
type; the kernel looks up which VMA (if any) covers that address; if none covers it, it's a genuine
invalid access and the process receives `SIGSEGV`; if a VMA covers it, the kernel determines the
correct resolution (allocate and zero a new anonymous page, fetch a file-backed page from the page
cache or issue I/O to populate it, perform a copy-on-write duplication, or swap in a page) updates the
page table entry accordingly, and returns to re-execute the faulting instruction, which now succeeds
transparently — the faulting program has no idea a fault even occurred.

### Key commands
```
ps -o min_flt,maj_flt -p <pid>       # cumulative minor/major fault counts for a process
/usr/bin/time -v ./program            # major/minor page faults reported for a full run
perf stat -e minor-faults,major-faults ./program   # live fault-rate measurement
strace -e trace=%memory <cmd>          # observe mmap/brk/page-fault-adjacent syscalls
```

## Demand Paging

Demand paging is the strategy of never loading a page into physical memory until the very moment it's
actually accessed, rather than eagerly loading an entire program or file up front. When a process
`execve()`s a new binary, the kernel does not read the entire executable file into memory immediately
— it maps the executable's segments into the process's address space as file-backed VMAs and returns
almost instantly, and only the pages the program actually touches during execution get faulted in on
demand, one page at a time, as minor or major faults. This is precisely why large programs start
quickly despite their on-disk size being large — a program's rarely-executed error-handling code paths
may never fault in their backing pages at all during a typical run — and why the very first access to
a given code or data page is measurably slower than subsequent accesses (paying the major-fault cost
once, then benefiting from the page remaining resident and eventually TLB-cached). The same principle
applies to `mmap()`ed files in general: mapping a multi-gigabyte file is a cheap, near-instantaneous
operation because it only establishes VMA bookkeeping, not actual I/O, and pages are pulled in lazily
exactly as the program reads/writes specific offsets within the mapping. Demand paging interacts
directly with memory overcommit: because pages are only actually backed by physical frames when
touched, the kernel can allow a process to reserve (map) far more virtual address space than physical
RAM exists, on the optimistic assumption that not every mapped page will be touched simultaneously —
this optimism is exactly what memory overcommit policy (see below) governs, and it's what makes the
OOM killer necessary as a last resort when that optimism turns out to be wrong and every promised page
really is being actively used at once.

### Key commands
```
cat /proc/<pid>/smaps_rollup           # total RSS/PSS across all mappings — how much is actually resident
strace -e trace=mmap ./program          # confirm mmap returns quickly regardless of mapped file size
```

## Copy-on-Write Pages

(See Section 2's Copy-on-Write entry for the `fork()`-centric explanation; this entry focuses on COW
as a general memory-management mechanism beyond process creation.) Copy-on-write is a general
optimization pattern applied anywhere the kernel can defer an expensive duplication until it's
provably necessary: `fork()`'s address-space duplication, `MAP_PRIVATE` file mappings (multiple
processes mapping the same file read-only share physical pages until one writes, at which point only
that process gets a private copy), and even some `mmap(MAP_ANONYMOUS)` patterns interacting with
`madvise()` hints. The shared mechanism is always the same: mark the shared page's page-table entry
read-only regardless of the mapping's "logical" writability, so any write attempt traps into the
kernel's page-fault handler, which recognizes the specific combination of "this VMA is logically
writable but the PTE is marked read-only for COW reasons" (distinct from a genuine permission
violation which would instead deliver `SIGSEGV`), allocates a new physical page, copies the original
content, remaps only the faulting process's translation to the new private page, and decrements the
shared page's reference count. The performance implication cuts both ways: COW makes read-heavy,
rarely-modified shared memory extremely cheap (many processes genuinely sharing the same physical
pages, visible in `/proc/<pid>/smaps` as high "Shared" byte counts), but a workload that touches a
large COW-shared region heavily right after a `fork()` (common in poorly-designed pre-fork server
architectures that both fork *and* immediately mutate large shared data structures) pays a burst of
page-fault-driven copy costs concentrated right after the fork, which can look like a mysterious
latency spike immediately following process creation if you don't know to attribute it to COW
resolution.

### Key commands
```
cat /proc/<pid>/smaps | grep -E 'Shared_Clean|Shared_Dirty|Private_Clean|Private_Dirty'   # COW-relevant breakdown
perf record -e page-faults -g ./program && perf report   # attribute COW fault bursts to specific call sites
```

## Buddy Allocator

The buddy allocator is the kernel's low-level physical-page allocator, responsible for satisfying
requests for contiguous runs of physical page frames (`alloc_pages()`), and its defining property is
efficient splitting and coalescing to combat external fragmentation. Free memory is tracked in
per-zone free lists organized by "order" — order 0 is a single 4KB page, order 1 is a 2-page (8KB)
block, order 2 is 4 pages (16KB), up to a maximum order (typically order 10, i.e. 4MB blocks) — with
each order's free list holding only blocks of exactly that size, naturally aligned to their own size
boundary. When a request for an order-N block arrives and none is free at that order, the allocator
looks at the next-higher order (N+1), and if a block is available there, it splits it in half — one
half satisfies the request (or is further split if still too large), the other half ("buddy") is
placed onto the order-N free list — recursively repeating from higher orders as needed until the
request is satisfied. Freeing works in reverse and is where the name comes from: when a block is
freed, the allocator computes its "buddy" address (found via a simple XOR of the block's address with
its size, since buddies are always adjacent, size-aligned pairs) and checks whether that buddy is
also currently free; if so, the two are merged ("coalesced") back into a single free block at the
next-higher order, and this coalescing repeats recursively upward as long as buddies keep turning out
to be free, actively fighting fragmentation by keeping free memory consolidated into larger blocks
whenever possible rather than left as many small, non-contiguous fragments. The buddy allocator's
practical limitation is that it can only ever hand out power-of-two-sized, physically contiguous
blocks — perfectly efficient for page-sized-and-larger allocations (used directly by the page cache,
huge pages, and DMA buffers needing physical contiguity), but wasteful and slow for the vast number of
small, sub-page kernel object allocations (a few hundred bytes for an inode, a task_struct, a network
packet buffer) that the kernel needs constantly — which is exactly the gap the slab/slub allocator,
built on top of the buddy allocator, exists to fill.

### Key commands
```
cat /proc/buddyinfo                # free block counts per order, per zone, per NUMA node
cat /proc/pagetypeinfo               # fragmentation detail by migrate type (movable/unmovable/reclaimable)
cat /proc/zoneinfo                   # detailed per-zone memory statistics
```

## Slab/Slub/Slob Allocators

The slab allocator sits above the buddy allocator specifically to efficiently serve the kernel's
enormous volume of small, fixed-size, frequently allocated/freed object types — `task_struct`s,
`inode`s, network socket buffers, dentries — where going through the buddy allocator's page-granularity
interface for every single small object would waste huge amounts of memory (internal fragmentation
from rounding every allocation up to a full page) and burn CPU cycles on repeated initialization
(zeroing/setting up complex structures from scratch every time). Instead, the slab allocator requests
whole pages from the buddy allocator in bulk ("slabs"), carves each slab into an array of same-sized
object slots matching a specific "kmem_cache" (one cache per distinct object type/size class — you can
see hundreds of them, one per structure type, in `/proc/slabinfo`), and maintains free lists of
ready-to-use object slots within each slab, so allocating a new `task_struct` is typically just
popping a pre-sized, often even pre-initialized ("constructor" callback run once at slab creation, not
per-allocation) slot off a free list — dramatically cheaper than a raw buddy allocation plus manual
initialization. SLUB (the default in modern kernels, "the unqueued slab allocator") is a simplified,
more cache-friendly reimplementation of the original SLAB allocator, removing several per-CPU queueing
layers that added complexity and cache-line contention, while SLOB (Simple List Of Blocks) is a
minimal, low-memory-overhead allocator intended for severely memory-constrained embedded systems,
trading allocation speed and fragmentation resistance for the smallest possible bookkeeping overhead —
virtually no production server kernel uses SLOB today, but it's a valid answer if asked to name all
three historically-available slab allocator implementations. Slab memory is directly visible and often
a surprisingly large fraction of "used" memory on a busy server — heavy filesystem metadata activity
(lots of inodes/dentries cached), for instance, can consume gigabytes of slab memory, which is
reclaimable under pressure but shows up separately from both application RSS and the page cache in
memory accounting, a frequent source of "where did my memory go" confusion when only looking at `free
-h`'s top-level numbers.

### Key commands
```
cat /proc/slabinfo | sort -k3 -n -r | head    # largest slab caches by total memory consumed
slabtop                                        # live, top-like view of slab cache usage
cat /proc/meminfo | grep -i slab                # total reclaimable + unreclaimable slab memory
```

## Kernel vs User Memory

Every process's virtual address space is split between a user-accessible region (where the process's
own code, data, heap, stack, and mmap'd regions live, subject to normal permission checks) and a
kernel-only region mapped identically (same physical backing) into *every* process's page tables but
protected so user-mode code cannot read, write, or execute it — this shared kernel mapping exists
specifically so that a syscall or interrupt doesn't require switching to an entirely separate address
space just to run kernel code, avoiding a full TLB flush on every single syscall entry/exit. On classic
x86_64 Linux, this split reserved the upper portion of the 48-bit canonical address range exclusively
for the kernel, leaving the lower portion for user-space — but this shared-mapping design is exactly
what the Meltdown vulnerability exploited: a malicious user-mode process could use CPU speculative
execution to transiently read kernel-space memory that was mapped (for performance) into its own page
tables despite permission bits nominally forbidding it, since certain CPUs didn't correctly enforce the
permission check before speculatively executing dependent instructions whose timing side-effects could
be measured to infer the "forbidden" data. KPTI (Kernel Page Table Isolation), the mitigation, largely
separates user-mode and kernel-mode page tables into two nearly-disjoint sets per process, switching
CR3 (paying a real, measurable performance cost, particularly for syscall-heavy workloads) on every
kernel entry/exit specifically so that even a successful speculative read cannot access kernel memory
that plainly isn't mapped in the user-mode table set at all anymore. Beyond this security dimension,
"kernel memory" also refers more broadly to memory the kernel allocates and manages internally
(slab caches, kernel stacks — a small, fixed-size per-thread stack used while executing in kernel mode
via syscalls/interrupts, distinct from the much larger user-mode stack — and page tables themselves),
none of which is swappable or directly visible to the owning process's own memory accounting, which is
why kernel memory pressure (from too many open files, too many processes, excessive dentry/inode
caching) can starve a system even when application-level `top`/`ps` numbers look unremarkable.

### Key commands
```
cat /proc/meminfo | grep -E 'KernelStack|SUnreclaim|PageTables'   # kernel-internal memory consumption
dmesg | grep -i kpti                 # confirm whether KPTI mitigation is active on this kernel/CPU
cat /sys/devices/system/cpu/vulnerabilities/meltdown   # kernel's own assessment of Meltdown exposure/mitigation
```

## mmap() Internals

`mmap()` is the syscall that establishes a new Virtual Memory Area in a process's address space,
either backed by a file (so reads/writes to the mapped region transparently read/write the underlying
file through the page cache, with changes visible to other processes mapping the same file and
optionally persisted back to disk) or anonymous (backed by nothing but zero-fill-on-demand pages,
used both directly by application code wanting large allocations and internally by `malloc()`
implementations for big requests). A file-backed `MAP_SHARED` mapping means multiple processes mapping
the same file region literally share the same physical page-cache pages — writes by one process are
immediately visible to others without any explicit IPC, which is a legitimate and fast (zero-copy,
no syscall-per-message) inter-process communication mechanism, distinct from `MAP_PRIVATE` which
gives each mapper its own copy-on-write view where writes are private and never written back to the
underlying file. `mmap()`'s appeal over `read()`/`write()` for large or randomly-accessed files is
that it avoids an explicit copy between kernel page-cache buffers and a userspace buffer — the
mapped memory *is* the page cache, accessed directly by CPU load/store instructions once faulted in
— at the cost of page-fault overhead per first access to each page (versus a single bulk `read()`
syscall's more predictable, batched cost), which is why `mmap()` tends to win for large files accessed
non-sequentially or repeatedly (databases memory-mapping their data files), while plain `read()`/
`write()` often wins for simple sequential, single-pass access where fault-per-page overhead outweighs
any zero-copy benefit. `mmap()` also underlies shared-library loading (the dynamic linker maps each
`.so`'s segments), `MAP_ANONYMOUS|MAP_SHARED` memory shared between related processes without a
backing file (a common alternative to System V shared memory), and huge-page-backed mappings via
`MAP_HUGETLB` or transparent huge page promotion, all funneling through the exact same VMA/page-fault
machinery described throughout this section.

### Key commands
```
strace -e trace=mmap,munmap ./program    # observe every mmap/munmap call and its flags/size
cat /proc/<pid>/maps | grep -v '\[' | head   # inspect file-backed mappings for a running process
lsof -p <pid>                              # cross-reference mapped files with open file descriptors
```

## brk() vs mmap() for heap growth

`malloc()`'s underlying allocator (glibc's ptmalloc2 by default) uses two different kernel mechanisms
to obtain memory from the OS depending on requested allocation size, and understanding this split
explains a lot of otherwise-surprising `malloc`/`free` memory-retention behavior. Small-to-medium
allocations are served from "the heap" in the traditional sense — a single, contiguous region grown
and shrunk via `brk()`/`sbrk()`, which simply moves a "program break" pointer up or down, extending or
shrinking one contiguous VMA; this is fast (no new VMA bookkeeping needed for each allocation, since
the allocator's own internal free-list/bin logic subdivides this single region) but has an important
constraint: `brk()` can only shrink the heap from its *current* top, so if a large allocation near the
top of the heap is freed but a smaller, still-in-use allocation sits above it in address order (address
order is a consequence of how the allocator happened to place things, not size order), the freed space
in the middle cannot actually be returned to the kernel via `brk()` — it stays reserved by the process,
explaining a common source of "freed memory but `ps`/`top` RSS didn't go down" confusion. Large
allocations (by default, requests at or above 128KB in glibc, tunable via `mallopt(M_MMAP_THRESHOLD)`)
instead go directly through `mmap(MAP_ANONYMOUS|MAP_PRIVATE)`, getting their own independent VMA that
can be `munmap()`'d and genuinely returned to the kernel immediately and completely upon `free()`,
with no risk of being "trapped" behind other still-live allocations the way heap-resident ones can be.
This is precisely why some memory-fragmentation-prone long-running processes benefit from tuning
`M_MMAP_THRESHOLD` lower (pushing more allocations through the cleanly-returnable `mmap()` path) or
from periodically calling `malloc_trim()` (which explicitly asks the allocator to return whatever
brk-heap space it safely can back to the kernel), and why observed process RSS can remain stubbornly
high long after an application believes it has freed the bulk of its memory.

### Key commands
```
strace -e trace=brk,mmap ./program        # observe which mechanism serves which allocation
cat /proc/<pid>/status | grep VmData        # traditional (brk-based) heap size for a process
mallinfo2() / malloc_stats()                # (in-process, via glibc) internal allocator arena stats
```

## Memory Overcommit and OOM Killer

Linux, by default, allows processes to collectively *request* (via `malloc()`/`mmap()`) more virtual
memory than the system could ever actually back with physical RAM plus swap — this is memory
overcommit, controlled by `/proc/sys/vm/overcommit_memory` with three modes: 0 (heuristic overcommit,
the default — the kernel uses a rough heuristic to reject clearly-insane requests but generally allows
most allocation requests to succeed optimistically, on the well-founded assumption that most processes
never actually touch every byte they allocate, thanks to demand paging), 1 (always overcommit, never
refuse any allocation request regardless of size, used for specialized workloads like certain
scientific/sparse-matrix applications that deliberately allocate huge, mostly-unused virtual regions),
and 2 (strict accounting — the kernel tracks total committed address space against a strict limit
derived from RAM plus swap plus a configurable overcommit ratio, refusing allocations that would
exceed it, trading application-level allocation failures you can catch and handle for the certainty of
never triggering the OOM killer due to overcommit specifically). The consequence of the default
heuristic-overcommit mode is that `malloc()`/`mmap()` returning successfully is not a guarantee that
memory is actually available — the real reckoning happens later, when pages are actually *touched* and
must be backed by real physical frames. When the system as a whole runs genuinely out of physical
memory and swap to satisfy actively-used pages, the kernel's OOM killer (`__oom_kill_process()` in
`mm/oom_kill.c`) is invoked as the last resort: rather than deadlocking the entire system waiting for
memory that will never appear, it selects a victim process (see OOM scoring below) and sends it
`SIGKILL`, forcibly freeing its memory to relieve the pressure. This design — optimistic overcommit
plus a reactive killer of last resort — trades the certainty of some individual process occasionally
being unexpectedly killed for much better overall memory utilization across the whole system in the
common case where most allocated-but-never-touched memory truly is never touched.

### Key commands
```
cat /proc/sys/vm/overcommit_memory        # current overcommit policy (0/1/2)
cat /proc/meminfo | grep Commit             # CommitLimit and Committed_AS — current overcommit accounting
dmesg | grep -i "killed process"            # OOM killer activity log, with victim PID/name/score
cat /proc/<pid>/oom_score                    # current OOM badness score for a process
```

## OOM Score and oom_score_adj

When the OOM killer must choose a victim, it computes a "badness" score for every eligible process,
primarily driven by the process's resident memory footprint (RSS plus swap usage) relative to total
available memory, on the reasonable premise that killing the single largest memory consumer relieves
the most pressure per process killed, while also factoring in adjustments for process runtime
(long-running, established processes are given a very slight preference over brand-new ones under
some heuristics), and explicit administrator overrides. `/proc/<pid>/oom_score` shows the currently
computed badness value (higher means more likely to be killed); the far more important knob for
operators is `/proc/<pid>/oom_score_adj`, a value from -1000 to +1000 that directly biases the score —
setting it to -1000 makes a process entirely immune to the OOM killer (used for genuinely
system-critical processes like `sshd` or a monitoring agent that must never be killed to preserve
the ability to diagnose the very incident that triggered OOM pressure in the first place), while a
positive value makes a process a preferred/earlier victim (commonly applied to best-effort batch jobs
or easily-restartable worker processes that are safe and cheap to lose compared to, say, a stateful
primary database process). systemd exposes this as the simpler-to-reason-about
`OOMScoreAdjust=` unit directive, and container orchestrators like Kubernetes set `oom_score_adj`
automatically and systematically based on a pod's QoS class (Guaranteed pods get the most negative,
least-likely-to-be-killed adjustment; BestEffort pods get the most positive, most-likely-to-be-killed
adjustment), which is exactly the mechanism behind Kubernetes's documented OOM-kill priority ordering
between QoS classes on a memory-pressured node — it's not a Kubernetes-specific kernel feature at all,
just Kubernetes correctly using the standard Linux `oom_score_adj` knob.

### Key commands
```
cat /proc/<pid>/oom_score                   # current computed badness score
cat /proc/<pid>/oom_score_adj                 # current adjustment value (-1000 to +1000)
echo -1000 > /proc/<pid>/oom_score_adj         # make a process immune to the OOM killer
choom -p <pid> -n -500                          # (util-linux) friendlier CLI to view/set oom_score_adj
```

## Swap Space and Swappiness

Swap space is disk (or increasingly, compressed-RAM via zram/zswap) storage used to hold pages that
are backed by anonymous memory (not file-backed, so they have no other on-disk representation to fall
back to) but haven't been recently used, freeing up physical RAM for more actively-used pages. The
kernel's `kswapd` background reclaim thread (and direct reclaim invoked synchronously by an allocating
process when free memory is critically low) selects "cold" pages using an approximation of
least-recently-used tracking (Linux maintains active/inactive LRU lists per memory zone/cgroup, moving
pages between them based on access patterns detected via the accessed bit and periodic scanning) and
writes them out to swap, marking their page table entries invalid so any subsequent access triggers a
major page fault that reads the page back in from swap (swapping in). Swappiness
(`/proc/sys/vm/swappiness`, 0-200 on modern kernels, historically capped at 100) is a tunable that
biases the kernel's relative preference for reclaiming/swapping anonymous pages versus reclaiming
file-backed page-cache pages instead — a low swappiness value (0-10, common on database servers)
tells the kernel to strongly prefer dropping clean, easily-re-readable file-cache pages under memory
pressure and only swap out anonymous memory as an absolute last resort, appropriate when swapping
would introduce unacceptable latency for a latency-sensitive, memory-resident workload; a higher value
allows more eager swapping of anonymous pages to preserve a larger file-cache working set, which can
benefit workloads whose performance depends heavily on cache hit rate for file I/O rather than raw
memory-resident data. Swapping to a traditional spinning disk is notoriously catastrophic for
interactive/latency-sensitive performance (I/O latency measured in milliseconds versus RAM's
nanoseconds, a roughly six-order-of-magnitude difference, and a system that starts thrashing —
constantly swapping pages back and forth — can spiral into near-total unresponsiveness, "swap death");
swap on fast NVMe or zswap/zram (which compresses pages in RAM itself rather than writing to a
physical device at all) substantially narrows but does not eliminate this latency cliff, and is
increasingly the preferred configuration on modern cloud/container hosts specifically to blunt the
worst-case impact of occasional memory pressure without disabling swap (and thus losing its benefits)
entirely.

### Key commands
```
free -h                                # swap total/used at a glance
cat /proc/sys/vm/swappiness              # current swappiness value
sysctl vm.swappiness=10                   # lower swappiness, favoring file-cache retention
swapon --show                              # active swap devices/files and their usage
vmstat 1                                    # 'si'/'so' columns: swap-in/swap-out rate, live
```

## Huge Pages (Transparent Huge Pages)

Standard 4KB pages mean a process with a large working set requires proportionally many page table
entries and TLB entries to cover it, increasing both page-table memory overhead and TLB miss rates
(since the TLB can only cache a limited number of entries regardless of how much memory each covers).
Huge pages (2MB, and 1GB "gigantic" pages on hardware/kernel configurations that support it) address
this by making each page table entry and TLB entry cover far more memory — a 2MB huge page needs the
same single TLB entry as a 4KB page would, but covers 512 times as much address space, dramatically
reducing TLB pressure and page-table memory overhead for large, memory-intensive workloads (databases,
JVMs with large heaps, in-memory caches). Explicit HugeTLB pages are pre-reserved by an administrator
(`/proc/sys/vm/nr_hugepages`) as a dedicated pool that applications must explicitly request via
`mmap(MAP_HUGETLB)` or a hugetlbfs mount — guaranteed available (not swappable, not subject to
fragmentation-driven allocation failure at request time since they're pre-reserved) but requiring
application awareness and explicit configuration. Transparent Huge Pages (THP) instead attempt to
provide the same TLB benefit automatically and transparently to any application, with the kernel
opportunistically promoting contiguous runs of regular 4KB pages into a single 2MB huge page in the
background (`khugepaged`) whenever it can find/create the necessary physically-contiguous, aligned run
of pages, with no application changes required. THP's transparency is also its main operational risk:
promoting pages requires finding contiguous physical memory, which under fragmented conditions forces
expensive memory compaction (moving other, unrelated pages around to create the needed contiguous
run) that can introduce unpredictable latency spikes — this exact behavior has caused enough
production incidents (especially for latency-sensitive databases like Redis and certain versions of
MongoDB/PostgreSQL) that disabling THP, or setting it to "madvise" mode (only promote pages for
regions an application explicitly opts into via `madvise(MADV_HUGEPAGE)`, rather than blanket
"always" promotion) is a very common, well-documented production tuning recommendation you should be
ready to justify from first principles in an interview, not just recite as folklore.

### Key commands
```
cat /sys/kernel/mm/transparent_hugepage/enabled     # current THP mode (always/madvise/never)
echo madvise > /sys/kernel/mm/transparent_hugepage/enabled   # restrict THP to explicit opt-in only
cat /proc/meminfo | grep -i huge                     # HugePages_Total/Free/Rsvd and AnonHugePages usage
grep -i thp /proc/vmstat                              # THP fault/collapse/split event counters
```

## Page Cache and Buffer Cache

The page cache is the kernel's transparent, automatic cache of file data in RAM, populated whenever a
process reads a file (the data is read from the block device into page-cache pages, then copied — or,
for `mmap()`, directly mapped — into the requesting process) and consulted on every subsequent read of
the same file offset before ever issuing a new block-device I/O, since a hit is served entirely from
RAM at memory-access speed. Crucially, page cache memory is not "wasted" or unavailable to
applications despite appearing as "used" in naive memory accounting — it's explicitly reclaimable:
under memory pressure, clean (unmodified since being read from disk) page-cache pages can simply be
dropped, since they can always be re-read from the underlying file if needed again, which is exactly
why `free -h`'s "available" column (not "free") is the number that actually matters for capacity
reasoning — a system showing very little in the "free" column but a large "available" figure (backed
mostly by reclaimable page cache) is not remotely under memory pressure, a distinction that trips up
even fairly experienced engineers reading `free` output for the first time. Historically, Linux
maintained a conceptually separate "buffer cache" for raw block-device I/O (filesystem metadata,
non-file-backed block reads) distinct from the page cache used for file content, but since kernel
2.4 these were unified — the buffer cache is now effectively just the page cache applied to block
device files directly, with `struct buffer_head` used as a smaller-granularity bookkeeping structure
layered on top of page-cache pages for filesystem metadata I/O rather than a wholly separate memory
pool, so modern `free -h`'s legacy "buffers" column is a comparatively small subset of overall
cache-related memory, with "cached" representing the bulk of it. Because the page cache is populated
lazily and opportunistically, a freshly-booted system or one that just had its cache dropped
(`echo 3 > /proc/sys/vm/drop_caches`) will show noticeably slower initial file-access performance
("cold cache") until repeated access patterns re-warm the cache, which is a real and expected
consideration when benchmarking (always distinguish and report cold-cache versus warm-cache results)
or when planning for the aftermath of a host reboot in a latency-sensitive service.

### Key commands
```
free -h                                    # 'available' column accounts for reclaimable cache correctly
cat /proc/meminfo | grep -E 'Cached|Buffers'   # page cache and buffer-head memory breakdown
vmstat -s | grep -i cache                    # cache-related counters over the system's uptime
echo 1 > /proc/sys/vm/drop_caches             # drop only page cache (2=dentries/inodes, 3=both) — diagnostic use only
```

## Dirty Page Writeback

A "dirty" page is a page-cache page that has been modified in memory but not yet written back to its
underlying storage — writes to file-backed memory (via `write()` or a writable `mmap()`) are, by
default, buffered entirely in the page cache and marked dirty rather than synchronously flushed to
disk immediately, which is what gives Linux its fast, asynchronous write performance for ordinary file
I/O (an application's `write()` call typically returns as soon as the data is copied into RAM, long
before it's durably on disk). Dirty pages are written back to storage by kernel writeback threads,
triggered by several independent policies working together: a periodic timer (`dirty_writeback_
centisecs`) that flushes any pages that have been dirty longer than a maximum age
(`dirty_expire_centisecs`), preventing data from sitting unwritten indefinitely and bounding potential
data loss on a crash; and a ratio-based threshold (`dirty_ratio`/`dirty_background_ratio`, expressed as
a percentage of total memory) where crossing `dirty_background_ratio` triggers background writeback
asynchronously without blocking application writes, while crossing the higher `dirty_ratio` forces
subsequent `write()` calls themselves to block synchronously until enough dirty pages are flushed,
acting as a hard backpressure mechanism preventing runaway memory consumption by an application writing
far faster than storage can absorb. This buffering means a plain `write()` returning successfully is
*not* a durability guarantee — an application requiring genuine durability (a database committing a
transaction) must explicitly call `fsync()`/`fdatasync()` (or open the file with `O_DSYNC`/`O_SYNC`)
to force those specific dirty pages to be written to stable storage and confirmed before proceeding,
which is exactly why database write-ahead-log implementations are built around careful, deliberate
`fsync()` placement rather than relying on ordinary buffered writes, and why an ungraceful power loss
can lose recently-written-but-not-yet-`fsync()`ed data even though the application's `write()` calls
all appeared to succeed.

### Key commands
```
cat /proc/meminfo | grep -i dirty              # current dirty page count system-wide
cat /proc/sys/vm/dirty_ratio                     # hard write-blocking threshold (% of memory)
cat /proc/sys/vm/dirty_background_ratio           # async background-writeback trigger threshold
sync                                                # force all dirty pages system-wide to be written back now
strace -e trace=fsync,fdatasync ./program            # confirm an application actually forces durability
```

## NUMA Architecture

(See Section 2's NUMA-aware scheduling entry for the CPU-scheduling angle; this entry focuses on the
memory-management side.) On a NUMA system, physical memory is partitioned across nodes, each directly
attached to one CPU socket's integrated memory controller, meaning memory access latency and bandwidth
differ depending on whether a CPU is accessing "local" memory on its own node or "remote" memory
attached to a different node, reachable only via a slower inter-socket interconnect. The memory
allocator is NUMA-aware and, by default, follows a "local allocation" policy — when a process touches
a page for the first time (first-touch allocation), the kernel preferentially allocates the backing
physical frame from the NUMA node local to the CPU that performed the touch, on the reasonable
assumption that the same CPU (or one on the same node) is likely to be the one accessing that memory
again soon. This first-touch policy has a subtle but important operational consequence: if a process's
initialization thread runs on CPU node 0 and allocates/zeroes a large buffer before worker threads on
node 1 actually use it, that memory ends up permanently node-0-local, and every subsequent access from
node-1 workers pays the remote-access penalty for the buffer's entire lifetime unless explicitly
migrated — a very common and easy-to-miss source of NUMA-related performance degradation in
multi-threaded applications that don't carefully control which thread performs the first touch of
each region. `numactl` lets an administrator override default policy explicitly: bind a process's
memory allocations to a specific node (`--membind`) regardless of first-touch behavior, prefer a node
without hard-failing if it's exhausted (`--preferred`), or interleave allocations round-robin across
multiple nodes (`--interleave`) for workloads that genuinely access a large shared structure evenly
from all nodes and would rather average out the remote-access penalty than concentrate it. `numastat`
reports the running tally of local versus remote (`numa_hit`/`numa_miss`/`other_node`) allocations for
a process, letting you empirically confirm whether NUMA locality assumptions actually held in
practice rather than guessing from topology alone.

### Key commands
```
numactl --hardware                       # NUMA node count, CPU membership, memory size per node
numastat -p <pid>                          # per-process local vs remote memory access breakdown
numactl --membind=0 --cpunodebind=0 command   # force strict node-0-only CPU and memory placement
cat /sys/devices/system/node/node0/meminfo   # detailed memory stats for one specific NUMA node
```

## Memory Cgroups and Limits

The memory cgroup controller (`memory` in cgroups v1, unified under cgroup v2's single hierarchy)
lets the kernel track and limit memory usage for an arbitrary group of processes as a single
accounting unit, which is the exact mechanism underlying container memory limits (`docker run -m`,
Kubernetes pod `resources.limits.memory`) — a container is, at its memory-accounting core, nothing
more than a cgroup with a `memory.max` (v2) / `memory.limit_in_bytes` (v1) limit applied to the set of
processes namespaced into that container. When a cgroup's memory usage approaches its limit, the
kernel first attempts reclaim scoped specifically to that cgroup's own pages (dropping that cgroup's
reclaimable page-cache pages, or swapping out its anonymous pages if swap is available and permitted)
before ever considering the more drastic step of invoking the OOM killer — and critically, cgroup-
scoped OOM kills are similarly scoped: exceeding a cgroup's own memory limit triggers the kernel to
kill a process *within that cgroup* (chosen by the same badness-scoring logic, but restricted to
cgroup membership) rather than considering the entire host's process list, which is exactly why one
noisy/leaking container can be killed without taking down unrelated containers or the host's own
system processes, as long as limits are configured correctly. cgroup v2 additionally exposes
`memory.high` as a *soft* throttling limit distinct from the hard `memory.max` kill-triggering limit —
crossing `memory.high` doesn't kill anything but does aggressively throttle the cgroup's memory
allocation rate (forcing synchronous reclaim work onto the offending processes themselves) as an early
backpressure signal, giving well-behaved applications a chance to respond to memory pressure
gracefully before ever reaching a hard, kill-triggering limit — a distinction increasingly used by
container orchestrators to differentiate "soft" resource requests from "hard" resource limits with
genuinely different, kernel-enforced backing behavior rather than being purely a scheduler-level
policy convention.

### Key commands
```
cat /sys/fs/cgroup/<path>/memory.max          # (cgroup v2) hard memory limit for a cgroup
cat /sys/fs/cgroup/<path>/memory.current       # current memory usage for a cgroup
cat /sys/fs/cgroup/<path>/memory.events         # oom_kill, high, max event counters for a cgroup
systemd-cgtop                                    # live top-like view of cgroup resource usage, including memory
```

---

### Interview Questions

**Conceptual/Internals (8)**

1. **What is the difference between a minor and major page fault, and why does the distinction
   matter for performance?**
   A minor fault is resolved without new disk I/O — the page is already resident in memory for some
   other reason (page cache hit, COW resolution) and only page-table bookkeeping needs updating. A
   major fault requires reading data from a block device (an uncached file page, or a swapped-out
   anonymous page), which blocks the process in uninterruptible sleep and costs orders of magnitude
   more time, making major-fault rate a direct, measurable contributor to application latency.

2. **Explain memory overcommit and why the OOM killer exists.**
   Linux's default overcommit policy allows processes to allocate more virtual memory than physical
   RAM plus swap could ever back, on the premise that demand paging means most allocated memory is
   never actually touched. When usage estimates turn out wrong and the system genuinely runs out of
   physical memory to back actively-used pages, the OOM killer selects and kills a victim process
   (scored primarily by memory footprint, adjustable via `oom_score_adj`) as a last resort to relieve
   pressure rather than deadlocking the whole system.

3. **Why is `free -h`'s "available" column more meaningful than its "free" column?**
   Page cache and other reclaimable memory (like clean, unused slab entries) show as "used" from a
   naive perspective but can be dropped instantly under pressure without any data loss since the
   underlying files can simply be re-read. "Available" accounts for this reclaimable memory, giving an
   accurate picture of memory truly usable by new allocations, whereas "free" alone dramatically
   understates real capacity on a healthy, well-cached system.

4. **How does the buddy allocator prevent external fragmentation?**
   It organizes free physical memory into power-of-two-sized blocks per order; allocation splits a
   larger free block in half when a smaller one isn't available, and freeing checks whether the
   freed block's "buddy" (its size-aligned adjacent counterpart) is also free, coalescing them back
   into a larger block recursively — actively consolidating free memory into larger contiguous chunks
   rather than leaving many small fragments scattered across address space.

5. **What problem do huge pages solve, and what's the trade-off with Transparent Huge Pages
   specifically?**
   Huge pages (2MB/1GB) reduce TLB and page-table pressure for large working sets by making each
   translation cover far more memory than a standard 4KB page. THP automates this transparently via
   background promotion (`khugepaged`), but promotion requires finding/creating physically contiguous
   memory, sometimes forcing expensive memory compaction that introduces latency spikes — a well-known
   trade-off that leads many latency-sensitive production databases to disable or restrict THP to
   `madvise`-only mode.

6. **Why does `malloc()`/`free()` not always return memory to the OS immediately, even after
   `free()` is called?**
   Small/medium allocations are served from a single contiguous heap region grown via `brk()`; `brk()`
   can only shrink from the current top of that region, so memory freed in the middle (behind other
   still-live allocations at higher addresses) cannot be returned to the kernel until everything above
   it is also freed. Large allocations instead go through `mmap()`, which can be `munmap()`'d and fully
   returned immediately and independently upon `free()`.

7. **What is copy-on-write and where does it apply beyond `fork()`?**
   COW defers duplicating memory until a write actually occurs, achieved by marking shared pages
   read-only and handling the resulting page fault on write by allocating a private copy for just the
   faulting process. Beyond `fork()`, it applies to `MAP_PRIVATE` file mappings (multiple processes
   sharing read-only pages of the same file until one writes) and other scenarios where the kernel can
   safely share physical pages optimistically.

8. **Explain how NUMA first-touch allocation works and why it can cause unexpected performance
   problems.**
   The kernel allocates the physical backing for a newly-touched virtual page from the NUMA node local
   to whichever CPU performed that first touch, assuming future accesses will come from the same
   locality. If an initialization thread on one node touches memory that worker threads on a different
   node will actually use, that memory remains permanently remote to those workers, causing ongoing
   cross-node access latency unless explicitly rebound or migrated.

**Scenario/Troubleshooting (6)**

9. **The OOM killer fired and killed an unexpected process on a host running several services. How
   do you determine why that specific process was chosen, and how do you prevent it recurring for a
   critical service?**
   Check `dmesg`/`journalctl` for the OOM kill log entry (shows victim PID, name, and computed badness
   score) and cross-reference each candidate process's `/proc/<pid>/oom_score` at the time. To protect
   a critical process going forward, set a strongly negative `oom_score_adj` (or the equivalent
   systemd `OOMScoreAdjust=`) so it is deprioritized as a kill target, while addressing the underlying
   memory pressure (right-sizing limits, fixing a leak) rather than relying on adjustment alone.

10. **A container is killed with an OOM error even though `free -h` on the host shows plenty of
    available memory. Why?**
    The container is almost certainly memory-limited via a cgroup (`memory.max`), and cgroup-scoped
    memory pressure triggers an OOM kill independent of host-wide memory availability — the container
    exceeded its own limit even though the host as a whole has ample free/reclaimable memory. Check
    `memory.events`/`memory.current` for that specific cgroup rather than host-wide `free -h`.

11. **After enabling Transparent Huge Pages fleet-wide, a latency-sensitive service starts showing
    periodic latency spikes it didn't have before. What's the likely cause and remediation?**
    THP's background promotion (`khugepaged`) can trigger memory compaction to assemble the
    contiguous physical memory needed for a 2MB huge page, and compaction can introduce unpredictable
    latency for the process being compacted around. Remediation is setting THP to `madvise` mode (only
    promote regions the application explicitly opts into) or disabling THP entirely for that
    workload, which is a common, well-documented tuning step for latency-sensitive databases.

12. **A long-running process's RSS keeps growing over days even though the application team insists
    it isn't leaking. How do you determine whether this is a real leak or expected behavior?**
    Inspect `/proc/<pid>/smaps` over time to distinguish page-cache-backed (file-mmap) growth from
    genuine anonymous/heap growth, check whether the allocator is retaining freed-but-unreturned
    `brk()`-heap memory (try `malloc_trim()` and observe whether RSS drops), and compare against actual
    live-object counts inside the application (heap profiler) rather than trusting RSS alone, since
    RSS reflects the allocator's retained memory, not strictly live application data.

13. **A database host was resized to a machine with more total cores and memory, but throughput
    dropped. NUMA is suspected — how do you confirm and fix it?**
    Run `numastat -p <pid>` for the database process and look for a high `numa_miss`/`other_node`
    ratio, confirming cross-node memory access is prevalent. Fix by restarting the process bound to a
    single NUMA node (`numactl --cpunodebind --membind`) if the working set fits within one node's
    memory, or by making the application NUMA-aware (partitioning data/threads per node) if it
    legitimately needs to span multiple nodes.

14. **An application performing bulk writes suddenly experiences a burst of `write()` calls blocking
    far longer than usual. What's a likely memory-subsystem explanation?**
    The dirty page ratio has likely crossed `vm.dirty_ratio` (the hard write-blocking threshold),
    forcing subsequent `write()` calls to block synchronously until enough dirty pages are flushed to
    storage — a backpressure mechanism kicking in because the application is generating dirty pages
    faster than the underlying storage can absorb writeback. Remediation includes tuning
    `dirty_ratio`/`dirty_background_ratio` more conservatively, spreading writes more evenly, or
    addressing an underlying storage throughput bottleneck.

**FAANG-level Deep Dive (6)**

15. **Trace, at the hardware and kernel level, exactly what happens when a process accesses a
    virtual address whose page table entry marks it not-present.**
    The MMU's page-table walk finds a not-present entry (or no entry at all at some level) and raises
    a page-fault exception, trapping into the kernel with the faulting address in CR2 and an error
    code describing the access type. The kernel's fault handler (`handle_mm_fault()`) looks up the
    VMA covering that address; if none exists, it delivers `SIGSEGV`. If a VMA exists, it determines
    the correct resolution — zero-fill a new anonymous page, fetch a file-backed page from cache or
    issue I/O, perform a COW duplication, or swap in a page — installs the resulting physical frame
    into the page table entry with correct permissions, and returns, causing the CPU to
    transparently re-execute the faulting instruction, which now succeeds.

16. **Why does KPTI (Kernel Page Table Isolation) impose a measurable performance cost specifically
    on syscall-heavy workloads, and what is it actually protecting against?**
    Before KPTI, kernel-space mappings were present (though permission-protected) in every process's
    page tables to avoid a full TLB flush on every kernel entry/exit. Meltdown exploited CPU
    speculative execution to transiently bypass that permission check and read kernel memory via
    timing side channels. KPTI's fix is to maintain almost entirely separate page table sets for user
    and kernel mode, requiring a CR3 reload (and associated TLB impact even with PCID tagging
    mitigating some of it) on every syscall/interrupt entry and exit — a cost paid proportionally to
    how frequently a workload crosses the user/kernel boundary, which is why syscall-heavy applications
    (many small I/O operations) see a larger relative slowdown than CPU-bound, syscall-light workloads.

17. **Explain precisely why `mmap(MAP_SHARED)` provides zero-copy IPC between processes, contrasting
    it with a pipe-based IPC mechanism.**
    A `MAP_SHARED` mapping of the same file (or shared memory object) by multiple processes results in
    their respective page table entries pointing at the exact same physical page-cache frames — a
    write by one process is a direct in-place modification of memory another process's own page table
    already resolves to, requiring no data movement or syscall at all for the actual data transfer
    (only initial setup and synchronization primitives like a futex or semaphore). A pipe, by
    contrast, requires the kernel to copy data from the writer's buffer into a kernel-internal pipe
    buffer, and again from that buffer into the reader's buffer on `read()` — genuine data copying on
    both ends, which `mmap(MAP_SHARED)` entirely avoids for the transfer itself.

18. **Why can a cgroup-scoped OOM kill happen even when `memory.max` accounting appears to have
    headroom at the moment of the kill, from an operator's perspective checking metrics a few
    seconds later?**
    Memory accounting and enforcement happen synchronously at allocation/charge time inside the
    kernel, not at whatever cadence an external monitoring/metrics scrape samples `memory.current`.
    A workload can spike its actual page allocations well past the limit in a burst faster than any
    external polling interval can observe, triggering an immediate cgroup-scoped reclaim-then-OOM
    sequence that resolves (killing a process, freeing its memory) before the next metrics sample even
    fires, which is why relying solely on periodic metrics scrapes to "catch" transient memory spikes
    before an OOM kill is fundamentally unreliable — `memory.events`' `oom_kill` counter, not a
    point-in-time usage graph, is the authoritative signal.

19. **Describe how first-touch NUMA policy interacts with copy-on-write pages after `fork()` on a
    NUMA system, and why this can produce surprising node placement for a forked worker process.**
    Copy-on-write means a forked child initially shares its parent's exact physical pages (whichever
    NUMA node they were originally allocated on), regardless of which NUMA node the child process's
    threads subsequently run on. Only when the child actually writes to (and thus COW-faults) a page
    does a *new* physical page get allocated, and that new allocation follows first-touch policy based
    on the *writing* CPU's node — meaning a forked worker pinned to a different NUMA node than its
    parent can end up with a "checkerboard" mix of pages: still-shared, unmodified pages remaining on
    the parent's original node (now remote to the child), and freshly COW-duplicated pages correctly
    local to the child's own node, a subtlety that purely static topology analysis without
    fork-timing awareness will miss.

20. **Why does `vm.overcommit_memory=2` (strict accounting) not simply eliminate the possibility of
    an OOM kill occurring?**
    Strict accounting only governs whether new allocation *requests* (`malloc`/`mmap` calls
    themselves) are permitted based on total committed address space against a computed limit — it
    prevents the *promise* of memory from exceeding what could theoretically be backed. It does not
    change what happens once already-committed, previously-successful allocations are actually
    *touched* and require real physical backing simultaneously; if legitimate, already-approved usage
    still exceeds available physical memory plus swap at that moment (e.g., swap becomes unavailable,
    or accounting didn't anticipate transient kernel-internal memory needs), the OOM killer can still
    be invoked — strict overcommit accounting reduces but does not categorically eliminate OOM risk.

### Hands-On Labs

**Lab 1: Observe RSS growth from mmap'd pages via smaps**
- Objective: Directly observe demand paging and RSS growth as pages are touched.
- Setup: A small C program and `strace`/`/proc` access.
- Tasks: `mmap()` a large anonymous region without touching it; check `/proc/<pid>/smaps_rollup` RSS
  (should be near zero); touch pages incrementally in a loop with sleeps between batches; observe RSS
  growing proportionally to pages actually touched, not the mapping's total size.
- Expected outcome: A clear, measured demonstration that `mmap()` size and RSS are independent until
  pages are faulted in.

**Lab 2: Trigger and observe the OOM killer safely in a cgroup**
- Objective: Reproduce a controlled OOM kill and interpret the resulting logs/scores.
- Setup: A disposable VM or container with cgroup v2 access.
- Tasks: Create a cgroup with a small `memory.max`; run a program inside it that allocates and touches
  memory past the limit; observe the kill in `dmesg` and `memory.events`; repeat with `oom_score_adj`
  set to protect one of two competing processes and confirm the other is killed instead.
- Expected outcome: A documented before/after showing how `oom_score_adj` changes victim selection.

**Lab 3: Measure THP's compaction-induced latency impact**
- Objective: Quantify the trade-off THP introduces for a latency-sensitive workload.
- Setup: A VM where you can toggle `/sys/kernel/mm/transparent_hugepage/enabled`.
- Tasks: Run a latency-measuring benchmark (e.g., `redis-benchmark` or a custom p99 latency test)
  under THP=always, THP=madvise, and THP=never; compare p50/p99 latency distributions across all three.
- Expected outcome: Quantified evidence supporting (or refuting, on your specific hardware/kernel) the
  common "disable THP for latency-sensitive databases" recommendation.

**Lab 4: Dirty page writeback backpressure**
- Objective: Reproduce `dirty_ratio`-triggered write blocking.
- Setup: A VM with a deliberately slow backing store (e.g., a loopback device with `dm-delay`, or a
  slow USB/network drive).
- Tasks: Lower `vm.dirty_ratio`/`vm.dirty_background_ratio` to small values; run a sustained
  sequential write workload (`dd` or `fio`) and measure `write()` call latency via `strace -T`; compare
  against default dirty ratio values.
- Expected outcome: Demonstrated correlation between dirty ratio thresholds and write-call blocking
  latency.

**Lab 5: NUMA first-touch and migration experiment**
- Objective: Empirically observe first-touch NUMA placement and its correction via `numactl`.
- Setup: A multi-node NUMA machine or emulated NUMA VM.
- Tasks: Allocate and touch a large buffer from a thread pinned to node 0; spawn worker threads pinned
  to node 1 accessing that same buffer; measure access latency/throughput; repeat with the buffer
  allocated via `numactl --interleave=all` or touched first from node-1-pinned threads instead.
- Expected outcome: Quantified performance difference attributable purely to first-touch placement.

### Production Incidents

**Incident 1: Cascading OOM kills across a shared host due to missing cgroup limits**
- Symptom: A single misbehaving batch job on a shared multi-tenant host triggers the OOM killer, which
  kills an unrelated critical service instead of the batch job itself.
- Investigation: `dmesg` shows the OOM killer selected the critical service based on badness score;
  neither process had cgroup-scoped memory limits configured, so the OOM killer considered the entire
  host's process list rather than being contained to the offending batch job's own resource group.
- Root cause: Workloads were run without per-workload cgroup memory limits, meaning a runaway batch
  job's memory growth pressured the whole host rather than being capped and killed within its own
  isolated accounting scope.
- Recovery: Restarted the killed critical service; immediately applied a `memory.max` limit to the
  batch job's cgroup to contain future incidents.
- Prevention: Mandate cgroup memory limits (or container resource limits, which are the same
  mechanism) for every workload on shared hosts, and set a strongly negative `oom_score_adj` for
  identified critical system services as defense in depth.

**Incident 2: Latency regression traced to Transparent Huge Page compaction stalls**
- Symptom: A latency-sensitive in-memory cache service begins exhibiting periodic multi-hundred-
  millisecond p99 latency spikes correlated with no obvious CPU or network anomaly.
- Investigation: `perf record` during a spike shows time spent in kernel memory-compaction code paths;
  `/sys/kernel/mm/transparent_hugepage/enabled` shows `always` mode active, and
  `/proc/vmstat`'s `thp_collapse_alloc`/`compact_stall` counters correlate directly with the observed
  latency spike timestamps.
- Root cause: THP's background `khugepaged` promotion was triggering memory compaction under
  moderate memory fragmentation, stalling the cache process's memory accesses during compaction windows.
- Recovery: Set THP to `madvise` mode fleet-wide for this workload class, immediately eliminating the
  correlated latency spikes.
- Prevention: Standardize THP=madvise (or disabled) as the default posture for all latency-sensitive
  production services, and add `compact_stall`/THP fault counters to the standard host dashboard so
  future regressions are caught proactively rather than via customer-facing latency complaints.

**Incident 3: Silent data loss after a power failure due to missing fsync in a custom write path**
- Symptom: After an unplanned datacenter power event, a subset of recently "successfully written"
  records are missing entirely from a custom-built storage service upon restart.
- Investigation: Code review of the write path shows records were written via buffered `write()` calls
  with no `fsync()`/`fdatasync()` call anywhere before acknowledging the write as successful to
  callers; the missing records correspond exactly to data that was still resident as dirty pages in
  the page cache, never yet flushed to disk, at the moment power was lost.
- Root cause: The application incorrectly treated a successful `write()` syscall return as a
  durability guarantee, when it only guarantees the data reached the page cache, not stable storage.
- Recovery: Restored from the last consistent backup/replica; accepted the narrow window of lost
  writes as unrecoverable for this incident.
- Prevention: Added explicit `fsync()`/`fdatasync()` calls (or `O_DSYNC` opens) at every point the
  application acknowledges a write as durable, and added a periodic chaos-testing procedure that
  simulates power loss against a test instance to catch future durability regressions before
  production.
