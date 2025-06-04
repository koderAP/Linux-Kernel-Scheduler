# Gang Scheduling Class for Linux Kernel (ARM64)

## Overview

This repository introduces a custom **gang scheduling policy** into the Linux kernel, implemented as a standalone scheduling class named `SCHED_GANG`. Gang scheduling is an efficient mechanism for synchronizing the execution of interdependent threads across multiple CPUs, making it ideal for high-performance computing (HPC) environments.

Unlike traditional schedulers (e.g., CFS or RT) that operate independently per task or CPU, `SCHED_GANG` ensures that all threads belonging to the same gang are executed **simultaneously and in a lockstep fashion**, significantly reducing communication delays and synchronization overheads.

---

## Key Features

* **New Scheduling Class (`SCHED_GANG`)**
  Inserted between `SCHED_RT` and `SCHED_FAIR` to prioritize coordinated HPC execution without preempting real-time tasks.

* **Custom Per-CPU Runqueues**
  Each CPU maintains a `gang_rq` structure that isolates scheduling metadata and maintains distinct queues for leader and member tasks.

* **Thread Coordination via IPI**
  Leader threads broadcast Inter-Processor Interrupts (IPIs) to signal synchronized execution of all member threads.

* **Three Custom Syscalls**

  * `register_gang(pid, gangid, exec_time)`
  * `exit_gang(pid)`
  * `list_gang(gangid, *pids)`

* **Safe Task Lifecycle Handling**
  Kprobe-based instrumentation on `do_exit()` ensures graceful gang cleanup during unexpected terminations.

* **Patch Format**
  Delivered as a patch file for clean application on top of the vanilla Linux kernel v6.13.4.

---

## System Architecture

### 1. Gang Runqueue (`gang_rq`)

Each CPU has its own `gang_rq`, comprising:

* `leader_rq`: Queue of leader tasks (one per gang).
* `member_rq`: Queue of member threads.
* `curr`, `next`, `prev`: State trackers to enforce gang synchronization.
* `counter`: Auxiliary tracking for scheduling transitions.

### 2. Execution Flow

* The **leader** registers the gang and is scheduled first.
* When selected, the leader uses `wake_up_gang()` to **broadcast IPIs**.
* All **members** begin executing only after receiving the IPI.
* On **preemption or exit** of the leader, all members are simultaneously descheduled.

---

## File Modifications

### Core Files Modified

* `kernel/sched/core.c`: Integration with the scheduler class hierarchy.
* `kernel/sched/sched.h`: Runqueue extensions.
* `include/uapi/linux/sched.h`: New scheduling constant.
* `include/asm-generic/vmlinux.lds.h`: Ensures correct linkage order.

### New Files Added

* `gang.c`, `gang.h`: Core gang scheduling logic and data structures.
* `gang_syscalls.c`: Definitions and handlers for syscall interfaces.

---

## Usage

### 1. Applying the Patch

```bash
patch -p1 < gang_sched.patch
```

### 2. Building the Kernel

Follow standard cross-compilation and deployment steps for the ARM64 architecture.

### 3. Affinitizing Threads

Use `sched_setaffinity()` to pin each thread to a dedicated CPU core:

```c
cpu_set_t cpuset;
CPU_ZERO(&cpuset);
CPU_SET(core_id, &cpuset);
sched_setaffinity(pid, sizeof(cpuset), &cpuset);
```

### 4. Syscall Interface

#### Register a Thread in a Gang

```c
int sys_register_gang(int pid, int gangid, int exec_time);
```

Registers a thread under a gang. The first thread becomes the **leader**.

#### Exit from a Gang

```c
int sys_exit_gang(int pid);
```

Deregisters a thread. If it is the last member, the gang is deallocated.

#### List Members of a Gang

```c
void sys_list_gang(int gangid, int* pids);
```

Populates an array with the PIDs of threads belonging to the specified gang.

---

## Example Program

```c
int main(int argc, char *argv[]) {
    int exec_time = atoi(argv[1]);
    int gangid = atoi(argv[2]);
    int pid = getpid();

    syscall(SYS_register_gang, pid, gangid, exec_time);
    perform_job(exec_time);
    syscall(SYS_exit_gang, pid);

    return 0;
}
```

---

## Design Considerations

* **Synchronization First**: Leader scheduling always precedes member activation.
* **Runqueue Separation**: Prevents premature or unsynchronized member execution.
* **IPI Coordination**: Ensures low-latency synchronized startup.
* **Safety via Kprobes**: Ensures no stale gang states or memory leaks.

---

## Testing and Validation

* Implemented and tested using the ARM64 Linux kernel on a macOS environment via UTM.
* Stress-tested using dummy HPC workloads simulating multi-threaded applications.
* Verified thread affinity, synchronization accuracy, and correct gang lifecycle behavior.

---

## Limitations and Future Work

* Assumes number of threads ≤ number of CPU cores.
* No per-gang time-quota enforcement (can be extended via `task_tick`).

---

## License

This repository is licensed under the GPL-V2 License. Please see the [LICENSE](./LICENSE) file for details.


