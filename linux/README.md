# Linux Interview Questions - Complete Guide

> **500+ Linux Interview Questions for Senior DevOps Engineer, SRE, and Platform Engineer roles at FAANG companies**

---

## 📋 Table of Contents

- [Linux Fundamentals](#linux-fundamentals)
- [Process Management](#process-management)
- [Memory Management](#memory-management)
- [Storage & Filesystems](#storage--filesystems)
- [Networking](#networking)
- [Performance Tuning](#performance-tuning)
- [Security](#security)
- [Systemd](#systemd)
- [Troubleshooting Scenarios](#troubleshooting-scenarios)
- [Command Reference](#command-reference)

---

## Linux Fundamentals

### 🟢 Basic Questions

#### Q1: Explain the Linux boot process.

**Basic Answer:**
BIOS/UEFI → Bootloader (GRUB) → Kernel → Init/Systemd → User Space Services

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    LINUX BOOT PROCESS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. BIOS/UEFI                                                   │
│     ├── POST (Power-On Self Test)                               │
│     ├── Hardware initialization                                 │
│     └── Locate and load bootloader from boot device             │
│                                                                  │
│  2. BOOTLOADER (GRUB2)                                          │
│     ├── Stage 1: MBR/GPT boot code (512 bytes)                  │
│     ├── Stage 1.5: Filesystem drivers                           │
│     ├── Stage 2: /boot/grub2/grub.cfg                           │
│     ├── Display menu, load kernel + initramfs                   │
│     └── Pass parameters to kernel (root=, quiet, etc.)          │
│                                                                  │
│  3. KERNEL INITIALIZATION                                       │
│     ├── Decompress and start kernel                             │
│     ├── Mount initramfs (temporary root)                        │
│     ├── Initialize hardware drivers                             │
│     ├── Mount real root filesystem                              │
│     └── Execute /sbin/init (PID 1)                              │
│                                                                  │
│  4. SYSTEMD (PID 1)                                             │
│     ├── Read default.target (multi-user, graphical)             │
│     ├── Start units based on dependencies                       │
│     │   ├── sysinit.target (early services)                     │
│     │   ├── basic.target (logging, timers)                      │
│     │   ├── multi-user.target (network, services)               │
│     │   └── graphical.target (display manager)                  │
│     └── Reach login prompt                                      │
│                                                                  │
│  Boot Timeline:                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ BIOS │ GRUB │ Kernel │ initramfs │ Systemd │  Services  │    │
│  │ ~2s  │ ~1s  │  ~3s   │    ~2s    │   ~5s   │    ~10s    │    │
│  └─────────────────────────────────────────────────────────┘    │
│  Total: ~20-30 seconds typical                                   │
│                                                                  │
│  Debug Boot:                                                     │
│  systemd-analyze                    # Total boot time            │
│  systemd-analyze blame              # Time per unit              │
│  systemd-analyze critical-chain     # Critical path              │
│  journalctl -b                      # Current boot logs          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Expert Answer:**

initramfs deep dive:
```bash
# View initramfs contents
lsinitramfs /boot/initramfs-$(uname -r).img

# Rebuild initramfs
dracut --force /boot/initramfs-$(uname -r).img $(uname -r)

# What's in initramfs:
# - Minimal shell (busybox or systemd)
# - Essential drivers (storage, filesystem, LVM, RAID)
# - Cryptsetup for encrypted disks
# - Scripts to locate and mount root filesystem

# Why initramfs?
# Kernel is compiled without drivers for all storage types
# initramfs contains drivers needed to access your specific root filesystem
```

---

## Process Management

### 🟢 Basic Questions

#### Q2: Explain Linux process states and lifecycle.

**Basic Answer:**
Processes can be Running (R), Sleeping (S/D), Stopped (T), or Zombie (Z). Created with fork(), transitioned with exec(), terminated with exit().

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    PROCESS STATES                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    ┌─────────────────┐                          │
│                    │    Created      │                          │
│                    │   (fork/clone)  │                          │
│                    └────────┬────────┘                          │
│                             │                                    │
│                             ▼                                    │
│                    ┌─────────────────┐                          │
│       ┌───────────▶│     READY       │◀───────────┐             │
│       │            │ (runnable, R)   │            │             │
│       │            └────────┬────────┘            │             │
│       │                     │ Scheduled           │             │
│       │                     ▼                     │             │
│       │            ┌─────────────────┐            │             │
│       │            │    RUNNING      │────────────┘             │
│       │            │   (on CPU, R)   │      Preempted           │
│       │            └────────┬────────┘                          │
│       │                     │                                    │
│       │     ┌───────────────┼───────────────┐                   │
│       │     │               │               │                   │
│       │     ▼               ▼               ▼                   │
│       │ ┌────────┐    ┌──────────┐    ┌──────────┐              │
│       │ │SLEEPING│    │ STOPPED  │    │  ZOMBIE  │              │
│       │ │  S/D   │    │    T     │    │    Z     │              │
│       │ └───┬────┘    └──────────┘    └──────────┘              │
│       │     │                                                    │
│       └─────┘ Wakeup                                            │
│                                                                  │
│  STATE CODES (ps aux):                                          │
│  R  Running or runnable (on run queue)                          │
│  S  Interruptible sleep (waiting for event)                     │
│  D  Uninterruptible sleep (waiting for I/O)                     │
│  T  Stopped (SIGSTOP/SIGTSTP)                                   │
│  Z  Zombie (terminated, waiting for parent to reap)             │
│  X  Dead (should never be seen)                                 │
│                                                                  │
│  Additional flags:                                               │
│  <  High priority (nice < 0)                                    │
│  N  Low priority (nice > 0)                                     │
│  L  Has pages locked in memory                                  │
│  s  Session leader                                              │
│  l  Multi-threaded                                              │
│  +  Foreground process group                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Expert Answer:**

Process creation deep dive:
```c
// fork() creates a copy of the process
pid_t pid = fork();
if (pid == 0) {
    // Child process
    execve("/bin/ls", args, envp);  // Replace with new program
} else {
    // Parent process
    waitpid(pid, &status, 0);  // Wait for child to exit
}

// Modern alternative: clone()
// Allows fine-grained control over what's shared
// Used to create threads (share memory) or containers (separate namespaces)
```

Process hierarchy:
```bash
# View process tree
pstree -p

# View all info about a process
cat /proc/<pid>/status
cat /proc/<pid>/maps     # Memory mappings
cat /proc/<pid>/fd/      # File descriptors
cat /proc/<pid>/environ  # Environment variables

# D state debugging (uninterruptible sleep)
cat /proc/<pid>/stack    # Kernel stack trace
echo w > /proc/sysrq-trigger  # Show blocked tasks
```

---

### 🟡 Intermediate Questions

#### Q3: Explain signals in Linux. How would you handle SIGTERM gracefully?

**Basic Answer:**
Signals are software interrupts sent to processes. SIGTERM requests graceful termination, SIGKILL forces immediate termination, SIGHUP traditionally reloads configuration.

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    COMMON SIGNALS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Signal    Number  Default    Description                       │
│  ──────────────────────────────────────────────────────────────  │
│  SIGHUP      1     Term       Terminal hangup / config reload   │
│  SIGINT      2     Term       Interrupt (Ctrl+C)                │
│  SIGQUIT     3     Core       Quit with core dump (Ctrl+\)      │
│  SIGKILL     9     Term       Force kill (cannot be caught)     │
│  SIGSEGV    11     Core       Segmentation fault                │
│  SIGTERM    15     Term       Graceful termination              │
│  SIGSTOP    19     Stop       Pause (cannot be caught)          │
│  SIGCONT    18     Cont       Resume paused process             │
│  SIGUSR1    10     Term       User-defined signal 1             │
│  SIGUSR2    12     Term       User-defined signal 2             │
│                                                                  │
│  Graceful Shutdown Pattern:                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  1. SIGTERM received                                     │    │
│  │     │                                                    │    │
│  │     ▼                                                    │    │
│  │  2. Stop accepting new requests                          │    │
│  │     │                                                    │    │
│  │     ▼                                                    │    │
│  │  3. Complete in-flight requests (with timeout)           │    │
│  │     │                                                    │    │
│  │     ▼                                                    │    │
│  │  4. Close connections and flush buffers                  │    │
│  │     │                                                    │    │
│  │     ▼                                                    │    │
│  │  5. Exit cleanly (exit code 0)                           │    │
│  │                                                          │    │
│  │  If timeout expires:                                     │    │
│  │  6. SIGKILL from orchestrator → immediate termination    │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

Python signal handling:
```python
import signal
import sys
import time

def graceful_shutdown(signum, frame):
    print(f"Received signal {signum}, shutting down gracefully...")
    # Stop accepting new requests
    server.shutdown()
    # Wait for in-flight requests
    server.wait_for_pending(timeout=30)
    # Cleanup
    server.cleanup()
    sys.exit(0)

# Register handlers
signal.signal(signal.SIGTERM, graceful_shutdown)
signal.signal(signal.SIGINT, graceful_shutdown)

# Run server
server.run()
```

---

## Memory Management

### 🟡 Intermediate Questions

#### Q4: Explain Linux memory management and what happens during OOM.

**Basic Answer:**
Linux uses virtual memory with pages, manages physical RAM plus swap, and when memory is exhausted, the OOM killer terminates processes to free memory.

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    MEMORY MANAGEMENT                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  MEMORY LAYOUT:                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  Total Memory = Physical RAM + Swap                      │    │
│  │                                                          │    │
│  │  RAM Usage:                                              │    │
│  │  ┌──────────────────────────────────────────────┐        │    │
│  │  │ Used │ Buffers │ Cache │ Free (Available)   │        │    │
│  │  └──────────────────────────────────────────────┘        │    │
│  │                                                          │    │
│  │  • Used: Memory used by applications                     │    │
│  │  • Buffers: Block device I/O cache                       │    │
│  │  • Cache: Page cache (file contents)                     │    │
│  │  • Free: Completely unused                               │    │
│  │  • Available: Free + reclaimable cache/buffers           │    │
│  │                                                          │    │
│  │  free -h output interpretation:                          │    │
│  │               total    used    free   shared  buff/cache │    │
│  │  Mem:          32G     10G     2G     500M       20G     │    │
│  │  Swap:          8G      0B     8G                        │    │
│  │                                                          │    │
│  │  Available memory ≈ free + buff/cache (reclaimable)      │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  OOM KILLER:                                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  When triggered:                                         │    │
│  │  • System runs out of memory + swap                      │    │
│  │  • Cannot reclaim cache pages                            │    │
│  │                                                          │    │
│  │  Selection criteria (oom_score):                         │    │
│  │  • Based on memory usage                                 │    │
│  │  • Adjusted by oom_score_adj (-1000 to +1000)            │    │
│  │  • Higher score = more likely to be killed               │    │
│  │                                                          │    │
│  │  /proc/<pid>/oom_score        # Current score            │    │
│  │  /proc/<pid>/oom_score_adj    # Adjustment (-1000=never) │    │
│  │                                                          │    │
│  │  Protect critical processes:                             │    │
│  │  echo -1000 > /proc/<pid>/oom_score_adj                  │    │
│  │                                                          │    │
│  │  View OOM events:                                        │    │
│  │  dmesg | grep -i "out of memory"                         │    │
│  │  journalctl -k | grep -i oom                             │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Expert Answer:**

Memory-related tuning:
```bash
# Key sysctls
vm.swappiness = 10              # Prefer RAM over swap (0-100)
vm.vfs_cache_pressure = 50      # Cache pressure (higher = free cache faster)
vm.dirty_ratio = 20             # % of RAM for dirty pages before sync
vm.dirty_background_ratio = 10  # Start background writeback
vm.overcommit_memory = 0        # 0=heuristic, 1=always, 2=never
vm.overcommit_ratio = 50        # % of RAM that can be overcommitted

# Check memory pressure (cgroups v2)
cat /sys/fs/cgroup/memory.pressure
# some avg10=5.00 avg60=2.00 avg300=1.50 total=123456

# Process memory breakdown
cat /proc/<pid>/smaps_rollup
# Or detailed: cat /proc/<pid>/smaps
```

---

## Networking

### 🟡 Intermediate Questions

#### Q5: Explain the Linux networking stack and how to troubleshoot network issues.

**Basic Answer:**
Linux networking follows the OSI model with network interfaces, IP addresses, routing tables, and iptables for firewalling. Use tools like ip, ss, tcpdump for troubleshooting.

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    LINUX NETWORK STACK                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Application Layer                                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Application (nginx, curl)                               │    │
│  │          │                                               │    │
│  │          ▼ socket()                                      │    │
│  │  ┌─────────────────────────────────────────────────┐     │    │
│  │  │  Socket Layer (struct socket)                    │     │    │
│  │  │  - bind(), listen(), connect(), send(), recv()   │     │    │
│  │  └─────────────────────────────────────────────────┘     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            │                                     │
│                            ▼                                     │
│  Transport Layer (TCP/UDP)                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  • Connection management (TCP)                           │    │
│  │  • Flow control, congestion control                      │    │
│  │  • Port multiplexing                                     │    │
│  │  • Checksum validation                                   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            │                                     │
│                            ▼                                     │
│  Network Layer (IP)                                             │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  • Routing decisions                                     │    │
│  │  • Fragmentation                                         │    │
│  │  • IP address handling                                   │    │
│  │  • netfilter hooks (iptables/nftables)                   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            │                                     │
│                            ▼                                     │
│  Link Layer (Ethernet)                                          │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  • ARP resolution                                        │    │
│  │  • Frame handling                                        │    │
│  │  • Network device drivers                                │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            │                                     │
│                            ▼                                     │
│  Physical (NIC)                                                 │
│                                                                  │
│  NETFILTER HOOKS (iptables/nftables):                           │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  Incoming:                                               │    │
│  │  ──▶ PREROUTING ──▶ [Routing] ──▶ INPUT ──▶ Local       │    │
│  │                        │                                 │    │
│  │                        ▼                                 │    │
│  │                    FORWARD ──▶                           │    │
│  │                        │                                 │    │
│  │  Outgoing:             ▼                                 │    │
│  │  Local ──▶ OUTPUT ──▶ POSTROUTING ──▶                    │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

Network troubleshooting commands:
```bash
# Interface and addressing
ip addr show                    # Interface IPs
ip link show                    # Interface status
ip route show                   # Routing table
ip neigh show                   # ARP table

# Connections and ports
ss -tlnp                        # TCP listening ports
ss -tnp state established       # Established connections
ss -s                           # Socket statistics

# DNS
dig +short example.com          # DNS lookup
cat /etc/resolv.conf            # DNS servers

# Connectivity testing
ping -c 3 8.8.8.8               # ICMP test
traceroute example.com          # Path tracing
mtr example.com                 # Combined ping/traceroute
curl -v https://example.com     # HTTP test

# Packet capture
tcpdump -i eth0 port 80         # Capture HTTP traffic
tcpdump -i any -w capture.pcap  # Save to file

# Firewall
iptables -L -n -v               # List rules
nft list ruleset                # nftables rules

# Network performance
iperf3 -s                       # Server mode
iperf3 -c server-ip             # Client test
```

---

## Performance Tuning

### 🔴 Advanced Questions

#### Q6: How would you investigate and resolve high CPU/memory/IO issues?

**Basic Answer:**
Use top/htop for CPU, free/vmstat for memory, iostat/iotop for disk I/O. Identify the resource bottleneck, find the responsible process, and either optimize or add resources.

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    PERFORMANCE INVESTIGATION                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  METHODOLOGY: USE (Utilization, Saturation, Errors)             │
│                                                                  │
│  1. CPU ANALYSIS                                                │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  # Utilization                                           │    │
│  │  mpstat -P ALL 1             # Per-CPU usage             │    │
│  │  sar -u 1 10                 # Historical CPU            │    │
│  │                                                          │    │
│  │  # Saturation (run queue)                                │    │
│  │  vmstat 1                    # r column = run queue      │    │
│  │  sar -q 1                    # Load average              │    │
│  │                                                          │    │
│  │  # Identify CPU consumers                                │    │
│  │  pidstat -u 1                # Per-process CPU           │    │
│  │  perf top                    # Live CPU profiling        │    │
│  │  perf record -g -p <pid>     # Record for analysis       │    │
│  │  perf report                 # View flame graph          │    │
│  │                                                          │    │
│  │  # CPU types:                                            │    │
│  │  %user   - User space                                    │    │
│  │  %system - Kernel space                                  │    │
│  │  %iowait - Waiting for I/O (CPU is idle)                │    │
│  │  %steal  - VM waiting for hypervisor                     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  2. MEMORY ANALYSIS                                             │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  # Utilization                                           │    │
│  │  free -h                     # Memory overview           │    │
│  │  vmstat -s                   # Memory statistics         │    │
│  │                                                          │    │
│  │  # Saturation (swapping)                                 │    │
│  │  vmstat 1                    # si/so columns (swap in/out)│    │
│  │  sar -B 1                    # Paging statistics         │    │
│  │                                                          │    │
│  │  # Per-process memory                                    │    │
│  │  ps aux --sort=-%mem | head  # Top memory users          │    │
│  │  pmap -x <pid>               # Process memory map        │    │
│  │  cat /proc/<pid>/smaps       # Detailed memory           │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  3. DISK I/O ANALYSIS                                           │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  # Utilization                                           │    │
│  │  iostat -xz 1                # Disk utilization          │    │
│  │                                                          │    │
│  │  # Key metrics:                                          │    │
│  │  %util   - Disk busy percentage                          │    │
│  │  await   - Average wait time (ms)                        │    │
│  │  avgqu-sz - Average queue length                         │    │
│  │  r/s, w/s - IOPS                                         │    │
│  │                                                          │    │
│  │  # Per-process I/O                                       │    │
│  │  iotop -o                    # Show only active I/O      │    │
│  │  pidstat -d 1                # Per-process disk I/O      │    │
│  │                                                          │    │
│  │  # I/O wait investigation                                │    │
│  │  sar -d 1                    # Disk activity             │    │
│  │  cat /proc/diskstats        # Raw statistics            │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  4. NETWORK ANALYSIS                                            │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  # Utilization                                           │    │
│  │  sar -n DEV 1                # Network interface stats   │    │
│  │  nload                       # Real-time bandwidth       │    │
│  │                                                          │    │
│  │  # Saturation (drops, errors)                            │    │
│  │  ip -s link show             # Interface statistics      │    │
│  │  netstat -s                  # Protocol statistics       │    │
│  │                                                          │    │
│  │  # Connection issues                                     │    │
│  │  ss -s                       # Socket summary            │    │
│  │  ss -tnp state time-wait     # TIME_WAIT connections     │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Troubleshooting Scenarios

### Scenario 1: High Load Average but Low CPU Usage

```
┌─────────────────────────────────────────────────────────────────┐
│          HIGH LOAD AVERAGE, LOW CPU TROUBLESHOOTING              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Symptoms:                                                       │
│  - Load average: 15, 12, 10                                     │
│  - CPU: 10% user, 5% system, 80% idle                           │
│                                                                  │
│  Root Cause: Usually I/O wait or D-state processes               │
│                                                                  │
│  Investigation:                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  1. Check for I/O wait                                   │    │
│  │     top                      # Look at %wa               │    │
│  │     vmstat 1                 # wa column                 │    │
│  │                                                          │    │
│  │  2. Find D-state processes (uninterruptible sleep)       │    │
│  │     ps aux | awk '$8 ~ /D/'                              │    │
│  │     while true; do ps -eo state,pid,cmd | grep "^D"; \   │    │
│  │       sleep 1; done                                      │    │
│  │                                                          │    │
│  │  3. Check disk I/O                                       │    │
│  │     iostat -xz 1                                         │    │
│  │     # High await = slow disk                             │    │
│  │     # High %util = disk saturated                        │    │
│  │                                                          │    │
│  │  4. Find I/O heavy processes                             │    │
│  │     iotop -o                                             │    │
│  │                                                          │    │
│  │  5. Check for NFS/network storage issues                 │    │
│  │     df -h                    # Hanging = NFS issue       │    │
│  │     mount | grep nfs                                     │    │
│  │                                                          │    │
│  │  Common causes:                                          │    │
│  │  - Slow/failed disk                                      │    │
│  │  - NFS server unresponsive                               │    │
│  │  - Excessive swapping                                    │    │
│  │  - RAID rebuild in progress                              │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Scenario 2: Cannot SSH to Server

```
┌─────────────────────────────────────────────────────────────────┐
│              SSH CONNECTION TROUBLESHOOTING                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Local connectivity test                                     │
│     ping <server-ip>           # Network reachability           │
│     traceroute <server-ip>     # Path issues                    │
│     nc -vz <server-ip> 22      # Port connectivity              │
│                                                                  │
│  2. If port unreachable:                                        │
│     # On server (via console/other access):                     │
│     systemctl status sshd      # SSH service running?           │
│     ss -tlnp | grep 22         # Listening on port?             │
│     iptables -L -n | grep 22   # Firewall blocking?             │
│                                                                  │
│  3. If connects but hangs:                                      │
│     ssh -v <user>@<server>     # Verbose output                 │
│     # Common issues:                                            │
│     - DNS reverse lookup timeout                                │
│     - PAM module hanging                                        │
│     - UseDNS yes in sshd_config                                 │
│                                                                  │
│  4. Authentication failures:                                    │
│     # Check server logs                                         │
│     journalctl -u sshd -f                                       │
│     tail -f /var/log/secure                                     │
│                                                                  │
│  5. Common fixes:                                               │
│     # Disable DNS lookup (faster connection)                    │
│     echo "UseDNS no" >> /etc/ssh/sshd_config                    │
│                                                                  │
│     # Fix permissions                                           │
│     chmod 700 ~/.ssh                                            │
│     chmod 600 ~/.ssh/authorized_keys                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Command Reference

### Essential Commands for SRE/DevOps

```bash
# Process Management
ps aux                     # All processes
pgrep -f "pattern"         # Find process by pattern
pkill -f "pattern"         # Kill by pattern
kill -TERM <pid>           # Graceful termination
kill -9 <pid>              # Force kill
nohup command &            # Run immune to hangups

# Resource Monitoring
top / htop                 # Interactive process viewer
vmstat 1                   # Virtual memory stats
iostat -xz 1               # Disk I/O stats
mpstat -P ALL 1            # Per-CPU stats
sar -n DEV 1               # Network stats

# Disk Operations
df -h                      # Disk space
du -sh /path/*             # Directory sizes
lsblk                      # Block devices
fdisk -l                   # Partition info
mount | column -t          # Mounted filesystems

# Network
ip addr                    # IP addresses
ip route                   # Routing table
ss -tlnp                   # Listening ports
ss -tnp                    # Active connections
tcpdump -i eth0            # Packet capture

# System Info
uname -a                   # Kernel version
cat /etc/os-release        # OS information
uptime                     # System uptime
dmesg | tail               # Kernel messages
journalctl -xe             # System logs

# File Operations
find / -name "*.log" -size +100M  # Find large files
lsof +D /path              # Open files in directory
lsof -p <pid>              # Files opened by process
```

---

## 📚 Documentation Links

- [Linux Kernel Documentation](https://www.kernel.org/doc/)
- [Red Hat System Administration Guide](https://access.redhat.com/documentation/)
- [Linux Performance](http://www.brendangregg.com/linuxperf.html)
- [The Linux Documentation Project](https://tldp.org/)
- [Arch Linux Wiki](https://wiki.archlinux.org/)

---

**[← Back to Main README](../README.md)** | **[Next: Terraform →](../terraform/README.md)**
