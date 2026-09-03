# Section 8: Observability, Performance & Troubleshooting

This section covers the practical toolchain for observing and diagnosing a running Linux system —
`/proc`/`/sys`, `strace`, `perf`, `bpftrace`, classic performance tools, core dumps, and the USE
method — tying together every kernel subsystem from earlier sections into an actionable
troubleshooting methodology.

## Subtopic Index
- [/proc and /sys Filesystems in Depth](#proc-and-sys-filesystems-in-depth)
- [strace and ltrace](#strace-and-ltrace)
- [perf (CPU profiling, flamegraphs)](#perf-cpu-profiling-flamegraphs)
- [bpftrace and eBPF tracing](#bpftrace-and-ebpf-tracing)
- [ftrace](#ftrace)
- [vmstat, iostat, mpstat, sar](#vmstat-iostat-mpstat-sar)
- [top/htop internals (how they read /proc)](#tophtop-internals-how-they-read-proc)
- [ss and netstat internals](#ss-and-netstat-internals)
- [lsof](#lsof)
- [dmesg and Kernel Logs](#dmesg-and-kernel-logs)
- [Core Dumps and Crash Analysis](#core-dumps-and-crash-analysis)
- [USE Method (Utilization, Saturation, Errors)](#use-method-utilization-saturation-errors)
- [Latency vs Throughput Analysis](#latency-vs-throughput-analysis)
- [Benchmarking Tools (fio, iperf, stress-ng)](#benchmarking-tools-fio-iperf-stress-ng)

---

## /proc and /sys Filesystems in Depth

`/proc` and `/sys` are the two synthetic filesystems that expose nearly all kernel and process state
to userspace as readable (and sometimes writable) text files, and fluency navigating both directly —
rather than relying purely on higher-level tools that merely parse them — is what separates surface-
level from genuinely deep troubleshooting ability, since every tool covered later in this section
(`top`, `ss`, `iostat`) is ultimately just a convenient formatter over exactly this same raw data.
`/proc/<pid>/` holds an entire directory per process: `status`/`stat` (state, memory, scheduling
counters), `maps`/`smaps` (virtual memory layout, covered in Section 3), `fd/` (open file descriptors,
each a symlink revealing what it actually points at), `cwd`/`root`/`exe` (symlinks to the process's
current directory, chroot root, and the actual executable binary on disk), `environ` (the process's
environment variables), `cgroup` (which cgroups this process belongs to across every active
hierarchy), and `task/<tid>/` (per-thread breakdowns of much of the same information, as discussed in
Section 2). System-wide, `/proc/meminfo`, `/proc/cpuinfo`, `/proc/interrupts`, `/proc/net/*`, and
`/proc/sys/*` (the sysctl tree, both readable and writable for live kernel tuning) round out the
picture. `/sys`, by contrast, is organized around the kernel's internal device/driver object model
rather than process state — every bus, device, driver, and class the kernel knows about appears as a
directory with attribute files, and it's the standard mechanism for both introspection (`/sys/class/
net/eth0/carrier` for link state) and live configuration (`/sys/block/sda/queue/scheduler`, as covered
in Section 4). Because both are regenerated live from in-kernel data structures rather than being
ordinary persisted files, they impose essentially zero disk I/O cost to read and always reflect
current, real-time state — but they are also **not** atomic snapshots across multiple reads: reading
`/proc/<pid>/stat` and then `/proc/<pid>/status` moments later can reflect the process's state at two
slightly different instants, an important subtlety for anyone writing tooling that needs genuinely
point-in-time-consistent multi-field process data.

### Key commands
```
cat /proc/<pid>/status              # human-readable process state summary
ls -l /proc/<pid>/fd/                 # open file descriptors and what they resolve to
cat /proc/meminfo                      # system-wide memory breakdown
find /sys/devices -name modalias | head   # explore the sysfs device tree directly
```

## strace and ltrace

`strace` intercepts and logs every syscall a process makes (using `ptrace()` to attach to and pause
the target at each syscall entry/exit, or, on newer configurations, seccomp-bpf-based fast paths for
lower overhead), showing the exact syscall name, arguments (resolved into human-readable flag names,
not raw integers), and return value/errno for each one — making it the single most valuable tool for
answering "what is this process actually doing at the kernel interaction level" when application-level
logs don't explain a hang, an unexpected error, or a permission failure. Because `strace` operates via
`ptrace()`, attaching to and single-stepping through every syscall imposes substantial overhead
(commonly reported as anywhere from 2x to 100x+ slowdown depending on syscall frequency), which makes
it excellent for diagnostic, short-duration investigation but a poor choice for profiling a
production workload under real load without careful, narrowly-scoped filtering (`-e trace=open,read`
to limit to specific syscalls of interest, `-p <pid>` to attach to an already-running process rather
than launching fresh, and `-c` for aggregated summary statistics rather than a full line-by-line trace
when you just need counts/timing rather than every individual call). `ltrace` is the library-call-level
analog, intercepting calls to shared library functions (`malloc`, `strcpy`, any dynamically-linked
function) rather than syscalls — genuinely useful for diagnosing issues at the application-logic level
above the syscall boundary (confirming a specific library function is even being called, with what
arguments), but generally considered less reliable and more invasive than `strace` (its interception
mechanism has more edge cases with statically-linked binaries, PLT/GOT manipulation, and
optimization-related inlining) and consequently less commonly reached for first in professional
troubleshooting compared to `strace`. A classic, must-know `strace` pattern: `strace -f -e trace=open,
openat -p <pid>` reveals exactly which file a process is trying (and failing) to open when the
application only reports a generic "permission denied" or "file not found" without specifying the
actual path — collapsing what could otherwise be extensive log-diving or guesswork into a single,
definitive, kernel-level answer.

### Key commands
```
strace -f -p <pid>                    # attach to a running process (and its threads/children) live
strace -e trace=network ./program       # trace only network-related syscalls
strace -c ./program                       # aggregated syscall count/time summary instead of a full trace
ltrace -f -e malloc+free ./program          # trace library-level calls (here, allocation-related)
```

## perf (CPU profiling, flamegraphs)

`perf` is the standard Linux profiling toolchain, built on top of the kernel's `perf_events`
subsystem, capable of both hardware-performance-counter-based sampling (cache misses, branch
mispredictions, instructions-per-cycle, using the CPU's own dedicated performance monitoring unit
registers) and software-event tracing (context switches, page faults, and arbitrary kernel
tracepoints/kprobes), all through one unified command-line interface. The most common workflow —
`perf record -g` (capturing call-graph/stack information alongside sampled events, typically
CPU-cycle-based by default) followed by `perf report` — periodically interrupts the running program
(at a configurable sampling frequency) and records the current instruction pointer plus a captured
call stack, building up a statistical profile of *where* CPU time is actually being spent across the
entire call graph without needing to instrument or recompile the target program at all (a substantial
advantage over traditional instrumenting profilers, which require code modification/relinking and
introduce more observer-effect overhead). Flame graphs (via Brendan Gregg's `FlameGraph` toolset,
consuming `perf record`'s raw output) render this sampled call-graph data as a visual, easily-scannable
stacked-bar chart — each function's box width proportional to the fraction of total samples where it
appeared somewhere in the captured call stack, with the vertical axis representing call-stack depth —
letting an engineer visually spot the widest boxes (the functions consuming the most aggregate CPU
time across the whole profile, whether as the actual leaf function executing or as a parent frame
whose descendants collectively consume significant time) in seconds rather than having to manually
parse a text-based call-tree report. `perf`'s low overhead (sampling-based, not exhaustive
instrumentation of every single call) makes it genuinely viable for careful use directly in production
environments (unlike `strace`'s much higher per-syscall overhead), which is precisely why it's the
default first tool reached for when investigating "why is this process consuming so much CPU" in a
live production incident rather than only in a controlled lab/staging reproduction.

### Key commands
```
perf record -g -p <pid> -- sleep 30    # sample a running process's call stacks for 30 seconds
perf report                              # interactive, text-based hierarchical report of captured samples
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg   # generate a visual flame graph
perf stat ./program                        # aggregate hardware counter summary (cycles, IPC, cache misses) for a run
```

## bpftrace and eBPF tracing

`bpftrace` is a high-level tracing language and runtime built on eBPF, letting an engineer write
concise, awk-like one-liners or short scripts that attach to kprobes (arbitrary kernel function entry/
exit points), uprobes (equivalent instrumentation points in userspace binaries/libraries),
tracepoints (stable, kernel-maintained instrumentation points that don't break across kernel version
upgrades the way raw kprobes attached to internal function names sometimes can), and USDT probes
(userspace statically-defined tracepoints some applications/runtimes deliberately expose), compiling
each script down to a verified, sandboxed eBPF program the kernel JIT-compiles and executes directly
in kernel context with minimal overhead — a meaningfully different, generally lower-overhead and more
flexible approach than either `strace`'s `ptrace()`-based interception or a custom kernel module would
require. A representative, genuinely useful one-liner:
`bpftrace -e 'kprobe:vfs_read { @[comm] = count(); }'` attaches to every `vfs_read()` call system-wide
and tallies a per-process-name count, answering "which processes are actually generating the most
read-syscall-driven VFS activity right now" with a single command and no application instrumentation
required at all. Because eBPF programs are verified before being allowed to load (bounded loops,
memory-access-safety checks, ensuring the program cannot crash or hang the kernel), `bpftrace` scripts
can be run with meaningful confidence directly against production systems for live investigation in a
way that writing and loading an ad-hoc custom kernel module for the same purpose never could be
trusted to be safe. This capability directly underlies the modern "extended BPF observability"
ecosystem more broadly (`bcc`/BPF Compiler Collection tools like `biolatency`, `tcplife`,
`execsnoop`, many of which are effectively pre-packaged, polished versions of exactly the kind of
ad-hoc `bpftrace` script described above), representing the current state of the art for
low-overhead, highly flexible, production-safe deep system tracing beyond what `strace`/`perf` alone
can conveniently express.

### Key commands
```
bpftrace -e 'kprobe:vfs_read { @[comm] = count(); }'   # tally VFS reads per process name, live
bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args->filename)); }'   # trace file opens
bpftrace -l 'kprobe:*tcp*'                                # list available kernel probe points matching a pattern
biolatency                                                  # (bcc tool) histogram of block I/O latency, built on eBPF
```

## ftrace

ftrace is the kernel's own built-in, lower-level tracing framework (predating and complementary to
`perf_events`/eBPF, accessed directly through `/sys/kernel/debug/tracing/` rather than a separate
userspace daemon), providing function-level tracing (recording every entry/exit of nearly any kernel
function, or a filtered subset), event tracing (structured tracepoints similar to those `bpftrace`
also consumes), and specialized tracers for specific latency-analysis use cases — `function_graph`
tracer produces an indented, call-graph-style view of kernel function call nesting and duration
directly usable for understanding exactly what a specific syscall does internally step by step;
`irqsoff`/`preemptoff` tracers specifically record the worst-observed duration the kernel spent with
interrupts or preemption disabled, invaluable for real-time/low-latency kernel tuning work where an
unexpectedly long non-preemptible section anywhere in the kernel could introduce an unacceptable
latency spike, exactly the class of problem `PREEMPT_RT` tuning (Section 2) needs to identify and
eliminate. While `perf` and `bpftrace` have become the more commonly reached-for tools for most modern
troubleshooting (offering friendlier interfaces and, for eBPF specifically, safety-verified custom
logic), ftrace remains directly and immediately available on essentially every Linux kernel with
debugfs mounted, requiring no additional tooling installation at all, and its specialized latency
tracers (`irqsoff`, `wakeup`, `wakeup_rt`) provide capabilities not conveniently duplicated elsewhere,
making it still a genuinely relevant tool specifically for kernel-level latency forensics on systems
where installing additional tracing tooling isn't practical or where these specific specialized
tracers are exactly what's needed.

### Key commands
```
echo function_graph > /sys/kernel/debug/tracing/current_tracer   # enable the function-graph tracer
cat /sys/kernel/debug/tracing/trace                                 # view captured trace output
echo 1 > /sys/kernel/debug/tracing/tracing_on                        # start/stop tracing without reconfiguring
trace-cmd record -p function_graph -F ./program                        # convenience wrapper around raw ftrace usage
```

## vmstat, iostat, mpstat, sar

These classic `sysstat`-family tools each provide a focused, time-series view of a specific resource
dimension, and together form the standard first-response toolkit for any performance investigation
before reaching for heavier tools like `perf`/`bpftrace`. `vmstat` gives a compact, single-screen
summary spanning process run-queue length (`r`), blocked/uninterruptible process count (`b`, directly
relevant to the load-average-vs-D-state discussion in Section 2), memory (free, buffer, cache),
swap activity (`si`/`so`), I/O (blocks in/out), and CPU time breakdown (user/system/idle/iowait/steal)
all in one view, making it an excellent first command to run when given no prior context about what
might be wrong. `iostat -x` provides much more detailed, per-block-device I/O statistics —
throughput, average request size, average queue length, `await` (average time a request spends
queued plus serviced, the single most useful field for spotting a genuinely saturated/struggling
storage device), and `%util` (the percentage of time the device had at least one outstanding request,
which, importantly, can reach 100% even on a device that could still accept more parallel requests if
it supports sufficient queue depth — a frequently-misread metric that doesn't necessarily mean the
device is at its absolute throughput ceiling, merely that it was never fully idle during the sampling
interval). `mpstat -P ALL` breaks CPU utilization down per individual core rather than an aggregate
system-wide average, essential for spotting a single-core bottleneck (a single-threaded process
pegging one core at 100% while the aggregate system-wide CPU utilization looks unremarkably low across
many cores) that an aggregate-only view would completely hide. `sar` is the umbrella historical-data
tool underlying all of the above, capable of continuously logging these same metrics over time
(commonly configured to run periodically via cron/systemd timer) specifically so that after an
incident has already passed, `sar -f /var/log/sa/saXX` can retroactively reconstruct exactly what
CPU/memory/I/O/network conditions looked like at the time of the incident, rather than requiring the
metric collection to have been actively, manually running at the exact moment the problem occurred.

### Key commands
```
vmstat 1                              # one-second-interval system-wide resource summary
iostat -x 1                             # detailed per-device I/O statistics, refreshed every second
mpstat -P ALL 1                           # per-core CPU utilization breakdown
sar -f /var/log/sa/sa15                     # retroactively review historical metrics from a specific past day
```

## top/htop internals (how they read /proc)

`top` and `htop` are, at their core, nothing more than a loop that periodically re-reads `/proc/<pid>/
stat` (and related files) for every process on the system, computes deltas between successive samples
(CPU time consumed since the last refresh, divided by wall-clock time elapsed, to derive the familiar
percentage-CPU-usage figure), and renders a sorted, formatted display — understanding this explicitly
demystifies several of `top`'s behaviors that otherwise seem like magic. The reported `%CPU` figure is
always a rate computed between two samples, never an instantaneous value read directly from the
kernel (there is no such thing as an "instantaneous CPU percentage" for a process, only a duration of
consumed CPU time over a measured wall-clock interval), which is exactly why a very short-lived,
extremely bursty process can be entirely invisible in `top`'s default refresh interval despite
consuming meaningful CPU during its brief life — if it starts and exits between two consecutive
`/proc` samples, `top` simply never observes it at all. `htop` extends `top`'s basic model with a more
readable, colorized, scrollable interface, native support for viewing a process tree, and per-thread
display, but is reading precisely the same underlying `/proc` data — no additional kernel privilege or
information source is involved, merely a friendlier presentation layer. Understanding this "it's just
`/proc` polling under the hood" reality is what lets an engineer correctly reason about `top`'s
limitations (it cannot show you anything `/proc` itself doesn't expose, it cannot see historical data
before it started running, and its default sort/refresh settings can hide short-lived or
low-average-but-high-peak resource consumers) and know precisely when a different tool (`pidstat` for
historical per-process time-series, `perf`/`bpftrace` for anything requiring kernel-internal detail
`/proc` simply doesn't surface at all) is actually the correct tool for a specific investigative
question `top` cannot answer.

### Key commands
```
top -d 1                              # refresh every 1 second (finer-grained sampling than the 3s default)
top -H -p <pid>                         # per-thread view for one specific process
htop                                      # friendlier interactive equivalent, same underlying /proc data source
pidstat 1                                  # historical, loggable per-process time-series (complements top's live-only view)
```

## ss and netstat internals

`ss` (socket statistics) and the older `netstat` both present a formatted view of the kernel's
internal socket tables (`/proc/net/tcp`, `/proc/net/udp`, `/proc/net/unix`, and their IPv6
equivalents), showing local/remote address-port pairs, connection state, and (with appropriate
privilege) the owning process — but `ss` is implemented using the more modern, efficient netlink
socket-diagnostic API (`NETLINK_SOCK_DIAG`) rather than `netstat`'s older approach of parsing the
`/proc/net/*` text files directly, which matters substantially on hosts with very large numbers of
active connections: `netstat`'s approach requires reading and parsing a potentially enormous flat text
file representing every single socket on the system for every single query, while `ss`'s netlink-based
querying can filter directly in the kernel before data is even returned to userspace, making `ss`
dramatically faster on connection-heavy hosts (load balancers, busy application servers) where
`netstat` can itself become a genuinely slow, resource-consuming command to run, ironically worst
exactly when you most need a *fast* diagnostic tool during a connection-related incident. `ss -tin`
(covered already in Section 5) additionally surfaces TCP-internals detail (congestion window,
retransmission counts, RTT estimates) that plain `netstat` never exposed at all, since it draws on the
same detailed kernel socket-diagnostic information `ss`'s netlink-based approach was specifically
designed to expose. Most modern distributions have deprecated or entirely removed `netstat` from
default installations in favor of `ss` (part of the actively-maintained `iproute2` package) plus `ip`
for routing/interface information, reflecting this real, measurable performance and capability
advantage rather than being merely a stylistic tooling preference — a genuinely relevant, practical
fact for anyone still reflexively reaching for `netstat` out of habit on a modern system.

### Key commands
```
ss -tanp                              # all TCP sockets, numeric addresses, with owning process
ss -tan state established | wc -l       # quick count of established connections
ss -s                                     # summary totals across all socket types/states
netstat -tanp                              # older equivalent, meaningfully slower on connection-heavy hosts
```

## lsof

`lsof` (list open files) enumerates every open file descriptor across every process on the system —
and, consistent with the "everything is a file" UNIX philosophy discussed in Section 1, this
includes not just ordinary regular files but also directories, character/block devices, network
sockets, pipes, and shared memory segments, since all of these are represented as file descriptors in
a process's `files_struct` regardless of what they actually connect to. This universality is exactly
what makes `lsof` valuable across so many different troubleshooting scenarios that might otherwise
seem unrelated: `lsof -i :443` finds which process is bound to a specific port (useful when a service
fails to start with "address already in use" and you need to identify the actual conflicting
process); `lsof /mount/point` finds every process with any open file on a specific filesystem (useful
before attempting to unmount it, since a busy filesystem refuses to unmount while any process still
holds a reference); `lsof +L1` finds files with a link count of zero — deleted but still open,
exactly the "disk full but `du` disagrees" scenario discussed in Section 4; and `lsof -p <pid>` gives
a complete inventory of everything one specific process currently has open, useful both for
understanding a process's resource footprint and for diagnosing file-descriptor-limit-related
failures (comparing the count of open descriptors against that process's configured `ulimit -n`).
Because `lsof` must enumerate the *entire* system's open files by default (walking every process's
`/proc/<pid>/fd/` directory), running it unscoped on a host with an enormous number of processes/open
files can itself be a surprisingly slow, resource-intensive operation — always preferring the most
specific applicable filter (`-p`, `-i`, a specific path) over an unscoped, full-system `lsof` invocation
is both faster and produces far more immediately actionable, less overwhelming output for whatever
specific question is actually being investigated.

### Key commands
```
lsof -i :443                          # find the process bound to a specific port
lsof -p <pid>                           # every open file descriptor for a specific process
lsof +L1                                  # find deleted-but-still-open files (disk space troubleshooting)
lsof /mount/point                          # every process holding a file open on a specific filesystem
```

## dmesg and Kernel Logs

`dmesg` displays the kernel's ring buffer — a fixed-size, in-memory circular buffer that the kernel
itself writes `printk()` messages into throughout its own execution, starting from the very earliest
boot messages (as discussed in Section 1) through to live, ongoing kernel-level events (driver errors,
OOM killer activity, hardware faults, filesystem errors, security-module denials surfaced via
`audit`/`dmesg` both) for as long as the system has been running. Because it's a *fixed-size* buffer,
older messages are eventually overwritten by newer ones once the buffer fills, which is exactly why
production systems forward kernel messages to persistent storage (`journalctl -k` for
systemd-journal-integrated kernel logs, which persist across reboots if journal storage is configured
persistently as discussed in Section 7, or a traditional syslog daemon configured to capture and
retain kernel facility messages) rather than relying on `dmesg`'s live in-memory buffer alone for any
message that might need to be reviewed well after the fact. `dmesg`'s output is prioritized (each
message tagged with a syslog-standard severity level from emergency down to debug), and filtering by
severity (`dmesg --level=err,crit,alert,emerg`) is the standard first move when scanning a busy
system's kernel log for genuinely actionable problems rather than routine informational messages —
`dmesg -T` additionally converts the raw, boot-relative timestamps kernel messages are natively stored
with into human-readable wall-clock time, essential for correlating a specific kernel event against
other timestamped evidence (application logs, monitoring alerts) gathered from entirely different
sources during an investigation. Because kernel-level events (OOM kills, driver errors, filesystem
corruption detection, hardware error-correction events) are frequently the *root cause* underlying
symptoms that first present at the application layer, checking `dmesg`/kernel logs early in any deep
troubleshooting investigation — not merely as an afterthought once application-level logs have already
been exhausted — is a genuinely important habit distinguishing efficient from inefficient
troubleshooting workflows.

### Key commands
```
dmesg -T --level=err,crit,alert,emerg    # human-readable timestamps, filtered to actionable severities
journalctl -k -b                            # kernel messages for the current boot, via persistent journal storage
dmesg | grep -i -E 'oom|killed process'       # quick scan for OOM killer activity
dmesg -w                                        # follow new kernel messages live, as they occur
```

## Core Dumps and Crash Analysis

A core dump is a snapshot of a process's memory (and register state) captured at the moment it
terminates abnormally (typically from an unhandled fatal signal like `SIGSEGV`, `SIGABRT`, or
`SIGBUS`), written to disk (or, on modern systemd-based systems, captured and stored by
`systemd-coredump` rather than a bare file dropped in the crashing process's working directory) for
later post-mortem analysis with a debugger, without needing to have caught the crash live or
reproduced it interactively under a debugger's direct control. `ulimit -c` governs whether core dumps
are even generated at all (defaulting to zero/disabled on many systems specifically to avoid
unexpectedly filling disk space with large dumps from routine crashes) and `/proc/sys/kernel/
core_pattern` controls exactly where/how a dump is written — a plain filename pattern for a
traditional flat-file dump, or, notably, a pipe syntax (`|/path/to/handler %p %u %g`) that routes the
raw core data through an external handler process entirely, which is precisely the mechanism
`systemd-coredump` uses to intercept every crash system-wide, compress and store it in a structured,
`coredumpctl`-queryable location, and automatically capture rich accompanying metadata (which
executable, which package version, a backtrace summary) alongside the raw memory dump itself. Once
captured, `gdb <executable> <core-file>` (or `coredumpctl debug` when using systemd's integrated
storage) lets an engineer load the dump and interactively inspect exactly the state the process was in
at the moment of the crash — the full call stack (`bt`/backtrace) across every thread, local variable
values in each frame, and raw memory contents — frequently pinpointing the exact line and even the
exact corrupted value responsible for a crash without ever needing to reproduce the failure live under
active observation, which is especially valuable for crashes that are rare, timing-dependent, or
otherwise difficult to deliberately reproduce on demand. For genuinely difficult, intermittent
production crashes, ensuring core dump capture is properly configured and retained (correctly-set
`core_pattern`, adequate `ulimit -c`, and systemd-coredump storage retention long enough to actually
review it after being paged) *before* the next occurrence is frequently the single highest-leverage
preparatory step available, since a crash that isn't captured at all when it happens cannot be
analyzed no matter how sophisticated the analysis tooling used afterward.

### Key commands
```
ulimit -c unlimited                  # allow core dump generation for the current shell session
cat /proc/sys/kernel/core_pattern      # confirm current core-dump routing configuration
coredumpctl list                        # (systemd-coredump) list captured crashes
coredumpctl debug <pid-or-exe>             # load a captured crash directly into gdb for analysis
```

## USE Method (Utilization, Saturation, Errors)

The USE Method, formalized by Brendan Gregg, is a systematic checklist-driven approach to performance
troubleshooting specifically designed to avoid the common failure mode of ad-hoc investigation missing
an entire resource dimension simply because no one thought to check it. For every resource in the
system (CPU, memory, each individual storage device, each network interface, and so on), the method
prescribes checking three distinct properties: Utilization (the percentage of time the resource was
busy servicing work, or the percentage of its capacity in use — a CPU's utilization percentage, a
storage device's `%util` from `iostat`), Saturation (the degree to which work is queued waiting for
the resource because it's already fully utilized — a CPU's run-queue length from `vmstat`'s `r`
column, a storage device's average queue length from `iostat -x`, both of which can reveal genuine
resource contention even when a naive utilization-only view might look merely "high but not maxed
out"), and Errors (the count of error events for that resource — NIC CRC errors from `ethtool -S`,
disk I/O errors from kernel logs, memory ECC correction counts) which can degrade performance or cause
failures through an entirely different mechanism than simple exhaustion (a resource experiencing
errors might show low utilization while nonetheless performing terribly due to constant
retry/recovery overhead). Applying USE systematically across every resource — rather than fixating
early on whichever single metric happened to catch attention first — is specifically designed to
surface the true bottleneck efficiently: a system exhibiting poor application performance with
low CPU utilization, low memory pressure, and low network utilization, but very high storage-device
saturation (a long, growing `iostat` queue length despite the device's raw utilization percentage not
yet reading a full 100%), correctly directs an investigation toward the storage subsystem specifically,
avoiding wasted time exhaustively investigating CPU or application code when the resource actually
under contention was never CPU at all. USE's discipline of checking utilization, saturation, *and*
errors for *every* resource, rather than stopping at the first metric that looks superficially
concerning, is precisely the structured rigor that separates a systematic, efficient troubleshooting
methodology from unstructured guess-and-check.

### Key commands
```
mpstat -P ALL 1                # CPU: utilization per core
vmstat 1                         # CPU/memory: utilization and saturation (r/b queue columns)
iostat -x 1                       # storage: utilization (%util) and saturation (avgqu-sz/aqu-sz)
ethtool -S eth0 | grep -i err       # network: error counters
```

## Latency vs Throughput Analysis

Latency (how long a single operation takes) and throughput (how many operations complete per unit
time) are related but distinct performance dimensions that can move independently of each other, and
conflating them is a common source of misdiagnosed performance problems. A system can have excellent
aggregate throughput while individual request latency is poor — a batching or queuing design that
processes many requests efficiently in bulk but makes each individual request wait in a queue before
being included in the next batch trades increased per-request latency for higher aggregate throughput,
a deliberate and often entirely correct engineering trade-off for bulk/batch workloads, but a poor fit
for latency-sensitive interactive workloads where consistent low per-request latency matters far more
than raw aggregate operation count. Conversely, a system can show excellent (low) median/average
latency while still having a serious problem visible only in the tail: reporting only a mean or
median latency figure systematically hides tail behavior (p95, p99, p99.9 percentiles) that
frequently matters most for user-perceived experience and SLA compliance, since a small fraction of
requests experiencing severe latency (perhaps due to occasional GC pauses, lock contention, or a
specific slow code path only triggered by certain input patterns) can still represent a very large
absolute number of poorly-served requests at any meaningful scale, entirely invisible if only
central-tendency statistics are examined. Correctly analyzing a performance problem requires being
explicit about which dimension actually matters for the workload/SLA in question (a batch ETL job
genuinely cares primarily about throughput and total completion time; a user-facing API cares
primarily about tail latency and can often tolerate comparatively modest aggregate throughput) and
using percentile-based, full-distribution analysis (histograms, percentile breakdowns) rather than
single summary statistics whenever tail behavior is operationally relevant — a mature interview answer
to "how do you think about performance" should explicitly distinguish these dimensions rather than
treating "performance" as a single undifferentiated concept.

### Key commands
```
ss -tin                          # per-connection RTT (a latency-relevant metric) alongside throughput-relevant cwnd
fio --output-format=json ... | jq '.jobs[0].read.clat_ns.percentile'   # full latency percentile breakdown from a benchmark
perf sched latency                 # scheduler-induced latency breakdown per task
```

## Benchmarking Tools (fio, iperf, stress-ng)

Reliable performance investigation and capacity planning both depend on being able to generate
controlled, repeatable synthetic load against a specific subsystem in isolation, rather than relying
solely on unpredictable, hard-to-reproduce real production traffic patterns for every measurement.
`fio` (Flexible I/O tester) is the standard tool for storage benchmarking, capable of precisely
configuring I/O pattern (sequential vs random, read vs write vs mixed ratios), block size, queue depth/
parallelism (`iodepth`, `numjobs`), and I/O engine (`libaio`/`io_uring` for genuinely asynchronous,
high-queue-depth testing representative of real database/high-performance-storage workloads, versus
simple synchronous engines more representative of naive application I/O patterns) — letting an
engineer directly answer questions like "what's this specific storage device/array's actual achievable
IOPS and latency under a 4K random-read workload at queue depth 32" with a controlled, repeatable
measurement rather than inferring it indirectly from production behavior alone. `iperf`/`iperf3` is
the equivalent standard for network throughput benchmarking, measuring achievable bandwidth (and,
with appropriate options, latency/jitter for UDP testing) between two hosts directly at the
TCP/UDP layer, isolating pure network-path capability from any application-level processing overhead
that might otherwise confound a measurement based purely on observing real application traffic.
`stress-ng` provides configurable synthetic CPU, memory, I/O, and even more exotic stressors (cache
contention, specific instruction-mix patterns) specifically for validating system behavior and
stability under controlled resource pressure — useful both for proactively validating a system handles
expected peak load gracefully before it's ever exposed to real traffic, and for deliberately
reproducing resource-exhaustion scenarios (as used in several of this guide's earlier hands-on labs)
in a controlled way rather than needing to wait for a genuine, unpredictable production incident to
observe the same failure mode. A consistent theme across all three tools: they exist specifically to
let an engineer isolate and measure one resource dimension precisely and repeatably, which is a
necessary complement to (not a replacement for) the observational tools covered throughout this
section — benchmarking answers "what is this subsystem's actual capability," while observability
tools answer "what is actually happening right now on this specific system."

### Key commands
```
fio --name=randread --ioengine=libaio --rw=randread --bs=4k --iodepth=32 --size=1G --numjobs=4 --runtime=60 --group_reporting
iperf3 -c <server-ip> -t 30            # measure achievable TCP throughput to a remote host for 30 seconds
stress-ng --cpu 4 --vm 2 --vm-bytes 1G --timeout 60s   # generate combined CPU and memory pressure
stress-ng --fork 0 --timeout 10s         # (careful, disposable VM only) simulate fork-bomb-like pressure for testing limits
```

---

### Interview Questions

**Conceptual/Internals (8)**

1. **Why does `strace` impose significant overhead, and how do you minimize its impact when
   investigating a production issue?**
   `strace` uses `ptrace()` to intercept and pause the target process at every single syscall
   entry/exit, and this attach-and-pause mechanism has real per-syscall cost, compounding heavily for
   syscall-frequent workloads. Minimize impact by scoping tightly with `-e trace=` to only the specific
   syscalls of interest, attaching to an already-running process rather than launching fresh, and
   preferring `-c` (aggregated summary) over a full line-by-line trace when only counts/timing are
   needed rather than the full sequence.

2. **What is a flame graph actually visualizing, and how is it generated?**
   A flame graph visualizes sampled call-stack data (typically from `perf record -g`), with each
   function represented as a box whose width is proportional to the fraction of total samples in which
   that function appeared anywhere in the captured call stack, and vertical position representing call
   stack depth. It's generated by collapsing `perf`'s raw sampled stack traces into a folded format and
   rendering that as a stacked, width-proportional bar chart (via tools like Brendan Gregg's
   FlameGraph scripts).

3. **Why is `ss` generally preferred over `netstat` on modern systems, especially for connection-heavy
   hosts?**
   `ss` uses the netlink socket-diagnostic API, allowing filtering to happen in-kernel before data is
   returned to userspace, while `netstat` parses the entire `/proc/net/*` text representation of every
   socket on the system for every query. On hosts with very large connection counts, this makes
   `netstat` itself a slow, resource-intensive command, ironically worst exactly when a fast diagnostic
   tool is most needed.

4. **Explain the USE Method and what problem it's specifically designed to prevent.**
   USE prescribes checking Utilization, Saturation, and Errors for every resource in a system
   systematically, specifically to prevent the common failure mode of fixating on whichever single
   metric happened to catch attention first and missing the true bottleneck in a resource dimension
   that was never checked at all — a resource can be performing poorly due to saturation or errors even
   while its raw utilization percentage looks unremarkable.

5. **Why can `top`'s reported %CPU value miss a very short-lived, bursty process entirely?**
   `top` computes %CPU as a rate between two successive `/proc/<pid>/stat` samples taken at its
   refresh interval; a process that starts and exits entirely between two consecutive samples is never
   observed by `top` at all, regardless of how much CPU it actually consumed during its brief
   lifetime, since there is no continuous background collection independent of `top`'s own polling
   interval.

6. **What is the difference between latency and throughput, and why can optimizing for one hurt the
   other?**
   Latency measures how long a single operation takes; throughput measures how many operations
   complete per unit time. Batching/queuing strategies commonly improve aggregate throughput precisely
   by increasing individual-request latency (holding requests briefly to process them together more
   efficiently in bulk), which is a reasonable trade-off for throughput-oriented batch workloads but
   harmful for latency-sensitive interactive workloads, making it essential to know which dimension
   actually matters for a given workload before optimizing.

7. **Why does `core_pattern`'s pipe syntax matter for how systemd-coredump captures crashes
   system-wide?**
   Setting `core_pattern` to a pipe (`|/path/to/handler ...`) routes raw core dump data through an
   external handler process at the moment of a crash, rather than writing a flat file directly to the
   crashing process's working directory. This is exactly the mechanism `systemd-coredump` uses to
   intercept every crash centrally, compress and store it in a structured, queryable location, and
   attach rich metadata, rather than requiring each application to independently manage its own core
   dump storage.

8. **Why is bpftrace generally considered lower-risk than writing an ad-hoc custom kernel module for
   the same investigative purpose?**
   bpftrace compiles scripts down to eBPF programs, which the kernel verifies before loading —
   checking for bounded loops, memory-access safety, and other properties that guarantee the program
   cannot crash or hang the kernel. A hand-written kernel module has no equivalent safety verification
   and runs with full, unchecked kernel privilege, making a bug in it capable of crashing or corrupting
   the entire system in ways a verified eBPF program is specifically designed to prevent.

**Scenario/Troubleshooting (6)**

9. **An application intermittently fails with a generic "permission denied" error with no further
    detail in its own logs. How do you find the exact cause quickly?**
    Attach `strace -f -e trace=open,openat -p <pid>` (or launch fresh under `strace` if reproducible on
    demand) to see exactly which file path the process is attempting to open at the moment of failure,
    collapsing what could be extensive log-diving or guesswork into a single definitive syscall-level
    answer, then cross-reference that path against both DAC permissions and, if applicable, SELinux/
    AppArmor denial logs from Section 6.

10. **A host shows high aggregate CPU utilization in a monitoring dashboard, but engineers can't
    identify which specific process is responsible via periodic `top` snapshots.**
    The responsible process is likely short-lived/bursty and falling between `top`'s sampling
    intervals; use `pidstat 1` (or a `perf record -a` system-wide capture) to get finer-grained,
    genuinely continuous historical per-process data rather than relying on `top`'s periodic point-in-
    time snapshots, which can systematically miss processes whose entire lifetime falls between
    samples.

11. **`iostat -x` shows a storage device at 100% `%util` but the application team insists throughput
    still has headroom. How do you reconcile this?**
    `%util` measures the percentage of time the device had at least one outstanding request, not
    necessarily that it's at its absolute throughput ceiling — a device supporting meaningful queue
    depth can still accept more parallel requests even at 100% util if `avgqu-sz`/`aqu-sz` isn't yet
    very high and `await` remains reasonable. Confirm true saturation by checking whether queue depth
    and average wait time are actually climbing under increased load, rather than relying on `%util`
    alone as a saturation indicator.

12. **A production service crashes intermittently, but no core dump is available for post-mortem
    analysis when it happens.**
    Check `ulimit -c` for the service's actual runtime user/systemd unit (core dumps are frequently
    disabled by default) and `/proc/sys/kernel/core_pattern` for correct routing; configure
    systemd-coredump (or an equivalent flat-file core_pattern with adequate storage) proactively before
    the next occurrence, since a crash that isn't captured when it happens cannot be analyzed
    afterward no matter how sophisticated the available tooling.

13. **A load balancer's monitoring shows low average request latency, but customer complaints about
    slowness persist. What's the likely gap in the monitoring, and what should you add?**
    Average/median latency systematically hides tail latency (p95/p99/p99.9) — a small fraction of
    requests experiencing severe latency can represent a large absolute number of poorly-served
    requests at scale while barely moving an average. Add percentile-based latency monitoring and
    alerting specifically on tail percentiles, not just mean/median, to make this class of problem
    visible.

14. **After enabling a new eBPF-based observability agent, a latency-sensitive service shows a small
    but consistent throughput regression. How would you validate whether the agent is the cause and
    what would you check?**
    Compare `perf stat`/hardware counters and `bpftool prog list`/`bpftool prog profile` (or the
    agent's own reported overhead) with the agent enabled versus disabled under an otherwise identical
    controlled benchmark (`fio`/`iperf3`/a representative synthetic load), since even verified,
    low-overhead eBPF programs are not literally free — attaching to very high-frequency hook points
    (like every syscall or every packet) can impose a small but measurable per-event cost that becomes
    significant in aggregate for extremely high-throughput, latency-sensitive workloads.

**FAANG-level Deep Dive (6)**

15. **Explain why `perf record`'s sampling-based profiling can produce a misleading picture for a
    workload dominated by very short-lived function calls, and what mitigation exists.**
    Sampling captures the current instruction pointer/call stack at a fixed frequency; a function whose
    individual invocations are shorter than the average interval between samples may be systematically
    under-represented (or entirely missed) in the resulting profile purely due to sampling granularity,
    even if it's called extremely frequently and its aggregate contribution to total runtime is
    significant. Mitigation includes increasing sampling frequency (`-F` in `perf record`, at the cost
    of higher observer-effect overhead) or supplementing sampling-based profiling with tracepoint/
    uprobe-based exact-count instrumentation (via `bpftrace`/`ftrace`) for specifically that function
    when sampling granularity is suspected to be hiding its true contribution.

16. **Why does netlink-based socket querying (used by `ss`) scale better than `/proc/net/*` text
    parsing (used by `netstat`) specifically as connection count grows, at a mechanistic level?**
    `/proc/net/tcp` and similar files must be fully generated (walking the kernel's entire socket hash
    table) and then fully parsed as text by the querying tool for every single invocation regardless of
    how narrow the actual query is, with cost scaling linearly with total socket count every time.
    Netlink's socket-diagnostic API allows the query itself (filters on state, address family, and
    other criteria) to be passed into the kernel, letting the kernel return only matching sockets
    directly in a structured binary format, avoiding both the full-table text generation and the
    full-table text parsing that `netstat`'s approach requires regardless of how selective the final
    displayed output is.

17. **Explain precisely why a fixed-size kernel ring buffer (as used by `dmesg`/`printk`) is the right
    design choice for kernel logging despite the data-loss risk of overwriting old messages, rather
    than an unbounded, dynamically-growing buffer.**
    An unbounded buffer risks unconstrained memory consumption specifically during pathological
    conditions (a runaway driver logging errors in a tight loop, or a genuine crash/panic scenario
    generating an enormous burst of diagnostic messages) — precisely the conditions under which kernel
    logging is most critical and memory may already be under severe pressure, making an unbounded
    buffer's own memory consumption a potential contributor to system instability rather than purely a
    diagnostic aid. A fixed-size ring buffer bounds this worst-case memory cost predictably regardless
    of message volume, at the cost of eventually overwriting older messages — a trade-off explicitly
    addressed operationally by forwarding messages to persistent, effectively-unbounded external
    storage (journald/syslog) for anything that must survive longer than the ring buffer's own fixed
    capacity allows.

18. **Why can two engineers investigating the same intermittent latency spike reach different
    conclusions if one relies solely on `perf record`'s CPU-cycle-based sampling while the other uses
    `bpftrace` to measure actual wall-clock syscall latency?**
    CPU-cycle-based sampling only captures where the CPU is actively executing instructions — a thread
    blocked in uninterruptible sleep waiting on slow I/O (as discussed in Section 2) consumes
    essentially zero CPU cycles during that wait and is therefore effectively invisible to a purely
    CPU-sampling-based profile, even though it may be the actual dominant contributor to the observed
    wall-clock latency spike. A `bpftrace` script measuring actual elapsed time between syscall entry
    and exit directly captures this blocked-waiting duration regardless of CPU activity, correctly
    attributing the latency to the syscall/I/O wait that a CPU-sampling-only view would systematically
    miss — this is exactly why latency investigation frequently requires combining CPU-profiling tools
    with wall-clock/syscall-latency tracing rather than relying on either alone.

19. **Why does benchmarking storage with `fio` using a synchronous I/O engine typically produce
    substantially different (and often misleading) results compared to `libaio`/`io_uring` for a
    workload meant to represent a real high-performance database?**
    A synchronous I/O engine issues one request, blocks until it completes, then issues the next —
    meaning the achievable queue depth is effectively always 1, regardless of any `iodepth` setting,
    which fails to exercise the storage device's/controller's ability to service many requests
    concurrently (a major source of achievable IOPS on modern SSD/NVMe hardware, whose internal
    parallelism is specifically designed to be exploited by concurrent, overlapping requests).
    `libaio`/`io_uring` genuinely submit multiple requests without blocking between them, correctly
    exercising this concurrency, which is why benchmark configuration must match the actual I/O
    concurrency pattern of the real target workload to produce results that meaningfully predict real
    application performance rather than measuring an artificially serialized, unrepresentative access
    pattern.

20. **Explain why a percentile-based latency SLA (e.g., "p99 < 200ms") can still be technically met
    while a meaningful fraction of *users* experience unacceptable latency, and what additional
    analysis reveals this gap.**
    A p99 latency figure describes the distribution of individual *requests*, not users — a small
    subset of users who happen to generate a disproportionate number of requests (a "power user"
    pattern, or a workload where a single problematic backend shard/dependency consistently serves a
    specific subset of users) can experience a much higher effective per-user latency even while the
    overall request-level p99 remains within SLA, because their poor-latency requests are diluted
    across the full population of otherwise-fine requests from everyone else. Revealing this requires
    analyzing latency distribution segmented by user/tenant/backend-shard rather than purely in
    aggregate across the entire undifferentiated request population, a genuinely important distinction
    between "the SLA metric looks fine in aggregate" and "every user is actually having a good
    experience."

### Hands-On Labs

**Lab 1: Diagnose a synthetic "permission denied" mystery with strace**
- Objective: Practice the canonical `strace`-based root-cause workflow.
- Setup: A test program deliberately misconfigured to fail opening a specific file due to a subtle
  permission or path issue, without printing the failing path itself.
- Tasks: Attach `strace -f -e trace=open,openat` to the failing program; identify the exact failing
  path and errno; fix the underlying permission/path issue and confirm success.
- Expected outcome: A documented root-cause identification purely from syscall-level tracing.

**Lab 2: Generate and interpret a CPU flame graph**
- Objective: Produce and read a real flame graph for a CPU-bound workload.
- Setup: A small CPU-intensive test program with an intentionally inefficient function.
- Tasks: Profile with `perf record -g`; generate a flame graph; identify the inefficient function as
  the widest box; optimize it and regenerate the flame graph to confirm the width (and total runtime)
  shrank.
- Expected outcome: A before/after flame graph pair demonstrating a measured, visually-confirmed
  optimization.

**Lab 3: Write a bpftrace one-liner for live syscall investigation**
- Objective: Get hands-on with bpftrace for a realistic investigative task.
- Setup: A Linux VM with bpftrace installed and a test workload generating file opens.
- Tasks: Write a one-liner tracing `openat` calls system-wide with process name and filename; run it
  while the test workload executes; confirm it captures every file open accurately.
- Expected outcome: A working, verified live syscall-tracing one-liner.

**Lab 4: USE Method walkthrough on an induced bottleneck**
- Objective: Apply the USE Method systematically to correctly identify an induced bottleneck.
- Setup: A VM where you deliberately induce one specific resource bottleneck (e.g., artificially
  throttled disk I/O via `dm-delay`, without telling yourself in advance which resource you throttled
  if practicing solo).
- Tasks: Systematically check Utilization/Saturation/Errors for CPU, memory, each storage device, and
  network; identify the actually-bottlenecked resource purely from this systematic checklist.
- Expected outcome: Correct identification of the induced bottleneck via disciplined USE Method
  application, not guesswork.

**Lab 5: Core dump capture and post-mortem analysis**
- Objective: Configure core dump capture and perform real post-mortem debugging.
- Setup: A small C program with a deliberate segfault bug, and `gdb`.
- Tasks: Confirm/set `ulimit -c unlimited` and appropriate `core_pattern`; run the program to crash;
  load the resulting core dump in `gdb` (or via `coredumpctl debug`); use `bt` to identify the exact
  line and cause of the crash without ever running the program under a live debugger.
- Expected outcome: A successful, documented post-mortem root-cause identification purely from a
  captured core dump.

### Production Incidents

**Incident 1: A critical alert with no clear cause traced to a bursty process invisible to standard
monitoring**
- Symptom: A monitoring dashboard shows brief, periodic CPU utilization spikes to 100% lasting only a
  few seconds each, with `top` snapshots taken during on-call investigation never showing any single
  process responsible.
- Investigation: Switched from periodic `top` snapshots to continuous `pidstat 1` logging and a
  system-wide `perf record -a` capture spanning several spike occurrences, revealing a short-lived
  batch/cron-triggered process whose entire lifetime (a few hundred milliseconds) fell reliably between
  `top`'s default refresh intervals.
- Root cause: A misconfigured cron job was launching a CPU-intensive but very short-lived subprocess
  far more frequently than intended, invisible to point-in-time `top` snapshots but real and
  significant in aggregate CPU consumption.
- Recovery: Corrected the cron job's frequency misconfiguration.
- Prevention: Added continuous, always-on `pidstat`/historical process accounting (rather than relying
  solely on live `top` snapshots during ad-hoc investigation) to standard monitoring infrastructure,
  specifically to catch this class of short-lived-process issue going forward.

**Incident 2: A critical crash went unanalyzable for weeks due to disabled core dumps**
- Symptom: A production service crashes roughly weekly with no clear pattern, and each occurrence is
  simply restarted by its supervisor with no root-cause analysis possible.
- Investigation: Confirmed `ulimit -c` for the service's systemd unit was effectively 0 (default,
  never explicitly configured) and no core dump was ever being generated at the moment of any crash,
  leaving nothing to analyze after the fact regardless of how many times the crash recurred.
- Root cause: Core dump capture was never explicitly configured for this service, a gap unnoticed
  precisely because the service's automatic restart made each individual crash operationally
  invisible/low-impact in isolation, removing the usual pressure to investigate root cause.
- Recovery: Configured `LimitCORE=infinity` on the systemd unit and verified `systemd-coredump`
  capture; the very next occurrence was successfully captured and root-caused via `gdb`/`coredumpctl`.
- Prevention: Added a standard checklist item to new-service onboarding requiring explicit core-dump
  capture verification before a service is considered production-ready, rather than relying on the
  platform default.

**Incident 3: Tail-latency SLA violations invisible in average-latency dashboards**
- Symptom: Customer complaints about intermittent slow responses persist for weeks despite the
  team's primary latency dashboard (showing average and median response time) remaining comfortably
  within SLA the entire time.
- Investigation: Added p95/p99/p99.9 percentile breakdowns to the same dashboard and immediately
  revealed a consistently elevated p99.9 tail correlated with the complaint reports, traced via
  `bpftrace`-based syscall latency tracing to occasional multi-second lock contention in a specific,
  rarely-exercised code path triggered only by a particular request pattern.
- Root cause: The team's dashboards and alerting had only ever been configured around
  average/median latency, which diluted the tail-latency problem into invisibility despite it being
  large and real for the specific affected requests.
- Recovery: Fixed the identified lock-contention code path; confirmed the p99.9 tail improvement in
  the newly-added percentile dashboard.
- Prevention: Standardized percentile-based (not average-based) latency monitoring and alerting as the
  required default for all customer-facing services going forward, explicitly informed by this
  incident's demonstrated gap.
