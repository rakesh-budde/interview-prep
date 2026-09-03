# Section 2: Process Management & Scheduling

This section covers how Linux represents, creates, schedules, and terminates processes and threads —
the kernel data structures and algorithms behind `fork()`, the Completely Fair Scheduler, signals, and
everything you need to reason about CPU-bound performance and "why is this process stuck" incidents.

## Subtopic Index
- [Processes vs Threads](#processes-vs-threads)
- [`task_struct` internals](#task_struct-internals)
- [Process Creation: fork(), vfork(), clone(), execve()](#process-creation-fork-vfork-clone-execve)
- [Copy-on-Write (COW)](#copy-on-write-cow)
- [Process States and State Transitions](#process-states-and-state-transitions)
- [Zombie and Orphan Processes](#zombie-and-orphan-processes)
- [Process Termination and Reaping (wait/waitpid)](#process-termination-and-reaping-waitwaitpid)
- [Process Groups and Sessions](#process-groups-and-sessions)
- [Signals and Signal Handling](#signals-and-signal-handling)
- [Signal Masking and Pending Signals](#signal-masking-and-pending-signals)
- [Context Switching](#context-switching)
- [Linux CPU Scheduler (CFS)](#linux-cpu-scheduler-cfs)
- [Scheduling Classes (SCHED_OTHER, SCHED_FIFO, SCHED_RR, SCHED_DEADLINE)](#scheduling-classes-sched_other-sched_fifo-sched_rr-sched_deadline)
- [Nice Values and Priorities](#nice-values-and-priorities)
- [Load Average vs CPU Utilization](#load-average-vs-cpu-utilization)
- [Preemption (voluntary/involuntary)](#preemption-voluntaryinvoluntary)
- [SMP and Multi-core Scheduling](#smp-and-multi-core-scheduling)
- [CPU Affinity and NUMA-aware Scheduling](#cpu-affinity-and-numa-aware-scheduling)
- [Real-Time Scheduling](#real-time-scheduling)

---

## Processes vs Threads

A process is a unit of resource ownership — it has its own virtual address space, file descriptor
table, signal handlers, and security context (UID/GID, capabilities) — while a thread is a unit of
execution that shares all of that with its sibling threads except for a private stack, register set,
and a small amount of thread-local state (TLS). On Linux, this distinction is almost entirely an
artifact of userspace convention rather than a hard kernel boundary: the kernel scheduler doesn't
actually have a first-class "process" object distinct from a "thread" object — both are represented by
the same `task_struct`, and what userspace calls a "thread" is simply a `task_struct` created via
`clone()` with flags that tell the kernel to *share* the parent's memory descriptor (`mm_struct`), file
descriptor table (`files_struct`), and signal handlers, rather than copying them. This is why Linux's
`ps -eLf` and `top -H` can show individual threads as separate schedulable entities with their own
kernel-assigned thread ID (visible as the `LWP` or via `gettid()`), while `getpid()` still returns the
shared, group-level PID that all threads of one process report. glibc's pthreads library is built
entirely on top of `clone()` with the right flag combination (`CLONE_VM|CLONE_FS|CLONE_FILES|
CLONE_SIGHAND|...`) — there is no separate "thread scheduler" in the kernel; the CFS scheduler simply
treats every `task_struct` as an independently schedulable unit, whether it happens to share an
address space with others or not, which is also why CPU-bound multi-threaded programs scale near-
linearly across cores (each thread genuinely runs on a separate core protected by real hardware
parallelism, not cooperative userspace switching). The practical interview distinction to hold onto:
threads are cheap to create and communicate through shared memory with no syscall overhead, but they
share fate — one thread's stray write can corrupt another thread's data in the same address space
since there's no memory protection between them, whereas a bug in one process can't directly corrupt
another process's memory because the MMU enforces separate page tables per `mm_struct`.

### Key commands
```
ps -eLf                      # list threads (LWP column) alongside process PID
top -H -p <pid>               # per-thread CPU usage within one process
cat /proc/<pid>/status | grep Threads    # thread count for a process
ls /proc/<pid>/task/          # one directory per thread, each with its own stack/stat info
```

## `task_struct` internals

Every process and every thread in the Linux kernel is represented by exactly one instance of `struct
task_struct` (defined in `include/linux/sched.h`), a large structure that is the kernel's complete
bookkeeping record for a schedulable entity. Key fields include: `pid`/`tgid` (the kernel-internal
thread ID and the userspace-visible "thread group ID" that `getpid()` actually returns — for a
single-threaded process these are equal, but for additional threads created via `clone()`, `pid` is
unique per thread while `tgid` stays the same across the whole process, which is the real mechanism
behind the processes-vs-threads distinction discussed above); `state`/`__state` (the current
scheduling state — running, interruptible sleep, uninterruptible sleep, stopped, zombie); `mm`
(pointer to the `mm_struct` describing the virtual address space — shared between threads of one
process, unique per process); `files` (pointer to the `files_struct` open file descriptor table,
similarly shared or private depending on clone flags); `sched_entity`/`sched_class` (the scheduler's
own bookkeeping — virtual runtime, scheduling class pointer determining which algorithm governs this
task); `signal`/`sighand` (pending signals, blocked signal mask, registered handlers); `cred`
(credentials — real/effective/saved UID and GID, capability sets, used for every permission check);
`thread_pid`/`real_parent`/`parent`/`children`/`sibling` (the process hierarchy links used for
reparenting on exit and `wait()` semantics); and `cgroups` (pointer to this task's cgroup membership
across all active cgroup hierarchies, which is how cgroup limits/accounting actually get enforced per
task at schedule/charge time). All `task_struct`s are linked into a circular doubly-linked list (the
"task list") walked by `ps`/`/proc` enumeration, and additionally indexed by PID in a radix tree
(`pid_hash`) for O(1)-ish lookup by PID during syscalls like `kill()`. Understanding that
`task_struct` is the *single* unifying representation for both processes and threads is the key that
unlocks a lot of otherwise-confusing Linux behavior — e.g., why `/proc/<pid>/task/<tid>/` exists as a
directory for every thread, each with its own independent `stat`/`status`/`stack` entries, because
each thread genuinely is a distinct `task_struct` with distinct scheduling and signal-delivery state.

### Key commands
```
cat /proc/<pid>/status         # human-readable dump of key task_struct-derived fields
cat /proc/<pid>/stat            # raw scheduling stats (state, priority, utime, stime, etc.)
cat /proc/<pid>/sched            # scheduler-specific fields (vruntime, nr_switches, etc.)
crash> struct task_struct <addr> # (crash utility, kernel debugging) dump the actual struct from a core/live kernel
```

## Process Creation: fork(), vfork(), clone(), execve()

`fork()` is the traditional UNIX process-creation primitive: it creates a new `task_struct` that is an
almost-exact duplicate of the calling process — same code, same data, same open file descriptors, same
signal handlers — differing only in PID, PPID, and the return value (0 in the child, the child's PID
in the parent), with execution resuming *twice*, once in each process, right after the `fork()` call
returns. Historically this duplicated the entire address space physically, which was expensive for
large processes; modern Linux instead implements `fork()` via `clone()` with copy-on-write semantics
(see below), making the actual duplication cost proportional to page-table entries, not memory
content, until pages are actually modified. `vfork()` is a rarely-used optimization predating
copy-on-write's maturity: it suspends the parent and has the child temporarily share the parent's
address space directly (no copying, not even page tables) under the strict contract that the child
will only call `execve()` or `_exit()` immediately, never write to memory or return normally — violating
this contract corrupts the parent, so `vfork()` is essentially obsolete now that COW `fork()` is
cheap, but you may still see it in latency-critical old code or busybox-style shells. `clone()` is the
actual, general-purpose syscall underlying both `fork()` and thread creation — it takes an explicit
bitmask of flags (`CLONE_VM` to share the address space instead of copying it, `CLONE_FILES` to share
the file descriptor table, `CLONE_FS` to share filesystem info like cwd/umask, `CLONE_SIGHAND` to
share signal handler tables, plus namespace flags like `CLONE_NEWPID`/`CLONE_NEWNET` used by container
runtimes) that lets a caller precisely choose what to share versus duplicate — pthreads passes nearly
every "share" flag, `fork()` passes none of them (full duplication, then COW), and container runtimes
pass namespace flags to build isolated execution contexts. `execve()` is conceptually separate from
process creation entirely: it doesn't create a new process, it replaces the calling process's entire
program image (code, data, stack, heap) with a new one loaded from an executable file, while
preserving the same PID, open file descriptors (unless marked `close-on-exec`), and process
group/session — the classic UNIX idiom of "fork, then exec" is precisely this: `fork()` cheaply
duplicates the current process to get a new PID/task_struct, and the child immediately calls
`execve()` to replace its own program image with the desired new program, at which point copy-on-write
pages inherited from the parent are simply discarded since the child's address space is entirely
replaced.

```
        fork()                          execve()
Parent ────────► Child (COW copy of      Child ────────► Child now runs a
   task_struct     parent's task_struct,   task_struct     completely different
   PID=100         same code/data via       PID=101         program image loaded
                   shared, marked           (same PID,      from disk; old code/
                   read-only pages)         PPID unchanged) data/heap discarded
```

### Key commands
```
strace -f -e trace=clone,fork,vfork,execve <cmd>   # observe exact syscalls used for a given program
cat /proc/<pid>/status | grep -E 'Threads|State'    # sanity check process vs thread relationships
ltrace -f <cmd>                                     # library-call level trace (glibc wrappers)
```

## Copy-on-Write (COW)

Copy-on-write is the optimization that makes `fork()` cheap despite conceptually duplicating an
entire address space. Instead of physically copying every page of the parent's memory into new
physical frames for the child, the kernel duplicates only the parent's page tables, marks every
mapped page in both parent and child as read-only, and increments a reference count on each physical
page frame — both processes' virtual addresses now point at the *same* physical pages, and since
they're marked read-only, either process attempting to write triggers a page fault. The kernel's page
fault handler recognizes this specific fault as a COW fault (distinguished by checking the page's
reference count and a VMA flag indicating it should have been writable), allocates a brand-new
physical page, copies the original page's contents into it, updates only the faulting process's page
table entry to point at the new private page (marked writable this time), and decrements the
reference count on the original shared page — the other process's mapping is untouched and continues
sharing the original page. This means the actual cost of `fork()` is proportional to the number of
page-table entries that need duplicating (fast, and increasingly optimized further with huge pages
reducing table size), not the size of the address space's *content*, and the cost of subsequent
writes is deferred and paid only for pages that are actually modified — a very common pattern like
`fork()` immediately followed by `execve()` (as in shell command execution) barely touches any
inherited pages at all before they're discarded, making COW's deferred-copy strategy essentially free
in that case. COW is also the same underlying mechanism used for `mmap(MAP_PRIVATE)` mappings of
files — multiple processes mapping the same file read-only share physical pages until one of them
writes, at which point that one process gets a private copy, which is exactly how shared library code
pages (`.so` files) are mapped identically (and once) across every process using them, while each
process's writable data segment remains private.

### Key commands
```
cat /proc/<pid>/smaps | grep -A2 Private   # distinguish private vs shared page counts per mapping
perf stat -e page-faults ./program          # count total page faults, including COW faults, for a workload
cat /proc/vmstat | grep -i cow              # (kernel version dependent) COW-related fault counters
```

## Process States and State Transitions

Every `task_struct` carries a scheduling state describing what the process is currently doing, and
transitions between these states are what the scheduler and various wakeup mechanisms drive. `R`
(TASK_RUNNING) means the task is either actually executing on a CPU right now or sitting on the run
queue ready to execute the instant the scheduler picks it — `ps`/`top` don't distinguish "running" from
"runnable-but-waiting-for-CPU," both show as `R`. `S` (TASK_INTERRUPTIBLE) is a sleeping state where the
task is waiting for some event (I/O completion, a mutex, a timer, data on a socket) but can be woken
early by a signal delivery — most idle processes waiting on `select()`/`poll()`/`read()` from a
terminal sit here, and this is the overwhelmingly common state for the vast majority of processes on
an idle-ish system. `D` (TASK_UNINTERRUPTIBLE) is the state that generates the most production
incidents: the task is sleeping waiting for I/O (typically block-device or NFS I/O) in a context where
the kernel considers it unsafe to interrupt with a signal — this state cannot be killed with `SIGKILL`
while it persists, and processes stuck in `D` state for a long time are the classic symptom of a
failing disk, an overloaded storage backend, or a hung NFS mount, and they directly inflate the "load
average" number even though the CPU itself may be sitting completely idle. `T`/`t` (TASK_STOPPED /
TASK_TRACED) means the process has been suspended by a `SIGSTOP`/`SIGTSTP` or is being traced by a
debugger via `ptrace()`. `Z` (EXIT_ZOMBIE) means the process has already terminated and released all
its resources but its `task_struct` remains in the task list solely to hold its exit status until the
parent calls `wait()`/`waitpid()` to collect it (see Zombie processes below). Transitions between
these states are driven by the scheduler (R↔runnable/running boundary), by wakeup functions called
from interrupt handlers or other processes (S/D → R, when the awaited event occurs — e.g., a disk
interrupt handler calling `wake_up()` on the tasks blocked waiting for that I/O), and by signal
delivery/`wait()` reaping for the T/Z states.

```
                     scheduled off CPU (voluntary or involuntary)
        ┌────────────────────────────────────────────┐
        ▼                                              │
   ┌─────────┐   blocks on I/O/lock/signal wait   ┌────┴────┐
   │ RUNNING │ ───────────────────────────────────▶│ S or D  │
   │ (R)     │◀─────────────────────────────────── │ sleeping│
   └─────────┘   event occurs, task woken up        └─────────┘
        │                                                │
        │ SIGSTOP                    process exits ──────┘
        ▼                                    │
   ┌─────────┐                                ▼
   │ STOPPED │                          ┌───────────┐   parent calls wait()
   │ (T)     │                          │  ZOMBIE   │──────────────────────▶ task_struct freed
   └─────────┘                          │  (Z)      │
```

### Key commands
```
ps -eo pid,stat,comm            # STAT column shows current state (R/S/D/T/Z + modifiers like <,N,s,l,+)
ps -eo pid,stat,comm | grep ' D' # find all uninterruptible-sleep processes — classic I/O-stall symptom
cat /proc/<pid>/stat | awk '{print $3}'   # raw state character for one process
watch -n1 'ps -eo stat= | sort | uniq -c'  # live histogram of all process states on the system
```

## Zombie and Orphan Processes

A zombie is a process that has called `exit()` (or been terminated by a signal) but whose exit status
has not yet been collected by its parent via `wait()`/`waitpid()`; the kernel keeps a minimal
`task_struct` around — no memory, no file descriptors, nothing but PID, exit status, and resource usage
accounting — purely so the parent can eventually retrieve that exit status, since UNIX semantics
guarantee a parent can always learn how its child exited. A process becomes a *permanent* zombie
problem when its parent is buggy or simply never calls `wait()` — the zombie itself consumes only a
tiny sliver of kernel memory (a `task_struct`) and no other resources, so a few zombies are harmless,
but a process that spawns thousands of children and never reaps any of them will eventually exhaust
the PID space or hit process-count `ulimit`s, which is a subtle but real production failure mode.
Zombies cannot be killed by `SIGKILL` because they aren't running anything to kill — the "kill" that
actually matters is fixing or killing the *parent*, at which point every remaining zombie (and every
still-alive child) is reparented. An orphan is the mirror-image scenario: a still-*running* process
whose original parent has died before it did. On Linux, orphans are reparented not necessarily to PID
1 as older UNIX folklore assumes, but to the nearest "subreaper" ancestor — by default that's `init`
(PID 1, systemd), but a process can mark itself as a subreaper via `prctl(PR_SET_CHILD_SUBREAPER)`
specifically so orphaned grandchildren are reparented to it instead of escaping all the way to PID 1;
container runtimes and process supervisors use this to make sure they, not the host's PID 1, are
responsible for reaping any orphaned descendants of a container's processes. This reparenting is also
precisely why a zombie's parent dying doesn't leave the zombie stuck forever: once reparented (to
init or a subreaper), the new parent is expected to periodically `wait()` on all its children,
collecting and clearing out any zombies that were handed to it.

### Key commands
```
ps -eo pid,ppid,stat,comm | grep 'Z'      # find zombie processes (STAT shows Z)
ps -o pid,ppid,stat,comm -p <ppid>         # inspect a suspected buggy parent not reaping children
pstree -p <ppid>                           # visualize the process tree to spot orphans/zombies
kill -0 <ppid>                             # check if the parent of a zombie is even still alive
```

## Process Termination and Reaping (wait/waitpid)

When a process exits — whether via a normal `return`/`exit()` call or being killed by a signal — the
kernel's `do_exit()` path releases essentially everything: it closes all open file descriptors
(decrementing reference counts, potentially triggering the actual close of underlying resources if no
other process/thread shares them), tears down the virtual memory mappings and decrements references
on the `mm_struct` (freed only once every thread sharing it has also exited), detaches from any
IPC/semaphore resources, and reparents any still-living children to the nearest subreaper. What it
deliberately does *not* do is fully free the `task_struct` itself — that's held in the zombie state
specifically to preserve the exit status (`WIFEXITED`/`WEXITSTATUS`, or `WIFSIGNALED`/`WTERMSIG`
information) until the parent retrieves it. `wait()` and `waitpid()` are the syscalls a parent uses to
retrieve this: `wait()` blocks until *any* child changes state (exits, is stopped, or is continued) and
returns that child's PID and status; `waitpid(pid, &status, options)` allows waiting on a specific
child, and critically supports `WNOHANG` for a non-blocking poll (used heavily in event-loop-based
process supervisors that can't afford to block), plus `WUNTRACED`/`WCONTINUED` to also be notified
about stop/continue transitions rather than only final exits. Once `wait()`/`waitpid()` successfully
retrieves a zombie's status, the kernel finally frees that `task_struct` entirely, removing the last
trace of the process. A well-behaved process supervisor (systemd, a shell, a container runtime acting
as PID 1) must register a `SIGCHLD` handler (or use `waitid()`/`signalfd` in an event loop) and call
`waitpid(..., WNOHANG)` in a loop every time `SIGCHLD` is delivered, since multiple children can exit
in a tight window and signals of the same type are not queued — missing this pattern is the single
most common root cause of zombie accumulation in custom-written supervisors or minimal container
`ENTRYPOINT` scripts that never handle reaping (this is exactly the class of bug that tools like
`tini`/`dumb-init` exist to fix when running a container without a full init system as PID 1).

### Key commands
```
strace -f -e trace=wait4,waitid <cmd>    # observe reaping behavior of a supervisor process live
ps -eo pid,ppid,stat,comm | grep defunct  # "defunct" is what some ps output calls zombies
echo $?                                    # shell builtin: last foreground command's exit status
```

## Process Groups and Sessions

Process groups and sessions are the kernel's mechanism for organizing related processes so that
signals (especially from terminal control keys) and terminal ownership can be managed collectively
rather than per-process. A process group is a set of one or more processes sharing a Process Group ID
(PGID), typically all the processes in one shell pipeline (`cmd1 | cmd2 | cmd3` are all placed in the
same new process group by the shell) — this lets the shell send a single signal to the entire pipeline
at once (e.g., `Ctrl-C` sends `SIGINT` to the whole foreground process group, not just one command in
the pipe). A session is a still-larger grouping: one or more process groups sharing a Session ID (SID),
typically created when a new login/terminal session begins (`setsid()`), and a session is associated
with at most one controlling terminal at a time. Within a session, exactly one process group is
designated the *foreground* process group for the controlling terminal — the kernel tracks this via
`tcsetpgrp()`/`tcgetpgrp()` — and only processes in the foreground group receive terminal-generated
signals like `SIGINT` (Ctrl-C) or `SIGTSTP` (Ctrl-Z); background process groups continue running
unaffected by terminal keystrokes, which is precisely the mechanism behind shell job control (`bg`,
`fg`, `&`). When the controlling terminal itself is closed or disconnects (e.g., an SSH connection
drops), the kernel sends `SIGHUP` to the session's foreground process group, which is the traditional
reason background jobs die when a terminal closes unless explicitly protected with `nohup` (which
simply ignores `SIGHUP`) or `disown`/`setsid` (which detaches a job into a new session with no
controlling terminal to lose in the first place) — this is also why `systemd`-managed daemons and
properly-daemonized background services always call `setsid()` early, so they have no controlling
terminal at all and are immune to this class of signal entirely.

### Key commands
```
ps -eo pid,ppid,pgid,sid,tty,comm    # see PGID/SID/controlling-tty relationships for every process
setsid command &                     # start a command detached in a brand-new session
nohup command &                      # ignore SIGHUP so command survives terminal disconnect
disown %1                            # detach an already-backgrounded job from the shell's job table
```

## Signals and Signal Handling

A signal is an asynchronous, software-generated notification delivered to a process to indicate an
event — anything from a user pressing Ctrl-C (`SIGINT`), a program dividing by zero or dereferencing
a bad pointer (`SIGFPE`, `SIGSEGV`), a timer expiring (`SIGALRM`), a child process changing state
(`SIGCHLD`), or an explicit request for termination (`SIGTERM`, `SIGKILL`). Each signal has a default
disposition (terminate, terminate-and-core-dump, stop, continue, or ignore), which a process can
override for most signals by registering a handler via `sigaction()` (the modern, POSIX-standardized
interface; the older `signal()` has portability quirks around whether the disposition resets after one
delivery and is generally discouraged in new code). Two signals are special-cased by the kernel and
cannot be caught, blocked, or ignored under any circumstances: `SIGKILL` (9) and `SIGSTOP` (19) — this
is a deliberate design guarantee that there always exists a way to unconditionally terminate or pause
any process regardless of how broken or hostile its own signal handling code is. When a signal is
delivered to a process currently executing in userspace, the kernel interrupts it at the next
opportunity (immediately if it's the currently running task, or upon being scheduled if not), saves the
interrupted user-mode register state, and forces execution to jump to the registered signal handler
running on (usually) the same stack, and once the handler returns, a special `sigreturn()` trampoline
restores the original saved register state so the interrupted code resumes exactly where it left off
— this is why signal handlers must be careful to only call "async-signal-safe" functions (a
POSIX-defined subset, notably excluding most of `stdio` and `malloc`), since the handler can interrupt
*any* point in the program's execution, including in the middle of a non-reentrant library call.
Signals delivered to a multi-threaded process are delivered to exactly one thread (chosen by the
kernel, though a thread can explicitly request/block specific signals via `pthread_sigmask`), except
truly process-directed signals that specifically target the whole thread group. In an interview,
be ready to distinguish process-terminating signals by their intent: `SIGTERM` requests graceful
shutdown (catchable, so applications typically flush state and clean up), `SIGKILL` is an
unconditional, uncatchable, immediate kill used only after `SIGTERM` fails to work within a timeout
(exactly how `systemctl stop`/Kubernetes pod termination escalates), and `SIGHUP` historically meant
"controlling terminal disconnected" but is commonly repurposed by daemons to mean "reload
configuration" as a convention, not a kernel-enforced meaning.

### Key commands
```
kill -l                          # list all signal names/numbers
kill -TERM <pid>                  # request graceful termination
kill -KILL <pid>                  # unconditional, uncatchable termination
kill -HUP <pid>                   # commonly used to ask a daemon to reload config
trap 'echo caught' TERM           # (shell) register a handler for a signal in a script
strace -e trace=rt_sigaction,kill <cmd>   # observe a program registering handlers / sending signals
```

## Signal Masking and Pending Signals

Every thread maintains a signal mask — a bitmask of signals currently *blocked* from delivery — set
via `sigprocmask()` (single-threaded) or `pthread_sigmask()` (per-thread in a multi-threaded process),
which is distinct from *ignoring* a signal (`SIG_IGN` disposition permanently discards it) because a
blocked signal is instead held pending: the kernel remembers it occurred (in a per-task pending-signal
bitmask, plus a real-time signal queue for `SIGRTMIN`-and-above signals which, unlike standard
signals, *do* queue multiple pending instances rather than collapsing repeats into one flag) and
delivers it as soon as the thread unblocks it. This distinction matters for a subtle but important
reason: standard signals (1-31) are not queued — if `SIGCHLD` arrives three times while blocked, only
one pending indication survives, which is precisely why event-loop code handling `SIGCHLD` must call
`waitpid(..., WNOHANG)` in a loop until it returns "no more children" rather than assuming one signal
means exactly one child exited. Blocking signals is a standard technique for writing correct
concurrent code — critical sections that must not be interrupted by an async handler mid-update
(e.g., updating a data structure that a signal handler also touches) block the relevant signal for
the duration, then unblock it afterward, at which point the kernel delivers any pending occurrence
immediately. `signalfd()` is a more modern alternative pattern favored in event-driven servers: instead
of installing a traditional async handler (with all the async-signal-safety restrictions), you block
the signals of interest with `sigprocmask()` and instead create a file descriptor via `signalfd()` that
becomes readable whenever one of those signals is pending, letting you handle signals synchronously
through the exact same `epoll()`/`select()` event loop as network I/O, entirely avoiding the
correctness hazards of true asynchronous signal handlers.

### Key commands
```
cat /proc/<pid>/status | grep -E 'SigPnd|SigBlk|SigIgn|SigCgt'   # pending/blocked/ignored/caught signal masks (hex bitmasks)
strace -e trace=rt_sigprocmask <cmd>     # observe a program blocking/unblocking signals live
```

## Context Switching

A context switch is the kernel operation that stops executing one task and resumes another on the
same CPU core, and understanding its real cost is essential for reasoning about scheduler-heavy or
syscall-heavy workloads. When the scheduler decides to switch (`schedule()` in `kernel/sched/core.c`),
it must: save the outgoing task's CPU register state (general-purpose registers, program counter,
stack pointer, and on x86 potentially FPU/SSE/AVX state if used) into that task's `task_struct`/
`thread_struct`; switch the memory management context if the new task belongs to a different address
space (`mm_struct`) — this means loading a new value into the CR3 register (x86) to point at the new
page tables, which invalidates address-space-specific entries in the Translation Lookaside Buffer
(TLB) unless the CPU supports tagged TLBs (PCID on modern x86, ASID on ARM) to avoid a full flush;
restore the incoming task's previously saved register state; and update scheduler bookkeeping (run
queue membership, statistics). The most expensive hidden cost isn't the register save/restore itself
(a handful of instructions) but the *indirect* cost of a cold cache and TLB after switching address
spaces — the incoming task's working set is very likely not resident in L1/L2 cache anymore, so it
pays a burst of cache misses re-warming its data, which is why context-switch-heavy workloads
(excessive threading, thrashing between too many runnable processes, or synchronous request/response
patterns causing constant blocking/waking) show up as high CPU time in "system" categories with lower
effective throughput even though the CPU appears busy. A context switch *between two threads of the
same process* is cheaper precisely because `CLONE_VM`-shared threads share the same `mm_struct` — no
CR3 reload, no address-space TLB invalidation needed, only the register-state and scheduler-metadata
portions of a full switch — which is one more concrete reason threads are cheaper than processes for
tightly-coupled concurrent work. Voluntary switches (a task blocks on I/O or a lock, calling
`schedule()` itself) and involuntary switches (the scheduler preempts a still-runnable task because
its time slice expired or a higher-priority task became runnable) are both counted separately in
`/proc/<pid>/status` (`voluntary_ctxt_switches`/`nonvoluntary_ctxt_switches`), and a high involuntary
count relative to voluntary is a signal of CPU contention (more runnable work than available cores).

### Key commands
```
cat /proc/<pid>/status | grep ctxt_switches   # voluntary vs involuntary switch counts for one process
vmstat 1                                       # 'cs' column: total context switches per second, system-wide
pidstat -w 1                                    # per-process context-switch rate over time
perf stat -e context-switches,cpu-migrations ./program   # low-level counters for a specific workload
```

## Linux CPU Scheduler (CFS)

The Completely Fair Scheduler is the default scheduling algorithm for ordinary (`SCHED_OTHER`)
processes, implemented in `kernel/sched/fair.c`, and its core idea is to model an idealized "perfectly
fair" CPU that could give every runnable task an infinitesimally thin, perfectly equal slice of CPU
time simultaneously, then approximate that ideal as closely as possible with a real, single-task-at-a-
time CPU. It does this via a per-task accounting value called `vruntime` ("virtual runtime") that
tracks how much CPU time a task has *effectively* consumed, weighted by its priority/nice value — a
task with a lower nice value (higher priority) accrues vruntime more slowly for the same real CPU time,
so it earns the right to run more often. All runnable tasks on a given CPU's run queue are kept in a
red-black tree keyed by `vruntime`, and the scheduler's core decision, `pick_next_task()`, is close to
"always run the task with the smallest vruntime" (the leftmost node in the tree, cached for O(1)
access) — a task that has run recently has a larger vruntime and sinks toward the right of the tree,
making room for tasks that have waited longer (smaller vruntime) to get their turn, which is precisely
what produces the "completely fair" emergent behavior without any fixed, rigid time-slice-per-task
schedule. When a task is scheduled, it's not given an unconditionally fixed quantum; instead its
"ideal" slice length is computed from a target scheduling latency period divided proportionally among
currently runnable tasks (more runnable tasks means shorter individual slices, keeping overall
responsiveness bounded), and it's preempted early if a newly-woken task has a substantially smaller
vruntime (meaning it deserves the CPU more, by fairness accounting) even before its computed slice
expires. Since Linux 6.6, CFS has begun to be replaced by EEVDF (Earliest Eligible Virtual Deadline
First), a related but more principled fairness algorithm addressing some of CFS's known
latency-under-load edge cases, but the vruntime/red-black-tree mental model remains the right
foundation for discussing the pre-6.6 scheduler that's still what most production kernels run today,
and EEVDF questions are increasingly common as a "have you kept up" FAANG-level probe.

```
Run queue (per-CPU) modeled as a red-black tree keyed by vruntime:

                (task C, vruntime=120)
               /                      \
   (task A, vruntime=80)        (task E, vruntime=200)
                        \
                (task B, vruntime=95)

pick_next_task() → leftmost node → task A (smallest vruntime = least CPU consumed so far, relatively)
```

### Key commands
```
cat /proc/sys/kernel/sched_latency_ns        # target scheduling latency period (tunable)
cat /proc/<pid>/sched                         # se.vruntime, nr_switches, and other CFS internals for a task
chrt -p <pid>                                  # show current scheduling policy/priority for a process
schedtool -v -n 0 <pid>                        # (older tool) inspect/adjust scheduling parameters
```

## Scheduling Classes (SCHED_OTHER, SCHED_FIFO, SCHED_RR, SCHED_DEADLINE)

Linux organizes scheduling into pluggable "scheduling classes," each implementing a common interface
(`pick_next_task`, `enqueue_task`, etc.) and checked in a strict priority order every time the
scheduler needs to choose the next task to run — this is why a real-time task can always preempt a
normal one regardless of vruntime accounting: the scheduler core simply asks the highest-priority
class first ("is there a runnable deadline task? no? is there a runnable FIFO/RR task? no? fall
through to CFS"), never even considering CFS's red-black tree if a real-time class has something
runnable. `SCHED_OTHER` (also called `SCHED_NORMAL`) is the default class governed by CFS/EEVDF fair
scheduling described above, appropriate for the overwhelming majority of ordinary processes.
`SCHED_FIFO` is a real-time, fixed-priority, run-to-completion class: a `SCHED_FIFO` task, once
scheduled, keeps the CPU indefinitely until it voluntarily yields, blocks, or a higher-or-equal-
priority real-time task becomes runnable — there is no time-slicing at all within the same priority
level, making it dangerous (a buggy infinite loop in a `SCHED_FIFO` task can starve the entire system,
including the kernel's own housekeeping, unless `RT throttling` — a safety-valve sysctl limiting
real-time task CPU share — is enabled). `SCHED_RR` is FIFO's time-sliced sibling: same fixed-priority
preemption model, but tasks at the same priority level are round-robined with a bounded time quantum
rather than one task running forever. `SCHED_DEADLINE` is the newest and most sophisticated real-time
class, based on the Earliest Deadline First (EDF) algorithm combined with Constant Bandwidth Server
(CBS) admission control: a task declares a runtime/period/deadline triple, and the kernel both
schedules strictly by nearest deadline and refuses to admit a new deadline task if doing so would make
the declared guarantees for existing deadline tasks mathematically infeasible, giving genuinely
provable latency guarantees appropriate for audio/video processing or industrial control loops running
on general-purpose Linux. Practically, `SCHED_FIFO`/`SCHED_RR` require `CAP_SYS_NICE` (or root) to set,
because an unprivileged process granted real-time priority is a straightforward denial-of-service
vector against the whole system.

### Key commands
```
chrt -f -p 50 <pid>            # set SCHED_FIFO with priority 50 on an existing process
chrt -r -p 20 <pid>            # set SCHED_RR with priority 20
chrt -d --sched-runtime 1000000 --sched-deadline 10000000 --sched-period 10000000 0 <cmd>   # SCHED_DEADLINE
cat /proc/sys/kernel/sched_rt_runtime_us    # real-time throttling safety valve (vs sched_rt_period_us)
```

## Nice Values and Priorities

The traditional UNIX "nice value" is a per-process hint, ranging from -20 (highest priority, least
"nice" to other processes) to +19 (lowest priority, most "nice," yielding the CPU to others more
readily), applying only within the `SCHED_OTHER`/CFS class — it has no effect on real-time scheduling
classes, which use an entirely separate priority scale (1-99) that always outranks any CFS task
regardless of nice value. Internally, CFS translates the nice value into a scheduling *weight* via a
lookup table (`sched_prio_to_weight[]`) where each step of nice value corresponds to roughly a 10%
change in effective CPU share when tasks are competing — this weight directly scales how quickly a
task's vruntime accumulates relative to real time consumed: a heavily-weighted (low nice value) task's
vruntime grows more slowly for the same wall-clock CPU time, so it remains the "smallest vruntime"
candidate and gets picked more often by the red-black-tree scheduling decision, achieving a
proportionally larger CPU share without any special-casing beyond this weight multiplier. Setting nice
values is done via `nice` (at process launch) or `renice` (for an already-running process); lowering a
process's nice value below its current level (making it higher-priority) requires `CAP_SYS_NICE`
(effectively root) since it could otherwise let unprivileged users unfairly monopolize CPU, while
raising your own nice value (deprioritizing yourself) is always allowed. Distinct from the classic
nice value is `ionice`, which sets I/O scheduling priority/class (best-effort, real-time, or idle) for
the block-layer I/O scheduler independently of CPU nice value — a CPU-nice-19 process can still be
I/O-priority-critical, and vice versa, since these are genuinely separate resource-scheduling
subsystems (CPU scheduler vs block I/O scheduler) with independently tunable priority mechanisms.

### Key commands
```
nice -n 10 command              # launch a command with nice value +10 (lower priority)
renice -n -5 -p <pid>            # change nice value of a running process (needs privilege to go negative)
ps -eo pid,ni,pri,comm           # NI (nice) and PRI (kernel-internal priority) columns
ionice -c2 -n7 -p <pid>          # set best-effort I/O class, lowest I/O priority level
```

## Load Average vs CPU Utilization

Load average — the three numbers reported by `uptime`/`w`/`top` (1, 5, and 15-minute exponentially
damped moving averages) — is one of the most misunderstood Linux metrics precisely because it is
*not* CPU utilization. Linux defines "load" as the number of tasks that are either currently running
on a CPU or in an uninterruptible/runnable state waiting for a resource — critically, this includes
`D`-state tasks blocked on I/O, not just CPU-bound `R`-state tasks competing for cores, a deliberate
design choice inherited from BSD's original load-average definition intended to capture "how much
demand is the system experiencing across any resource," not narrowly CPU alone. This is precisely why
you can see a load average of 40 on an 8-core box that appears almost entirely CPU-idle in `top`: if
40 processes are all blocked in `D` state waiting on a slow or failing storage backend, they all count
toward load even though zero CPU cycles are being consumed servicing them — this is the single most
common "load average lies to you" interview trap, and correctly diagnosing it means immediately
cross-referencing `ps -eo stat` for a pile of `D`-state processes rather than assuming a CPU bottleneck
just because load is high. CPU utilization, in contrast, is a point-in-time (or interval-averaged)
measure of how busy the CPUs actually are, broken down into user time, system (kernel) time, I/O-wait
time (CPU idle specifically *because* it's waiting on outstanding I/O, still counted as "idle" for
scheduling purposes but reported separately as `%wa` because it hints at an I/O bottleneck), and
steal time (relevant on virtualized/cloud hosts — time the hypervisor gave to *other* tenants instead
of your VM, which looks like mysteriously "missing" CPU capacity you're being billed for but not
receiving). A mature interview answer to "the load average is high, is the system in trouble?" is:
"it depends entirely on *why* — check whether it's CPU-bound (utilization near 100%, few D-state
tasks) or I/O-bound (D-state pileup, `iostat` showing high `await`/`%util` on a device, CPU relatively
idle) because the remediation is completely different."

### Key commands
```
uptime                          # the three load-average numbers
mpstat -P ALL 1                  # per-CPU user/system/iowait/steal breakdown over time
vmstat 1                          # 'r' (runnable) and 'b' (blocked/uninterruptible) queue length columns
ps -eo stat= | sort | uniq -c     # quick histogram distinguishing R-heavy vs D-heavy load
iostat -x 1                       # device-level %util/await to confirm an I/O-bound hypothesis
```

## Preemption (voluntary/involuntary)

Preemption is the mechanism by which the kernel takes the CPU away from a currently running task
before it voluntarily gives it up. Voluntary preemption happens when a task itself calls into the
kernel in a way that can block — issuing a blocking syscall (`read()` on an empty pipe, waiting on a
mutex/futex, sleeping) — at which point the task calls `schedule()` on its own behalf, the scheduler
picks a different runnable task, and the original task is moved off the CPU with its own cooperation.
Involuntary preemption is what actually gives Linux (and any general-purpose OS) fairness and
responsiveness guarantees despite badly-behaved or purely CPU-bound programs: the kernel's timer
interrupt fires periodically (the scheduling "tick," historically 100-1000Hz depending on kernel
config, though `NO_HZ`/tickless configurations suppress unnecessary ticks on idle or single-task cores
to save power) and, on each tick, the scheduler checks whether the currently running task has
exhausted its fair-share time slice or whether a higher-priority/smaller-vruntime task has since become
runnable — if so, it sets a "need resched" flag, and at the next safe opportunity (returning from the
interrupt, or the next kernel-preemption-safe point) the currently running task is forcibly switched
out even though it never asked to give up the CPU. Kernel preemption itself is configurable at build
time (`CONFIG_PREEMPT_NONE`/`VOLUNTARY`/`PREEMPT`/`PREEMPT_RT`): fully preemptible kernels
(`CONFIG_PREEMPT` or the `PREEMPT_RT` real-time patch set, now largely merged upstream) allow even code
*executing inside the kernel itself* (not just userspace) to be preempted at nearly any point (except
genuinely non-preemptible critical sections holding a spinlock), which is essential for low-latency
and real-time workloads, at some throughput cost from the added preemption-check overhead, whereas
`CONFIG_PREEMPT_NONE` (common on servers optimizing for raw throughput) only preempts at explicit,
well-defined kernel checkpoints, favoring fewer context switches and better cache locality over worst-
case latency.

### Key commands
```
zcat /proc/config.gz | grep CONFIG_PREEMPT     # which preemption model this kernel was built with
cat /proc/sys/kernel/sched_latency_ns           # target latency guiding involuntary preemption decisions
cat /proc/<pid>/status | grep nonvoluntary_ctxt_switches   # count of involuntary preemptions for a task
```

## SMP and Multi-core Scheduling

Symmetric Multi-Processing (SMP) means every CPU core is treated as an equal, generic scheduling
resource capable of running any runnable task, and Linux's scheduler maintains a separate run queue
per CPU core (rather than one global run queue) specifically to avoid the lock-contention bottleneck
that a single shared queue would create as core counts scale into the dozens or hundreds. Since work
naturally becomes imbalanced over time (some cores idle while others have several runnable tasks
queued), the scheduler runs periodic and event-driven load balancing (`kernel/sched/fair.c`'s
`load_balance()`), which considers moving tasks from a busier CPU's run queue to an idler one — but
this migration isn't free: moving a task to a different core means it loses all its warm cache state
(L1/L2 cache lines, and potentially L3 depending on core topology) and must re-populate that working
set from scratch on the new core, so the load balancer explicitly weighs migration cost against
imbalance severity using CPU topology information (Linux models cores into "scheduling domains" —
SMT/hyperthread siblings, cores sharing an L2/L3 cache, NUMA nodes — and prefers migrating within a
"cheap" domain like SMT siblings sharing cache over an "expensive" cross-NUMA-node migration).
Simultaneous Multi-Threading (SMT, Intel Hyper-Threading) complicates this further: two "logical CPUs"
on one physical core share nearly all execution resources (ALUs, cache), so the scheduler's topology
awareness tries to spread independent tasks across *different physical cores* first before doubling up
two tasks onto SMT siblings of the same core, since two CPU-bound tasks sharing one physical core's
resources will contend and run slower than if each had a whole separate physical core to itself — this
"SMT-aware" placement is also central to security-driven core scheduling features (grouping only
mutually-trusting tasks onto SMT siblings of the same core to mitigate cross-thread side-channel
attacks like L1TF/MDS).

### Key commands
```
lscpu                              # CPU topology: sockets, cores per socket, threads per core, NUMA nodes
cat /proc/schedstat                 # per-CPU scheduler statistics including migration counts
cat /sys/devices/system/cpu/cpu0/topology/thread_siblings_list   # SMT sibling mapping for one core
mpstat -P ALL 1                     # confirm actual load distribution across cores in practice
```

## CPU Affinity and NUMA-aware Scheduling

CPU affinity is an explicit constraint, set via `sched_setaffinity()` (or the `taskset` CLI), that
restricts which subset of CPUs the scheduler is allowed to run a given task on, overriding the
scheduler's normal freedom to migrate it anywhere for load-balancing purposes. Pinning is used for two
main reasons: performance (keeping a latency-sensitive or cache-sensitive task glued to one core avoids
migration-induced cache cold-start costs entirely, valuable for high-frequency trading systems, audio
processing, or busy-polling network I/O threads) and isolation (dedicating specific cores exclusively
to a critical workload, combined with `isolcpus`/`nohz_full` kernel boot parameters that remove those
cores from the general scheduler's load-balancing domain and from periodic timer-tick housekeeping
entirely, minimizing any "noisy neighbor" jitter from unrelated kernel or system activity landing on
those cores). NUMA (Non-Uniform Memory Access) architecture matters enormously for scheduling on
multi-socket servers: each CPU socket has its own directly-attached memory controller and DRAM
("local" memory, low latency), while accessing memory attached to a *different* socket ("remote"
memory) must traverse an inter-socket interconnect (Intel QPI/UPI, AMD Infinity Fabric), adding
meaningfully higher latency and lower bandwidth. The scheduler is NUMA-aware specifically to minimize
this penalty: it models NUMA nodes as another (the outermost, most "expensive to cross") scheduling
domain level and strongly prefers keeping a task on the same NUMA node as the memory it's actually
using, and Linux's "automatic NUMA balancing" (`numa_balancing`) goes further by periodically
unmapping a task's pages, catching the resulting page faults to observe which node is actually
accessing them, and migrating either the task to the memory's node or the memory to the task's node
to converge toward locality over time. For workloads where you know the topology in advance (a
database sized to fit one NUMA node's local memory, for instance), explicit control via `numactl`
(binding both CPU and memory allocation to a specific node) usually outperforms relying on automatic
balancing's converge-over-time heuristics, especially for short-lived or bursty workloads that don't
run long enough for automatic balancing to pay off.

### Key commands
```
taskset -c 2,3 command            # restrict a command to CPUs 2 and 3
taskset -pc 4-7 <pid>               # change CPU affinity of a running process
numactl --hardware                  # show NUMA node topology and memory sizes
numactl --cpunodebind=0 --membind=0 command   # pin both CPU and memory allocation to NUMA node 0
cat /proc/<pid>/numa_maps           # per-VMA NUMA placement for a running process
```

## Real-Time Scheduling

"Real-time" in the scheduling sense does not mean "fast" — it means *predictable and bounded*
worst-case latency, even if average-case throughput is sacrificed to guarantee it. Linux's real-time
scheduling classes (`SCHED_FIFO`, `SCHED_RR`, `SCHED_DEADLINE`, discussed above) always take strict
priority over `SCHED_OTHER`/CFS tasks in the scheduler's class-ordering, giving userspace real-time
processes a guarantee that they will preempt any normal task the instant they become runnable. But
scheduling class alone doesn't guarantee real-time behavior end-to-end unless the rest of the kernel
also behaves predictably: a stock (`CONFIG_PREEMPT_NONE`/`VOLUNTARY`) kernel has long
non-preemptible sections (holding a spinlock, or executing certain interrupt/softirq handling code)
where even a `SCHED_FIFO` task cannot preempt, introducing unpredictable latency spikes — this is
precisely what the `PREEMPT_RT` patch set (now substantially merged upstream as the `PREEMPT_RT`
config option) addresses, converting most spinlocks into preemptible sleeping locks, running most
interrupt handling in preemptible kernel threads instead of true hardware-interrupt context, and
generally minimizing the kernel's own worst-case non-preemptible windows so that real-time userspace
tasks get genuinely bounded scheduling latency, not just scheduling *priority*. Building a real
low-latency system involves several coordinated techniques beyond just picking `SCHED_FIFO`: CPU
isolation (`isolcpus`, `nohz_full`) to remove scheduler-tick and load-balancing interference on
dedicated cores; IRQ affinity tuning (`/proc/irq/<n>/smp_affinity`) to steer hardware interrupt
handling away from those isolated cores; disabling CPU frequency scaling/C-states that introduce
latency spikes when a core wakes from a deep sleep state; locking memory pages (`mlockall()`) to
prevent page faults from a real-time thread accidentally touching a swapped-out or not-yet-faulted-in
page mid-critical-section; and priority inheritance on mutexes (a standard real-time technique where
a low-priority task holding a lock a high-priority task is waiting on is temporarily boosted to the
high-priority task's level, preventing "priority inversion" where an unrelated medium-priority task
preempts the lock holder and indirectly blocks the high-priority waiter indefinitely). This full
picture — priority *and* bounded kernel-internal latency *and* controlled interrupt/frequency
behavior — is what a genuinely deep interview answer distinguishes from the surface-level "just use
SCHED_FIFO" answer.

### Key commands
```
uname -v                              # check whether this is a PREEMPT_RT-patched kernel
chrt -f -p 80 <pid>                    # assign SCHED_FIFO priority 80 to a latency-critical process
cyclictest -p 80 -n -m -l 100000       # standard real-time latency measurement/benchmark tool
cat /proc/irq/<n>/smp_affinity_list    # confirm/adjust which CPUs handle a given hardware interrupt
```

---

### Interview Questions

**Conceptual/Internals (8)**

1. **What is the actual kernel-level relationship between a "process" and a "thread"?**
   Both are represented by the identical `task_struct` structure; there is no separate kernel object
   for "thread" versus "process." A thread is simply a `task_struct` created via `clone()` with flags
   telling the kernel to share the parent's `mm_struct` (address space), file descriptor table, and
   signal handlers rather than duplicating them, while all threads of one process share the same
   `tgid` (thread group ID) that `getpid()` reports, even though each has a distinct kernel-internal
   `pid`.

2. **Explain copy-on-write and why it makes `fork()` cheap.**
   `fork()` duplicates only page tables, marking every page read-only and shared between parent and
   child with an incremented reference count, rather than physically copying memory content. A write
   by either process triggers a page fault that allocates a new private physical page, copies the
   content, and remaps only the faulting process's page table entry — deferring (and often entirely
   avoiding, e.g. before an immediate `execve()`) the cost of copying memory that's never actually
   modified.

3. **What is the difference between an S-state and a D-state sleeping process, and why does the
   distinction matter operationally?**
   `S` (interruptible sleep) can be woken early by signal delivery; `D` (uninterruptible sleep) cannot
   be interrupted by a signal — including `SIGKILL` — while it persists, because the kernel considers
   it unsafe to abandon that wait (typically active block-device or NFS I/O). Long-lived `D`-state
   processes are the classic symptom of failing storage or an unresponsive NFS server, and they cannot
   be killed until the underlying I/O either completes or times out.

4. **How does the CFS scheduler decide which task to run next?**
   Every runnable task's `vruntime` (virtual runtime, weighted by nice value/priority) is tracked in a
   per-CPU red-black tree keyed by that value; the scheduler picks the leftmost node — the task with
   the smallest vruntime, meaning it has received proportionally the least CPU time so far — giving an
   emergent "fair" distribution of CPU time without a fixed round-robin schedule.

5. **Why can load average be high while CPUs show mostly idle in `top`?**
   Linux load average counts not just CPU-runnable tasks but also tasks in uninterruptible (`D`) sleep
   waiting on I/O. A pile of processes blocked on a slow disk or hung NFS mount inflates load average
   substantially while consuming essentially zero CPU cycles, which is why load average must always be
   cross-referenced with process state and I/O metrics, not treated as a pure CPU-pressure indicator.

6. **What's the difference between voluntary and involuntary preemption?**
   Voluntary preemption occurs when a task itself calls into the kernel in a way that blocks (a
   syscall that sleeps, waiting on a lock) and calls `schedule()` cooperatively. Involuntary preemption
   is forced by the kernel's timer tick noticing the running task has exhausted its fair-share slice
   or that a more deserving task has become runnable, switching the task out even though it never
   asked to yield the CPU.

7. **Why can't `SIGKILL` and `SIGSTOP` be caught, blocked, or ignored?**
   This is a deliberate kernel design guarantee ensuring there is always an unconditional way to
   terminate or pause any process regardless of how broken, hostile, or buggy its own signal-handling
   code is — without this guarantee, a process could make itself permanently unkillable by installing
   a handler that ignores every termination request.

8. **What happens to a zombie process's parent dying before the parent calls `wait()`?**
   The zombie is reparented to the nearest ancestor marked as a subreaper (via
   `PR_SET_CHILD_SUBREAPER`), or to PID 1/systemd by default if no closer subreaper exists; the new
   parent is then responsible for eventually calling `wait()`/`waitpid()` to collect the zombie's exit
   status and allow the kernel to free its remaining `task_struct`.

**Scenario/Troubleshooting (6)**

9. **A process is stuck in `D` state and `kill -9` does nothing. How do you actually resolve this?**
   You generally cannot force-kill a genuinely uninterruptible-sleep process; you must find and fix the
   underlying I/O stall — check `iostat -x` for a saturated/failing block device, check whether an NFS
   mount is hung (`nfsstat`, checking server reachability), and in the worst case the process only
   clears once the I/O subsystem itself recovers, times out, or (for NFS) the mount is force-unmounted
   with `umount -f`/hard vs soft mount options reconsidered for the future.

10. **A monitoring script notices hundreds of zombie processes accumulating under one long-running
    parent daemon. How do you diagnose and fix it?**
    Confirm with `ps -eo pid,ppid,stat,comm | grep Z` that the zombies share one PPID; the parent
    daemon has a bug where it forks children but never calls `wait()`/`waitpid()` (or doesn't handle
    `SIGCHLD` correctly, missing exits when several arrive close together since standard signals don't
    queue). Short-term, restarting the parent reparents its zombies to init/a subreaper which reaps
    them; the real fix is adding a proper `SIGCHLD` handler doing `waitpid(..., WNOHANG)` in a loop, or
    switching the daemon to run under a real init/supervisor (`tini`, systemd) that handles reaping.

11. **A container's `ENTRYPOINT` runs a shell script that spawns a background process, and that
    process becomes a zombie/orphan mess as the container churns. Why, and what's the standard fix?**
    A shell script used as PID 1 inside a container doesn't have proper signal-forwarding or child-
    reaping behavior — it isn't a real init system. The standard fix is running a minimal init like
    `tini` or `dumb-init` as PID 1 (or using the container runtime's built-in equivalent, e.g. Docker's
    `--init` flag), which correctly reaps zombies and forwards signals to the actual application
    process.

12. **CPU utilization looks low, but a latency-sensitive application still exhibits periodic
    millisecond-scale stalls. What scheduling-related causes would you investigate?**
    Check for involuntary context switches and CPU migrations (`/proc/<pid>/status`,
    `perf stat -e context-switches,cpu-migrations`) possibly caused by other processes/interrupts
    landing on the same cores; check NUMA locality (`numa_maps`) for cross-node memory access latency;
    check for CPU frequency scaling/C-state transitions adding wake-up latency; and consider whether
    the process needs explicit CPU pinning (`taskset`) or isolated cores (`isolcpus`) to eliminate
    scheduler-induced jitter.

13. **A newly-launched batch job unexpectedly starves an important interactive service on the same
    host of CPU. What do you check and how do you fix it?**
    Check the batch job's scheduling class/nice value (`ps -eo pid,cls,ni,comm`) — if it was
    mistakenly launched with an elevated real-time class (`SCHED_FIFO`/`SCHED_RR`) it will always
    preempt normal `SCHED_OTHER` tasks regardless of nice value. The fix is either correcting the
    scheduling class back to `SCHED_OTHER` with an appropriately high (deprioritizing) nice value via
    `renice`, or explicitly reserving/pinning cores for the interactive service using `taskset`/
    `isolcpus` so it's insulated from batch workload contention entirely.

14. **A multi-threaded application shows far worse throughput on a 2-socket NUMA server than a
    single-socket server with fewer total cores. What's the likely explanation and remediation?**
    Threads and the memory they access are likely spread across both NUMA nodes without locality
    awareness, causing frequent remote-memory accesses across the slower inter-socket interconnect.
    Remediation is binding the process (or per-thread, if the workload partitions data) to a single
    NUMA node with `numactl --cpunodebind --membind` when the working set fits in one node's memory, or
    redesigning the application to be NUMA-aware (partitioning data structures per node) if it must
    span multiple nodes.

**FAANG-level Deep Dive (6)**

15. **Explain exactly how CFS computes a task's vruntime accrual rate from its nice value, and why
    this produces proportional CPU sharing without any explicit "give task X 20% of the CPU" logic.**
    CFS looks up a scheduling weight from a fixed table (`sched_prio_to_weight[]`) based on nice value,
    where each nice-value step corresponds to roughly a 10/11 or 11/10 multiplicative weight change.
    vruntime advances as `actual_runtime * (NICE_0_WEIGHT / task_weight)` — a heavier-weighted (lower
    nice) task's vruntime grows more slowly per unit of real CPU time consumed, so it stays the
    "smallest vruntime" (leftmost in the red-black tree) more often and gets picked to run more
    frequently; the proportional CPU share emerges purely from this weighting interacting with the
    "always run smallest vruntime" selection rule, with no explicit percentage-based logic anywhere.

16. **Why does EEVDF (replacing CFS as of Linux 6.6) address a genuine CFS shortcoming, and what's
    the core algorithmic difference?**
    CFS's "run smallest vruntime" rule can let a task that has been waiting build up a very negative
    relative vruntime advantage and then get an unfairly long run before yielding, or conversely can
    make it hard to reason about worst-case latency guarantees for latency-sensitive tasks under heavy
    load, since CFS's fairness is only asymptotically achieved, not deadline-bounded per task. EEVDF
    (Earliest Eligible Virtual Deadline First) assigns each task an explicit virtual deadline derived
    from its weight/slice request and schedules by nearest deadline among *eligible* tasks (those whose
    fair share entitlement has caught up to real time), giving more direct, tunable control over
    latency versus throughput trade-offs per task than CFS's purely emergent vruntime-ordering
    behavior.

17. **Why does a context switch between two threads of the same process cost meaningfully less than a
    switch between two unrelated processes?**
    Threads of the same process share the same `mm_struct`, meaning the CR3 register (pointing at the
    active page table) does not need to be reloaded and the CPU's tagged-TLB entries (PCID on x86)
    remain valid, avoiding the address-space-transition costs entirely. Only the register set and
    scheduler bookkeeping need saving/restoring, whereas a cross-process switch additionally pays for a
    potential TLB/cache-locality disruption from the address-space change, which is the dominant hidden
    cost in most real-world context-switch overhead measurements.

18. **Describe priority inversion and how Linux's real-time subsystem prevents it.**
    Priority inversion occurs when a low-priority task holds a lock that a high-priority task needs,
    and an unrelated medium-priority task preempts the low-priority lock holder (since it outranks it),
    indirectly blocking the high-priority task far longer than the lock's actual critical section
    should require. Linux addresses this with priority-inheritance mutexes (`pthread_mutex` with the
    `PTHREAD_PRIO_INHERIT` protocol, and the kernel's own `rt_mutex` used internally): while a
    high-priority task waits on a lock, the current holder's effective priority is temporarily boosted
    to match, preventing medium-priority tasks from preempting it during the critical section, then
    reverting the boost once the lock is released.

19. **Why does `PREEMPT_RT` require converting most kernel spinlocks into sleeping locks, and what
    trade-off does that impose?**
    A traditional spinlock busy-waits with preemption disabled, which is fine for genuinely
    microsecond-scale critical sections but becomes a source of unbounded worst-case latency for any
    real-time task trying to preempt in when a lower-priority task (or interrupt context) is holding
    one during a longer operation. `PREEMPT_RT` converts most spinlocks (except a small, carefully
    audited set that must remain true spinlocks, like those protecting the scheduler's own core
    run-queue data) into priority-inheriting sleeping mutexes, letting a waiting high-priority task be
    correctly preempted-in via priority inheritance instead of busy-waiting — at the cost of somewhat
    higher average-case overhead and code complexity versus the simpler traditional spinlock model.

20. **Why does automatic NUMA balancing sometimes perform worse than doing nothing for short-lived,
    bursty workloads, and when should you disable it in favor of explicit `numactl` placement?**
    Automatic NUMA balancing works by periodically unmapping pages, deliberately taking page faults to
    observe access patterns, and migrating pages/tasks toward better locality over multiple sampling
    intervals — a process that converges toward good locality this way needs to run long enough to
    amortize the cost of those induced faults and migrations. A short-lived or highly bursty workload
    may finish before convergence completes, paying the full cost of induced faults and migration
    overhead while gaining little to none of the locality benefit, which is why database and HPC
    workloads with well-understood, stable memory footprints typically disable automatic balancing
    (`numa_balancing=0`) and instead pin CPU and memory explicitly and permanently via `numactl` at
    launch time.

### Hands-On Labs

**Lab 1: Observe fork/exec and COW behavior directly**
- Objective: Confirm copy-on-write behavior empirically rather than just conceptually.
- Setup: Any Linux machine with `strace` and a compiler.
- Tasks: Write a small C program that allocates and touches a large buffer, then `fork()`s; in the
  child, modify a few pages of the buffer and sleep; observe `/proc/<child_pid>/smaps` for private vs
  shared page counts before and after the modification; trace with `strace -f` to see the `clone`
  syscall.
- Expected outcome: You can show shared page counts dropping and private/dirty page counts rising
  specifically for the pages the child modified, not the whole buffer.

**Lab 2: Reproduce and diagnose zombie accumulation**
- Objective: Build a deliberately buggy parent that doesn't reap children, then fix it.
- Setup: Any Linux shell/compiler access.
- Tasks: Write a parent program that forks 20 short-lived children and never calls `wait()`; observe
  zombies accumulating with `ps -eo pid,ppid,stat,comm`; add a proper `SIGCHLD` handler calling
  `waitpid(..., WNOHANG)` in a loop and confirm zombies no longer accumulate.
- Expected outcome: A before/after comparison demonstrating the exact fix for zombie leaks.

**Lab 3: Scheduling class and priority experiment**
- Objective: Observe real-time scheduling classes preempting CFS in practice.
- Setup: A disposable VM (real-time priority changes can affect system responsiveness).
- Tasks: Launch several CPU-bound `SCHED_OTHER` loops; launch one additional CPU-bound loop under
  `chrt -f 50`; observe via `top`/`pidstat` that the `SCHED_FIFO` task dominates CPU time versus the
  normal tasks despite equal nice values.
- Expected outcome: Measured, explained CPU-share difference attributable purely to scheduling class.

**Lab 4: Load average vs CPU utilization divergence**
- Objective: Reproduce the classic "high load, idle CPU" scenario safely.
- Setup: A disposable VM with a slow/throttled block device (e.g., a loopback device with `dm-delay`,
  or simply many concurrent `dd` reads from a slow disk).
- Tasks: Launch many processes performing blocking reads against an artificially slow device; observe
  `uptime` load average climbing while `mpstat`/`top` shows CPUs mostly idle; confirm the processes are
  in `D` state with `ps -eo stat`.
- Expected outcome: A concrete demonstration and written explanation of why load average and CPU
  utilization diverge.

**Lab 5: NUMA-aware placement benchmark**
- Objective: Measure the real performance impact of NUMA locality.
- Setup: A multi-socket NUMA machine or NUMA-emulating VM (`qemu -numa node,...`).
- Tasks: Run a memory-bandwidth benchmark (e.g., `stress-ng --vm`) once with no placement control, once
  pinned to a single NUMA node's CPU and memory via `numactl --cpunodebind --membind`, and once
  deliberately cross-bound (CPU on node 0, memory on node 1); compare throughput/latency.
- Expected outcome: Quantified evidence of local vs remote NUMA access performance difference.

### Production Incidents

**Incident 1: Fleet-wide "load average" false alarm during a storage backend degradation**
- Symptom: Automated alerting pages on-call for dozens of hosts reporting load average above 100,
  suggesting massive CPU exhaustion, but application response times are only mildly degraded.
- Investigation: `mpstat` on affected hosts shows CPUs largely idle; `ps -eo stat` shows hundreds of
  application worker processes stuck in `D` state; `iostat -x` reveals a shared network storage backend
  with `await` times spiking into the seconds.
- Root cause: A backend storage array was degraded (a failed drive triggering RAID rebuild I/O
  contention), causing application I/O to queue and workers to block in uninterruptible sleep, which
  inflated load average without corresponding CPU exhaustion.
- Recovery: Engaged storage team to address the degraded array; in the interim, reduced application
  worker concurrency to lower outstanding I/O queue depth against the struggling backend.
- Prevention: Split alerting into separate CPU-utilization and I/O-wait/D-state-count signals instead
  of alerting on raw load average alone, and add storage-layer health metrics to the same dashboard so
  responders see the real bottleneck immediately instead of chasing a CPU red herring.

**Incident 2: A misconfigured deployment tool escalated to real-time priority and froze a production
node**
- Symptom: A production Kubernetes node becomes completely unresponsive over SSH and to its kubelet
  health checks, requiring a hard reboot; no obvious OOM or disk-full condition in the initial triage.
- Investigation: Post-reboot log analysis (`journalctl` from before the freeze) shows a batch data-
  processing job's container was launched with an unintended `--cap-add=SYS_NICE` and application code
  that called `sched_setscheduler(SCHED_FIFO, 99)` on itself, intended for a different, isolated
  benchmarking environment.
- Root cause: A `SCHED_FIFO` priority-99, run-to-completion task with a tight busy-loop bug ran without
  ever yielding, monopolizing a CPU core indefinitely and — because the kernel's real-time throttling
  safety valve had been disabled system-wide for an unrelated latency-tuning experiment — starved even
  kernel housekeeping threads on that core, hanging the whole node.
- Recovery: Hard reboot was required since the node was unresponsive to any control-plane input;
  post-recovery, the offending container's capability grant was removed.
- Prevention: Restrict `CAP_SYS_NICE`/real-time scheduling grants to explicitly approved, isolated
  workloads only, re-enable real-time throttling (`sched_rt_runtime_us`) fleet-wide as a hard safety
  net, and add a node-level watchdog alert specifically for unexpected `SCHED_FIFO`/`SCHED_RR` task
  creation outside approved namespaces.

**Incident 3: NUMA imbalance silently doubled p99 latency after a database host was resized**
- Symptom: After migrating a database to larger, dual-socket instances (more total cores/memory
  expected to improve performance), p99 query latency instead got measurably worse.
- Investigation: `numastat` showed a heavily skewed remote-memory-access ratio; the database process
  had been started without any NUMA-aware configuration and its memory allocations, plus its worker
  threads, were scattered across both sockets by the default scheduler/allocator behavior, with a large
  fraction of memory accesses crossing the inter-socket interconnect.
- Root cause: The database's connection-handling threads were spawned without CPU affinity and the
  buffer pool was allocated before automatic NUMA balancing had converged, freezing in a
  poor-locality state that balancing alone couldn't fully correct under continuous load.
- Recovery: Restarted the database bound to a single NUMA node via `numactl` (sized to fit the working
  set within that node's local memory), immediately restoring and improving on the original single-
  socket latency baseline.
- Prevention: Standardized the database deployment runbook to always explicitly set NUMA CPU/memory
  binding on multi-socket hosts rather than relying on default placement or automatic balancing
  convergence, and added `numastat` remote-access-ratio monitoring to the standard host dashboard.
