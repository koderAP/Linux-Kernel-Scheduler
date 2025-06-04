
# Gang Scheduling Class for Linux Kernel (ARM64)

## Overview

This repository introduces a custom Linux kernel scheduling class named `SCHED_GANG`, designed specifically for tightly-coupled, parallel workloads. Unlike traditional schedulers that manage threads independently, gang scheduling ensures that all threads belonging to the same "gang" are scheduled to run simultaneously across separate CPU cores. This model is ideal for high-performance computing (HPC) environments where synchronization between threads is critical.

The implementation targets the ARM64 architecture and provides a fully functional kernel patch that integrates with the standard Linux scheduler.

---

## Key Features

* **Custom Scheduling Class**: Adds `SCHED_GANG` between the RT (real-time) and FAIR schedulers in the Linux scheduler hierarchy.
* **Coordinated Parallel Execution**: All threads in a gang execute simultaneously using Inter-Processor Interrupts (IPIs).
* **Per-CPU Runqueues**: Introduces a per-CPU `gang_rq` structure with separate queues for gang leaders and members.
* **System Call Interface**: Provides three new system calls to register, exit, and list gang members.
* **Lock-Step Preemption**: Threads in a gang are collectively preempted and resumed to maintain execution consistency.
* **Supports N ≤ M Threads**: Ensures the number of threads in a gang does not exceed the number of available CPU cores.

---

## Directory Structure

* `kernel/sched/gang.c` – Core scheduling logic.
* `kernel/sched/gang_syscalls.c` – Gang-related syscall handlers.
* `include/linux/sched.h` – Definitions for gang structures and constants.
* `arch/arm64/tools/syscall_*.tbl` – System call table entries for ARM64.
* `include/uapi/linux/sched.h` – Scheduling policy constants.
* `include/asm-generic/vmlinux.lds.h` – Updated linker script to insert the new scheduling class.
* Patch file: `gang_sched.patch`

---

## Getting Started

### 1. Apply the Patch

```bash
patch -p1 < gang_sched.patch
```

### 2. Build and Install the Kernel (ARM64)

```bash
make ARCH=arm64 defconfig
make ARCH=arm64 -j$(nproc)
```

Install the compiled kernel on your target ARM64 system and reboot into it.

---

## System Calls

#### `int sys_register_gang(int pid, int gangid, int exec_time)`

Registers a process into a gang. The first thread becomes the leader. Ensures that the gang size does not exceed the number of CPU cores.

* `pid`: PID of the current process.
* `gangid`: Unique identifier for the gang.
* `exec_time`: Expected execution time in seconds.

#### `int sys_exit_gang(int pid)`

Removes a process from its gang and resets its scheduling policy. If it is the last member, the gang is destroyed.

#### `int sys_list_gang(int gangid, int* pids)`

Populates the provided array with PIDs of all active members in the gang.

---

## Gang Scheduling Mechanism

### Role-Based Execution

* **Leader**: The first task to register; scheduled first.
* **Members**: Other threads in the gang; awakened via IPIs after the leader is scheduled.

### Per-CPU Runqueues

Each CPU has:

* `leader_rq`: Queue of leader tasks.
* `member_rq`: Queue of gang members.

Tasks are enqueued based on their gang role and CPU affinity.

### Synchronization

When a leader is scheduled:

* Sends IPIs to CPUs of all member threads.
* Sets scheduling flags to trigger immediate execution of gang members.

When preempted:

* All gang members are also descheduled.
* Preemption state is tracked and reset in the `put_prev_task` hook.

---

## Example Usage

### User Program

```c
int pid = getpid();
int gangid = 101;
int exec_time = 10;

cpu_set_t mask;
CPU_ZERO(&mask);
CPU_SET(core_id, &mask);
sched_setaffinity(0, sizeof(mask), &mask);

syscall(472, pid, gangid, exec_time);  // register_gang
perform_job(exec_time);
syscall(473, pid);                     // exit_gang
```

### Expected Behavior

* All threads registered with the same gang ID will run simultaneously.
* If a leader is preempted or exits, all member threads are also paused or removed.
* Gang synchronization is enforced by kernel-level logic.

---

## Testing and Validation

* Register threads from different processes with a common gang ID.
* Assign unique CPUs using `sched_setaffinity`.
* Validate synchronized execution via `printk` logs or timing measurements.
* Use `sys_list_gang` to verify PID membership during execution.

---

## Design Highlights

* **Round-Robin Leader Rotation**: Gang leaders are rotated to distribute execution control.
* **IPI-Based Wakeups**: Enables real-time responsiveness and coordination.
* **Safe Exit Handling**: Threads are properly removed from gangs even on unexpected termination.
* **Integration with Scheduler Chain**: Seamlessly inserted into the Linux scheduling pipeline.

---

## Assumptions and Future Work

* Assumes each gang thread is pinned to a unique CPU core.
* Does not enforce per-gang runtime limits or quotas (can be extended).
* Limited to ARM64 architecture in this version.

---

## License

This project is released under the GPL v2 License.

