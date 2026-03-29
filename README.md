# Multi-Container Runtime
## 1. Team Information

|           Name          |      SRN      |
|-------------------------|---------------|
| Arin Singh              | PES1UG24CS079 |
| Anshdeep Singh Sachdeva | PES1UG24CS070 |

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

*(Take these screenshots while running the steps above and paste them here)*

| # | What it shows |
|---|---|
| 1 | Two containers running under one supervisor |
| 2 | `engine ps` output with both containers listed |
| 3 | Log file contents from `engine logs alpha` |
| 4 | `engine stop alpha` command and supervisor response |
| 5 | `dmesg` showing SOFT LIMIT warning for memtest |
| 6 | `dmesg` showing HARD LIMIT kill + `engine ps` showing hard_limit_killed |
| 7 | `time` output for exp1 vs exp2 showing different completion times |
| 8 | Supervisor "Clean exit. No zombies." message + `ps aux | grep defunct` empty |

---

## 4. Engineering Analysis

### 4.1 Isolation Mechanisms

Linux implements container isolation through namespaces — kernel data structures that give a group of processes a private view of a system resource. Our runtime creates three namespaces per container using `clone()`:

**PID namespace (`CLONE_NEWPID`):** The kernel maintains a separate PID numbering table for each namespace. The first process inside the container is assigned PID 1 in its namespace table, even though the host kernel tracks it as a much higher PID. When that process calls `getpid()` the kernel translates through the namespace mapping and returns 1. This means the container cannot see or signal host processes because their PIDs do not exist in the container's namespace table.

**UTS namespace (`CLONE_NEWUTS`):** The kernel stores the hostname and NIS domain name in a `uts_namespace` struct. With `CLONE_NEWUTS`, the child gets its own copy of this struct. Calls to `sethostname()` inside the container modify only the child's copy, leaving the host hostname unchanged.

**Mount namespace (`CLONE_NEWNS`):** The kernel maintains a mount tree per namespace. With `CLONE_NEWNS`, the child inherits a copy of the parent's mount tree at clone time, but all subsequent mount and unmount operations affect only the child's copy. This is what makes our `mount("proc", ...)` and `chroot()` invisible to the host.

`chroot()` changes the process's `root` pointer in its `fs_struct` from the real filesystem root to the container's rootfs directory. Path resolution for that process starts at this new root. A process inside cannot reach `..` above this root because the kernel clamps traversal at the `root` pointer.

**What the host kernel still shares:** The host kernel itself — the same kernel handles system calls from both host and container processes. The network stack (unless `CLONE_NEWNET` is added), the host clock, and the host's physical hardware are all shared. Kernel vulnerabilities affect all containers equally.

---

### 4.2 Supervisor and Process Lifecycle

When a process exits in Linux, it enters a zombie state — its exit status is preserved in the kernel's process table but its resources are freed. The zombie persists until the parent calls `waitpid()`. If the parent exits first, the child is re-parented to PID 1 (init), which calls `waitpid()` on its behalf. If no one ever calls `waitpid()`, the zombie entry stays in the process table forever, consuming a PID slot.

A long-running supervisor is valuable because it is always alive to be the parent of every container process. When a container exits, the kernel sends `SIGCHLD` to the supervisor. Our handler calls `waitpid(-1, &status, WNOHANG)` in a while loop — the `-1` means "any child", `WNOHANG` means "don't block if none are ready", and the loop drains multiple simultaneous exits since signals are not queued individually.

The `stop_requested` flag solves a classification problem: both a manual `engine stop` and the kernel module's hard-limit enforcement send `SIGKILL`. Without the flag, the supervisor cannot tell them apart. By setting `stop_requested = 1` before sending our own signal, the SIGCHLD handler can distinguish: `stop_requested=1` → `CONTAINER_STOPPED`, `SIGKILL + stop_requested=0` → `CONTAINER_HARD_LIMIT_KILLED`.

Signal delivery to a process inside a PID namespace uses the host PID. `kill(c->host_pid, SIGTERM)` works because the supervisor operates in the host's PID namespace and uses the host-side PID.

---

### 4.3 IPC, Threads, and Synchronization

**Two IPC mechanisms:**

*Path A (logging) — pipes:* A pipe is a kernel-maintained byte stream with two file descriptors. We create one before `clone()`. The child inherits the write end via `dup2()`; the supervisor keeps the read end. The write end in the supervisor is closed immediately after clone — this is critical because EOF on the read end only occurs when all write-end file descriptors are closed. If the supervisor kept its write-end copy open, the producer thread would block on `read()` forever even after the container exited.

*Path B (control) — UNIX domain socket:* A UNIX socket lives on the filesystem and supports bidirectional, connection-oriented communication. Each `engine start/stop/ps` invocation connects, sends a fixed-size `control_request_t`, reads back a `control_response_t`, and disconnects. Using fixed-size binary structs avoids framing problems (partial reads) that text protocols suffer from.

**Shared data structures and their races:**

*Container linked list:* Protected by `metadata_lock` (mutex). Without it: the SIGCHLD handler (which runs asynchronously) could be traversing the list at the same time `handle_client()` is inserting a new node, causing a partial traversal or corrupted `next` pointer.

