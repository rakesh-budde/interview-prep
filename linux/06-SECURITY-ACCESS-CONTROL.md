# Section 6: Linux Security & Access Control

This section covers Linux's layered security model — traditional DAC permissions, Mandatory Access
Control (SELinux/AppArmor), capabilities, seccomp, PAM, sudo internals, kernel hardening, and audit —
the material behind hardening, compliance, and "why was this access denied" interview questions.

## Subtopic Index
- [Users, Groups, UID/GID, /etc/passwd, /etc/shadow](#users-groups-uidgid-etcpasswd-etcshadow)
- [Discretionary Access Control (DAC)](#discretionary-access-control-dac)
- [Mandatory Access Control (MAC): SELinux, AppArmor](#mandatory-access-control-mac-selinux-apparmor)
- [Linux Capabilities](#linux-capabilities)
- [seccomp and seccomp-bpf](#seccomp-and-seccomp-bpf)
- [PAM (Pluggable Authentication Modules)](#pam-pluggable-authentication-modules)
- [sudo Internals](#sudo-internals)
- [chroot and pivot_root](#chroot-and-pivot_root)
- [Namespaces as Isolation Primitive (recap in security context)](#namespaces-as-isolation-primitive-recap-in-security-context)
- [cgroups for Resource Isolation](#cgroups-for-resource-isolation)
- [Kernel Hardening (KASLR, SMEP/SMAP, stack canaries)](#kernel-hardening-kaslr-smepsmap-stack-canaries)
- [Audit Framework (auditd)](#audit-framework-auditd)
- [SSH Security and Key-based Authentication](#ssh-security-and-key-based-authentication)
- [Firewalls (iptables/nftables/firewalld)](#firewalls-iptablesnftablesfirewalld)
- [File Integrity Monitoring](#file-integrity-monitoring)
- [Rootkits and Detection](#rootkits-and-detection)

---

## Users, Groups, UID/GID, /etc/passwd, /etc/shadow

Every Linux process runs with a security identity centered on a numeric UID (user ID) and one or more
GIDs (group IDs) — the kernel itself only ever checks numeric IDs for permission decisions; usernames
are purely a userspace convenience resolved via NSS (`/etc/passwd` locally, or LDAP/SSSD in enterprise
environments) for human readability. `/etc/passwd` maps each username to its UID, primary GID, home
directory, and login shell, and is world-readable by design since usernames and UIDs are not
themselves secret and many tools need to resolve them. Password hashes are deliberately kept out of
`/etc/passwd` (which historically stored them directly, a serious weakness once the file needed to
remain world-readable for other purposes) and instead live in `/etc/shadow`, readable only by root
(or via the setuid `passwd`/`login`/`sshd` binaries), storing a salted, iterated hash (commonly
SHA-512-crypt or, increasingly, yescrypt) alongside password aging metadata (last change date,
minimum/maximum age, warning period, inactivity/expiration). UID 0 is always root, universally
granted the ability to bypass essentially all DAC permission checks (though not necessarily MAC
checks — see SELinux/AppArmor below, which can meaningfully constrain even root); UIDs below a
distribution-specific threshold (typically 1000) are conventionally reserved for system/service
accounts, most of which are deliberately configured with no valid login shell (`/sbin/nologin` or
`/bin/false`) specifically so they can own and run their own processes/files without being usable for
interactive login at all. Group membership determines the secondary DAC permission check (the "group"
bits of the traditional rwx model) and is recorded in `/etc/group`, with a process's full set of
group memberships established at login/session start and generally requiring a new login session (not
just `usermod`) to take effect for already-running processes, since group membership is captured into
a process's credentials at authentication time, not dynamically re-queried on every access check.

### Key commands
```
id                                # current process's UID, GID, and full supplementary group list
getent passwd <user>                # resolve a user via NSS (works with LDAP/SSSD too, not just /etc/passwd)
chage -l <user>                      # password aging policy/status for a user
awk -F: '$3<1000{print $1,$3}' /etc/passwd   # list system/service accounts by UID convention
```

## Discretionary Access Control (DAC)

DAC is the traditional UNIX permission model where the *owner* of a resource decides who else may
access it — a file's owner can grant or restrict read/write/execute access to themselves, their
group, and everyone else via the classic rwx bits (extended by ACLs for more granular control, as
covered in Section 4), and critically, this discretion is entirely the owning user's choice: nothing
in the DAC model itself prevents an owner from making their own file world-writable if they choose to,
regardless of whether that's a wise security decision. This "owner discretion" property is precisely
DAC's fundamental limitation from a security-hardening perspective: a compromised process running as
a legitimate, unprivileged user can still access or modify anything that user is permitted to touch,
and a poorly-configured or careless application can accidentally over-grant access to its own files
with no system-wide policy stopping it — the kernel enforces whatever permission bits exist, but has
no independent opinion about whether those bits represent a *sound* security policy. This is exactly
the gap Mandatory Access Control exists to close: MAC adds a second, independent permission check
layered on top of (never replacing) DAC, enforced by system policy rather than resource-owner
discretion, so that even a process running as root, or a file whose owner has (perhaps mistakenly)
granted overly broad DAC permissions, remains additionally constrained by rules a system administrator
defines centrally and that ordinary users/processes cannot override no matter what they do with their
own DAC permission bits. Understanding this DAC-then-MAC layering (both checks must pass; either one
failing denies access) is essential for correctly diagnosing "permission denied" errors on a
MAC-enabled system — checking `ls -l` permission bits alone is insufficient, since a DAC-permitted
access can still be denied by SELinux/AppArmor policy, a distinction that trips up many
engineers unfamiliar with MAC-enabled systems the first time they encounter it.

### Key commands
```
ls -l <file>                     # traditional DAC permission bits
namei -l /path/to/file             # walk and show DAC permissions at every component of a path
stat <file>                         # full DAC metadata: owner, group, mode, in one view
```

## Mandatory Access Control (MAC): SELinux, AppArmor

Mandatory Access Control enforces a system-wide security policy that ordinary users and even root
cannot override through DAC permission changes alone, implemented on Linux primarily through two
distinct, non-interoperable frameworks. SELinux (Security-Enhanced Linux, developed originally by the
NSA, default on RHEL/Fedora/CentOS) implements Type Enforcement: every process runs within a security
"domain" and every file/resource carries a security "type" (both stored as a `security.selinux`
extended attribute, as noted in Section 4), and policy rules explicitly and exhaustively state which
domains may perform which operations (read, write, execute, connect, and more) on which types — by
default, anything not explicitly permitted by policy is denied (a genuine default-deny, allowlist-only
model), which is both SELinux's greatest strength (a compromised process, even running as root within
its confined domain, cannot access resources its domain's policy never granted regardless of DAC
permissions) and its steepest operational learning curve (writing/troubleshooting policy requires
understanding domains, types, and the (often large, generated) policy rule set, rather than adjusting
a comparatively simple set of path-based rules). AppArmor (default on Ubuntu/SUSE) takes a
simpler, path-based approach: profiles are attached per-binary (not stored as file metadata at all,
avoiding the xattr-preservation pitfalls SELinux labeling can run into during backup/restore) and
explicitly list which file paths, capabilities, and network operations that specific program may use,
which is generally easier to read, write, and reason about for a specific application's confinement
but offers somewhat coarser-grained, path-based (rather than SELinux's more abstract type-based)
policy expression. Both frameworks support a permissive/complain mode (log what *would* be denied
without actually enforcing it, essential for developing and testing new policy without breaking
production) alongside full enforcing mode, and both are frequently the actual root cause behind
confusing "permission denied" errors on services that appear to have entirely correct DAC ownership
and mode bits — checking `audit.log`/`dmesg` for AVC denial (SELinux) or DENIED (AppArmor) messages is
an essential, often-skipped first troubleshooting step whenever DAC permissions look correct but
access still fails on a MAC-enabled system.

### Key commands
```
getenforce                          # current SELinux mode (Enforcing/Permissive/Disabled)
ausearch -m avc -ts recent            # find recent SELinux denials from the audit log
sealert -a /var/log/audit/audit.log    # (setroubleshoot) human-readable explanation of SELinux denials
aa-status                              # current AppArmor profile status (enforce/complain per profile)
aa-complain /path/to/profile             # switch an AppArmor profile to complain (log-only) mode
```

## Linux Capabilities

Traditional UNIX privilege was binary: a process either ran as root (UID 0, able to bypass essentially
every DAC and many other kernel permission checks) or as an unprivileged user (subject to full
permission checking with no special abilities at all) — an all-or-nothing model that forced any
program needing even one narrow privileged operation (binding to a port below 1024, changing file
ownership, adjusting the system clock) to run fully as root, vastly over-granting privilege relative
to its actual needs. Linux capabilities decompose root's traditionally monolithic privilege into
roughly 40 distinct, independently grantable units — `CAP_NET_BIND_SERVICE` (bind to privileged
ports), `CAP_SYS_TIME` (change the system clock), `CAP_CHOWN` (change file ownership regardless of
DAC), `CAP_SYS_ADMIN` (a notoriously broad, catch-all capability covering many miscellaneous
privileged operations that were never cleanly separated, effectively still very close to full root
for many practical purposes and a common target of container-escape research specifically because of
its breadth), `CAP_NET_ADMIN` (network configuration changes), and many more — letting a specific
binary or process be granted exactly the narrow slice of privilege it actually needs via file
capabilities (`setcap`, stored as an extended attribute on the executable, analogous in spirit to the
older, cruder setuid-root pattern but far more precisely scoped) rather than requiring full root.
Capabilities are tracked per-process across several distinct sets — the permitted set (capabilities
the process is allowed to use, a ceiling), effective set (currently active/in-use, checked at the
actual point of a privileged operation), inheritable set (which capabilities survive across
`execve()` to a child process), and (since capability-aware execution matured) an ambient set enabling
capabilities to be inherited by unprivileged child processes in specific controlled scenarios — and
container runtimes make heavy, explicit use of capability dropping (`--cap-drop=ALL --cap-add=...`) as
a core hardening technique, since a container process running with the full default Docker capability
set (still a meaningfully reduced set relative to true root, but broader than most application
containers actually need) represents a larger attack surface than one explicitly reduced to only the
handful of capabilities its specific workload genuinely requires.

### Key commands
```
getcap /path/to/binary               # show capabilities granted to a specific executable
setcap cap_net_bind_service=+ep /path/to/binary   # grant a specific capability to a binary
capsh --print                          # show the current shell/process's full capability sets
cat /proc/<pid>/status | grep Cap        # raw hex-encoded capability sets for a running process
```

## seccomp and seccomp-bpf

seccomp (secure computing mode) restricts which syscalls a process is permitted to invoke at all,
providing defense-in-depth specifically against the scenario where an attacker has already achieved
arbitrary code execution within a process but the actual damage they can do is bounded by which
syscalls remain available to them — even a fully compromised, code-executing process cannot escalate
further through a syscall its seccomp filter simply refuses to allow, regardless of what permission
checks that syscall would otherwise pass. The original "strict mode" seccomp allowed only `read`,
`write`, `_exit`, and `sigreturn` — extremely safe but far too restrictive for almost any real
application. seccomp-bpf, the mode used in practice today (by Docker, systemd's `SystemCallFilter=`,
Chrome's sandboxing, and most container runtimes), instead attaches a small, kernel-verified BPF
program (structurally similar in spirit to classic packet-filter BPF, predating and distinct from
modern eBPF's more general kernel-hook framework, though sharing the same underlying instruction
verification philosophy) that inspects each attempted syscall's number and arguments and returns a
decision — allow, deny with an error, kill the process, or trap into a monitoring process for more
complex policy decisions — letting a much more nuanced allowlist (or denylist) of specific syscalls
(and even specific argument value patterns) be enforced with minimal per-syscall overhead since the
filter executes directly in the kernel at syscall entry rather than requiring a context switch to a
separate monitoring process for every single check. Container runtimes ship a default seccomp profile
(Docker's default profile blocks several dozen rarely-needed, higher-risk syscalls like
`clone` with dangerous namespace flags, kernel module loading syscalls, and various obscure/legacy
syscalls) specifically to reduce the effective kernel attack surface exposed to a containerized
process without needing to know in advance which specific vulnerability an attacker might otherwise
exploit through an unnecessary syscall — this is a genuinely different, complementary security layer
from capabilities (which govern *what a syscall is allowed to do* once invoked) and from MAC (which
governs *what resources* a process may access), addressing instead *which syscalls exist at all* as
an attack surface.

### Key commands
```
docker run --security-opt seccomp=/path/to/profile.json ...   # apply a custom seccomp profile to a container
systemctl show <unit> -p SystemCallFilter                       # check a systemd unit's seccomp syscall filter
strace -f -e trace=seccomp ./program                              # observe seccomp filter installation
cat /proc/<pid>/status | grep Seccomp                               # confirm seccomp mode active for a process (0=off,1=strict,2=filter)
```

## PAM (Pluggable Authentication Modules)

PAM decouples authentication *policy* (how a user proves who they are, and under what additional
conditions — time-of-day restrictions, account lockout after failed attempts, password complexity
requirements) from the individual applications (login, `sshd`, `sudo`, `su`, display managers) that
need to authenticate users, letting administrators change or layer authentication mechanisms system-
wide by editing PAM configuration rather than modifying every application's own code. Each PAM-aware
application consults its own configuration file under `/etc/pam.d/` (or falls back to `/etc/pam.d/
other` if none exists), which lists a stack of PAM modules grouped into four management groups —
`auth` (verify identity: password checking, and increasingly things like 2FA/hardware-key modules
stacked alongside or instead of password modules), `account` (non-authentication account validity
checks: is the account expired, locked, or restricted from logging in at this time), `password`
(handling actual password changes and enforcing complexity/history policy), and `session` (setup/
teardown work to perform around a session's lifetime — mounting a home directory, setting resource
limits, writing to `lastlog`, running `pam_systemd` to register the session with `logind`) — each
entry tagged with a control value (`required`, `requisite`, `sufficient`, `optional`) governing exactly
how that module's success or failure affects the overall stack's final decision, allowing genuinely
sophisticated authentication policies (e.g., "succeed if either a valid password OR a valid
hardware-key challenge is provided, but always still check the account isn't locked regardless of
which auth method succeeded") to be expressed declaratively. Because PAM sits underneath so many
distinct login/privilege-elevation paths simultaneously, it's also the standard mechanism for
system-wide policies like enforcing password complexity (`pam_pwquality`), locking an account after N
consecutive failed attempts (`pam_faillock`/`pam_tally2`), integrating centralized authentication
(`pam_sss` for SSSD-backed LDAP/AD integration, `pam_ldap`), and enforcing time-based or resource-based
restrictions (`pam_time`, `pam_limits` setting per-user `ulimit` values at login) — a misconfigured
PAM stack (a typo in a module path, an overly strict `requisite` entry failing unexpectedly) is a
uniquely dangerous class of misconfiguration since it can lock out *every* authentication path on a
system simultaneously, including the ability to `su`/`sudo` to fix it, which is exactly why PAM
configuration changes are conventionally tested in a still-open secondary session before the original
session is ever closed.

### Key commands
```
cat /etc/pam.d/sshd                # PAM stack configuration for a specific service
cat /etc/pam.d/common-auth            # (Debian-family) shared auth stack included by multiple services
pamtester login someuser authenticate   # test a PAM stack's authentication path directly, without a real login
faillock --user someuser                 # (pam_faillock) check/reset a user's failed-login lockout status
```

## sudo Internals

`sudo` lets an authorized user execute a command as another user (typically root) without needing that
target user's own password, governed by rules in `/etc/sudoers` (and `/etc/sudoers.d/` drop-in files,
the now-preferred way to add rules without directly editing the main file, always validated with
`visudo` which parses and syntax-checks before saving, specifically to prevent a broken sudoers file
from locking out all privilege escalation). Internally, `sudo` is itself a setuid-root binary — when
invoked, it runs with effective UID 0 regardless of the invoking user's real UID, giving it the actual
kernel-level privilege needed to eventually `execve()` the target command as the requested user, but
before doing so it authenticates the invoking user (by default, requiring their *own* password, not
the target user's, then caching that successful authentication for a configurable timeout — commonly
5 or 15 minutes — via a per-user, per-terminal timestamp file under `/var/run/sudo/`, which is exactly
why repeated `sudo` invocations within that window don't re-prompt for a password) and consults the
parsed sudoers policy to determine whether the requested command, as the requested target user, on
this specific host, is actually permitted for this invoking user or one of their groups. Sudoers rules
can be scoped extremely granularly — specific command paths with specific arguments, specific target
users/groups, specific hosts (relevant for a shared sudoers file distributed to many machines via
configuration management), and modifiers like `NOPASSWD` (skip the password re-prompt entirely for
matching rules, common for narrowly-scoped automation-friendly rules but a meaningfully increased risk
if applied broadly) — and every successful or failed `sudo` invocation is logged (to syslog/journald,
and optionally to a dedicated `sudo` I/O log capturing the full session transcript via `Defaults
log_input,log_output`), which is precisely the audit trail that makes `sudo`-based privilege
escalation preferable to widely sharing the root password directly: every elevation is individually
attributable to the specific user who invoked it, not merely "someone who knew the root password."

### Key commands
```
sudo -l                            # list the commands the current user is permitted to run via sudo
visudo                              # safely edit /etc/sudoers with syntax validation before saving
sudo -k                              # invalidate the cached authentication timestamp immediately
journalctl -u sudo / grep sudo /var/log/auth.log   # audit trail of sudo invocations
```

## chroot and pivot_root

`chroot()` changes a process's apparent filesystem root — after calling it, the process (and its
children) can no longer reference any path outside the new root via absolute paths, since the kernel
resolves `/` itself to the new location for that process going forward. This was the original, most
primitive form of filesystem-level process isolation, historically used to sandbox network-facing
daemons (a classic pattern being an FTP or DNS server chrooted into a minimal directory containing
only what it needs) and still used today as one ingredient (among namespaces and cgroups) in
constructing container isolation, but `chroot()` alone is a notoriously weak, incomplete security
boundary on its own: a process running as root inside a chroot can often escape it entirely (classic
techniques include creating device nodes to access raw disk devices directly, or using `chroot()`
itself a second time combined with directory-traversal tricks to break out), which is exactly why
modern container isolation never relies on `chroot()` alone, always combining it with mount
namespaces (so the process's mount table itself, not just its apparent root, is genuinely isolated),
user namespaces (removing genuine root privilege even if escape were otherwise possible), and other
namespace/cgroup primitives layered on top. `pivot_root()` (used by `switch_root` during the initramfs
boot sequence, as covered in Section 1, and internally by some container runtime implementations)
is a related but distinct and more robust operation: rather than merely changing what path resolves to
`/`, it actually swaps the process's current root mount with a new one, moving the *old* root to a
specified location (where it can then be explicitly unmounted and detached) rather than leaving it
merely inaccessible-but-still-present the way `chroot()` does — this is a meaningfully stronger
operation specifically because the old root filesystem can be genuinely, completely unmounted
afterward, closing off the escape vectors that rely on the old root still being mounted (just
unreachable via normal path resolution) somewhere in the mount namespace.

### Key commands
```
chroot /path/to/newroot /bin/bash    # manually chroot into a directory (testing/rescue use)
unshare --mount --pivot-root=/new / bash   # combine a mount namespace with pivot_root for stronger isolation
cat /proc/<pid>/root                   # symlink showing a process's actual chroot'd root, if any
```

## Namespaces as Isolation Primitive (recap in security context)

(Namespaces themselves are covered in networking/virtualization detail elsewhere; this entry focuses
specifically on their role as *security* isolation primitives.) From a security perspective, the most
important namespace is the user namespace (`CLONE_NEWUSER`), because it's the one namespace type that
can make a process's apparent root privilege genuinely meaningless outside its own namespace: a
process can have UID 0 (root) *inside* its own user namespace — able to perform operations that
normally require root within the scope of what that namespace controls — while being mapped to an
entirely unprivileged, ordinary UID on the host system outside it, via an explicit UID/GID mapping
(`/proc/<pid>/uid_map`) established by whatever privileged process created the namespace. This is what
makes "rootless containers" possible: a container process can believe it's root (satisfying
applications that hard-require root for certain operations, like binding privileged ports or changing
file ownership within its own container filesystem) while a genuine host-level compromise of that
"root" only grants the attacker the underlying, unprivileged host UID's actual privileges, dramatically
limiting the blast radius compared to a container actually running with real host-root privilege. The
other namespace types (PID, network, mount, UTS, IPC, and cgroup) each independently isolate a specific
resource *view* rather than a privilege level, meaning correctly reasoning about a container's true
security posture requires considering the full combination in use, not any single namespace type
alone — a container with an isolated PID namespace but no user namespace (still running as genuine host
root) provides essentially zero meaningful privilege isolation despite the process tree looking
isolated, which is exactly why user namespace adoption (historically slower than the other namespace
types due to real compatibility friction with some existing tooling/filesystems) is considered one of
the most security-relevant, and historically most under-deployed, hardening steps available for
container workloads.

### Key commands
```
unshare --user --map-root-user bash    # create a user namespace where you appear as root, but aren't on the host
cat /proc/<pid>/uid_map                  # inspect the UID mapping for a process's user namespace
podman run --userns=auto ...               # example of a container runtime defaulting toward rootless/user-namespaced execution
lsns -t user                                # list active user namespaces on the system
```

## cgroups for Resource Isolation

(cgroups' resource-*limiting* mechanics are covered in Sections 2/3/9; this entry focuses on their
role in the security/isolation model specifically.) From a security standpoint, cgroups' primary value
is not access control in the traditional permission sense but availability/denial-of-service
protection: without resource limits, any single process (whether malicious, buggy, or simply a noisy
neighbor in a multi-tenant environment) can consume unbounded CPU, memory, PIDs, or I/O bandwidth,
degrading or entirely denying service to every other legitimate workload on the same host — cgroups
close this gap by letting an administrator enforce hard, kernel-verified ceilings per workload that no
amount of application-level misbehavior can exceed, regardless of DAC/MAC permission outcomes for that
same process. The PID controller specifically deserves note as a frequently-overlooked but genuinely
important hardening measure: without a `pids.max` limit, a fork-bomb (a process that repeatedly forks
itself with no bound) can exhaust the entire system's PID space, effectively denying service to every
other process on the host (including the ability to even spawn a new shell to diagnose or fix the
problem) — a per-cgroup PID limit contains this failure to the offending cgroup alone, letting the
rest of the system continue operating normally while the runaway cgroup itself simply fails to fork
further once it hits its own ceiling. Combined with namespaces (providing *view* isolation) and MAC/
capabilities (providing *permission* isolation), cgroups round out the three complementary pillars
container security actually rests on — no single one of these three mechanisms alone constitutes
meaningful container isolation, and a security review of any containerized/multi-tenant environment
should explicitly verify all three are configured, not just assume "it's in a container" implies
comprehensive isolation by default.

### Key commands
```
cat /sys/fs/cgroup/<path>/pids.max         # PID limit for a cgroup (fork-bomb containment)
cat /sys/fs/cgroup/<path>/pids.current       # current process count in a cgroup
systemd-run --scope -p PIDsLimit=100 command   # launch a command in a cgroup with an enforced PID limit
```

## Kernel Hardening (KASLR, SMEP/SMAP, stack canaries)

Beyond access-control policy, the kernel itself implements several defense-in-depth mitigations
specifically against memory-corruption-based exploitation techniques, on the premise that some
vulnerabilities (buffer overflows, use-after-free bugs) will inevitably exist and the goal is making
them substantially harder to reliably exploit rather than assuming they'll never occur. KASLR (Kernel
Address Space Layout Randomization) randomizes the kernel's own load address in memory at each boot,
specifically defeating exploitation techniques that depend on knowing a fixed, predictable kernel
code/data address to redirect execution toward (return-oriented programming gadgets, for instance,
require knowing exactly where useful instruction sequences live in memory) — without knowing the
randomized base address, an attacker's otherwise-working exploit for a memory corruption bug typically
fails outright rather than succeeding, though various information-disclosure side-channel bugs have
historically been used specifically to defeat KASLR by leaking the actual randomized base address
before then chaining a separate memory-corruption exploit. SMEP (Supervisor Mode Execution Prevention)
and SMAP (Supervisor Mode Access Prevention) are CPU-hardware features (not purely kernel-software
mitigations) that the kernel enables to prevent itself from ever executing code (SMEP) or dereferencing
data (SMAP) located in user-space memory while running in kernel/supervisor mode — directly closing
off a once-common exploitation technique where an attacker plants malicious "kernel-mode" shellcode in
ordinary, easily-controlled user-space memory and then merely needs to redirect a vulnerable kernel
code path's execution there, since without SMEP/SMAP the kernel would otherwise happily execute or
read/write that attacker-controlled user-space memory as if it were legitimate kernel data/code. Stack
canaries are a compiler-inserted (not kernel-specific, though the kernel itself is compiled with them
too) mitigation against classic stack-buffer-overflow attacks: a random, secret value is placed on the
stack between local variables and the saved return address at function entry, and checked for
corruption immediately before the function returns — a buffer overflow attempting to overwrite the
return address to redirect execution must first overwrite this canary value in the process, and a
mismatched canary triggers immediate, controlled process termination rather than allowing the
corrupted return address to actually be used, converting what would otherwise be a potentially
exploitable memory-corruption bug into a reliable crash instead.

### Key commands
```
cat /proc/sys/kernel/kptr_restrict     # controls whether kernel addresses are hidden from unprivileged /proc reads
dmesg | grep -i "kernel base"            # (if exposed) confirm KASLR randomized load address differs across boots
cat /proc/cpuinfo | grep -o 'smep\|smap'   # confirm CPU/kernel support for SMEP/SMAP
readelf -d <binary> | grep -i stack        # (indirectly) confirm stack-protector related symbols in a compiled binary
```

## Audit Framework (auditd)

The Linux Audit subsystem provides fine-grained, kernel-level logging of security-relevant events —
syscalls matching configured rules, file access to specifically-watched paths, and authentication
events surfaced by PAM — producing a tamper-evident (when properly configured with immutable log
rotation and remote log shipping) record essential for compliance regimes (PCI-DSS, HIPAA, common
criteria certifications) and genuine incident forensics, distinct from and complementary to ordinary
application/syslog logging since it captures kernel-level truth about what actually happened
(which syscalls were invoked, by which UID, against which specific file) rather than whatever an
application chose to log about its own higher-level view of events. Audit rules are configured via
`auditctl` (or persisted in `/etc/audit/rules.d/` for rules that must survive a reboot) and fall into
two main categories: syscall rules (watch for specific syscalls, optionally filtered by architecture,
specific arguments, or the UID/UID-range of the calling process — a very common hardening rule watches
every `execve` call by UID 0, or every syscall attempting to change a file's ownership/permissions
system-wide) and file-watch rules (`-w /etc/shadow -p wa -k identity` style rules watching a specific
path for write/attribute-change access, tagged with a searchable key for later correlation). The audit
daemon (`auditd`) receives these events from the kernel and writes them to `/var/log/audit/audit.log`
in a structured, `ausearch`/`aureport`-queryable format specifically designed for forensic correlation
across many related events (a single logical action, like a file access denial, often generates several
related audit records that need to be correlated by a shared event ID/timestamp to reconstruct the
full picture) — and because a full, unfiltered audit configuration can generate enormous log volume at
significant performance cost, real-world audit rule design is a deliberate balancing act between
capturing genuinely security-relevant events comprehensively and avoiding overwhelming log storage/
processing capacity with excessive, low-value noise from routine, benign activity.

### Key commands
```
auditctl -w /etc/shadow -p wa -k shadow_changes   # watch a specific file for write/attribute-change access
ausearch -k shadow_changes                          # search audit log for events matching a specific rule key
aureport --auth --summary                            # summarized report of authentication events
ausearch -m avc -ts today                              # search for today's SELinux AVC denial events specifically
```

## SSH Security and Key-based Authentication

SSH is the standard secure remote access protocol for Linux, and its authentication model's most
important security property is public-key authentication's asymmetry: a user generates a public/
private key pair, keeps the private key secret (ideally itself encrypted with a passphrase and/or
held in a hardware security key/TPM rather than as a bare file on disk), and places only the public
key in the target account's `~/.ssh/authorized_keys` — authentication then proceeds via a
challenge-response exchange where the server, holding only the public key, can verify that the
connecting client possesses the corresponding private key (by checking a signature the client
computes over server-provided challenge data) without the private key itself ever being transmitted
or exposed to the server at any point, meaningfully stronger than password authentication where the
secret itself must be transmitted (even if only within an encrypted channel) and is vulnerable to
guessing/brute-force/credential-stuffing attacks in a way a sufficiently large private key simply
isn't. `sshd_config` hardening conventionally disables password authentication entirely
(`PasswordAuthentication no`, forcing key-based auth for all interactive access), disables direct root
login (`PermitRootLogin no`, forcing administrators to authenticate as an unprivileged user and
`sudo`/`su` afterward, preserving individual accountability rather than a shared, anonymous root
login path), and often restricts which users/groups may connect at all (`AllowUsers`/`AllowGroups`).
SSH agent forwarding (`ForwardAgent yes`) lets a private key held only on a user's local machine be
used to authenticate onward from an intermediate jump host without ever copying the private key to
that intermediate host — a genuine convenience, but a meaningfully real security risk if the
intermediate host is compromised, since a malicious root user there could, for the duration of the
forwarded session, request the forwarding agent to sign arbitrary further authentication challenges
on the original user's behalf without ever obtaining the actual private key bytes; `ProxyJump`
(replacing older, clunkier manual double-hop `ssh` invocations) is the generally preferred modern
alternative for reaching hosts behind a bastion, since it establishes a direct, end-to-end encrypted
tunnel through the jump host without needing agent forwarding's broader trust extension at all. Host
key verification (the "authenticity of host ... can't be established" prompt on first connection,
recorded thereafter in `~/.ssh/known_hosts`) exists specifically to detect man-in-the-middle attacks
substituting an attacker-controlled host for the genuine intended destination — blindly accepting
unknown host keys (`StrictHostKeyChecking no`, sometimes used carelessly in automation) defeats this
protection entirely and should be replaced with pre-provisioning known, trusted host keys through a
secure out-of-band channel wherever automation genuinely needs non-interactive SSH connections.

### Key commands
```
ssh-keygen -t ed25519 -a 100          # generate a modern, strong key pair (Ed25519, high KDF work factor)
ssh-copy-id user@host                   # securely install a public key into a remote account's authorized_keys
sshd -T | grep -iE 'passwordauth|permitrootlogin'   # confirm effective (post-include-merge) sshd hardening settings
ssh -J bastion-host target-host           # ProxyJump through a bastion without needing agent forwarding
```

## Firewalls (iptables/nftables/firewalld)

(The packet-filtering engine itself, netfilter, is covered in networking detail in Section 5; this
entry focuses on the operational/security-policy layer built on top of it.) A host-based firewall's
job is enforcing a security policy about which network traffic is permitted to/from a host, and on
Linux this is ultimately always implemented via netfilter hooks, whether configured directly through
raw `iptables`/`nftables` rules or through a higher-level management layer. `firewalld` is the
default higher-level firewall management daemon on many modern distributions (RHEL/Fedora/CentOS
family in particular), providing a "zone"-based abstraction (predefined trust levels like `public`,
`internal`, `trusted`, `dmz`, each with different default policies) and dynamically reloadable
configuration (rule changes can be applied without dropping already-established connections, unlike a
naive full `iptables-restore` of an entirely new rule set, which briefly clears all state including
active connection tracking) — under the hood it still ultimately generates and manages the same
underlying nftables/iptables rules, but organizes them around a more operationally friendly, service-
and-zone-oriented mental model rather than requiring administrators to hand-write raw chain/rule
syntax directly for routine changes. A well-hardened host firewall policy follows default-deny
principles: reject/drop all inbound traffic by default, explicitly allow-list only the specific ports/
services genuinely needed (SSH, and whatever application ports the host's actual role requires), and
apply source-address restrictions wherever the set of legitimate clients is known and bounded (an
internal database server, for instance, has no legitimate reason to accept connections from the
public internet at all, and a source-restricted rule specifically closes that unnecessary exposure
regardless of whether the application itself has its own authentication). Firewalls are explicitly a
defense-in-depth layer, not a substitute for application-level authentication/authorization — a
correctly-configured firewall reduces the *exposed attack surface* (which services are even reachable
at all, from where) but does nothing to protect a genuinely vulnerable, exposed service from
exploitation by a client the firewall does legitimately permit to connect, which is exactly why
firewall hardening is always paired with, never a replacement for, the access-control mechanisms
(DAC/MAC/capabilities/seccomp) covered elsewhere in this section.

### Key commands
```
firewall-cmd --list-all               # current firewalld zone configuration and allowed services/ports
firewall-cmd --permanent --add-service=https --zone=public   # persistently allow a service in a zone
firewall-cmd --reload                   # apply persistent changes without dropping existing connections
nft list ruleset                          # inspect the actual underlying nftables rules firewalld manages
```

## File Integrity Monitoring

File Integrity Monitoring (FIM) detects unauthorized or unexpected changes to critical system files —
binaries, configuration files, kernel modules — by maintaining a trusted baseline of cryptographic
hashes (and other metadata: permissions, ownership, size, timestamps) for a defined set of monitored
paths, then periodically (or, for more sophisticated tools, in near-real-time via kernel-level file
access hooks) recomputing and comparing against that baseline to surface any drift. Tools like AIDE
(Advanced Intrusion Detection Environment) and Tripwire build and store this baseline database
(critically, stored somewhere the monitored system itself cannot tamper with — ideally on read-only or
off-host storage, since a baseline database stored on the same, potentially-compromised host provides
no real integrity guarantee if an attacker with sufficient privilege can simply update the baseline to
match their own malicious changes) and are typically run on a scheduled basis (a cron job or systemd
timer triggering a scan and diff against the baseline, with results reviewed by security/operations
staff or fed into a SIEM for automated alerting). FIM is specifically valuable for detecting a category
of compromise that purely network/process-based monitoring can miss entirely: a rootkit or backdoor
that modifies a legitimate system binary in place (replacing `/bin/ps` or `/usr/sbin/sshd` with a
trojaned version that behaves normally for most purposes but hides the attacker's processes or
provides a hidden backdoor login path) would otherwise be extremely difficult to detect through normal
system observation, since the compromised binary is specifically designed to lie convincingly to
whatever's asking — but its cryptographic hash will not match the known-good baseline regardless of
how convincingly it otherwise behaves, which is exactly the property FIM relies on and why establishing
the baseline itself, at a moment of known-good system state and via a trustworthy, tamper-resistant
mechanism, is the single most operationally critical step in making FIM meaningful at all.

### Key commands
```
aide --init                          # build the initial trusted baseline database
aide --check                           # compare current filesystem state against the baseline, report drift
sha256sum /bin/ps /usr/sbin/sshd         # manual, ad-hoc integrity spot-check against known-good hashes
rpm -Va / dpkg --verify <package>          # package-manager-native integrity verification against installed package manifests
```

## Rootkits and Detection

A rootkit is malicious software specifically engineered to maintain privileged, persistent access to a
compromised system while actively hiding its own presence from normal administrative observation —
the defining characteristic distinguishing a rootkit from ordinary malware is this active concealment
effort, not merely the malicious capability itself. Userspace rootkits typically work by replacing
common system binaries (`ps`, `ls`, `netstat`) with trojaned versions that filter their own malicious
processes/files/connections out of the displayed output, or by using `LD_PRELOAD` to inject a
malicious shared library that intercepts and filters the results of common libc calls
(`readdir()`, `opendir()`) system-wide for every process that loads it — both approaches are
detectable via file integrity monitoring (the replaced binaries/injected library won't match
known-good hashes) and by cross-checking observations through independent tools/methods that the
rootkit didn't anticipate needing to also filter (comparing `ps` output against a raw `/proc` directory
listing, for instance, since a userspace rootkit filtering `ps` output specifically often fails to
also correctly filter every possible alternative way of enumerating `/proc`). Kernel-level rootkits are
substantially more dangerous and harder to detect, operating as a malicious loadable kernel module (or
via other kernel-memory-patching techniques) that can lie about *any* information the kernel itself
provides to any userspace tool whatsoever — including, if sufficiently sophisticated, subverting
`/proc` and `/sys` themselves at the source, meaning no purely userspace-level cross-checking technique
can reliably detect it, since every information source userspace could possibly consult is itself
under the compromised kernel's control. Detecting a genuinely sophisticated kernel-level rootkit
generally requires either offline/out-of-band analysis (booting from trusted, external media and
inspecting the suspect system's disk without ever executing its potentially-compromised kernel, or
comparing memory/disk state against a trusted baseline from outside the running, possibly-compromised
system entirely) or specialized kernel integrity tooling (Secure Boot with kernel lockdown, discussed
in Section 1, specifically to prevent unsigned/unauthorized kernel modules from loading in the first
place, functioning as *prevention* rather than after-the-fact detection) — which is precisely why the
mature security posture for genuinely high-value systems emphasizes prevention (Secure Boot, kernel
lockdown, mandatory module signing, minimizing attack surface via the earlier sections' hardening
techniques) far more heavily than any purely reactive, after-the-compromise rootkit-hunting technique,
since a sufficiently capable kernel-level compromise can, in the worst case, make detection from within
the running system fundamentally unreliable.

### Key commands
```
rkhunter --check                    # userspace rootkit-hunting tool: known-signature and heuristic checks
chkrootkit                            # alternative userspace rootkit scanner
lsmod                                  # inspect loaded kernel modules for anything unexpected/unsigned
mokutil --list-enrolled                 # confirm which keys are trusted for module signing (prevention layer)
```

---

### Interview Questions

**Conceptual/Internals (8)**

1. **What is the fundamental difference between DAC and MAC, and why does a system need both?**
   DAC lets a resource's owner decide access at their own discretion (traditional rwx bits/ACLs), with
   no independent check on whether that discretion represents sound security policy. MAC layers a
   second, system-policy-enforced check (SELinux/AppArmor) that ordinary users, and even root, cannot
   override through DAC changes alone — both checks must pass for access to be granted, closing the
   gap where a compromised process or careless owner could otherwise over-grant access purely through
   DAC.

2. **Explain Linux capabilities and the problem they solve versus traditional root/non-root.**
   Traditional UNIX privilege was all-or-nothing: any program needing even one privileged operation
   had to run fully as root. Capabilities decompose root's monolithic privilege into ~40 independently
   grantable units (binding privileged ports, changing ownership, adjusting the clock, etc.), letting a
   binary be granted only the narrow privilege it actually needs via file capabilities, substantially
   reducing the blast radius of a compromise compared to full root.

3. **What does seccomp actually restrict, and how does it differ from capabilities?**
   Capabilities govern what a syscall is allowed to *do* once invoked (fine-grained privilege).
   Seccomp instead restricts *which syscalls can be invoked at all*, providing defense-in-depth for the
   scenario where an attacker already has code execution — even with full capabilities, a seccomp
   filter can block an entire syscall from being called, closing off attack surface regardless of what
   that syscall's own permission checks would otherwise allow.

4. **How does public-key SSH authentication avoid ever transmitting the private key, and why is
    that stronger than password authentication?**
   The server holds only the public key and issues a challenge; the client signs it using the private
   key locally and returns the signature, which the server verifies using the public key alone — the
   private key itself never leaves the client. This avoids the fundamental weakness of password
   authentication, where the secret itself (even encrypted in transit) must ultimately be presented and
   is vulnerable to guessing, reuse, and credential-stuffing attacks in a way a sufficiently strong key
   pair isn't.

5. **What is a user namespace, and why is it considered the most security-critical namespace type
    for containers?**
    A user namespace lets a process have UID 0 (root) inside its own namespace while being mapped to an
    unprivileged UID on the host outside it. This is what makes rootless containers possible — a
    genuine host-level compromise of a containerized "root" process only grants the attacker the
    underlying unprivileged host UID's actual privileges, whereas without a user namespace a
    containerized root process really is host root, regardless of how isolated its other namespaces
    appear.

6. **Why do fork bombs require a PID cgroup limit specifically, rather than just CPU/memory limits,
    to be contained?**
    A fork bomb's damage comes from exhausting the finite global PID space, not primarily from CPU or
    memory consumption per process — a CPU or memory limit alone doesn't prevent a process from
    successfully forking enormous numbers of tiny, near-zero-resource children until PIDs are
    exhausted system-wide. A `pids.max` cgroup limit directly caps the number of processes a cgroup
    may create, containing this specific failure mode regardless of how little CPU/memory each
    individual forked process consumes.

7. **What's the difference between SELinux and AppArmor's approach to policy?**
   SELinux implements Type Enforcement: every process runs in a security domain and every resource has
   a type, with policy exhaustively defining allowed domain-to-type interactions in a default-deny
   model, offering fine-grained but complex policy. AppArmor uses simpler, path-based profiles attached
   per-binary listing explicitly allowed paths/capabilities/network operations, generally easier to
   write and reason about but coarser-grained than SELinux's type-based model.

8. **Why is `/etc/shadow` separate from `/etc/passwd`, and what does that separation protect
    against?**
   `/etc/passwd` must remain world-readable since many tools need to resolve UIDs/usernames, but it
   historically also stored password hashes directly, exposing them to any local user for offline
   brute-force attack. Moving hashes into `/etc/shadow`, readable only by root and privileged setuid
   binaries, removes that exposure while preserving the necessary world-readability of the
   non-sensitive user/UID mapping data in `/etc/passwd`.

**Scenario/Troubleshooting (6)**

9. **A service fails to bind to a file/socket despite correct DAC ownership and permission bits.
    What should you check next on a system running SELinux?**
    Check `ausearch -m avc -ts recent` (or `dmesg`) for AVC denial messages — SELinux enforces an
    independent, policy-driven check on top of DAC, so correct ownership/permission bits alone do not
    guarantee access if the process's SELinux domain isn't permitted to interact with that resource's
    type. `sealert` can provide a human-readable explanation and often a suggested policy fix (e.g., a
    `semanage fcontext`/`restorecon` correction) once the specific denial is identified.

10. **After a routine sudoers change, a specific team can no longer run any sudo commands, including
    the ones needed to fix the sudoers file itself. What's the safe recovery process, and how should
    such changes be tested going forward?**
    Recovery requires access via an already-open root session (or single-user/rescue mode) to correct
    the sudoers file directly, since the broken rule blocks the normal sudo path entirely. Going
    forward, sudoers changes should always be made via `visudo` (syntax validation before saving) and
    tested in a still-open secondary session before closing the session used to make the change, so a
    mistake never fully locks out privilege escalation.

11. **A container process explicitly runs as UID 0 inside the container, and a security review flags
    this as high risk. How would you determine whether this is actually a serious problem or a
    non-issue?**
    Check whether the container is running with a user namespace mapping (`podman run --userns=auto`
    or equivalent, and inspect `/proc/<pid>/uid_map` for the actual container process) — if a genuine
    UID mapping is in place, the container's "root" is mapped to an unprivileged host UID and the risk
    is substantially mitigated. If no user namespace is in use, the container process really is host
    root, and the finding is a genuine, serious risk requiring remediation (enabling user namespaces,
    or at minimum dropping capabilities and applying strict seccomp/MAC policy as partial
    compensating controls).

12. **`rpm -Va`/`dpkg --verify` reports a critical system binary's hash no longer matches the
    package manifest, but the file's permissions and ownership look completely normal. What's the
    appropriate immediate response?**
    Treat this as a strong potential rootkit/compromise indicator rather than routine drift — because a
    userspace rootkit specifically aims to look normal to casual inspection (permissions/ownership),
    the hash mismatch from an independent, trusted source (the package manager's own manifest) is far
    more reliable evidence. The appropriate response is isolating the host from the network, preserving
    forensic evidence (ideally via offline/out-of-band analysis rather than continuing to trust the
    potentially-compromised running kernel/userspace), and rebuilding from known-good media rather than
    attempting to "clean" the existing installation in place.

13. **An `sshd_config` audit finds `PermitRootLogin yes` and `PasswordAuthentication yes` still
    enabled on a production host, contrary to organizational policy. What's the remediation and what
    should you verify before applying it?**
    Before changing, confirm every legitimate user/automation account has working key-based
    authentication already configured and tested (to avoid an accidental full lockout), then set
    `PermitRootLogin no` and `PasswordAuthentication no`, reload `sshd`, and verify continued access via
    a still-open secondary session before closing the session used to make the change — the same
    "verify in a second session before closing the first" discipline applies here as with sudoers
    changes, since a mistake in SSH hardening can be just as lockout-prone.

14. **A file integrity monitoring tool reports drift on dozens of files after a routine, approved
    package update. How do you distinguish expected drift from a genuine concern?**
    Cross-reference the flagged files against the package manager's own manifest/changelog for the
    update in question (`rpm -q --changelog`, `dpkg -L`/package changelogs) to confirm the changes
    correspond to files the update legitimately modified. Any drift on files *not* accounted for by
    the approved update (or on files the package manager itself reports as already verified/unmodified)
    warrants deeper investigation as potentially unrelated, unauthorized change.

**FAANG-level Deep Dive (6)**

15. **Explain precisely why SELinux's default-deny Type Enforcement model provides meaningfully
    stronger containment than AppArmor's path-based profiles for a compromised, privilege-escalated
    process.**
    SELinux ties policy to abstract types and domains rather than concrete paths, meaning even if an
    attacker manages to relocate, rename, or hard-link a resource to an unexpected path, its security
    type (stored as a filesystem xattr, tied to the object itself, not derived from its current path)
    still governs access — a path-based system evaluating rules purely against the resource's current
    path can potentially be circumvented by path manipulation tricks that don't change the underlying
    object's semantic type, whereas SELinux's abstraction is specifically designed to be robust against
    this exact class of evasion.

16. **Why does seccomp-bpf's kernel-resident filter execution model impose meaningfully lower
    overhead than an equivalent policy enforced via `ptrace()`-based syscall interception?**
    A `ptrace()`-based approach requires a full context switch to a separate tracing process for every
    single intercepted syscall, which must then inspect the syscall and explicitly permit or deny it
    before the traced process can proceed — a substantial, per-syscall overhead cost. seccomp-bpf's
    filter program executes directly in-kernel at syscall entry, making its allow/deny decision without
    ever leaving kernel context or requiring a separate process round-trip, which is precisely why
    seccomp-bpf-based sandboxing (used by Chrome, container runtimes) remains practical for
    high-syscall-frequency workloads where `ptrace()`-based interception would be prohibitively slow.

17. **Why can a rootkit that only modifies userspace binaries/libraries (not the kernel) still often
    be reliably detected, while a kernel-level rootkit is fundamentally much harder to detect from
    within the running system?**
    A userspace rootkit relies on trojaning specific binaries or intercepting specific library calls,
    and independent verification paths (a package manager's own manifest hash check, direct inspection
    of `/proc` bypassing a trojaned `ps`, or file integrity monitoring against an externally-stored
    baseline) exist that the rootkit's authors may not have anticipated or successfully subverted. A
    kernel-level rootkit can, in principle, control the very mechanisms (`/proc`, `/sys`, even the
    behavior of hash/checksum syscalls themselves) that any userspace-level detection technique would
    need to rely on, meaning every information source available to a userspace check is potentially
    already under the compromised kernel's control — genuine detection at that point requires stepping
    entirely outside the running system's own trust boundary (offline analysis from trusted external
    media).

18. **Explain why SMEP/SMAP specifically target a different exploitation technique than stack
    canaries, and why both are still needed together as complementary mitigations.**
    Stack canaries specifically detect (after the fact, via a check at function return) whether a
    stack-based buffer overflow has corrupted a saved return address, converting an otherwise
    potentially-exploitable overflow into a reliable crash — but they do nothing to prevent an
    already-successful redirection of execution flow via some other memory-corruption technique
    (heap corruption, use-after-free) that doesn't involve overflowing a stack buffer at all. SMEP/SMAP
    instead prevent the kernel from executing/accessing attacker-controlled user-space memory
    regardless of *how* execution flow redirection was achieved, closing off an entire category of
    "plant shellcode in user memory, then redirect kernel execution there" techniques independent of
    the specific memory-corruption bug used to achieve that redirection — the two mitigations operate
    at different points in a typical exploit chain and neither substitutes for the other.

19. **Why does PAM's `sufficient` control value require careful ordering within a module stack to
    avoid accidentally weakening authentication policy?**
    A `sufficient` module, if it succeeds, immediately satisfies the entire stack's authentication
    requirement without necessarily evaluating subsequent modules (subject to no prior `requisite`
    module having already failed) — if a weaker or more permissive authentication method is placed as
    `sufficient` earlier in the stack than a stronger, intended-to-be-mandatory method, a user (or
    attacker) satisfying only the weaker method can bypass the stronger one entirely. Correct stack
    ordering must ensure any `sufficient` module genuinely represents an acceptable, fully-equivalent
    path to authentication on its own, and that mandatory checks (like account-lockout/expiration
    validation) are expressed as `required`, not `sufficient`, so they cannot be bypassed by an earlier
    module's success.

20. **Why is a user namespace mapping alone insufficient to fully secure a "rootless" container, and
    what additional layers are still required for genuinely robust isolation?**
    A user namespace changes what a process's UID 0 actually means in terms of host-level privilege,
    but it does not, by itself, restrict which syscalls are available (seccomp's job), which
    files/resources are accessible under system-wide MAC policy (SELinux/AppArmor's job), or how much
    CPU/memory/PIDs the process may consume (cgroups' job) — a "rootless" container relying on user
    namespace mapping alone but with permissive seccomp, no MAC policy, and no resource limits still
    presents a substantial attack surface and potential for resource-exhaustion denial-of-service
    against its host, even though a genuine host-root privilege escalation specifically is meaningfully
    harder to achieve. Robust container isolation requires all of namespaces, cgroups, capabilities,
    seccomp, and MAC policy configured together, not any single mechanism treated as sufficient on its
    own.

### Hands-On Labs

**Lab 1: Write and enforce a custom SELinux/AppArmor policy**
- Objective: Experience MAC policy authoring and enforcement firsthand.
- Setup: A VM with SELinux (RHEL/Fedora-family) or AppArmor (Ubuntu) available.
- Tasks: Write a minimal custom profile/policy module restricting a test binary to only its own
  working directory; test in permissive/complain mode first, reviewing generated denial logs; switch
  to enforcing mode and confirm the restriction is actually applied.
- Expected outcome: A working, enforced custom MAC policy with a documented before/after access test.

**Lab 2: Capability-drop a privileged binary**
- Objective: Replace a setuid-root pattern with narrowly-scoped Linux capabilities.
- Setup: Any Linux host with a compiler.
- Tasks: Write a small program that binds to a privileged port (<1024) as an unprivileged user, first
  observing it fail with `EACCES`; grant it `CAP_NET_BIND_SERVICE` via `setcap` instead of making it
  setuid-root; confirm it now succeeds while `id`/`whoami` inside the program still report the
  unprivileged user.
- Expected outcome: A demonstrated, narrowly-scoped privilege grant achieving the same functional
  outcome as setuid-root without the broad privilege exposure.

**Lab 3: Build and test a seccomp filter**
- Objective: Directly experience syscall-level sandboxing.
- Setup: A Linux VM with `libseccomp` (or Docker for a simpler custom-profile test).
- Tasks: Write a seccomp-bpf filter (or a Docker `--security-opt seccomp=...json` profile) that blocks
  a specific syscall (e.g., `ptrace` or `mount`); run a test program/container attempting that syscall
  and confirm it's denied/killed as configured, while other normal syscalls continue to work.
- Expected outcome: A working, verified seccomp filter with a clear before/after demonstration.

**Lab 4: Rootless container user namespace verification**
- Objective: Confirm and understand user namespace UID mapping in practice.
- Setup: A host with `podman` or `unshare` available.
- Tasks: Launch a rootless container (or `unshare --user --map-root-user`) appearing as root inside;
  from the host, inspect `/proc/<pid>/uid_map` and confirm the actual host-level UID; attempt a
  privileged host-level operation from inside the "root" container context and confirm it fails at the
  host boundary.
- Expected outcome: A concrete demonstration distinguishing apparent in-namespace root from real host
  privilege.

**Lab 5: File integrity monitoring baseline and drift detection**
- Objective: Build and validate an FIM workflow end-to-end.
- Setup: A disposable VM with AIDE installed.
- Tasks: Initialize an AIDE baseline; modify a monitored binary/config file (simulating unauthorized
  change); run a check and confirm the drift is detected and reported; document the baseline's storage
  location and why it must be protected from tampering by the monitored system itself.
- Expected outcome: A working FIM baseline/check cycle with a documented, reasoned explanation of
  baseline-integrity requirements.

### Production Incidents

**Incident 1: Privilege escalation via an overly broad `CAP_SYS_ADMIN` grant in a container**
- Symptom: A security assessment discovers a production container was granted `CAP_SYS_ADMIN`
  (intended to allow a specific mount operation the application needed) and demonstrates a
  proof-of-concept container escape using that capability.
- Investigation: Reviewing the container's deployment manifest confirms `CAP_SYS_ADMIN` was added
  broadly to resolve a permission error during initial rollout, without investigating which much
  narrower capability was actually required for the specific mount operation involved.
- Root cause: `CAP_SYS_ADMIN` is a notoriously broad, catch-all capability effectively granting
  near-root privilege for many practical purposes; it was used as a quick fix rather than identifying
  the true minimal capability set needed.
- Recovery: Identified the specific narrow capability actually required, removed `CAP_SYS_ADMIN`,
  and validated the application still functioned correctly with the minimal grant.
- Prevention: Added a capability-review gate to the container image/deployment approval pipeline
  specifically flagging any request for `CAP_SYS_ADMIN` (or other broad capabilities) for mandatory
  security review and justification before approval.

**Incident 2: SSH agent forwarding enabled lateral movement after a jump-host compromise**
- Symptom: During incident response for a compromised bastion/jump host, investigators find evidence
  the attacker used SSH sessions transiting that host to authenticate onward to several additional
  production hosts, despite the attacker never obtaining any private key file.
- Investigation: Confirmed several administrators had `ForwardAgent yes` configured for connections
  through the bastion, and the compromised host's root access allowed the attacker to use those
  forwarded agent connections to sign authentication challenges on the legitimate administrators'
  behalf during their active sessions, without ever needing the actual private key bytes.
- Root cause: Broad use of agent forwarding through a bastion host extended trust further than
  necessary, and the bastion itself was not sufficiently hardened/monitored to prevent the initial
  compromise that enabled this lateral movement technique.
- Recovery: Rotated all potentially-exposed credentials/keys, rebuilt the compromised bastion host, and
  migrated administrator access to `ProxyJump`-based direct tunneling instead of agent forwarding.
- Prevention: Disabled agent forwarding organization-wide in favor of `ProxyJump`, and substantially
  increased bastion host hardening and monitoring given its now-recognized high-value position as a
  pivot point for lateral movement.

**Incident 3: Fork bomb from a misconfigured CI job caused a fleet-wide PID exhaustion outage**
- Symptom: Multiple CI worker hosts become completely unresponsive, including refusing new SSH
  sessions, with no corresponding CPU or memory exhaustion alerts.
- Investigation: On a host recovered via out-of-band console access, `cat /proc/sys/kernel/pid_max`
  compared against process counts confirmed PID space exhaustion; reviewing recently-run CI jobs
  identified a build script with a recursive retry loop that, due to a logic bug, spawned an
  unbounded number of child processes rather than the intended bounded retry count.
- Root cause: CI worker cgroups had no `pids.max` limit configured, so the buggy job's runaway forking
  was able to exhaust the entire host's global PID space, denying service to every other process
  (including the ability to spawn a diagnostic shell) despite CPU/memory remaining largely available.
- Recovery: Hard-rebooted affected hosts (the only reliable recovery path once PID space was fully
  exhausted), fixed the CI script's retry logic bug.
- Prevention: Applied a `pids.max` cgroup limit to all CI job execution environments fleet-wide as a
  standard containment measure, and added PID-usage-percentage as a monitored/alerted metric alongside
  existing CPU/memory monitoring.
