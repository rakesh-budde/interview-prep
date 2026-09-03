# Section 7: systemd & Service Management

This section covers systemd's architecture in depth — units, dependency graphs, socket activation,
journald logging, cgroup integration, timers, and the network-facing daemons systemd ships alongside
init. This is core material for service reliability, boot performance, and daemon-management interview
questions.

## Subtopic Index
- [systemd Architecture](#systemd-architecture)
- [Units (service, socket, target, mount, timer, path)](#units-service-socket-target-mount-timer-path)
- [Unit Dependencies (Wants, Requires, After, Before)](#unit-dependencies-wants-requires-after-before)
- [systemd Targets vs Runlevels](#systemd-targets-vs-runlevels)
- [Socket Activation](#socket-activation)
- [journald and Structured Logging](#journald-and-structured-logging)
- [systemd Cgroup Integration](#systemd-cgroup-integration)
- [systemd Timers vs Cron](#systemd-timers-vs-cron)
- [systemd-resolved, systemd-networkd, systemd-udevd](#systemd-resolved-systemd-networkd-systemd-udevd)
- [Service Restart Policies and Failure Handling](#service-restart-policies-and-failure-handling)
- [Masking, Enabling, Disabling Units](#masking-enabling-disabling-units)

---

## systemd Architecture

systemd is PID 1 on nearly every modern Linux distribution, and its architecture is best understood as
a dependency-graph-driven unit manager rather than a simple sequential script runner. Every manageable
resource — a service, a mount point, a device, a socket, a timer, a slice of cgroup-managed resource
limits — is modeled as a "unit," each unit has a well-defined type-specific configuration file, and
systemd's core job is computing and executing a transaction (an ordered activation/deactivation plan)
across the dependency graph formed by all currently-relevant units whenever the target system state
changes (at boot, on an explicit `systemctl start/stop`, or in reaction to some triggering event like a
device appearing). Beyond pure unit management, systemd bundles a substantial and often-debated set of
additional daemons under the same project umbrella specifically because their functionality was judged
tightly coupled enough to service/session lifecycle to benefit from shared implementation:
`systemd-journald` (structured logging, see below), `systemd-logind` (session/seat management,
tracking which users are logged in on which terminals/displays and handling power-key/lid-close
policy), `systemd-udevd` (device event handling and `/dev` population, inheriting `udev`'s
historically-separate role), `systemd-networkd`/`systemd-resolved` (optional network configuration
and DNS resolution, coexisting with or replacing NetworkManager/other tools depending on distribution
choice), and `systemd-timesyncd` (basic NTP client functionality). This consolidation gives systemd a
uniform mechanism for cross-cutting concerns that were previously implemented inconsistently across
independent projects — every unit, regardless of type, benefits from the same dependency resolution,
the same cgroup-based process tracking, and the same journald logging integration — at the real,
frequently-debated cost of a single project controlling a much larger fraction of a Linux system's
core behavior than any single init system historically did, a genuine and still-active architectural
controversy in the Linux community that a mature interview answer should be able to represent fairly
from both sides rather than treating as settled.

### Key commands
```
systemctl --version                 # confirm systemd version and compiled-in feature list
systemd-analyze                       # boot time breakdown: firmware, loader, kernel, userspace
systemctl list-units --all             # every currently-loaded unit and its state
ps -p 1 -o comm=                        # confirm systemd is genuinely PID 1
```

## Units (service, socket, target, mount, timer, path)

Each systemd unit type models a distinct kind of manageable resource with its own type-specific
configuration section. A `.service` unit describes a managed process/daemon — its `ExecStart=`
command, its `Type=` (governing exactly how systemd determines the service has successfully started,
covered further below), restart policy, and resource/security sandboxing directives. A `.socket` unit
describes a listening socket (TCP/UDP port, UNIX socket, or FIFO) that systemd itself creates and
listens on *independently* of whether the associated service is currently running, enabling socket
activation (below). A `.target` unit is a pure synchronization/grouping point with no executable
content of its own — it exists purely to let other units express "I want to be active by the time
this named milestone is reached" (`multi-user.target`, `network-online.target`) without needing a
literal script to run for the milestone itself. A `.mount` unit describes a filesystem mount point,
letting systemd manage mounting/unmounting with the same dependency-ordering machinery as any other
unit type — genuinely useful for expressing "this service requires this specific mount to be active
first," including automatically generating implicit mount units from `/etc/fstab` entries so
traditional fstab-based configuration continues to integrate with dependency ordering. A `.timer` unit
pairs with a same-named unit (typically a `.service`) to trigger it on a schedule (calendar-based or
relative to boot/a previous run), serving as systemd's native cron replacement. A `.path` unit
triggers a paired unit based on filesystem path changes (a new file appearing, a path being modified),
letting event-driven activation happen on filesystem changes without the triggered service needing to
implement its own file-watching logic itself. Every unit type shares a common structural convention —
a `[Unit]` section for dependency/description metadata, a type-specific section (`[Service]`,
`[Socket]`, `[Mount]`, `[Timer]`, `[Path]`), and an `[Install]` section governing what `systemctl
enable` actually wires up — which is precisely what lets systemd apply the same dependency-resolution
and lifecycle machinery uniformly across such structurally different underlying resource types.

### Key commands
```
systemctl cat <unit>                  # show the fully-resolved unit file content (including drop-ins)
systemctl list-units --type=socket       # list active socket units
systemctl list-units --type=mount          # list active mount units, including fstab-generated ones
systemctl list-timers                        # list timer units and their next/last trigger times
```

## Unit Dependencies (Wants, Requires, After, Before)

systemd's dependency directives are deliberately split into two independent axes that are frequently
confused: *ordering* (which unit's start/stop actions must happen before or after another's) and
*requirement* (whether one unit's activation should pull in, or be blocked by the failure of, another
unit at all) — critically, specifying a requirement relationship (`Wants=`/`Requires=`) does *not* by
itself imply any ordering, and specifying an ordering relationship (`After=`/`Before=`) does *not* by
itself imply any requirement; the two must be combined explicitly (`Wants=foo.service` alongside
`After=foo.service`) to express the intuitive "start foo first, and pull it in as a dependency" — a
unit declaring only `After=foo.service` without a corresponding `Wants=`/`Requires=` will happily start
even if `foo.service` never starts at all, merely ensuring that *if* both end up starting, this one
starts after, which is a genuinely common source of "why didn't my dependency actually get started"
confusion for engineers new to systemd. `Requires=` is a hard requirement — if the required unit fails
to start (or is stopped later), the depending unit is also stopped, treating the dependency as
essential; `Wants=` is a soft requirement — the wanted unit is started alongside the wanting unit as a
best-effort action, but the wanting unit proceeds regardless of whether the wanted unit actually
succeeds, making `Wants=` the generally-recommended default for most real-world dependencies unless a
genuinely hard failure-propagation relationship is actually intended. `Conflicts=` expresses mutual
exclusivity (starting this unit stops any conflicting unit that's currently active), and
`BindsTo=`/`PartOf=` express tighter coupling variants (a unit bound to another is stopped if that
other unit stops for *any* reason, including a crash, not just an explicit administrative stop, unlike
plain `Requires=` which only propagates an explicit stop action). Correctly modeling these
relationships is what allows systemd's parallel startup to actually respect real-world correctness
constraints (a database service genuinely must not start before its data volume is mounted) while
still maximizing concurrency for genuinely independent units that have no real ordering constraint
between them at all.

### Key commands
```
systemctl list-dependencies <unit>        # visualize a unit's full dependency tree
systemctl list-dependencies --reverse <unit>   # show what depends ON this unit
systemd-analyze dot <unit> | dot -Tsvg > deps.svg   # render a unit's dependency graph visually
systemctl show <unit> -p Wants,Requires,After,Before   # raw dependency directive values for a unit
```

## systemd Targets vs Runlevels

(Covered in depth in Section 1's boot-process context; recapped here specifically as a systemd-
architecture concept.) Targets are systemd's generalization of the old SysVinit runlevel concept into
arbitrary, composable synchronization points within the broader unit dependency graph, rather than a
fixed, mutually-exclusive numbered state the whole system is in. Because a target is just another unit
type participating in the same `Wants=`/`After=` dependency machinery as everything else, custom
targets can be defined for arbitrary application-specific synchronization needs (a "database-ready"
target that several unrelated services all declare a dependency on, without needing to hardcode a
direct dependency on the database service itself, decoupling the concept "the data layer is ready"
from exactly which concrete unit currently provides it) — a flexibility with no clean equivalent in
the old fixed-numbered-runlevel model. `systemctl isolate <target>` activates exactly the units that
target (transitively) requires/wants while stopping units not needed for it, providing the same
"switch to a different overall system state" capability `telinit N` provided, but computed dynamically
from the dependency graph rather than looked up from a static, pre-built directory listing for that
specific numbered runlevel.

### Key commands
```
systemctl get-default                  # current default target
systemctl list-units --type=target        # all currently active targets
systemctl isolate multi-user.target         # switch to a target immediately, stopping unneeded units
```

## Socket Activation

Socket activation lets systemd itself own and listen on a service's socket (TCP port, UNIX socket,
FIFO) independently of whether that service's actual process is currently running, deferring the cost
of starting the service until the very first connection actually arrives — and, just as importantly,
allowing the *socket itself* to exist and accept (queue) connections even before the service starts,
so a client connecting during the brief window while the service is still initializing experiences a
short connection delay rather than an outright connection-refused error, since the kernel-level socket
is already listening and queuing regardless of the backing service's readiness. Mechanically, systemd
creates the listening socket described by a `.socket` unit at boot (or whenever that socket unit is
started), and when a connection arrives on a socket that has no currently-running paired service
behind it, systemd starts that service and — depending on the socket unit's configuration — either
passes the already-accepted connection's file descriptor directly to the newly-started service process
(inherited via a well-known, fixed file descriptor number, letting the service skip its own
`socket()`/`bind()`/`listen()` setup entirely and just start reading/writing immediately) or simply
signals the service to start and lets it independently bind its own socket once running, with systemd
then handing off ownership of the pre-existing listening socket to it. Beyond the startup-latency
deferral benefit, socket activation provides genuine resilience properties: if a service crashes,
systemd (still holding the listening socket independently of the crashed service process) can restart
it without ever dropping already-queued or new incoming connections during the restart window, and
multiple services can even be activated from the same shared socket set for advanced load-distribution
patterns. This is architecturally similar in spirit to (and directly inspired by) macOS's launchd and,
historically, inetd's superserver model, but integrated natively into systemd's broader unit/dependency
framework rather than existing as a separate, bolted-on subsystem.

### Key commands
```
systemctl list-sockets                 # all socket units and their current listening state
systemctl status <service>.socket        # confirm a socket unit's activation state independent of its service
ss -tlnp | grep systemd                    # confirm a socket is held open by systemd itself, pre-service-start
journalctl -u <service> -u <service>.socket   # correlated logs across both the socket and service units
```

## journald and Structured Logging

`systemd-journald` is systemd's native logging daemon, collecting log data from multiple sources
simultaneously — the kernel ring buffer (`dmesg`-equivalent messages), standard syslog-protocol
messages (for compatibility with applications/daemons still logging via the traditional syslog
mechanism), and, most distinctively, structured messages submitted directly via `sd_journal_print()`/
`sd_journal_send()` calls or captured automatically from a systemd-managed service's own stdout/stderr
— and storing all of it in a binary, indexed, structured format rather than plain text log files. This
structured format is precisely what enables journald's most valuable query capabilities: every log
entry automatically carries rich metadata (the originating unit, PID, UID, boot ID, SELinux context,
and more) without any application needing to explicitly format or embed that metadata into its own log
message text, and `journalctl` can efficiently filter on any of these fields (`journalctl -u
myservice`, `journalctl _PID=1234`, `journalctl -b -1` for the previous boot specifically) far more
reliably than `grep`-based parsing of loosely-structured plain-text log files ever could, since the
metadata is a first-class, indexed field rather than something that has to be pattern-matched out of
free-form text. Journal storage can be configured as volatile (`/run/log/journal`, RAM-backed,
cleared on reboot — the default on some minimal/embedded configurations) or persistent
(`/var/log/journal`, surviving reboots, with configurable size caps and automatic rotation via
`SystemMaxUse=`/`RuntimeMaxUse=` settings in `journald.conf`), and journald can additionally forward
everything it receives to a traditional syslog daemon (rsyslog/syslog-ng) running alongside it
specifically for organizations still standardized on traditional flat-file/remote-syslog-based log
pipelines, or forward directly to a remote collector via `systemd-journal-remote` for centralized,
structured log aggregation without needing a separate syslog forwarder at all. A frequently-tested
practical detail: journald rate-limits messages per-unit by default (to prevent a single misbehaving,
log-spamming service from consuming disproportionate disk/CPU resources or drowning out other
services' logs), which can surprise engineers debugging a verbose service who see gaps ("N messages
suppressed") in the log stream unless rate-limiting is explicitly adjusted or disabled for that specific
unit's debugging session.

### Key commands
```
journalctl -u myservice -f              # follow logs for a specific unit live
journalctl -b -1                          # logs from the previous boot
journalctl --disk-usage                    # current journal storage consumption
journalctl -u myservice -p err              # filter to error-priority-and-above messages for a unit
journalctl --vacuum-size=500M                # manually shrink journal storage to a target size
```

## systemd Cgroup Integration

Every systemd-managed unit that runs processes (services, scopes, and user sessions via `logind`) is
automatically placed into its own dedicated cgroup, organized hierarchically under systemd's own
top-level cgroup tree (visible under `/sys/fs/cgroup/system.slice/<unit>.service/` on a cgroup-v2
system) — this is precisely what makes `systemctl stop` genuinely reliable in a way a traditional
SysVinit script (which typically just tracked and killed one recorded PID) never could: since every
process a service ever forks, no matter how deeply nested or how many times it's re-forked, remains a
member of that service's cgroup unless it deliberately escapes into a different one, `systemctl stop`
can simply signal every process in the entire cgroup at once, guaranteeing no orphaned descendant
process survives the stop, regardless of the service's own internal process-tracking bugs. This same
cgroup placement is the mechanism behind systemd's native resource-control directives
(`CPUQuota=`, `MemoryMax=`, `TasksMax=`, `IOWeight=` in a unit's `[Service]` section) — these are not
systemd-invented resource controls at all, merely a convenient, declarative unit-file interface for
configuring the exact same kernel cgroup controllers discussed elsewhere in this guide, letting an
administrator express "this service may use at most 50% of one CPU and 512MB of memory" directly in
the unit file rather than needing separate `cgcreate`/`cgset` tooling invoked out-of-band. `slices`
(`.slice` units, like `system.slice`, `user.slice`, or custom-defined ones) provide an additional
grouping layer above individual units specifically for applying resource limits to a whole category of
related units collectively (e.g., capping the combined resource consumption of *all* user sessions
under `user.slice`, regardless of how many individual users are currently logged in), giving
administrators hierarchical resource governance that mirrors cgroups' own hierarchical structure
directly through systemd's unit configuration model.

### Key commands
```
systemd-cgtop                          # live top-like view of resource usage per systemd cgroup
systemctl status <unit>                  # shows the unit's cgroup path and member processes
systemctl set-property <unit> MemoryMax=512M   # apply a resource limit live, without restarting the unit
cat /sys/fs/cgroup/system.slice/<unit>.service/memory.current   # raw current usage for a unit's cgroup
```

## systemd Timers vs Cron

systemd timers (`.timer` units, paired with a same-named `.service` unit they trigger) are the native
systemd replacement for cron-based scheduled jobs, and while functionally overlapping with cron for
the basic "run this on a schedule" use case, they offer several meaningfully different operational
properties. A timer can be calendar-based (`OnCalendar=`, using a flexible syntax supporting things
like `*-*-* 02:00:00` for daily-at-2am, or more complex recurring patterns) exactly like cron's
schedule expressions, but can also be monotonic/relative (`OnBootSec=`, `OnUnitActiveSec=`, triggering
some duration after boot or after the paired unit's last activation finishes, respectively) — a
scheduling model cron has no native equivalent for at all, useful for "run every N hours starting from
whenever the system last booted" style requirements rather than a fixed wall-clock schedule. Because a
triggered timer ultimately just starts an ordinary systemd service unit, it automatically inherits
every other systemd service feature for free: full journald-integrated logging (queryable via
`journalctl -u <job>.service` rather than needing cron's own separate, comparatively primitive mail-
on-output or redirect-to-file conventions), resource limits via the same cgroup-integration mechanisms
just discussed, dependency ordering (a timer's paired service can declare `After=network-
online.target` to ensure it doesn't run before network connectivity is actually available, something
a bare crontab entry has no native way to express at all), and `Persistent=true` semantics
(specifically solving cron's classic "the system was powered off during the scheduled run time, so
the job silently never ran at all" problem — a persistent timer records its last trigger time and, on
next boot, catches up any missed run if the calculated next-scheduled-time has already passed).
`systemd-run --on-calendar=...` even allows ad-hoc, one-off timer creation directly from the command
line without needing to author persistent unit files at all, useful for quick, transient scheduling
needs. The trade-off is that timers require authoring (or at minimum understanding) two separate unit
files (the `.timer` and its paired `.service`) rather than cron's single-line-per-job simplicity,
representing a genuinely real increase in configuration verbosity for the most trivial scheduling
use cases, even as it provides substantially more capability and better observability for anything
beyond the most basic case.

### Key commands
```
systemctl list-timers --all             # all timers, their next/last trigger times, and paired units
systemctl status myjob.timer               # timer unit status
journalctl -u myjob.service                  # logs from a timer-triggered job, exactly like any other service
systemd-run --on-calendar='*-*-* 03:00:00' --unit=adhoc-job /path/to/script   # ad-hoc scheduled job
```

## systemd-resolved, systemd-networkd, systemd-udevd

(`systemd-resolved` is covered in DNS-resolution detail in Section 5; this entry focuses on all three
daemons' shared architectural role.) These three daemons represent systemd's optional extension into
network and device configuration management, each replacing or complementing an area historically
handled by separate, independent tooling. `systemd-networkd` provides declarative network interface
configuration (static/DHCP addressing, VLANs, bridges, bonds) via simple `.network`/`.netdev` config
files, positioned as a lighter-weight alternative to NetworkManager for server/embedded use cases that
don't need NetworkManager's more interactive, desktop-oriented feature set (Wi-Fi roaming UI
integration, captive-portal detection) — most server distributions let administrators choose between
`systemd-networkd`, NetworkManager, or traditional distribution-specific scripts (`ifupdown`,
`network-scripts`), and only one should typically be actively managing a given interface at a time to
avoid conflicting configuration attempts. `systemd-resolved` provides the local caching/forwarding DNS
stub resolver discussed in Section 5, uniquely valuable for its per-interface DNS configuration
support (relevant for hosts with multiple simultaneously-active network connections needing different
DNS behavior per interface, like a VPN). `systemd-udevd` inherits (and is largely a direct continuation
of) the historically-separate `udev` project's role: reacting to kernel `uevent` notifications
(hardware appearing/disappearing) to create/remove `/dev` device nodes with correct permissions and
apply naming/symlink policy (predictable network interface names based on physical bus location rather
than a nondeterministic kernel-assigned enumeration order, `/dev/disk/by-uuid/...` convenience
symlinks) — its integration into the broader systemd project specifically lets device-triggered events
participate in the same unit-dependency and `.path`/`.device` unit machinery as everything else systemd
manages, letting a service declare a dependency on a specific device becoming available
(`After=dev-sda1.device`) using the exact same dependency-graph mechanism used for every other unit
type in this section.

### Key commands
```
networkctl status                    # systemd-networkd's view of interface configuration/state
resolvectl status                      # systemd-resolved's per-interface DNS configuration
udevadm monitor                          # watch live kernel uevents and udev processing in real time
udevadm info /dev/sda                      # inspect udev-assigned properties/symlinks for a device
```

## Service Restart Policies and Failure Handling

systemd's `Restart=` directive governs whether and when a service is automatically restarted after its
main process exits, with several distinct trigger conditions (`no` — never automatically restart;
`on-success` — only if it exited cleanly; `on-failure` — only on a non-zero exit code, signal
termination, or timeout, the most commonly used production setting for services expected to run
indefinitely; `on-abnormal`; `always` — restart unconditionally regardless of exit reason, appropriate
mainly for services with their own internal, carefully-designed exit semantics) combined with
`RestartSec=` (a delay before each restart attempt, avoiding a tight, resource-consuming crash loop)
and `StartLimitIntervalSec=`/`StartLimitBurst=` (a rate-limiting circuit breaker — if a unit is
restarted more than `StartLimitBurst` times within `StartLimitIntervalSec`, systemd stops attempting
further automatic restarts entirely and marks the unit failed, specifically preventing an
unrecoverably-broken service from consuming resources in an infinite rapid restart loop forever, a
scenario a naive "always restart" policy without this circuit breaker could otherwise produce). The
`Type=` directive fundamentally determines how systemd knows a service has successfully started at
all, which directly affects both dependency-ordering correctness and restart/failure detection
accuracy: `Type=simple` (the default) considers the unit started the instant its main process is
exec'd, with no actual readiness verification; `Type=forking` expects the initial process to fork a
background daemon and then exit, with systemd tracking the *forked* child (identified via a PID file
if `PIDFile=` is specified, or by cgroup inspection otherwise) as the real, ongoing service process;
`Type=notify` (the most robust option for services that support it) requires the service to explicitly
call `sd_notify(READY=1)` via a private, unit-specific socket once it has genuinely finished
initializing (opened its listening sockets, loaded its configuration, whatever "ready" actually means
for that application), letting systemd accurately delay dependent units' startup until true readiness
rather than merely "the process was exec'd," which is a meaningfully more correct signal than
`Type=simple` provides for services with non-trivial startup/warm-up time. `OnFailure=` can additionally
trigger an entirely separate unit specifically in response to this unit's failure (commonly used to
trigger an alerting/notification service), providing a native systemd mechanism for failure-driven
automation without needing an external process-monitoring tool layered on top purely to detect and
react to systemd-managed service failures.

### Key commands
```
systemctl show <unit> -p Restart,RestartSec,StartLimitBurst   # current restart policy configuration
systemctl reset-failed <unit>          # clear a unit's "failed" state and start-limit counter after a fix
journalctl -u <unit> -p err              # review failure-related log entries for a repeatedly-failing unit
systemd-notify --ready                    # (from within a service) manually signal readiness under Type=notify
```

## Masking, Enabling, Disabling Units

`systemctl enable`/`disable` and `systemctl mask`/`unmask` are frequently confused but operate at
different levels of the unit activation system. `enable` creates the symlinks described by a unit's
`[Install]` section (typically `WantedBy=multi-user.target`, meaning enabling creates a symlink in
`multi-user.target.wants/` pointing back at the unit file) so the unit is automatically started at the
next boot (or whenever the relevant target is reached) — but `enable` alone does *not* start the unit
immediately in the currently-running system (a common point of confusion; `systemctl enable --now` is
the combined "enable for future boots and also start right now" convenience form). `disable` removes
those same symlinks, meaning the unit will no longer be automatically started at the relevant target,
but a unit that's already running when disabled keeps running until explicitly stopped, and — more
importantly — a disabled-but-not-masked unit can still be started manually or pulled in as a
dependency by some *other* unit's `Wants=`/`Requires=`, since disabling only removes its own
`[Install]`-driven auto-start wiring, not its ability to be activated at all. `mask` is a
fundamentally stronger operation: it replaces the unit file entirely with a symlink to `/dev/null`,
making the unit impossible to start under any circumstances whatsoever — not via `systemctl start`,
not via another unit's dependency pulling it in, nothing — until explicitly `unmask`ed, which is
precisely the tool for genuinely preventing a problematic unit from ever running again (a legacy
service being replaced that some other package's dependency declaration keeps accidentally
re-activating, for instance) rather than merely deprioritizing its automatic startup the way `disable`
does. Understanding this distinction precisely — enable/disable governs automatic activation at
boot/target-reached time; mask/unmask governs whether activation can happen *at all*, by any means —
is exactly the kind of precise, easily-glossed-over detail that separates a surface-level from a
genuinely deep systemd interview answer.

### Key commands
```
systemctl enable --now myservice     # enable at boot AND start immediately
systemctl disable myservice            # remove auto-start wiring, but leave it startable manually
systemctl mask myservice                 # make the unit impossible to start by any means
systemctl unmask myservice                 # reverse a mask, restoring normal startability
```

---

### Interview Questions

**Conceptual/Internals (8)**

1. **Explain the difference between `Wants=`/`Requires=` and `After=`/`Before=` in systemd unit
   files.**
   `Wants=`/`Requires=` express *requirement* (whether one unit should pull in, or have its own
   activation blocked by the failure of, another) with no implied ordering. `After=`/`Before=` express
   pure *ordering* (which unit starts/stops first, if both are going to be active) with no implied
   requirement. Achieving the intuitive "start X first, and also depend on X" behavior requires
   specifying both directives together explicitly.

2. **How does socket activation improve both startup latency and reliability?**
   systemd creates and listens on a service's socket independently of the service process itself, so
   the socket can accept and queue connections even before the backing service has started or after it
   crashes, deferring service startup until first connection and avoiding dropped connections during a
   restart, since the listening socket persists across the service process's lifecycle rather than
   being tied to it.

3. **Why is `Type=notify` considered a more accurate readiness signal than `Type=simple`?**
   `Type=simple` considers a service "started" the instant its process is exec'd, with no actual
   verification that it has finished initializing. `Type=notify` requires the service to explicitly
   call `sd_notify(READY=1)` once genuinely ready (sockets bound, config loaded), letting systemd
   correctly delay dependent units' startup and readiness-dependent health checks until true readiness
   rather than mere process existence.

4. **What makes `systemctl stop` more reliable at fully terminating a service than a traditional
   SysVinit script tracking a single PID?**
   Every process a systemd-managed unit ever forks is placed into that unit's dedicated cgroup;
   `systemctl stop` can signal every process in that cgroup at once regardless of how deeply nested or
   how many times the service re-forked, guaranteeing no orphaned descendant survives — a traditional
   script tracking only one recorded PID has no such guarantee if the service's own process management
   has bugs.

5. **What is the practical difference between `disable` and `mask`?**
   `disable` removes a unit's automatic-activation symlinks (so it won't auto-start at the relevant
   target) but leaves it startable manually or as a pulled-in dependency of another unit. `mask`
   replaces the unit file with a symlink to `/dev/null`, making it impossible to start by any means at
   all until explicitly unmasked.

6. **Why do systemd timers solve a problem cron cannot for scheduled jobs on systems that may be
   powered off at the scheduled time?**
   A `Persistent=true` timer records its last trigger time and, on the next boot, checks whether the
   calculated next-scheduled run time has already passed while the system was off, triggering a
   catch-up run if so. Plain cron has no native equivalent — a job scheduled during a period the
   system was powered off simply never runs at all with standard cron.

7. **What is `StartLimitBurst`/`StartLimitIntervalSec` for, and what failure mode does it prevent?**
   It's a circuit breaker limiting how many times a unit may be automatically restarted within a given
   interval before systemd stops attempting further restarts and marks it failed. This prevents a
   persistently, unrecoverably broken service under an `always`/`on-failure` restart policy from
   consuming resources in an infinite rapid crash-restart loop forever.

8. **Why does systemd bundle udev, resolved, networkd, and journald under one project rather than
   keeping them fully independent, and what's the trade-off?**
   Bundling lets every unit type share the same dependency-graph, cgroup-tracking, and logging
   infrastructure uniformly (a device event can participate in ordinary unit dependencies, every
   service's logs flow through the same structured journald pipeline) rather than each subsystem
   reinventing its own. The trade-off, a genuine and ongoing point of debate, is that a single project
   now controls a much larger fraction of core system behavior, increasing the blast radius of any
   systemd-level bug and reducing the modularity/swappability earlier, more loosely-coupled Linux
   init/logging/device-management tooling provided.

**Scenario/Troubleshooting (6)**

9. **A service that should auto-start at boot doesn't, despite `systemctl enable` having been run
    previously and the unit starting fine when triggered manually.**
    Confirm the unit file's `[Install]` section actually specifies a `WantedBy=`/`RequiredBy=`
    target matching the system's actual default target (`systemctl get-default`), and confirm the
    expected symlink genuinely exists (`ls` the relevant `.wants/` directory) — a unit file edited or
    replaced *after* `enable` was originally run doesn't automatically regenerate stale symlinks; a
    fresh `systemctl daemon-reload` followed by re-running `enable` is required if the `[Install]`
    section itself changed.

10. **A service repeatedly restarts in a tight loop after a bad deployment, and `systemctl status`
    now reports it as failed with no further restart attempts, even after the underlying bug is
    fixed.**
    The unit likely hit its `StartLimitBurst`/`StartLimitIntervalSec` circuit breaker during the crash
    loop and is now in a "failed, no further auto-restart" state independent of whether the underlying
    issue has since been fixed. `systemctl reset-failed <unit>` clears this state (alongside restarting
    the now-fixed unit), which is the standard remediation step after resolving the root cause.

11. **A dependent service occasionally starts before its declared dependency has actually finished
    initializing, despite a correct `After=`/`Wants=` pairing in its unit file.**
    `After=` only guarantees ordering relative to the dependency unit being considered "started" by
    systemd's own definition, which for `Type=simple` (the default) means merely "process was exec'd,"
    not "finished initializing." If the dependency needs genuine readiness-gated ordering, it should be
    converted to `Type=notify` with an explicit `sd_notify(READY=1)` call once truly ready, or paired
    with an explicit health-check/wait mechanism in the dependent unit if the dependency's own code
    cannot be modified to support notify-type readiness signaling.

12. **A scheduled job configured via `.timer`/`.service` units silently didn't run at its expected
    time, and no error is visible in `journalctl -u <job>.service`.**
    Check `systemctl status <job>.timer` and `systemctl list-timers` first — since the service unit
    never actually ran, its own logs will naturally show nothing; the failure is more likely in timer
    activation itself (a syntax error in `OnCalendar=`, the timer unit not being enabled/started, or a
    dependency the timer itself declared not being satisfied) rather than in the paired service.

13. **After migrating log storage configuration, `journalctl -b -3` (three boots ago) no longer
    returns any results, though the service was confirmed running at that time via external
    monitoring.**
    Check whether journal storage is configured as volatile (`/run/log/journal`, cleared on every
    reboot) rather than persistent (`/var/log/journal`) in `journald.conf`'s `Storage=` setting —
    volatile storage by design cannot retain logs across a reboot at all, which would fully explain
    missing historical-boot data despite the service genuinely having run and logged normally at the
    time.

**FAANG-level Deep Dive (6)**

15. **Explain precisely how systemd computes a "transaction" when processing a unit activation
    request, and why this can result in units outside the directly-requested one being started or
    stopped.**
    Starting a unit isn't evaluated in isolation — systemd computes the full transitive closure of
    `Requires=`/`BindsTo=`/`Conflicts=` relationships reachable from the requested unit, building a
    transaction that may include starting required dependencies not yet active and stopping any
    currently-active conflicting units, then verifies the resulting transaction doesn't contain
    unresolvable ordering cycles or contradictions before executing it atomically — which is exactly
    why activating one unit can visibly start or stop several others that weren't directly named in
    the original request.

16. **Why can two units sharing an `After=` ordering relationship but no `Wants=`/`Requires=`
    relationship still both fail to start correctly under certain boot conditions, and what does this
    reveal about a common unit-file authoring mistake?**
    Without a requirement relationship, systemd has no reason to actually pull in and activate the
    "after" unit at all in a given boot's transaction if nothing else independently wants it — the
    ordering constraint only applies *if* both units end up being activated by the overall transaction
    for unrelated reasons, meaning a unit relying purely on `After=` for a dependency it actually needs
    can silently run before (or never coincide at all with) that dependency being active, a subtle
    trap that reveals the author intended a requirement relationship but only encoded an ordering one.

17. **Why does socket-activated file descriptor passing (rather than the service independently
    binding its own socket after being told to start) provide a meaningfully stronger zero-downtime
    restart guarantee?**
    When systemd itself owns the listening socket and passes the already-bound (and, for the first
    connection, already-accepted) file descriptor directly to the newly-started service process, the
    socket's listen backlog and any already-queued connections persist across the service process's
    entire restart transition, since the kernel-level socket object was never closed at any point — a
    service that instead independently re-binds its own socket after being merely signaled to start
    necessarily has a brief window where no listener exists at all between the old process exiting and
    the new process completing its own bind/listen setup, a real (if often small) availability gap the
    fd-passing model avoids entirely.

18. **Explain why a unit's cgroup-based process tracking can still fail to catch every descendant
    process in certain edge cases, despite systemd's design intent.**
    A process can escape its unit's cgroup if it has sufficient privilege to directly write to
    `cgroup.procs` in a different cgroup path (moving itself out), or in namespaced/nested-cgroup
    scenarios where a process creates and moves into its own child cgroup that systemd's stop logic
    doesn't anticipate needing separate handling for — while systemd's default behavior correctly
    signals every process remaining in the tracked cgroup tree, a sufficiently privileged and
    deliberately evasive process can still relocate itself outside that tree entirely, which is why
    cgroup-based tracking, while far more reliable than legacy single-PID tracking, is not an absolute,
    unconditional guarantee against every possible process-escape technique.

19. **Why might enabling both `systemd-networkd` and NetworkManager simultaneously on the same
    interface produce unpredictable network behavior, and how would you diagnose which daemon is
    actually managing a given interface?**
    Both daemons independently attempt to apply their own configuration (addressing, routing) to
    interfaces they believe they're responsible for, and without careful mutual exclusion
    configuration (each explicitly told which interfaces to manage, or one entirely disabled),
    conflicting configuration attempts on the same interface can produce flapping addresses, routes
    being added and removed by different daemons in sequence, or simply unpredictable final state
    depending on daemon startup/reconciliation timing. Diagnosis involves checking
    `networkctl status <iface>` and NetworkManager's own `nmcli device status` together, and confirming
    via each daemon's own configuration (`.network` files' `Match=` sections, NetworkManager's
    `unmanaged-devices` setting) exactly which interfaces each has actually claimed responsibility for.

20. **Why does journald's rate-limiting behavior potentially mask a genuine incident's diagnostic
    signal, and how should rate-limiting be handled for services under active incident
    investigation?**
    Default per-unit rate-limiting silently drops (while noting a suppressed-count summary message)
    log entries beyond a configured burst threshold within a time window, specifically to protect
    overall system logging capacity from a single misbehaving service — but during an active incident
    where a service is legitimately producing an unusually high volume of genuinely diagnostic error
    messages, this same protection can silently discard exactly the detailed information an
    investigator most needs. The correct practice is temporarily raising or disabling rate limits
    (`journald.conf`'s `RateLimitIntervalSec=`/`RateLimitBurst=`, or per-unit overrides) for a service
    under active investigation, and being aware to check for "N messages suppressed" notices in the
    journal output itself as a signal that the visible log may not be complete.

### Hands-On Labs

**Lab 1: Author a complete custom service with proper dependency ordering**
- Objective: Practice writing a production-quality unit file from scratch.
- Setup: A Linux VM with systemd.
- Tasks: Write a `.service` unit for a simple long-running script, with `Type=notify` (having the
  script call `systemd-notify --ready` once "initialized"), `Restart=on-failure`, `RestartSec=5`, and
  correct `After=`/`Wants=` on `network-online.target`; enable and verify with `systemctl status`.
- Expected outcome: A correctly-behaving custom service demonstrating notify-type readiness and
  restart-on-failure behavior.

**Lab 2: Socket activation from scratch**
- Objective: Build and verify a socket-activated service.
- Setup: A simple TCP echo script and a systemd-capable VM.
- Tasks: Write paired `.socket` and `.service` units; start only the socket unit; confirm via `ss -tln`
  that the port is listening with no service process running yet; connect a client and confirm the
  service starts on-demand and handles the connection.
- Expected outcome: A demonstrated, working on-demand socket-activated service.

**Lab 3: systemd timer with persistent catch-up**
- Objective: Verify `Persistent=true` catch-up behavior for a missed scheduled run.
- Setup: A VM you can safely power off/on.
- Tasks: Configure a daily timer with `Persistent=true`; power off the VM during its scheduled window;
  power back on later and confirm (via `journalctl -u <job>.service`) that a catch-up run occurred.
- Expected outcome: A verified demonstration of systemd timers' advantage over plain cron for
  missed-schedule recovery.

**Lab 4: cgroup resource limiting via systemd unit properties**
- Objective: Apply and verify live resource limits on a running service.
- Setup: A CPU/memory-intensive test service.
- Tasks: Start a service without limits and observe its resource consumption; apply
  `systemctl set-property <unit> CPUQuota=20% MemoryMax=256M` live; confirm enforcement by observing
  throttling/OOM behavior under load, and verify the underlying cgroup files directly.
- Expected outcome: A demonstrated, verified live resource limit applied without restarting the
  service.

**Lab 5: Boot time analysis and targeted optimization**
- Objective: Use systemd's own tooling to measurably improve boot time.
- Setup: Any systemd VM you can reboot freely.
- Tasks: Run `systemd-analyze blame`/`critical-chain`; identify the slowest non-essential unit; add an
  appropriate `After=`/reorder it out of the critical path, or disable it if genuinely unnecessary at
  boot; reboot and compare timings.
- Expected outcome: A measured, explained boot-time improvement backed by before/after
  `systemd-analyze` output.

### Production Incidents

**Incident 1: A "fixed" service never actually restarted after hitting the start-limit circuit
breaker**
- Symptom: After deploying a fix for a service that had been crash-looping, the service still shows as
  down, with no further restart attempts visible in recent logs.
- Investigation: `systemctl status` shows the unit in a "failed" state with restart attempts
  exhausted; `journalctl` confirms the crash loop from before the fix triggered
  `StartLimitBurst`/`StartLimitIntervalSec`, after which systemd stopped attempting automatic restarts
  entirely, independent of the underlying code fix already being deployed.
- Root cause: The deployment/remediation runbook didn't include `systemctl reset-failed` as a required
  step after resolving a crash-loop root cause, an easy step to overlook since the service "should"
  just start working again once the bug is fixed.
- Recovery: Ran `systemctl reset-failed <unit>` followed by `systemctl start <unit>`, restoring normal
  operation immediately.
- Prevention: Updated the incident/deployment runbook to explicitly include `reset-failed` as a
  standard step whenever remediating a previously crash-looping systemd service.

**Incident 2: Silent log loss during an active incident due to journald rate limiting**
- Symptom: During an ongoing production incident, engineers investigating via `journalctl -u
  affected-service -f` notice gaps in the timeline that don't match the observed application
  behavior, with occasional "N messages suppressed" notices easy to miss in a fast-scrolling log
  stream.
- Investigation: Confirmed the affected service's error-logging volume during the incident far
  exceeded journald's default per-unit rate limit, silently dropping a meaningful fraction of the very
  diagnostic detail needed to root-cause the issue.
- Root cause: Default journald rate-limiting settings, appropriate for normal operating conditions,
  were never adjusted for the unusually high legitimate log volume generated by a service actively
  failing during an incident.
- Recovery: Temporarily raised the affected unit's rate limit, immediately restoring full-fidelity
  logging for the remainder of the investigation.
- Prevention: Added an incident-response runbook step to proactively raise or disable journald rate
  limiting for any service under active investigation, and added monitoring for "suppressed" message
  notices as their own alertable signal.

**Incident 3: Conflicting network configuration after both systemd-networkd and NetworkManager were
left enabled post-migration**
- Symptom: A newly-provisioned host intermittently loses and regains its primary network address,
  with connectivity flapping every few minutes without any physical link issue.
- Investigation: `networkctl status` and `nmcli device status` both showed active management claims on
  the same physical interface; logs showed both daemons periodically reapplying their own
  (subtly different) DHCP-derived configuration to the same interface, each overwriting the other's
  settings in turn.
- Root cause: A provisioning template migration accidentally left both `systemd-networkd` and
  NetworkManager enabled and unconfigured to explicitly exclude each other's managed interfaces,
  something the previous template had correctly handled but the migrated version omitted.
- Recovery: Disabled `systemd-networkd` (the non-standard choice for this fleet, which had
  standardized on NetworkManager) and confirmed stable, non-flapping connectivity afterward.
- Prevention: Added an explicit provisioning-template validation check confirming exactly one network
  management daemon is active and correctly scoped to the expected interfaces before a host is marked
  ready for service.
