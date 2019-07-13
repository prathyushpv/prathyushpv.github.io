---
layout: post
title: "Building usefull tools with eBPF: Part 2, Tracing the Locks in Linux Kernel"
tags: [Linux, eBPF]
---

 Linux kernel has different types of locks for synchronization. Spin locks, Semaphores, Futexes are some examples. These locks have different behavior and if they are not used properly, it can create performance degradation. For example, spinlocks have least locking and unlocking times but they waste CPU cycles. Spinlocks are used when the waiting time is known to be small. But if a kernel thread is waiting for a spinlock for a considerable amount of time, CPU time is wasted and system performance will be severely affected.
 
## Lock contentions
 There can be situations in which threads are waiting for locks for long time periods and the majority of the time is spent on waiting rather than doing useful work. This is a lock contention and it should be avoided. Sometimes these contentions are unforeseen during the development of the system. So it's very important to quickly identify lock contentions in systems and fix that.
 
 With the help of eBPF, we can identify contentions in the Linux kernel. In this post, I will explain how to trace all locking operations on read-write locks in the Linux kernel, and see if there is any lock being contented for.
 
## Read-Write in Linux Kernel
 Read-write locks provide simultaneous read access to threads, but when a thread needs to get write access, it gets exclusive access. This type of lock is usually used to protect a data structure which is read by many threads at the same time and modified infrequently. 
 
 In the Linux kernel, there are two types of read-write locks, the one using spinlock and the one using semaphores. We will try to find out the waiting times of kernel threads on read-write locks that use spinlocks.
 
## Locking function of read-write locks
 
 Now we need to figure out which function in the Linux kernel will wait on the read-write locks. For simplicity, we will only consider the read lock waiting times. We can list out all the possible kernel functions that eBPF can trace by using the command
 
``` bash
sudo bpftrace -l
```

We can search for the functions that have read_lock in it by using the command

```bash
 sudo bpftrace -l|grep read_lock
```

This command will give us the output
```bash
tracepoint:kprobes:r__raw_read_lock_bcc_15726
tracepoint:kprobes:p__raw_read_lock_bcc_15726
kprobe:cpus_read_lock
kprobe:usermodehelper_read_lock_wait
kprobe:queued_read_lock_slowpath
kprobe:__srcu_read_lock
kprobe:acpi_ut_acquire_read_lock
kprobe:acpi_ut_release_read_lock
kprobe:device_links_read_lock
kprobe:dax_read_lock
kprobe:_raw_read_lock
kprobe:_raw_read_lock_bh
kprobe:_raw_read_lock_irq
kprobe:_raw_read_lock_irqsave
kprobe:btrfs_read_lock_root_node
kprobe:btrfs_assert_tree_read_locked.part.0
kprobe:btrfs_tree_read_lock
kprobe:btrfs_tree_read_lock_atomic
kprobe:btrfs_try_tree_read_lock
```

In this output, the function we are looking for is *_raw_read_lock*. The actual function for acquiring a read lock in the kernel is read_lock() function, but that function internally calls *_raw_read_lock*. So it is sufficient to trace the function *_raw_read_lock*.

## Tracing a locking event
 
 For finding the waiting time of a thread for a particular read lock acquisition, we have to figure out how much time that thread spent on *_raw_read_lock()* function call. For this when a thread enters this function, we have to keep track of that time and when the thread returns from that function, we have to record that time also. Then we can find the waiting time as 

*waiting time = function return timestamp - function entry timestamp*

