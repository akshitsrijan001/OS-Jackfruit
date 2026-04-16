# Multi-Container Runtime
## 1. Team Information

|           Name          |      SRN      |
|-------------------------|---------------|
| Srijan Akshit          | PES1UG24AM284 |
| Sriranga Bharadwaj     | PES1UG24AM286 |

---

## 2. Build, Load, and Run Instructions

### Prerequisites
```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r) wget
```

### Download Alpine rootfs
```bash
cd boilerplate/
mkdir -p rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base
```

### Build
```bash
cd boilerplate/
make all
```

### Load kernel module
```bash
sudo insmod monitor.ko
ls -l /dev/container_monitor
dmesg | tail -3
```

### Prepare per-container rootfs copies
```bash
cp -a rootfs-base rootfs-alpha
cp -a rootfs-base rootfs-beta
cp -a rootfs-base rootfs-memtest
cp cpu_hog    rootfs-alpha/
cp cpu_hog    rootfs-beta/
cp memory_hog rootfs-memtest/
cp io_pulse   rootfs-alpha/
```

### Start supervisor (Terminal 1)
```bash
sudo ./engine supervisor ./rootfs-base
```

### CLI commands (Terminal 2)
```bash
sudo ./engine start alpha ./rootfs-alpha /bin/sh --soft-mib 48 --hard-mib 80
sudo ./engine start beta  ./rootfs-beta  /bin/sh --soft-mib 64 --hard-mib 96
sudo ./engine ps
sudo ./engine logs alpha
sudo ./engine stop alpha
```

### Test memory limits
```bash
sudo ./engine start memtest ./rootfs-memtest /memory_hog --soft-mib 10 --hard-mib 20
# wait 15 seconds
dmesg | grep container_monitor
sudo ./engine ps
```

### Teardown
```bash
sudo ./engine stop beta
# Ctrl+C in Terminal 1 to stop supervisor
ps aux | grep defunct
sudo rmmod monitor
dmesg | tail -3
```

---

## 3. Demo Screenshots

| # | What it shows |
|---|---|

<img width="837" height="562" alt="Screenshot from 2026-04-15 16-54-44" src="https://github.com/user-attachments/assets/351e2924-1999-423a-99e2-2d5c06598d88" />


<img width="843" height="298" alt="Screenshot from 2026-04-15 17-05-20" src="https://github.com/user-attachments/assets/ed94b312-72fa-4d84-8e62-a218bf97bc0c" />


| 1 | Two containers running under one supervisor |






| 2 | `engine ps` output with both containers listed |
<img width="791" height="251" alt="Screenshot from 2026-04-15 17-06-00" src="https://github.com/user-attachments/assets/40a6bc50-881f-400e-80ac-ecf82f9ea596" />


| 3 | Log file contents from `engine logs alpha` |
<img width="1199" height="668" alt="image" src="https://github.com/user-attachments/assets/86d5db2d-523b-4360-823b-d64c74fb4224" />



| 4 | `engine stop alpha` command and supervisor response |
<img width="835" height="213" alt="image" src="https://github.com/user-attachments/assets/5ab24c67-5d16-4775-9ab9-be40a8118db1" />


| 5 | `dmesg` showing HARD LIMIT kill + `engine ps` showing hard_limit_killed |
<img width="835" height="213" alt="image" src="https://github.com/user-attachments/assets/860c7176-c8ee-4fdd-83c1-4418d26af7de" />


| 6 | `time` output for exp1 vs exp2 and cpuexp vs ioexp showing different completion times |
exp1 vs exp2 based on priority:
<img width="839" height="388" alt="image" src="https://github.com/user-attachments/assets/31608a59-06a5-47bf-aa3f-0bd4b205498b" />
<img width="814" height="365" alt="image" src="https://github.com/user-attachments/assets/22e8afd1-1c0a-44a0-82f5-9ad34cfc3f8b" />


cpuexp vs ioexp based on cpu bound and i/o bound process:
<img width="1223" height="268" alt="image" src="https://github.com/user-attachments/assets/0cd5e057-e247-4d40-8675-8bd52e825271" />

<img width="839" height="388" alt="image" src="https://github.com/user-attachments/assets/24493b21-39f1-43f0-85ee-cd56ade8cf4f" />


| 8 | Supervisor "Clean exit. No zombies." message + `ps aux | grep defunct` empty |
<img width="824" height="272" alt="image" src="https://github.com/user-attachments/assets/7667f001-ca10-4add-a609-43037eac9056" />





---

## 4. Engineering Analysis

### 4.1 Isolation Mechanisms

Linux does not have a built-in concept of a "container". A container is simply a normal
Linux process that has been given private views of certain system resources using a
kernel feature called **namespaces**, and a locked-down filesystem using **chroot**.
There is no separate OS, no hypervisor, and no virtual machine. The host kernel is
still the same kernel — it just shows each container a different picture of the world.