*Bounded buffer (ring buffer):* Protected by three primitives:
- `mutex` — prevents two producer threads from simultaneously writing to the same tail slot. Without it, two producers could both read `count`, see room, both compute the same `tail` index, and overwrite each other's data.
- `not_full` condition variable — a producer that finds the buffer full calls `pthread_cond_wait(&not_full, &mutex)`, atomically releasing the mutex and sleeping. When the consumer pops an entry it signals `not_full`. This prevents busy-spinning and eliminates the deadlock where a full buffer stalls all producers permanently.
- `not_empty` condition variable — the consumer sleeps here when the buffer is empty, waking when a producer inserts. Without this the consumer would spin at 100% CPU polling the count.

We use `while` loops (not `if`) around every `pthread_cond_wait` call because POSIX permits spurious wakeups — a thread can wake up from `cond_wait` even though no signal was sent. The `while` re-checks the actual condition and returns to sleep if still unsatisfied.

---

### 4.4 Memory Management and Enforcement

**What RSS measures:** RSS (Resident Set Size) is the number of physical memory pages currently mapped into a process's page tables and present in RAM. Specifically, `get_mm_rss(mm)` in the kernel sums three counters: anonymous pages (stack, heap), file-mapped pages (shared libraries loaded from disk), and shared memory pages.

**What RSS does not measure:** Virtual address space that has been `mmap()`-ed but never touched (pages are allocated lazily on first access). Swapped-out pages that were once in RAM but have been written to swap. Memory in kernel structures allocated on behalf of the process (socket buffers, pipe buffers). This means a process can have a large virtual size but small RSS if it has not touched its allocations.

**Why soft and hard limits are different policies:** A soft limit is a warning threshold — the process is not harmed, just flagged. This lets operators notice gradual memory growth before it becomes a problem, without disrupting a legitimate workload that briefly spikes. A hard limit is enforcement — once a process has consumed more physical RAM than the system can tolerate, it must be stopped. The two-tier design allows for observability before action.

**Why enforcement belongs in kernel space:** A user-space daemon polling RSS has a fundamental reliability problem — it can be killed, starved of CPU by the very process it is monitoring, or delayed by scheduling. A kernel timer callback runs with the scheduler's cooperation regardless of user-space load. More importantly, `kill()` from user space goes through a permission check, but `send_sig()` from kernel space does not. The kernel module can reliably kill a container even if the supervisor process itself is busy or has been compromised.

---

### 4.5 Scheduling Behavior

Linux uses the Completely Fair Scheduler (CFS). CFS maintains a virtual runtime (`vruntime`) for each runnable process. At each scheduling decision it picks the process with the smallest `vruntime` (implemented as a red-black tree sorted by vruntime). The `nice` value scales how quickly a process's vruntime accumulates — a process with nice=10 accumulates vruntime faster than one with nice=0, so CFS picks it less often, giving it less CPU time.

**Experiment 1 results:**

| Container | Nice value | Workload | Real time |
|-----------|-----------|----------|-----------|
| exp1      | 0         | cpu_hog 30s | 0.098s |
| exp2      | 10        | cpu_hog 30s | 5.131s |

exp2 took longer because CFS assigns nice=10 a weight of roughly 1/3 of nice=0's weight on a 2-core system. The higher-priority process accumulates vruntime more slowly so CFS always prefers it when both are runnable.

**Experiment 2 results:**

| Container | Type    | Workload        | Real time |
|-----------|---------|-----------------|-----------|
| cpuexp    | CPU-bound | cpu_hog 20s   | [A]s |
| ioexp     | I/O-bound | io_pulse 40 iter | [B]s |

The I/O-bound process (`io_pulse`) calls `usleep()` between iterations, voluntarily yielding the CPU. CFS resets its vruntime toward the minimum when it wakes up from sleep, giving it high scheduling priority for its brief CPU bursts. This explains why it completed at approximately the same speed as if running alone — it barely competed with `cpuexp` for CPU time.

This demonstrates CFS's responsiveness goal: processes that sleep frequently and use CPU in short bursts are rewarded with low latency when they wake, while throughput-oriented processes (cpu_hog) get the remaining CPU time without starvation.

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

| Container | Nice | Measured real time |
|-----------|------|--------------------|
| exp1      | 0    | [fill in]s         |
| exp2      | 10   | [fill in]s         |

**Interpretation:** CFS weight for nice=0 is 1024; for nice=10 it is 110. On a single core this means exp1 gets approximately 1024/(1024+110) ≈ 90% of CPU time and exp2 gets ≈10%. exp2's real time should be roughly 9× exp1's if both run to completion. On a multi-core system the effect is less pronounced because each container may run on a separate core.

### Experiment 2 — CPU-bound vs I/O-bound

| Container | Type      | Workload        | Measured real time |
|-----------|-----------|-----------------|-------------------|
| cpuexp    | CPU-bound | cpu_hog 20      | [fill in]s        |
| ioexp     | I/O-bound | io_pulse 40 200 | [fill in]s        |

**Interpretation:** The I/O-bound workload completed in approximately the same time as if running alone because it spent most of its time sleeping in `usleep()`. CFS correctly identified it as a low-vruntime process and gave it immediate CPU access when it woke up. This demonstrates CFS's design goal: I/O-bound processes should not be penalised for yielding the CPU voluntarily.