So we need to store the timestamp when a thread enters *_raw_read_lock()* function for a particular read-write lock. For storing this, we can use the [BPF_HASH](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#2-bpf_hash) data structure provided by eBPF. It is basically a hash table and we can access items using a key.

We need to store the timestamp for a particular locking event. So the key we use should be unique for a lock acquire function call. We cannot use the lock address as the key because many threads may try to acquire the same lock. We cannot use the thread id of the thread, because the same thread may acquire different locks. The perfect candidate for the key is the combination of both lock virtual address and the thread id. We can use c structs as key. The key we are going to use is,
```c
struct key_t {
    u64 pid;
    raw_rwlock_t* lock;
};
```

Along with the timestamp, we can store some additional information, like the PID, command name, etc so that can identify the process which is waiting for the lock.

```c
struct data_t {
    u32 pid;
    u32 tid;
    u64 ts;
    char comm[TASK_COMM_LEN];
    u64 lock;
    u64 lock_time;
    u64 present_time;
    u64 diff;
    u64 stack_id;
    u32 lock_count;
};
```

The different fields in the struct are:
1. pid: PID of the process
2. tid: TID of the thread
3. ts: The time at which the thread enters lock acquire
4. comm: Name of the process
5. lock: Virtual Address of the lock
6. lock_time: Time spent waiting on the lock
7. present_time: The present timestamp
8. diff: The difference between entry time and return time of the lock acquire function
9. stack_id: Used to store the stack trace of the lock acquire function call
10. lock_count: The number of times this lock was acquired

Also, we have to declare the data structures for storing this with the following code
```c
BPF_STACK_TRACE(stack_traces, 102400);
BPF_PERF_OUTPUT(output);
BPF_HASH(lock_events, struct key_t, struct data_t, 102400);
```
* BPF_STACK_TRACE will declare a data structure [BPF_STACK_TRACE](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#5-bpf_stack_trace) for saving the stack traces
* BPF_PERF_OUTPUT will create a [perf ring buffer](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#2-bpf_perf_output) to stream the info to userspace
* BPF_HASH will create the hashmap [BPF_HASH](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#2-bpf_hash) to save struct data_t. It uses key_t as key.
## Code to collect data from kernel

Now we have the data structure to collect information from the kernel. We have to run code in the eBPF virtual machine whenever any thread in the machine enters or leaves *_raw_read_lock()*. The code to be executed when any thread enters the *_raw_read_lock()* is :
```c
int lock(struct pt_regs *ctx, raw_spinlock_t *lock) {

    u32 current_pid = bpf_get_current_pid_tgid();
        
    struct data_t data = {};
    struct key_t key = {bpf_get_current_pid_tgid(), lock};
    struct data_t *data_ptr;
    data_ptr = lock_events.lookup(&key);
    if(data_ptr)
    {
        data_ptr->ts = bpf_ktime_get_ns();
        data_ptr->lock_count += 1;
        data_ptr->stack_id = stack_traces.get_stackid(ctx, BPF_F_REUSE_STACKID);
    }
    else
    {
        data.pid = bpf_get_current_pid_tgid();
        data.tid = bpf_get_current_pid_tgid() >> 32;
        data.ts = bpf_ktime_get_ns();
        bpf_get_current_comm(&data.comm, sizeof(data.comm));
        data.lock = (u64)lock;
        data.lock_count = 1;
        data.stack_id = stack_traces.get_stackid(ctx, BPF_F_REUSE_STACKID);
        lock_events.insert(&key, &data);
    }
    return 0;
}
```

This function simply collects the information we need.
* *bpf_get_current_pid_tgid()* function will return the current thread's pid and tid. It will return a 64-bit integer. Most significant 32 bits represent pid and least significant 32 bits represent the thread id.
* *lock_events.lookup(&key)* function checks if the key is already present in the hash table lock_events. If it is present, it returns a pointer to the struct. If not, it returns NULL.
* *bpf_ktime_get_ns()* function return the current time in nanoseconds.
* *stack_traces.get_stackid(ctx, BPF_F_REUSE_STACKID)*  will get the stack trace of the function call and saves its id in stack_traces. We are not using it right now.
* *bpf_get_current_comm()* will fetch the name of the thread

The code to be executed when the function return is,
```c
int release(struct pt_regs *ctx, raw_spinlock_t *lock) {
    u64 present = bpf_ktime_get_ns();
    
    u32 current_pid = bpf_get_current_pid_tgid();
        
    struct data_t *data;
    struct key_t key = {bpf_get_current_pid_tgid(), lock};
    data = lock_events.lookup(&key);
    if(data)
    {
        data->lock_time += (present - data->ts);
        data->present_time = present;
        data->diff = present - data->ts;
        data->type = 1;
        output.perf_submit(ctx, data, sizeof(struct data_t));
    }
    return 0;
}
```

This function retrieves the structure saved in the hashmap when the thread enters the lock acquire function. Calculate the waiting time and then it will stream the information to the [perf ring buffer](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#2-bpf_perf_output) *output*.

The above code has to be compiled and inserted to the eBPF virtual machine. For this, we can use the bcc python library. If we save the above code in a python string prog, the python code will be:

```python
from __future__ import print_function
from bcc import BPF
import datetime


def print_event(cpu, data, size):
    global start
    event = b['output'].event(data)
    # for lock in locks:
    if start == 0:
        start = event.ts
    time_s = (float(event.ts - start)) / 1000000000
    print("%-18.9f %-16s %-6d %-6d %-6d %-6f %-15f %-6d" % (time_s, event.comm, event.pid, event.tid,
                                                                 event.lock,
                                                                 (float(event.present_time - start)) / 1000000000,
                                                                 event.lock_time, event.diff))

b = BPF(text=prog)
b.attach_kprobe(event="_raw_read_lock", fn_name="lock")
b.attach_kretprobe(event="_raw_read_lock", fn_name="release")

print("Tracing locks for %d seconds" % 10)

# process event
start = 0

# loop with callback to print_event
b['output'].open_perf_buffer(print_event)
start_time = datetime.datetime.now()

while 1:
    b.perf_buffer_poll()
    time_elapsed = datetime.datetime.now() - start_time
    if time_elapsed.seconds > 5:
        exit()
```

* b = BPF(text=prog): This will create an object of BPF. The code to be executed has to be passed as a string to the constructor
* b.attach_kprobe(event="_raw_read_lock", fn_name="lock"): This line will attach our function lock to the start of kernel function _raw_read_lock.
*  b.attach_kretprobe(event="_raw_read_lock", fn_name="release"): This line will attach our function release to the end of kernel function _raw_read_lock.
*  b['output'].open_perf_buffer(print_event): This line will tell the library to execute the function print_event when a new element is added to the perf ring buffer.
*  The wile loop will poll the perf buffer for 10 seconds.
*  print_event function will extract details from the ring buffer and display it on the terminal. We will get the c structure data_t as a python object. We can access the member of struct using the dot operator.

When everything is put together, the final script is: 

This script will track individual wait times of all locking operations and displays it on the terminal. But from this information, we will not be able to identify the locks that are contented for easily. For this, we need to process the data we are extracting and should display it in a way that is more useful. 

The next part will explain how to create an HTML report from this information have.
