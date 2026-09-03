# Linux Interview Preparation — Mastery Guide

A complete, in-depth Linux interview preparation curriculum for Senior Linux Engineer, SRE, Platform
Engineer, DevOps Engineer, Kernel Engineer, Staff Engineer, and FAANG/MANGA-level interviews. Every
section teaches from absolute fundamentals through expert-level kernel internals — kernel data
structures, syscalls, algorithms, and real production incident scenarios — not just command
memorization. Each file includes a depth-first narrative per topic, key commands, 20 interview
questions (8 conceptual + 6 scenario/troubleshooting + 6 FAANG-level deep dives), hands-on exercises,
and production incident case studies.

The source specification for this guide is [prompt.txt](prompt.txt).

## How to use this guide

Work through the sections in order for a structured 3-6 month preparation timeline, or jump directly
to a specific section to fill a targeted gap. Every section builds on kernel primitives introduced in
Sections 1-3, so if you're short on time, do not skip the fundamentals, process/scheduling, and memory
management sections — nearly every later section (networking, storage, containers, security) directly
references concepts first established there.

## Sections

| # | Section | Covers |
|---|---------|--------|
| 1 | [Fundamentals & History](01-FUNDAMENTALS-HISTORY.md) | UNIX/Linux history, GNU philosophy, kernel vs distro, monolithic vs microkernel, full boot process (BIOS/UEFI → GRUB → initramfs → kernel init → systemd), FHS, shells, environment variables |
| 2 | [Process Management & Scheduling](02-PROCESS-MANAGEMENT-SCHEDULING.md) | `task_struct`, fork/clone/exec, copy-on-write, process states, zombies/orphans, signals, context switching, CFS scheduler, scheduling classes, NUMA-aware scheduling, real-time scheduling |
| 3 | [Memory Management](03-MEMORY-MANAGEMENT.md) | Virtual memory, paging, page faults, buddy/slab allocators, mmap, memory overcommit and the OOM killer, swap, huge pages, page cache, NUMA, memory cgroups |
| 4 | [Filesystems & Storage](04-FILESYSTEMS-STORAGE.md) | VFS, inodes/dentries, ext4/XFS/Btrfs internals, journaling, permissions/ACLs/xattrs, partitioning, LVM, RAID, I/O schedulers, blk-mq, the full disk I/O path |
| 5 | [Networking Stack](05-NETWORKING-STACK.md) | NIC-to-socket packet flow, network namespaces, netfilter/conntrack, TCP/IP internals, TCP state machine, congestion control, routing, veth/bridge/macvlan, DNS, eBPF/XDP, traffic control, load balancing |
| 6 | [Security & Access Control](06-SECURITY-ACCESS-CONTROL.md) | DAC vs MAC (SELinux/AppArmor), Linux capabilities, seccomp, PAM, sudo internals, chroot/pivot_root, namespaces/cgroups as security primitives, kernel hardening, auditd, SSH security, firewalls, rootkits |
| 7 | [systemd & Service Management](07-SYSTEMD-SERVICE-MANAGEMENT.md) | Unit types and dependency graphs, socket activation, journald, cgroup integration, timers vs cron, network-facing systemd daemons, restart policies, masking/enabling |
| 8 | [Observability, Performance & Troubleshooting](08-OBSERVABILITY-PERFORMANCE-TROUBLESHOOTING.md) | `/proc`/`/sys`, strace/ltrace, perf and flame graphs, bpftrace/eBPF, ftrace, classic sysstat tools, core dumps, the USE method, latency vs throughput, benchmarking |
| 9 | [Virtualization & Containers](09-VIRTUALIZATION-CONTAINERS.md) | KVM/QEMU architecture, virtio, all seven Linux namespaces, cgroups v1 vs v2, OverlayFS for containers, runc/containerd/CRI-O, a full `docker run` trace, nested virtualization |
| 10 | [Shell Scripting & Automation Mastery](10-SHELL-SCRIPTING-AUTOMATION.md) | POSIX vs bash, arrays/subshells, process/command substitution, fd redirection internals, `set -euo pipefail`, trap-based cleanup, text processing, xargs parallelism, idempotent production scripting |
| 11 | [System Design & Production Architecture](11-SYSTEM-DESIGN-PRODUCTION.md) | HA fleet design, kernel/sysctl tuning at scale, fd/connection limits, capacity planning, database and low-latency tuning, immutable infrastructure, kernel live patching, disaster recovery |
| 12 | [FAANG Behavioral & Leadership](12-FAANG-BEHAVIORAL.md) | STAR method for SRE incidents, blameless postmortems, leading major outage response, mentoring on internals, reliability vs feature velocity — plus 15 full model STAR answers |
| 13 | [Hands-On Labs & Cheat Sheet](13-HANDS-ON-LABS-CHEATSHEET.md) | 15 cross-cutting capstone labs spanning multiple sections, plus a master command-line cheat sheet organized by subject area |

## Suggested study timeline (3-6 months)

- **Weeks 1-4**: Sections 1-3 (fundamentals, process/scheduling, memory) — the kernel primitives
  everything else depends on.
- **Weeks 5-8**: Sections 4-5 (filesystems/storage, networking) — the two largest, most
  interview-dense subsystems.
- **Weeks 9-11**: Sections 6-7 (security, systemd) — access control and service management depth.
- **Weeks 12-14**: Section 8 (observability/troubleshooting) — practice diagnosing real symptoms using
  every tool covered so far.
- **Weeks 15-17**: Sections 9-10 (virtualization/containers, shell scripting) — tie primitives into
  containers and production automation.
- **Weeks 18-20**: Section 11 (system design) — apply everything at fleet scale.
- **Weeks 21-24**: Section 12 (behavioral) plus Section 13's capstone labs — mock interviews, full
  incident-to-prevention exercises, and a final pass through every section's 20 interview questions.

Do not skip the hands-on labs embedded in every section — reading the internals narrative builds
recognition, but running the labs builds the recall speed a live interview actually demands.