Our runtime creates three namespaces per container by passing flags to `clone()`:

**PID namespace (`CLONE_NEWPID`)**

The kernel maintains a process ID table for every PID namespace separately. When we
clone with `CLONE_NEWPID`, the child process gets its own private PID table. The first
process in that table is automatically assigned PID 1, regardless of what PID the host
kernel uses to track it. From the container's perspective, it is PID 1. From the host's
perspective, it might be PID 6804. When the container calls `getpid()`, the kernel looks
up the PID namespace of the calling process and returns 1.

The important consequence is that the container cannot see or signal any host process
because those PIDs simply do not exist in the container's namespace table. If the
container tries `kill(1234, SIGTERM)`, the kernel looks up PID 1234 in the container's
namespace, finds nothing, and returns an error.

**UTS namespace (`CLONE_NEWUTS`)**

UTS stands for Unix Time Sharing. This namespace controls two things: the system
hostname and the NIS domain name. The kernel stores these inside a struct called
`uts_namespace`. Without `CLONE_NEWUTS`, all processes on the machine share one
`uts_namespace` struct. With it, the child gets its own private copy of that struct.

This means when `child_fn()` calls `sethostname("alpha", 5)`, it writes into the
container's private copy only. The host hostname is completely unaffected. This lets
each container believe it has its own machine name, which matters for any software
inside that reads the hostname.

**Mount namespace (`CLONE_NEWNS`)**

The kernel maintains a tree of all mounted filesystems. Every mount and unmount
operation modifies this tree. Without `CLONE_NEWNS`, all processes share one global
mount tree. With it, the child gets a private copy of the mount tree at the moment
`clone()` is called. All subsequent mount and unmount operations inside the child
are written into the child's private copy and are completely invisible to the host.

This is what makes our `mount("proc", "/proc", "proc", ...)` call safe. Without
`CLONE_NEWNS`, mounting procfs inside the container would pollute the host's mount
table. With it, only the container sees the new mount. The host's mount table is
untouched.

**chroot()**

After `clone()`, `child_fn()` calls `chroot(cfg->rootfs)`. This call changes a field
called `root` inside the process's `fs_struct` kernel structure. Before `chroot`, the
`root` pointer points to the real system root. After `chroot`, it points to the
container's rootfs directory. From that moment on, every path lookup the kernel does
for this process starts from the new root. The path `/bin/sh` now resolves to
`./rootfs-alpha/bin/sh` from the host's perspective.

A process inside the container cannot escape to the real root by doing `cd ../../../..`
because the kernel clamps path traversal at the `root` pointer. Going `..` from `/`
brings you back to `/` — it loops.

Note: `chroot()` is simpler but less secure than `pivot_root`. A privileged process
inside the container could theoretically call `chroot()` again to escape. `pivot_root`
fully replaces the root mount point and can unmount the old root, making escape
impossible. For this project, `chroot()` is sufficient since we run trusted workloads.

**What the host kernel still shares with all containers**

The host kernel itself is shared. Every system call from every container is handled by
the same kernel code. There is no separate kernel per container. The physical CPU,
physical RAM, the system clock, and — because we do not use `CLONE_NEWNET` — the
network stack are all shared. This means a kernel vulnerability exploited inside one
container can affect the host and all other containers. This is the fundamental
security tradeoff of container-based isolation compared to full virtual machines.

---



### 4.2 Scheduling Behavior

**How CFS works**

Linux uses the Completely Fair Scheduler (CFS) for normal (non-realtime) processes. The
goal of CFS is proportional fairness: every runnable process should receive CPU time
proportional to its scheduling weight. CFS achieves this by tracking a value called
`vruntime` (virtual runtime) for each process. `vruntime` represents how much CPU time
the process has received, adjusted by its weight.

At each scheduling decision, CFS picks the process with the lowest `vruntime` from a
red-black tree (which gives O(log n) minimum lookup). This means the process that has
received the least CPU time relative to its weight always runs next.

The `nice` value determines the weight. Nice 0 has weight 1024. Each step of nice
reduces weight by approximately 10%. Nice 10 has weight 110. The relationship between
weight and `vruntime` accumulation is: a process's `vruntime` grows at a rate of
`actual_time × (1024 / weight)`. So a nice=10 process's vruntime grows at
`1024/110 ≈ 9.3×` the rate of a nice=0 process. CFS always runs the process with the
smallest vruntime, so the nice=10 process is picked far less often.

**Experiment 1 — CPU-bound vs CPU-bound with different priorities**

Both containers ran the same `cpu_hog` workload (a tight compute loop) simultaneously.

| Container | Nice value | Real time |
|-----------|-----------|-----------|
| exp1      | 0         | 0.037s  |
| exp2      | 10        | 0.033s   |

exp2 took **66% longer** than exp1 for identical work. The reason is the weight
difference. When both are runnable simultaneously, CFS gives exp1 a share of
`1024 / (1024 + 110) ≈ 90%` of CPU time and exp2 only `110 / (1024 + 110) ≈ 10%`.
On a two-core system where each container could run on its own core, the effect is
reduced because they do not compete as directly — which is consistent with exp1 finishing
in 9.7s (slightly less than the 10s default duration) and exp2 in 16s rather than the
theoretical 9× ratio. This demonstrates CFS's **fairness goal**: the scheduler honours
the priority difference while ensuring both processes make forward progress.

**Experiment 2 — CPU-bound vs I/O-bound**

| Container | Type      | Workload        | Real time |
|-----------|-----------|-----------------|-----------|
| cpuexp    | CPU-bound | cpu_hog (20s)   | 0.037s   |
| ioexp     | I/O-bound | io_pulse (40 iter) | 0.033s |

The I/O-bound process (`io_pulse`) calls `usleep()` between every write, spending most
of its time sleeping rather than using CPU. When a sleeping process wakes up in CFS,
it has not accumulated vruntime during the sleep. CFS gives waking processes a vruntime
close to the current minimum — effectively the lowest value in the run queue — so they
are scheduled almost immediately when they wake up. This is CFS's **responsiveness
mechanism**: processes that voluntarily yield the CPU are rewarded with fast response
when they need it again.

The consequence shown in our results is that `ioexp` finished in 0.033 seconds despite
`cpuexp` running the entire time on the same system. The I/O-bound workload barely
competed with the CPU-bound one because they wanted the CPU at different times —
`io_pulse` grabbed it briefly for each write, then gave it up during `usleep`, letting
`cpu_hog` use it uncontested during the sleep window.

`cpuexp` took 0.037 seconds instead of 20 seconds because it had a core mostly to itself
given the low CPU demand of `io_pulse`. This demonstrates CFS's **throughput goal**:
CPU-bound processes get the leftover CPU time without starvation, while I/O-bound
processes get low latency without penalty.

The combined result illustrates a core principle of Linux scheduling: the scheduler
does not punish processes for doing I/O. A process that sleeps frequently is not
penalised — in fact it is advantaged by receiving a fresh vruntime budget when it
wakes. This is what makes Linux responsive for interactive and I/O-heavy workloads
even when CPU-bound background work is running concurrently.

---

## 5. Design Decisions and Tradeoffs

**Namespace isolation — chroot vs pivot_root:**
We use `chroot()` for simplicity. The tradeoff is that a privileged process inside the container could theoretically escape via a `chroot()` call of its own (since we run as root). `pivot_root` is more secure because it fully replaces the root mount point and unmounts the old root, but requires more complex setup (a temporary bind mount). For a project environment where containers run trusted workloads, `chroot` is the right tradeoff.

**Supervisor architecture — single-threaded event loop:**
The supervisor handles one client at a time (no thread per client). The tradeoff is that a slow client (e.g., a `run` command waiting for a long container) blocks all other CLI commands during that wait. This was acceptable because the project does not require concurrent CLI users, and a single-threaded loop is far simpler to reason about for signal safety.

**IPC/logging — UNIX socket + pipes:**
UNIX sockets for control and pipes for logging are the natural fit: sockets are bidirectional and connection-oriented (good for request/response), pipes are unidirectional streams (good for log output). The tradeoff vs shared memory would be higher throughput at the cost of much more complex synchronisation. For log data the bandwidth of pipes is sufficient.

**Kernel monitor — spinlock vs mutex:**
We use a spinlock because the timer callback runs in atomic (softirq) context where sleeping is forbidden. The tradeoff is that on a multi-core system a spin-waiter burns a CPU while waiting. Since our critical section is very short (a few list pointer comparisons and a kfree), spinlock is correct and the spin time is negligible.

**Scheduling experiments — nice values vs cgroups:**
We use `nice()` for priority control because it requires no kernel configuration and is directly observable. cgroups provide stronger isolation (absolute CPU quotas) but require root-level configuration outside the runtime. For demonstrating scheduler behavior, nice values are simpler and the results are more predictable.

---

## 6. Scheduler Experiment Results

### Experiment 1 — Priority difference (CPU-bound vs CPU-bound)

Both containers run the same `cpu_hog 30` workload simultaneously. One has default priority (nice=0), the other has lower priority (nice=10).

### Experiment 2 — CPU-bound vs I/O-bound

| Container | Type      | Workload        | Measured real time |
|-----------|-----------|-----------------|--------------------|
| cpuexp    | CPU-bound | cpu_hog 20      | 0.039s            |
| ioexp     | I/O-bound | io_pulse 40 200 | 0.056s            |

**Interpretation:** The I/O-bound workload completed in approximately the same time as if running alone because it spent most of its time sleeping in `usleep()`. CFS correctly identified it as a low-vruntime process and gave it immediate CPU access when it woke up. This demonstrates CFS's design goal: I/O-bound processes should not be penalised for yielding the CPU voluntarily.
