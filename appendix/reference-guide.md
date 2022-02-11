# Reference Guide

This chapter is a reference for the Pintos code. The reference guide does not cover all of the code in Pintos, but it does cover those pieces that students most often find troublesome. You may find that you want to read each part of the reference guide as you work on the project where it becomes important.

We recommend using "tags" to follow along with references to function and variable names (see section [F.1 Tags](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_12.html#SEC170)).

## A.1 Loading

This section covers the Pintos loader and basic kernel initialization.

### A.1.1 The Loader

The first part of Pintos that runs is the loader, in threads/loader.S. The PC BIOS loads the loader into memory. The loader, in turn, is responsible for finding the kernel on disk, loading it into memory, and then jumping to its start. It's not important to understand exactly how the loader works, but if you're interested, read on. You should probably read along with the loader's source. You should also understand the basics of the 80x86 architecture as described by chapter 3, "Basic Execution Environment," of \[ [IA32-v1](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_14.html#IA32-v1)].

The PC BIOS loads the loader from the first sector of the first hard disk, called the _master boot record_ (MBR). PC conventions reserve 64 bytes of the MBR for the partition table, and Pintos uses about 128 additional bytes for kernel command-line arguments. This leaves a little over 300 bytes for the loader's own code. This is a severe restriction that means, practically speaking, the loader must be written in assembly language.

The Pintos loader and kernel don't have to be on the same disk, nor does is the kernel required to be in any particular location on a given disk. The loader's first job, then, is to find the kernel by reading the partition table on each hard disk, looking for a bootable partition of the type used for a Pintos kernel.

### A.1.2 Low-Level Kernel Initialization

The loader's last action is to transfer control to the kernel's entry point, which is `start()` in threads/start.S. The job of this code is to switch the CPU from legacy 16-bit "real mode" into the 32-bit "protected mode" used by all modern 80x86 operating systems.

The startup code's first task is actually to obtain the machine's memory size, by asking the BIOS for the PC's memory size. The simplest BIOS function to do this can only detect up to 64 MB of RAM, so that's the practical limit that Pintos can support. The function stores the memory size, in pages, in global variable `init_ram_pages`.

The first part of CPU initialization is to enable the A20 line, that is, the CPU's address line numbered 20. For historical reasons, PCs boot with this address line fixed at 0, which means that attempts to access memory beyond the first 1 MB (2 raised to the 20th power) will fail. Pintos wants to access more memory than this, so we have to enable it.

Next, the loader creates a basic page table. This page table maps the 64 MB at the base of virtual memory (starting at virtual address 0) directly to the identical physical addresses. It also maps the same physical memory starting at virtual address `LOADER_PHYS_BASE`, which defaults to 0xc0000000 (3 GB). The Pintos kernel only wants the latter mapping, but there's a chicken-and-egg problem if we don't include the former: our current virtual address is roughly 0x20000, the location where the loader put us, and we can't jump to 0xc0020000 until we turn on the page table, but if we turn on the page table without jumping there, then we've just pulled the rug out from under ourselves.

After the page table is initialized, we load the CPU's control registers to turn on protected mode and paging, and set up the segment registers. We aren't yet equipped to handle interrupts in protected mode, so we disable interrupts. The final step is to call `main()`.

### A.1.3 High-Level Kernel Initialization

The kernel proper starts with the `main()` function. The `main()` function is written in C, as will be most of the code we encounter in Pintos from here on out.

When `main()` starts, the system is in a pretty raw state. We're in 32-bit protected mode with paging enabled, but hardly anything else is ready. Thus, the `main()` function consists primarily of calls into other Pintos modules' initialization functions. These are usually named `module_init()`, where module is the module's name, module.c is the module's source code, and module.h is the module's header.

The first step in `main()` is to call `bss_init()`, which clears out the kernel's "BSS", which is the traditional name for a segment that should be initialized to all zeros. In most C implementations, whenever you declare a variable outside a function without providing an initializer, that variable goes into the BSS. Because it's all zeros, the BSS isn't stored in the image that the loader brought into memory. We just use `memset()` to zero it out.

Next, `main()` calls `read_command_line()` to break the kernel command line into arguments, then `parse_options()` to read any options at the beginning of the command line. (Actions specified on the command line execute later.)

`thread_init()` initializes the thread system. We will defer full discussion to our discussion of Pintos threads below. It is called so early in initialization because a valid thread structure is a prerequisite for acquiring a lock, and lock acquisition in turn is important to other Pintos subsystems. Then we initialize the console and print a startup message to the console.

The next block of functions we call initializes the kernel's memory system. `palloc_init()` sets up the kernel page allocator, which doles out memory one or more pages at a time (see section [A.5.1 Page Allocator](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC123)). `malloc_init()` sets up the allocator that handles allocations of arbitrary-size blocks of memory (see section [A.5.2 Block Allocator](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC124)). `paging_init()` sets up a page table for the kernel (see section [A.7 Page Table](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC126)).

In projects 2 and later, `main()` also calls `tss_init()` and `gdt_init()`.

The next set of calls initializes the interrupt system. `intr_init()` sets up the CPU's _interrupt descriptor table_ (IDT) to ready it for interrupt handling (see section [A.4.1 Interrupt Infrastructure](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC119)), then `timer_init()` and `kbd_init()` prepare for handling timer interrupts and keyboard interrupts, respectively. `input_init()` sets up to merge serial and keyboard input into one stream. In projects 2 and later, we also prepare to handle interrupts caused by user programs using `exception_init()` and `syscall_init()`.

Now that interrupts are set up, we can start the scheduler with `thread_start()`, which creates the idle thread and enables interrupts. With interrupts enabled, interrupt-driven serial port I/O becomes possible, so we use `serial_init_queue()` to switch to that mode. Finally, `timer_calibrate()` calibrates the timer for accurate short delays.

If the file system is compiled in, as it will starting in project 2, we initialize the IDE disks with `ide_init()`, then the file system with `filesys_init()`.

Boot is complete, so we print a message.

Function `run_actions()` now parses and executes actions specified on the kernel command line, such as `run` to run a test (in project 1) or a user program (in later projects).

Finally, if -q was specified on the kernel command line, we call `shutdown_power_off()` to terminate the machine simulator. Otherwise, `main()` calls `thread_exit()`, which allows any other running threads to continue running.

### A.1.4 Physical Memory Map

@headitem Memory Range

| Owner              | Contents |                                                                               |
| ------------------ | -------- | ----------------------------------------------------------------------------- |
| 00000000--000003ff | CPU      | Real mode interrupt table.                                                    |
| 00000400--000005ff | BIOS     | Miscellaneous data area.                                                      |
| 00000600--00007bff | --       | ---                                                                           |
| 00007c00--00007dff | Pintos   | Loader.                                                                       |
| 0000e000--0000efff | Pintos   | Stack for loader; kernel stack and `struct thread` for initial kernel thread. |
| 0000f000--0000ffff | Pintos   | Page directory for startup code.                                              |
| 00010000--00020000 | Pintos   | Page tables for startup code.                                                 |
| 00020000--0009ffff | Pintos   | Kernel code, data, and uninitialized data segments.                           |
| 000a0000--000bffff | Video    | VGA display memory.                                                           |
| 000c0000--000effff | Hardware | Reserved for expansion card RAM and ROM.                                      |
| 000f0000--000fffff | BIOS     | ROM BIOS.                                                                     |
| 00100000--03ffffff | Pintos   | Dynamic memory allocation.                                                    |

## A.2 Threads

### A.2.1 `struct thread`

The main Pintos data structure for threads is `struct thread`, declared in threads/thread.h.

#### Structure: **struct thread**

Represents a thread or a user process. In the projects, you will have to add your own members to `struct thread`. You may also change or delete the definitions of existing members.

Every `struct thread` occupies the beginning of its own page of memory. The rest of the page is used for the thread's stack, which grows downward from the end of the page. It looks like this:

```
                  4 kB +---------------------------------+
                       |          kernel stack           |
                       |                |                |
                       |                |                |
                       |                V                |
                       |         grows downward          |
                       |                                 |
                       |                                 |
                       |                                 |
                       |                                 |
                       |                                 |
                       |                                 |
                       |                                 |
                       |                                 |
sizeof (struct thread) +---------------------------------+
                       |              magic              |
                       |                :                |
                       |                :                |
                       |              status             |
                       |               tid               |
                  0 kB +---------------------------------+
```

This has two consequences. First, `struct thread` must not be allowed to grow too big. If it does, then there will not be enough room for the kernel stack. The base `struct thread` is only a few bytes in size. It probably should stay well under 1 kB.

Second, kernel stacks must not be allowed to grow too large. If a stack overflows, it will corrupt the thread state. Thus, kernel functions should not allocate large structures or arrays as non-static local variables. Use dynamic allocation with `malloc()` or `palloc_get_page()` instead (see section [A.5 Memory Allocation](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC122)).

#### Member of `struct thread`: tid\_t **tid**

The thread's thread identifier or _tid_. Every thread must have a tid that is unique over the entire lifetime of the kernel. By default, `tid_t` is a `typedef` for `int` and each new thread receives the numerically next higher tid, starting from 1 for the initial process. You can change the type and the numbering scheme if you like.

#### Member of `struct thread`: enum thread\_status **status**

The thread's state, one of the following:

#### Thread State: **`THREAD_RUNNING`**

The thread is running. Exactly one thread is running at a given time. `thread_current()` returns the running thread.

#### Thread State: **`THREAD_READY`**

The thread is ready to run, but it's not running right now. The thread could be selected to run the next time the scheduler is invoked. Ready threads are kept in a doubly linked list called `ready_list`.

#### Thread State: **`THREAD_BLOCKED`**

The thread is waiting for something, e.g. a lock to become available, an interrupt to be invoked. The thread won't be scheduled again until it transitions to the `THREAD_READY` state with a call to `thread_unblock()`. This is most conveniently done indirectly, using one of the Pintos synchronization primitives that block and unblock threads automatically (see section [A.3 Synchronization](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC111)).

There is no _a priori_ way to tell what a blocked thread is waiting for, but a backtrace can help (see section [E.4 Backtraces](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_11.html#SEC160)).

#### Thread State: **`THREAD_DYING`**

The thread will be destroyed by the scheduler after switching to the next thread.

#### Member of `struct thread`: char **name\[16]**

The thread's name as a string, or at least the first few characters of it.

#### Member of `struct thread`: uint8\_t \***stack**

Every thread has its own stack to keep track of its state. When the thread is running, the CPU's stack pointer register tracks the top of the stack and this member is unused. But when the CPU switches to another thread, this member saves the thread's stack pointer. No other members are needed to save the thread's registers, because the other registers that must be saved are saved on the stack.

When an interrupt occurs, whether in the kernel or a user program, an `struct intr_frame` is pushed onto the stack. When the interrupt occurs in a user program, the `struct intr_frame` is always at the very top of the page. See section [A.4 Interrupt Handling](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC118), for more information.

#### Member of `struct thread`: int **priority**

A thread priority, ranging from `PRI_MIN` (0) to `PRI_MAX` (63). Lower numbers correspond to lower priorities, so that priority 0 is the lowest priority and priority 63 is the highest. Pintos as provided ignores thread priorities, but you will implement priority scheduling in project 1 (see section [3.3.3 Priority Scheduling](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_3.html#SEC37)).

#### Member of `struct thread`: `struct list_elem` **allelem**

This "list element" is used to link the thread into the list of all threads. Each thread is inserted into this list when it is created and removed when it exits. The `thread_foreach()` function should be used to iterate over all threads.

#### Member of `struct thread`: `struct list_elem` **elem**

A "list element" used to put the thread into doubly linked lists, either `ready_list` (the list of threads ready to run) or a list of threads waiting on a semaphore in `sema_down()`. It can do double duty because a thread waiting on a semaphore is not ready, and vice versa.

#### Member of `struct thread`: uint32\_t \***pagedir**

Only present in project 2 and later. See section [5.1.2.3 Page Tables](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_5.html#SEC70).

#### Member of `struct thread`: unsigned **magic**

Always set to `THREAD_MAGIC`, which is just an arbitrary number defined in threads/thread.c, and used to detect stack overflow. `thread_current()` checks that the `magic` member of the running thread's `struct thread` is set to `THREAD_MAGIC`. Stack overflow tends to change this value, triggering the assertion. For greatest benefit, as you add members to `struct thread`, leave `magic` at the end.

### A.2.2 Thread Functions

threads/thread.c implements several public functions for thread support. Let's take a look at the most useful:

#### Function: void **thread\_init** (void)

Called by `main()` to initialize the thread system. Its main purpose is to create a `struct thread` for Pintos's initial thread. This is possible because the Pintos loader puts the initial thread's stack at the top of a page, in the same position as any other Pintos thread.

Before `thread_init()` runs, `thread_current()` will fail because the running thread's `magic` value is incorrect. Lots of functions call `thread_current()` directly or indirectly, including `lock_acquire()` for locking a lock, so `thread_init()` is called early in Pintos initialization.

#### Function: void **thread\_start** (void)

Called by `main()` to start the scheduler. Creates the idle thread, that is, the thread that is scheduled when no other thread is ready. Then enables interrupts, which as a side effect enables the scheduler because the scheduler runs on return from the timer interrupt, using `intr_yield_on_return()` (see section [A.4.3 External Interrupt Handling](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC121)).

#### Function: void **thread\_tick** (void)

Called by the timer interrupt at each timer tick. It keeps track of thread statistics and triggers the scheduler when a time slice expires.

#### Function: void **thread\_print\_stats** (void)

Called during Pintos shutdown to print thread statistics.

#### Function: tid\_t **thread\_create** (const char \*name, int priority, thread\_func \*func, void \*aux)

Creates and starts a new thread named name with the given priority, returning the new thread's tid. The thread executes func, passing aux as the function's single argument.

`thread_create()` allocates a page for the thread's `struct thread` and stack and initializes its members, then it sets up a set of fake stack frames for it (see section [A.2.3 Thread Switching](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC110)). The thread is initialized in the blocked state, then unblocked just before returning, which allows the new thread to be scheduled (see [Thread States](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#Thread%20States)).

#### Type: **void thread\_func (void \*aux)**

This is the type of the function passed to `thread_create()`, whose aux argument is passed along as the function's argument.

#### Function: void **thread\_block** (void)

Transitions the running thread from the running state to the blocked state (see [Thread States](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#Thread%20States)). The thread will not run again until `thread_unblock()` is called on it, so you'd better have some way arranged for that to happen. Because `thread_block()` is so low-level, you should prefer to use one of the synchronization primitives instead (see section [A.3 Synchronization](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC111)).

#### Function: void **thread\_unblock** (struct thread \*thread)

Transitions thread, which must be in the blocked state, to the ready state, allowing it to resume running (see [Thread States](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#Thread%20States)). This is called when the event that the thread is waiting for occurs, e.g. when the lock that the thread is waiting on becomes available.

#### Function: struct thread \***thread\_current** (void)

Returns the running thread.

#### Function: tid\_t **thread\_tid** (void)

Returns the running thread's thread id. Equivalent to `thread_current ()->tid`.

#### Function: const char \***thread\_name** (void)

Returns the running thread's name. Equivalent to `thread_current ()->name`.

#### Function: void **thread\_exit** (void) `NO_RETURN`

Causes the current thread to exit. Never returns, hence `NO_RETURN` (see section [E.3 Function and Parameter Attributes](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_11.html#SEC159)).

#### Function: void **thread\_yield** (void)

Yields the CPU to the scheduler, which picks a new thread to run. The new thread might be the current thread, so you can't depend on this function to keep this thread from running for any particular length of time.

#### Function: void **thread\_foreach** (thread\_action\_func \*action, void \*aux)

Iterates over all threads t and invokes `action(t, aux)` on each. action must refer to a function that matches the signature given by `thread_action_func()`:

#### Type: **void thread\_action\_func (struct thread \*thread, void \*aux)**

Performs some action on a thread, given aux.

#### Function: int **thread\_get\_priority** (void)

#### Function: void **thread\_set\_priority** (int new\_priority)

Stub to set and get thread priority. See section [3.3.3 Priority Scheduling](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_3.html#SEC37).

#### Function: int **thread\_get\_nice** (void)

#### Function: void **thread\_set\_nice** (int new\_nice)

#### Function: int **thread\_get\_recent\_cpu** (void)

#### Function: int **thread\_get\_load\_avg** (void)Stubs for the advanced scheduler. See section [B. 4.4BSD Scheduler](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_8.html#SEC142).

### A.2.3 Thread Switching

`schedule()` is responsible for switching threads. It is internal to threads/thread.c and called only by the three public thread functions that need to switch threads: `thread_block()`, `thread_exit()`, and `thread_yield()`. Before any of these functions call `schedule()`, they disable interrupts (or ensure that they are already disabled) and then change the running thread's state to something other than running.

`schedule()` is short but tricky. It records the current thread in local variable cur, determines the next thread to run as local variable next (by calling `next_thread_to_run()`), and then calls `switch_threads()` to do the actual thread switch. The thread we switched to was also running inside `switch_threads()`, as are all the threads not currently running, so the new thread now returns out of `switch_threads()`, returning the previously running thread.

`switch_threads()` is an assembly language routine in threads/switch.S. It saves registers on the stack, saves the CPU's current stack pointer in the current `struct thread`'s `stack` member, restores the new thread's `stack` into the CPU's stack pointer, restores registers from the stack, and returns.

The rest of the scheduler is implemented in `thread_schedule_tail()`. It marks the new thread as running. If the thread we just switched from is in the dying state, then it also frees the page that contained the dying thread's `struct thread` and stack. These couldn't be freed prior to the thread switch because the switch needed to use it.

Running a thread for the first time is a special case. When `thread_create()` creates a new thread, it goes through a fair amount of trouble to get it started properly. In particular, the new thread hasn't started running yet, so there's no way for it to be running inside `switch_threads()` as the scheduler expects. To solve the problem, `thread_create()` creates some fake stack frames in the new thread's stack:

* The topmost fake stack frame is for `switch_threads()`, represented by `struct switch_threads_frame`. The important part of this frame is its `eip` member, the return address. We point `eip` to `switch_entry()`, indicating it to be the function that called `switch_entry()`.
* The next fake stack frame is for `switch_entry()`, an assembly language routine in threads/switch.S that adjusts the stack pointer,[(4)](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_fot.html#FOOT4) calls `thread_schedule_tail()` (this special case is why `thread_schedule_tail()` is separate from `schedule()`), and returns. We fill in its stack frame so that it returns into `kernel_thread()`, a function in threads/thread.c.
* The final stack frame is for `kernel_thread()`, which enables interrupts and calls the thread's function (the function passed to `thread_create()`). If the thread's function returns, it calls `thread_exit()` to terminate the thread.

## A.3 Synchronization

If sharing of resources between threads is not handled in a careful, controlled fashion, the result is usually a big mess. This is especially the case in operating system kernels, where faulty sharing can crash the entire machine. Pintos provides several synchronization primitives to help out.

### A.3.1 Disabling Interrupts

The crudest way to do synchronization is to disable interrupts, that is, to temporarily prevent the CPU from responding to interrupts. If interrupts are off, no other thread will preempt the running thread, because thread preemption is driven by the timer interrupt. If interrupts are on, as they normally are, then the running thread may be preempted by another at any time, whether between two C statements or even within the execution of one.

Incidentally, this means that Pintos is a "preemptible kernel," that is, kernel threads can be preempted at any time. Traditional Unix systems are "nonpreemptible," that is, kernel threads can only be preempted at points where they explicitly call into the scheduler. (User programs can be preempted at any time in both models.) As you might imagine, preemptible kernels require more explicit synchronization.

You should have little need to set the interrupt state directly. Most of the time you should use the other synchronization primitives described in the following sections. The main reason to disable interrupts is to synchronize kernel threads with external interrupt handlers, which cannot sleep and thus cannot use most other forms of synchronization (see section [A.4.3 External Interrupt Handling](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC121)).

Some external interrupts cannot be postponed, even by disabling interrupts. These interrupts, called _non-maskable interrupts_ (NMIs), are supposed to be used only in emergencies, e.g. when the computer is on fire. Pintos does not handle non-maskable interrupts.

Types and functions for disabling and enabling interrupts are in threads/interrupt.h.

Type: **enum intr\_level**One of `INTR_OFF` or `INTR_ON`, denoting that interrupts are disabled or enabled, respectively.Function: enum intr\_level **intr\_get\_level** (void)Returns the current interrupt state.Function: enum intr\_level **intr\_set\_level** (enum intr\_level level)Turns interrupts on or off according to level. Returns the previous interrupt state.Function: enum intr\_level **intr\_enable** (void)Turns interrupts on. Returns the previous interrupt state.Function: enum intr\_level **intr\_disable** (void)Turns interrupts off. Returns the previous interrupt state.

### A.3.2 Semaphores

A _semaphore_ is a nonnegative integer together with two operators that manipulate it atomically, which are:

* "Down" or "P": wait for the value to become positive, then decrement it.
* "Up" or "V": increment the value (and wake up one waiting thread, if any).

A semaphore initialized to 0 may be used to wait for an event that will happen exactly once. For example, suppose thread A starts another thread B and wants to wait for B to signal that some activity is complete. A can create a semaphore initialized to 0, pass it to B as it starts it, and then "down" the semaphore. When B finishes its activity, it "ups" the semaphore. This works regardless of whether A "downs" the semaphore or B "ups" it first.

A semaphore initialized to 1 is typically used for controlling access to a resource. Before a block of code starts using the resource, it "downs" the semaphore, then after it is done with the resource it "ups" the resource. In such a case a lock, described below, may be more appropriate.

Semaphores can also be initialized to values larger than 1. These are rarely used.

Semaphores were invented by Edsger Dijkstra and first used in the THE operating system (\[ [Dijkstra](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_14.html#Dijkstra)]).

Pintos' semaphore type and operations are declared in threads/synch.h.

Type: **struct semaphore**Represents a semaphore.Function: void **sema\_init** (struct semaphore \*sema, unsigned value)Initializes sema as a new semaphore with the given initial value.Function: void **sema\_down** (struct semaphore \*sema)Executes the "down" or "P" operation on sema, waiting for its value to become positive and then decrementing it by one.Function: bool **sema\_try\_down** (struct semaphore \*sema)Tries to execute the "down" or "P" operation on sema, without waiting. Returns true if sema was successfully decremented, or false if it was already zero and thus could not be decremented without waiting. Calling this function in a tight loop wastes CPU time, so use `sema_down()` or find a different approach instead.Function: void **sema\_up** (struct semaphore \*sema)Executes the "up" or "V" operation on sema, incrementing its value. If any threads are waiting on sema, wakes one of them up.

Unlike most synchronization primitives, `sema_up()` may be called inside an external interrupt handler (see section [A.4.3 External Interrupt Handling](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC121)).

Semaphores are internally built out of disabling interrupt (see section [A.3.1 Disabling Interrupts](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC112)) and thread blocking and unblocking (`thread_block()` and `thread_unblock()`). Each semaphore maintains a list of waiting threads, using the linked list implementation in lib/kernel/list.c.

### A.3.3 Locks

A _lock_ is like a semaphore with an initial value of 1 (see section [A.3.2 Semaphores](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC113)). A lock's equivalent of "up" is called "release", and the "down" operation is called "acquire".

Compared to a semaphore, a lock has one added restriction: only the thread that acquires a lock, called the lock's "owner", is allowed to release it. If this restriction is a problem, it's a good sign that a semaphore should be used, instead of a lock.

Locks in Pintos are not "recursive," that is, it is an error for the thread currently holding a lock to try to acquire that lock.

Lock types and functions are declared in threads/synch.h.

Type: **struct lock**Represents a lock.Function: void **lock\_init** (struct lock \*lock)Initializes lock as a new lock. The lock is not initially owned by any thread.Function: void **lock\_acquire** (struct lock \*lock)Acquires lock for the current thread, first waiting for any current owner to release it if necessary.Function: bool **lock\_try\_acquire** (struct lock \*lock)Tries to acquire lock for use by the current thread, without waiting. Returns true if successful, false if the lock is already owned. Calling this function in a tight loop is a bad idea because it wastes CPU time, so use `lock_acquire()` instead.Function: void **lock\_release** (struct lock \*lock)Releases lock, which the current thread must own.Function: bool **lock\_held\_by\_current\_thread** (const struct lock \*lock)Returns true if the running thread owns lock, false otherwise. There is no function to test whether an arbitrary thread owns a lock, because the answer could change before the caller could act on it.

### A.3.4 Monitors

A _monitor_ is a higher-level form of synchronization than a semaphore or a lock. A monitor consists of data being synchronized, plus a lock, called the _monitor lock_, and one or more _condition variables_. Before it accesses the protected data, a thread first acquires the monitor lock. It is then said to be "in the monitor". While in the monitor, the thread has control over all the protected data, which it may freely examine or modify. When access to the protected data is complete, it releases the monitor lock.

Condition variables allow code in the monitor to wait for a condition to become true. Each condition variable is associated with an abstract condition, e.g. "some data has arrived for processing" or "over 10 seconds has passed since the user's last keystroke". When code in the monitor needs to wait for a condition to become true, it "waits" on the associated condition variable, which releases the lock and waits for the condition to be signaled. If, on the other hand, it has caused one of these conditions to become true, it "signals" the condition to wake up one waiter, or "broadcasts" the condition to wake all of them.

The theoretical framework for monitors was laid out by C. A. R. Hoare (\[ [Hoare](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_14.html#Hoare)]). Their practical usage was later elaborated in a paper on the Mesa operating system (\[ [Lampson](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_14.html#Lampson)]).

Condition variable types and functions are declared in threads/synch.h.

Type: **struct condition**Represents a condition variable.Function: void **cond\_init** (struct condition \*cond)Initializes cond as a new condition variable.Function: void **cond\_wait** (struct condition \*cond, struct lock \*lock)Atomically releases lock (the monitor lock) and waits for cond to be signaled by some other piece of code. After cond is signaled, reacquires lock before returning. lock must be held before calling this function.

Sending a signal and waking up from a wait are not an atomic operation. Thus, typically `cond_wait()`'s caller must recheck the condition after the wait completes and, if necessary, wait again. See the next section for an example.

Function: void **cond\_signal** (struct condition \*cond, struct lock \*lock)If any threads are waiting on cond (protected by monitor lock lock), then this function wakes up one of them. If no threads are waiting, returns without performing any action. lock must be held before calling this function.Function: void **cond\_broadcast** (struct condition \*cond, struct lock \*lock)Wakes up all threads, if any, waiting on cond (protected by monitor lock lock). lock must be held before calling this function.

#### **A.3.4.1 Monitor Example**

The classical example of a monitor is handling a buffer into which one or more "producer" threads write characters and out of which one or more "consumer" threads read characters. To implement this we need, besides the monitor lock, two condition variables which we will call not\_full and not\_empty:

```
char buf[BUF_SIZE];     /* Buffer. */
size_t n = 0;           /* 0 <= n <= BUF_SIZE: # of characters in buffer. */
size_t head = 0;        /* buf index of next char to write (mod BUF_SIZE). */
size_t tail = 0;        /* buf index of next char to read (mod BUF_SIZE). */
struct lock lock;       /* Monitor lock. */
struct condition not_empty; /* Signaled when the buffer is not empty. */
struct condition not_full; /* Signaled when the buffer is not full. */

...initialize the locks and condition variables...

void put (char ch) {
  lock_acquire (&lock);
  while (n == BUF_SIZE)            /* Can't add to buf as long as it's full. */
    cond_wait (&not_full, &lock);
  buf[head++ % BUF_SIZE] = ch;     /* Add ch to buf. */
  n++;
  cond_signal (&not_empty, &lock); /* buf can't be empty anymore. */
  lock_release (&lock);
}

char get (void) {
  char ch;
  lock_acquire (&lock);
  while (n == 0)                  /* Can't read buf as long as it's empty. */
    cond_wait (&not_empty, &lock);
  ch = buf[tail++ % BUF_SIZE];    /* Get ch from buf. */
  n--;
  cond_signal (&not_full, &lock); /* buf can't be full anymore. */
  lock_release (&lock);
}
```

Note that `BUF_SIZE` must divide evenly into `SIZE_MAX + 1` for the above code to be completely correct. Otherwise, it will fail the first time `head` wraps around to 0. In practice, `BUF_SIZE` would ordinarily be a power of 2.

### A.3.5 Optimization Barriers

An _optimization barrier_ is a special statement that prevents the compiler from making assumptions about the state of memory across the barrier. The compiler will not reorder reads or writes of variables across the barrier or assume that a variable's value is unmodified across the barrier, except for local variables whose address is never taken. In Pintos, threads/synch.h defines the `barrier()` macro as an optimization barrier.

One reason to use an optimization barrier is when data can change asynchronously, without the compiler's knowledge, e.g. by another thread or an interrupt handler. The `too_many_loops()` function in devices/timer.c is an example. This function starts out by busy-waiting in a loop until a timer tick occurs:

```
/* Wait for a timer tick. */
int64_t start = ticks;
while (ticks == start)
  barrier ();
```

Without an optimization barrier in the loop, the compiler could conclude that the loop would never terminate, because `start` and `ticks` start out equal and the loop itself never changes them. It could then "optimize" the function into an infinite loop, which would definitely be undesirable.

Optimization barriers can be used to avoid other compiler optimizations. The `busy_wait()` function, also in devices/timer.c, is an example. It contains this loop:

```
while (loops-- > 0)
  barrier ();
```

The goal of this loop is to busy-wait by counting `loops` down from its original value to 0. Without the barrier, the compiler could delete the loop entirely, because it produces no useful output and has no side effects. The barrier forces the compiler to pretend that the loop body has an important effect.

Finally, optimization barriers can be used to force the ordering of memory reads or writes. For example, suppose we add a "feature" that, whenever a timer interrupt occurs, the character in global variable `timer_put_char` is printed on the console, but only if global Boolean variable `timer_do_put` is true. The best way to set up x to be printed is then to use an optimization barrier, like this:

```
timer_put_char = 'x';
barrier ();
timer_do_put = true;
```

Without the barrier, the code is buggy because the compiler is free to reorder operations when it doesn't see a reason to keep them in the same order. In this case, the compiler doesn't know that the order of assignments is important, so its optimizer is permitted to exchange their order. There's no telling whether it will actually do this, and it is possible that passing the compiler different optimization flags or using a different version of the compiler will produce different behavior.

Another solution is to disable interrupts around the assignments. This does not prevent reordering, but it prevents the interrupt handler from intervening between the assignments. It also has the extra runtime cost of disabling and re-enabling interrupts:

```
enum intr_level old_level = intr_disable ();
timer_put_char = 'x';
timer_do_put = true;
intr_set_level (old_level);
```

A second solution is to mark the declarations of `timer_put_char` and `timer_do_put` as volatile. This keyword tells the compiler that the variables are externally observable and restricts its latitude for optimization. However, the semantics of volatile are not well-defined, so it is not a good general solution. The base Pintos code does not use volatile at all.

The following is _not_ a solution, because locks neither prevent interrupts nor prevent the compiler from reordering the code within the region where the lock is held:

```
lock_acquire (&timer_lock);     /* INCORRECT CODE */
timer_put_char = 'x';
timer_do_put = true;
lock_release (&timer_lock);
```

The compiler treats invocation of any function defined externally, that is, in another source file, as a limited form of optimization barrier. Specifically, the compiler assumes that any externally defined function may access any statically or dynamically allocated data and any local variable whose address is taken. This often means that explicit barriers can be omitted. It is one reason that Pintos contains few explicit barriers.

A function defined in the same source file, or in a header included by the source file, cannot be relied upon as a optimization barrier. This applies even to invocation of a function before its definition, because the compiler may read and parse the entire source file before performing optimization.

## A.4 Interrupt Handling

An _interrupt_ notifies the CPU of some event. Much of the work of an operating system relates to interrupts in one way or another. For our purposes, we classify interrupts into two broad categories:

* _Internal interrupts_, that is, interrupts caused directly by CPU instructions. System calls, attempts at invalid memory access (_page faults_), and attempts to divide by zero are some activities that cause internal interrupts. Because they are caused by CPU instructions, internal interrupts are _synchronous_ or synchronized with CPU instructions. `intr_disable()` does not disable internal interrupts.
* _External interrupts_, that is, interrupts originating outside the CPU. These interrupts come from hardware devices such as the system timer, keyboard, serial ports, and disks. External interrupts are _asynchronous_, meaning that their delivery is not synchronized with instruction execution. Handling of external interrupts can be postponed with `intr_disable()` and related functions (see section [A.3.1 Disabling Interrupts](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC112)).

The CPU treats both classes of interrupts largely the same way, so Pintos has common infrastructure to handle both classes. The following section describes this common infrastructure. The sections after that give the specifics of external and internal interrupts.

If you haven't already read chapter 3, "Basic Execution Environment," in \[ [IA32-v1](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_14.html#IA32-v1)], it is recommended that you do so now. You might also want to skim chapter 5, "Interrupt and Exception Handling," in \[ [IA32-v3a](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_14.html#IA32-v3a)].

### A.4.1 Interrupt Infrastructure

When an interrupt occurs, the CPU saves its most essential state on a stack and jumps to an interrupt handler routine. The 80x86 architecture supports 256 interrupts, numbered 0 through 255, each with an independent handler defined in an array called the _interrupt descriptor table_ or IDT.

In Pintos, `intr_init()` in threads/interrupt.c sets up the IDT so that each entry points to a unique entry point in threads/intr-stubs.S named `intrNN_stub()`, where NN is the interrupt number in hexadecimal. Because the CPU doesn't give us any other way to find out the interrupt number, this entry point pushes the interrupt number on the stack. Then it jumps to `intr_entry()`, which pushes all the registers that the processor didn't already push for us, and then calls `intr_handler()`, which brings us back into C in threads/interrupt.c.

The main job of `intr_handler()` is to call the function registered for handling the particular interrupt. (If no function is registered, it dumps some information to the console and panics.) It also does some extra processing for external interrupts (see section [A.4.3 External Interrupt Handling](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC121)).

When `intr_handler()` returns, the assembly code in threads/intr-stubs.S restores all the CPU registers saved earlier and directs the CPU to return from the interrupt.

The following types and functions are common to all interrupts.

#### Type: **void intr\_handler\_func (struct intr\_frame \*frame)**

This is how an interrupt handler function must be declared. Its frame argument (see below) allows it to determine the cause of the interrupt and the state of the thread that was interrupted.

#### Type: **struct intr\_frame**

The stack frame of an interrupt handler, as saved by the CPU, the interrupt stubs, and `intr_entry()`. Its most interesting members are described below.

#### Member of `struct intr_frame`: uint32\_t **edi**

#### Member of `struct intr_frame`: uint32\_t **esi**

#### Member of `struct intr_frame`: uint32\_t **ebp**

#### Member of `struct intr_frame`: uint32\_t **esp\_dummy**

#### Member of `struct intr_frame`: uint32\_t **ebx**

#### Member of `struct intr_frame`: uint32\_t **edx**

#### Member of `struct intr_frame`: uint32\_t **ecx**

#### Member of `struct intr_frame`: uint32\_t **eax**

#### Member of `struct intr_frame`: uint16\_t **es**

#### Member of `struct intr_frame`: uint16\_t **ds**

Register values in the interrupted thread, pushed by `intr_entry()`. The `esp_dummy` value isn't actually used (refer to the description of `PUSHA` in \[ [IA32-v2b](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_14.html#IA32-v2b)] for details).

#### Member of `struct intr_frame`: uint32\_t **vec\_no**

The interrupt vector number, ranging from 0 to 255.

#### Member of `struct intr_frame`: uint32\_t **error\_code**

The "error code" pushed on the stack by the CPU for some internal interrupts.

#### Member of `struct intr_frame`: void **(\*eip)** (void)

The address of the next instruction to be executed by the interrupted thread.

#### Member of `struct intr_frame`: void \***esp**

The interrupted thread's stack pointer.

#### Function: const char \***intr\_name** (uint8\_t vec)

Returns the name of the interrupt numbered vec, or `"unknown"` if the interrupt has no registered name.

### A.4.2 Internal Interrupt Handling

Internal interrupts are caused directly by CPU instructions executed by the running kernel thread or user process (from project 2 onward). An internal interrupt is therefore said to arise in a "process context."

In an internal interrupt's handler, it can make sense to examine the `struct intr_frame` passed to the interrupt handler, or even to modify it. When the interrupt returns, modifications in `struct intr_frame` become changes to the calling thread or process's state. For example, the Pintos system call handler returns a value to the user program by modifying the saved EAX register (see section [4.5.2 System Call Details](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_4.html#SEC63)).

There are no special restrictions on what an internal interrupt handler can or can't do. Generally they should run with interrupts enabled, just like other code, and so they can be preempted by other kernel threads. Thus, they do need to synchronize with other threads on shared data and other resources (see section [A.3 Synchronization](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC111)).

Internal interrupt handlers can be invoked recursively. For example, the system call handler might cause a page fault while attempting to read user memory. Deep recursion would risk overflowing the limited kernel stack (see section [A.2.1 `struct thread`](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC108)), but should be unnecessary.

#### Function: void **intr\_register\_int** (uint8\_t vec, int dpl, enum intr\_level level, intr\_handler\_func \*handler, const char \*name)

Registers handler to be called when internal interrupt numbered vec is triggered. Names the interrupt name for debugging purposes.

If level is `INTR_ON`, external interrupts will be processed normally during the interrupt handler's execution, which is normally desirable. Specifying `INTR_OFF` will cause the CPU to disable external interrupts when it invokes the interrupt handler. The effect is slightly different from calling `intr_disable()` inside the handler, because that leaves a window of one or more CPU instructions in which external interrupts are still enabled. This is important for the page fault handler; refer to the comments in userprog/exception.c for details.

dpl determines how the interrupt can be invoked. If dpl is 0, then the interrupt can be invoked only by kernel threads. Otherwise dpl should be 3, which allows user processes to invoke the interrupt with an explicit INT instruction. The value of dpl doesn't affect user processes' ability to invoke the interrupt indirectly, e.g. an invalid memory reference will cause a page fault regardless of dpl.

### A.4.3 External Interrupt Handling

External interrupts are caused by events outside the CPU. They are asynchronous, so they can be invoked at any time that interrupts have not been disabled. We say that an external interrupt runs in an "interrupt context."

In an external interrupt, the `struct intr_frame` passed to the handler is not very meaningful. It describes the state of the thread or process that was interrupted, but there is no way to predict which one that is. It is possible, although rarely useful, to examine it, but modifying it is a recipe for disaster.

Only one external interrupt may be processed at a time. Neither internal nor external interrupt may nest within an external interrupt handler. Thus, an external interrupt's handler must run with interrupts disabled (see section [A.3.1 Disabling Interrupts](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC112)).

An external interrupt handler must not sleep or yield, which rules out calling `lock_acquire()`, `thread_yield()`, and many other functions. Sleeping in interrupt context would effectively put the interrupted thread to sleep, too, until the interrupt handler was again scheduled and returned. This would be unfair to the unlucky thread, and it would deadlock if the handler were waiting for the sleeping thread to, e.g., release a lock.

An external interrupt handler effectively monopolizes the machine and delays all other activities. Therefore, external interrupt handlers should complete as quickly as they can. Anything that require much CPU time should instead run in a kernel thread, possibly one that the interrupt triggers using a synchronization primitive.

External interrupts are controlled by a pair of devices outside the CPU called _programmable interrupt controllers_, _PICs_ for short. When `intr_init()` sets up the CPU's IDT, it also initializes the PICs for interrupt handling. The PICs also must be "acknowledged" at the end of processing for each external interrupt. `intr_handler()` takes care of that by calling `pic_end_of_interrupt()`, which properly signals the PICs.

The following functions relate to external interrupts.

#### Function: void **intr\_register\_ext** (uint8\_t vec, intr\_handler\_func \*handler, const char \*name)

Registers handler to be called when external interrupt numbered vec is triggered. Names the interrupt name for debugging purposes. The handler will run with interrupts disabled.

#### Function: bool **intr\_context** (void)

Returns true if we are running in an interrupt context, otherwise false. Mainly used in functions that might sleep or that otherwise should not be called from interrupt context, in this form:

```
ASSERT (!intr_context ());
```

#### Function: void **intr\_yield\_on\_return** (void)

When called in an interrupt context, causes `thread_yield()` to be called just before the interrupt returns. Used in the timer interrupt handler when a thread's time slice expires, to cause a new thread to be scheduled.

## A.5 Memory Allocation

Pintos contains two memory allocators, one that allocates memory in units of a page, and one that can allocate blocks of any size.

### A.5.1 Page Allocator

The page allocator declared in threads/palloc.h allocates memory in units of a page. It is most often used to allocate memory one page at a time, but it can also allocate multiple contiguous pages at once.

The page allocator divides the memory it allocates into two pools, called the kernel and user pools. By default, each pool gets half of system memory above 1 MB, but the division can be changed with the -ul kernel command line option (see [Why PAL\_USER?](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_5.html#Why%20PAL\_USER?)). An allocation request draws from one pool or the other. If one pool becomes empty, the other may still have free pages. The user pool should be used for allocating memory for user processes and the kernel pool for all other allocations. This will only become important starting with project 3. Until then, all allocations should be made from the kernel pool.

Each pool's usage is tracked with a bitmap, one bit per page in the pool. A request to allocate n pages scans the bitmap for n consecutive bits set to false, indicating that those pages are free, and then sets those bits to true to mark them as used. This is a "first fit" allocation strategy (see [Wilson](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_14.html#Wilson)).

The page allocator is subject to fragmentation. That is, it may not be possible to allocate n contiguous pages even though n or more pages are free, because the free pages are separated by used pages. In fact, in pathological cases it may be impossible to allocate 2 contiguous pages even though half of the pool's pages are free. Single-page requests can't fail due to fragmentation, so requests for multiple contiguous pages should be limited as much as possible.

Pages may not be allocated from interrupt context, but they may be freed.

When a page is freed, all of its bytes are cleared to 0xcc, as a debugging aid (see section [E.8 Tips](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_11.html#SEC168)).

Page allocator types and functions are described below.

#### Function: void \***palloc\_get\_page** (enum palloc\_flags flags)

#### Function: void \***palloc\_get\_multiple** (enum palloc\_flags flags, size\_t page\_cnt)

Obtains and returns one page, or page\_cnt contiguous pages, respectively. Returns a null pointer if the pages cannot be allocated.

The flags argument may be any combination of the following flags:

#### Page Allocator Flag: **`PAL_ASSERT`**

If the pages cannot be allocated, panic the kernel. This is only appropriate during kernel initialization. User processes should never be permitted to panic the kernel.

#### Page Allocator Flag: **`PAL_ZERO`**

Zero all the bytes in the allocated pages before returning them. If not set, the contents of newly allocated pages are unpredictable.

#### Page Allocator Flag: **`PAL_USER`**

Obtain the pages from the user pool. If not set, pages are allocated from the kernel pool.

#### Function: void **palloc\_free\_page** (void \*page)

#### Function: void **palloc\_free\_multiple** (void \*pages, size\_t page\_cnt)

Frees one page, or page\_cnt contiguous pages, respectively, starting at pages. All of the pages must have been obtained using `palloc_get_page()` or `palloc_get_multiple()`.

### A.5.2 Block Allocator

The block allocator, declared in threads/malloc.h, can allocate blocks of any size. It is layered on top of the page allocator described in the previous section. Blocks returned by the block allocator are obtained from the kernel pool.

The block allocator uses two different strategies for allocating memory. The first strategy applies to blocks that are 1 kB or smaller (one-fourth of the page size). These allocations are rounded up to the nearest power of 2, or 16 bytes, whichever is larger. Then they are grouped into a page used only for allocations of that size.

The second strategy applies to blocks larger than 1 kB. These allocations (plus a small amount of overhead) are rounded up to the nearest page in size, and then the block allocator requests that number of contiguous pages from the page allocator.

In either case, the difference between the allocation requested size and the actual block size is wasted. A real operating system would carefully tune its allocator to minimize this waste, but this is unimportant in an instructional system like Pintos.

As long as a page can be obtained from the page allocator, small allocations always succeed. Most small allocations do not require a new page from the page allocator at all, because they are satisfied using part of a page already allocated. However, large allocations always require calling into the page allocator, and any allocation that needs more than one contiguous page can fail due to fragmentation, as already discussed in the previous section. Thus, you should minimize the number of large allocations in your code, especially those over approximately 4 kB each.

When a block is freed, all of its bytes are cleared to 0xcc, as a debugging aid (see section [E.8 Tips](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_11.html#SEC168)).

The block allocator may not be called from interrupt context.

The block allocator functions are described below. Their interfaces are the same as the standard C library functions of the same names.

#### Function: void \***malloc** (size\_t size)

Obtains and returns a new block, from the kernel pool, at least size bytes long. Returns a null pointer if size is zero or if memory is not available.

#### Function: void \***calloc** (size\_t a, size\_t b)

Obtains a returns a new block, from the kernel pool, at least `a * b` bytes long. The block's contents will be cleared to zeros. Returns a null pointer if a or b is zero or if insufficient memory is available.

#### Function: void \***realloc** (void \*block, size\_t new\_size)

Attempts to resize block to new\_size bytes, possibly moving it in the process. If successful, returns the new block, in which case the old block must no longer be accessed. On failure, returns a null pointer, and the old block remains valid.

A call with block null is equivalent to `malloc()`. A call with new\_size zero is equivalent to `free()`.

#### Function: void **free** (void \*block)

Frees block, which must have been previously returned by `malloc()`, `calloc()`, or `realloc()` (and not yet freed).

## A.6 Virtual Addresses

A 32-bit virtual address can be divided into a 20-bit _page number_ and a 12-bit _page offset_ (or just _offset_), like this:

```
               31               12 11        0
              +-------------------+-----------+
              |    Page Number    |   Offset  |
              +-------------------+-----------+
                       Virtual Address
```

Header threads/vaddr.h defines these functions and macros for working with virtual addresses:

#### Macro: **PGSHIFT**

#### Macro: **PGBITS**

The bit index (0) and number of bits (12) of the offset part of a virtual address, respectively.

#### Macro: **PGMASK**

A bit mask with the bits in the page offset set to 1, the rest set to 0 (0xfff).

#### Macro: **PGSIZE**

The page size in bytes (4,096).

#### Function: unsigned **pg\_ofs** (const void \*va)

Extracts and returns the page offset in virtual address va.

#### Function: uintptr\_t **pg\_no** (const void \*va)

Extracts and returns the page number in virtual address va.

#### Function: void \***pg\_round\_down** (const void \*va)

Returns the start of the virtual page that va points within, that is, va with the page offset set to 0.

#### Function: void \***pg\_round\_up** (const void \*va)

Returns va rounded up to the nearest page boundary.

Virtual memory in Pintos is divided into two regions: user virtual memory and kernel virtual memory (see section [4.1.4 Virtual Memory Layout](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_4.html#SEC48)). The boundary between them is `PHYS_BASE`:

#### Macro: **PHYS\_BASE**

Base address of kernel virtual memory. It defaults to 0xc0000000 (3 GB), but it may be changed to any multiple of 0x10000000 from 0x80000000 to 0xf0000000.

User virtual memory ranges from virtual address 0 up to `PHYS_BASE`. Kernel virtual memory occupies the rest of the virtual address space, from `PHYS_BASE` up to 4 GB.

#### Function: bool **is\_user\_vaddr** (const void \*va)

#### Function: bool **is\_kernel\_vaddr** (const void \*va)

Returns true if va is a user or kernel virtual address, respectively, false otherwise.

The 80x86 doesn't provide any way to directly access memory given a physical address. This ability is often necessary in an operating system kernel, so Pintos works around it by mapping kernel virtual memory one-to-one to physical memory. That is, virtual address `PHYS_BASE` accesses physical address 0, virtual address `PHYS_BASE` + 0x1234 accesses physical address 0x1234, and so on up to the size of the machine's physical memory. Thus, adding `PHYS_BASE` to a physical address obtains a kernel virtual address that accesses that address; conversely, subtracting `PHYS_BASE` from a kernel virtual address obtains the corresponding physical address. Header threads/vaddr.h provides a pair of functions to do these translations:

#### Function: void \***ptov** (uintptr\_t pa)

Returns the kernel virtual address corresponding to physical address pa, which should be between 0 and the number of bytes of physical memory.

#### Function: uintptr\_t **vtop** (void \*va)

Returns the physical address corresponding to va, which must be a kernel virtual address.

## A.7 Page Table

The code in pagedir.c is an abstract interface to the 80x86 hardware page table, also called a "page directory" by Intel processor documentation. The page table interface uses a `uint32_t *` to represent a page table because this is convenient for accessing their internal structure.

The sections below describe the page table interface and internals.

### A.7.1 Creation, Destruction, and Activation

These functions create, destroy, and activate page tables. The base Pintos code already calls these functions where necessary, so it should not be necessary to call them yourself.

#### Function: uint32\_t \***pagedir\_create** (void)

Creates and returns a new page table. The new page table contains Pintos's normal kernel virtual page mappings, but no user virtual mappings.

Returns a null pointer if memory cannot be obtained.

#### Function: void **pagedir\_destroy** (uint32\_t \*pd)

Frees all of the resources held by pd, including the page table itself and the frames that it maps.

#### Function: void **pagedir\_activate** (uint32\_t \*pd)

Activates pd. The active page table is the one used by the CPU to translate memory references.

### A.7.2 Inspection and Updates

These functions examine or update the mappings from pages to frames encapsulated by a page table. They work on both active and inactive page tables (that is, those for running and suspended processes), flushing the TLB as necessary.

#### Function: bool **pagedir\_set\_page** (uint32\_t \*pd, void \*upage, void \*kpage, bool writable)

Adds to pd a mapping from user page upage to the frame identified by kernel virtual address kpage. If writable is true, the page is mapped read/write; otherwise, it is mapped read-only.

User page upage must not already be mapped in pd.

Kernel page kpage should be a kernel virtual address obtained from the user pool with `palloc_get_page(PAL_USER)` (see [Why PAL\_USER?](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_5.html#Why%20PAL\_USER?)).

Returns true if successful, false on failure. Failure will occur if additional memory required for the page table cannot be obtained.

#### Function: void \***pagedir\_get\_page** (uint32\_t \*pd, const void \*uaddr)

Looks up the frame mapped to uaddr in pd. Returns the kernel virtual address for that frame, if uaddr is mapped, or a null pointer if it is not.

#### Function: void **pagedir\_clear\_page** (uint32\_t \*pd, void \*page)

Marks page "not present" in pd. Later accesses to the page will fault.

Other bits in the page table for page are preserved, permitting the accessed and dirty bits (see the next section) to be checked.

This function has no effect if page is not mapped.

### A.7.3 Accessed and Dirty Bits

80x86 hardware provides some assistance for implementing page replacement algorithms, through a pair of bits in the page table entry (PTE) for each page. On any read or write to a page, the CPU sets the _accessed bit_ to 1 in the page's PTE, and on any write, the CPU sets the _dirty bit_ to 1. The CPU never resets these bits to 0, but the OS may do so.

Proper interpretation of these bits requires understanding of _aliases_, that is, two (or more) pages that refer to the same frame. When an aliased frame is accessed, the accessed and dirty bits are updated in only one page table entry (the one for the page used for access). The accessed and dirty bits for the other aliases are not updated.

See section [5.1.5.1 Accessed and Dirty Bits](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_5.html#SEC75), on applying these bits in implementing page replacement algorithms.

#### Function: bool **pagedir\_is\_dirty** (uint32\_t \*pd, const void \*page)

#### Function: bool **pagedir\_is\_accessed** (uint32\_t \*pd, const void \*page)

Returns true if page directory pd contains a page table entry for page that is marked dirty (or accessed). Otherwise, returns false.

#### Function: void **pagedir\_set\_dirty** (uint32\_t \*pd, const void \*page, bool value)

#### Function: void **pagedir\_set\_accessed** (uint32\_t \*pd, const void \*page, bool value)

If page directory pd has a page table entry for page, then its dirty (or accessed) bit is set to value.

### A.7.4 Page Table Details

The functions provided with Pintos are sufficient to implement the projects. However, you may still find it worthwhile to understand the hardware page table format, so we'll go into a little detail in this section.

#### **A.7.4.1 Structure**

The top-level paging data structure is a page called the "page directory" (PD) arranged as an array of 1,024 32-bit page directory entries (PDEs), each of which represents 4 MB of virtual memory. Each PDE may point to the physical address of another page called a "page table" (PT) arranged, similarly, as an array of 1,024 32-bit page table entries (PTEs), each of which translates a single 4 kB virtual page to a physical page.

Translation of a virtual address into a physical address follows the three-step process illustrated in the diagram below:[(5)](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_fot.html#FOOT5)

1. The most-significant 10 bits of the virtual address (bits 22...31) index the page directory. If the PDE is marked "present," the physical address of a page table is read from the PDE thus obtained. If the PDE is marked "not present" then a page fault occurs.
2. The next 10 bits of the virtual address (bits 12...21) index the page table. If the PTE is marked "present," the physical address of a data page is read from the PTE thus obtained. If the PTE is marked "not present" then a page fault occurs.
3. The least-significant 12 bits of the virtual address (bits 0...11) are added to the data page's physical base address, yielding the final physical address.

```
 31                  22 21                  12 11                   0
+----------------------+----------------------+----------------------+
| Page Directory Index |   Page Table Index   |    Page Offset       |
+----------------------+----------------------+----------------------+
             |                    |                     |
     _______/             _______/                _____/
    /                    /                       /
   /    Page Directory  /      Page Table       /    Data Page
  /     .____________. /     .____________.    /   .____________.
  |1,023|____________| |1,023|____________|    |   |____________|
  |1,022|____________| |1,022|____________|    |   |____________|
  |1,021|____________| |1,021|____________|    \__\|____________|
  |1,020|____________| |1,020|____________|       /|____________|
  |     |            | |     |            |        |            |
  |     |            | \____\|            |_       |            |
  |     |      .     |      /|      .     | \      |      .     |
  \____\|      .     |_      |      .     |  |     |      .     |
       /|      .     | \     |      .     |  |     |      .     |
        |      .     |  |    |      .     |  |     |      .     |
        |            |  |    |            |  |     |            |
        |____________|  |    |____________|  |     |____________|
       4|____________|  |   4|____________|  |     |____________|
       3|____________|  |   3|____________|  |     |____________|
       2|____________|  |   2|____________|  |     |____________|
       1|____________|  |   1|____________|  |     |____________|
       0|____________|  \__\0|____________|  \____\|____________|
                           /                      /
```

Pintos provides some macros and functions that are useful for working with raw page tables:

#### Macro: **PTSHIFT**

#### Macro: **PTBITS**

The starting bit index (12) and number of bits (10), respectively, in a page table index.

#### Macro: **PTMASK**

A bit mask with the bits in the page table index set to 1 and the rest set to 0 (0x3ff000).

#### Macro: **PTSPAN**

The number of bytes of virtual address space that a single page table page covers (4,194,304 bytes, or 4 MB).

#### Macro: **PDSHIFT**

#### Macro: **PDBITS**

The starting bit index (22) and number of bits (10), respectively, in a page directory index.

#### Macro: **PDMASK**

A bit mask with the bits in the page directory index set to 1 and other bits set to 0 (0xffc00000).

#### Function: uintptr\_t **pd\_no** (const void \*va)

#### Function: uintptr\_t **pt\_no** (const void \*va)

Returns the page directory index or page table index, respectively, for virtual address va. These functions are defined in threads/pte.h.

#### Function: unsigned **pg\_ofs** (const void \*va)

Returns the page offset for virtual address va. This function is defined in threads/vaddr.h.

#### **A.7.4.2 Page Table Entry Format**

You do not need to understand the PTE format to do the Pintos projects, unless you wish to incorporate the page table into your supplemental page table (see section [5.1.4 Managing the Supplemental Page Table](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_5.html#SEC73)).

The actual format of a page table entry is summarized below. For complete information, refer to section 3.7, "Page Translation Using 32-Bit Physical Addressing," in \[ [IA32-v3a](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_14.html#IA32-v3a)].

```
 31                                   12 11 9      6 5     2 1 0
+---------------------------------------+----+----+-+-+---+-+-+-+
|           Physical Address            | AVL|    |D|A|   |U|W|P|
+---------------------------------------+----+----+-+-+---+-+-+-+
```

Some more information on each bit is given below. The names are threads/pte.h macros that represent the bits' values:

#### Macro: **PTE\_P**

Bit 0, the "present" bit. When this bit is 1, the other bits are interpreted as described below. When this bit is 0, any attempt to access the page will page fault. The remaining bits are then not used by the CPU and may be used by the OS for any purpose.

#### Macro: **PTE\_W**

Bit 1, the "read/write" bit. When it is 1, the page is writable. When it is 0, write attempts will page fault.

#### Macro: **PTE\_U**

Bit 2, the "user/supervisor" bit. When it is 1, user processes may access the page. When it is 0, only the kernel may access the page (user accesses will page fault).

Pintos clears this bit in PTEs for kernel virtual memory, to prevent user processes from accessing them.

#### Macro: **PTE\_A**

Bit 5, the "accessed" bit. See section [A.7.3 Accessed and Dirty Bits](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC129).

#### Macro: **PTE\_D**

Bit 6, the "dirty" bit. See section [A.7.3 Accessed and Dirty Bits](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC129).

#### Macro: **PTE\_AVL**

Bits 9...11, available for operating system use. Pintos, as provided, does not use them and sets them to 0.

#### Macro: **PTE\_ADDR**

Bits 12...31, the top 20 bits of the physical address of a frame. The low 12 bits of the frame's address are always 0.

Other bits are either reserved or uninteresting in a Pintos context and should be set to@tie{}0.

Header threads/pte.h defines three functions for working with page table entries:

#### Function: uint32\_t **pte\_create\_kernel** (uint32\_t \*page, bool writable)

Returns a page table entry that points to page, which should be a kernel virtual address. The PTE's present bit will be set. It will be marked for kernel-only access. If writable is true, the PTE will also be marked read/write; otherwise, it will be read-only.

#### Function: uint32\_t **pte\_create\_user** (uint32\_t \*page, bool writable)

Returns a page table entry that points to page, which should be the kernel virtual address of a frame in the user pool (see [Why PAL\_USER?](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_5.html#Why%20PAL\_USER?)). The PTE's present bit will be set and it will be marked to allow user-mode access. If writable is true, the PTE will also be marked read/write; otherwise, it will be read-only.

#### Function: void \***pte\_get\_page** (uint32\_t pte)

Returns the kernel virtual address for the frame that pte points to. The pte may be present or not-present; if it is not-present then the pointer returned is only meaningful if the address bits in the PTE actually represent a physical address.

#### **A.7.4.3 Page Directory Entry Format**

Page directory entries have the same format as PTEs, except that the physical address points to a page table page instead of a frame. Header threads/pte.h defines two functions for working with page directory entries:

#### Function: uint32\_t **pde\_create** (uint32\_t \*pt)

Returns a page directory that points to page, which should be the kernel virtual address of a page table page. The PDE's present bit will be set, it will be marked to allow user-mode access, and it will be marked read/write.

#### Function: uint32\_t \***pde\_get\_pt** (uint32\_t pde)

Returns the kernel virtual address for the page table page that pde, which must be marked present, points to.

## A.8 Hash Table

Pintos provides a hash table data structure in lib/kernel/hash.c. To use it you will need to include its header file, lib/kernel/hash.h, with `#include <hash.h>`. No code provided with Pintos uses the hash table, which means that you are free to use it as is, modify its implementation for your own purposes, or ignore it, as you wish.

Most implementations of the virtual memory project use a hash table to translate pages to frames. You may find other uses for hash tables as well.

### A.8.1 Data Types

A hash table is represented by `struct hash`.

#### Type: **struct hash**

Represents an entire hash table. The actual members of `struct hash` are "opaque." That is, code that uses a hash table should not access `struct hash` members directly, nor should it need to. Instead, use hash table functions and macros.

The hash table operates on elements of type `struct hash_elem`.

#### Type: **struct hash\_elem**

Embed a `struct hash_elem` member in the structure you want to include in a hash table. Like `struct hash`, `struct hash_elem` is opaque. All functions for operating on hash table elements actually take and return pointers to `struct hash_elem`, not pointers to your hash table's real element type.

You will often need to obtain a `struct hash_elem` given a real element of the hash table, and vice versa. Given a real element of the hash table, you may use the & operator to obtain a pointer to its `struct hash_elem`. Use the `hash_entry()` macro to go the other direction.

#### Macro: type \***hash\_entry** (struct hash\_elem \*elem, type, member)

Returns a pointer to the structure that elem, a pointer to a `struct hash_elem`, is embedded within. You must provide type, the name of the structure that elem is inside, and member, the name of the member in type that elem points to.

For example, suppose `h` is a `struct hash_elem *` variable that points to a `struct thread` member (of type `struct hash_elem`) named `h_elem`. Then, `hash_entry@tie{`(h, struct thread, h\_elem)} yields the address of the `struct thread` that `h` points within.

See section [A.8.5 Hash Table Example](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC139), for an example.

Each hash table element must contain a key, that is, data that identifies and distinguishes elements, which must be unique among elements in the hash table. (Elements may also contain non-key data that need not be unique.) While an element is in a hash table, its key data must not be changed. Instead, if need be, remove the element from the hash table, modify its key, then reinsert the element.

For each hash table, you must write two functions that act on keys: a hash function and a comparison function. These functions must match the following prototypes:

#### Type: **unsigned hash\_hash\_func (const struct hash\_elem \*element, void \*aux)**

Returns a hash of element's data, as a value anywhere in the range of `unsigned int`. The hash of an element should be a pseudo-random function of the element's key. It must not depend on non-key data in the element or on any non-constant data other than the key. Pintos provides the following functions as a suitable basis for hash functions.

#### Function: unsigned **hash\_bytes** (const void \*buf, size\_t \*size)

Returns a hash of the size bytes starting at buf. The implementation is the general-purpose [Fowler-Noll-Vo hash](http://en.wikipedia.org/wiki/Fowler\_Noll\_Vo\_hash) for 32-bit words.

#### Function: unsigned **hash\_string** (const char \*s)

Returns a hash of null-terminated string s.

#### Function: unsigned **hash\_int** (int i)

Returns a hash of integer i.

If your key is a single piece of data of an appropriate type, it is sensible for your hash function to directly return the output of one of these functions. For multiple pieces of data, you may wish to combine the output of more than one call to them using, e.g., the ^ (exclusive or) operator. Finally, you may entirely ignore these functions and write your own hash function from scratch, but remember that your goal is to build an operating system kernel, not to design a hash function.

See section [A.8.6 Auxiliary Data](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC140), for an explanation of aux.

#### Type: **bool hash\_less\_func (const struct hash\_elem \*a, const struct hash\_elem \*b, void \*aux)**

Compares the keys stored in elements a and b. Returns true if a is less than b, false if a is greater than or equal to b.

If two elements compare equal, then they must hash to equal values.

See section [A.8.6 Auxiliary Data](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC140), for an explanation of aux.

See section [A.8.5 Hash Table Example](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC139), for hash and comparison function examples.

A few functions accept a pointer to a third kind of function as an argument:

#### Type: **void hash\_action\_func (struct hash\_elem \*element, void \*aux)**

Performs some kind of action, chosen by the caller, on element.

See section [A.8.6 Auxiliary Data](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC140), for an explanation of aux.

### A.8.2 Basic Functions

These functions create, destroy, and inspect hash tables.

#### Function: bool **hash\_init** (struct hash \*hash, hash\_hash\_func \*hash\_func, hash\_less\_func \*less\_func, void \*aux)

Initializes hash as a hash table with hash\_func as hash function, less\_func as comparison function, and aux as auxiliary data. Returns true if successful, false on failure. `hash_init()` calls `malloc()` and fails if memory cannot be allocated.

See section [A.8.6 Auxiliary Data](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC140), for an explanation of aux, which is most often a null pointer.

#### Function: void **hash\_clear** (struct hash \*hash, hash\_action\_func \*action)

Removes all the elements from hash, which must have been previously initialized with `hash_init()`.

If action is non-null, then it is called once for each element in the hash table, which gives the caller an opportunity to deallocate any memory or other resources used by the element. For example, if the hash table elements are dynamically allocated using `malloc()`, then action could `free()` the element. This is safe because `hash_clear()` will not access the memory in a given hash element after calling action on it. However, action must not call any function that may modify the hash table, such as `hash_insert()` or `hash_delete()`.

#### Function: void **hash\_destroy** (struct hash \*hash, hash\_action\_func \*action)

If action is non-null, calls it for each element in the hash, with the same semantics as a call to `hash_clear()`. Then, frees the memory held by hash. Afterward, hash must not be passed to any hash table function, absent an intervening call to `hash_init()`.

#### Function: size\_t **hash\_size** (struct hash \*hash)

Returns the number of elements currently stored in hash.

#### Function: bool **hash\_empty** (struct hash \*hash)

Returns true if hash currently contains no elements, false if hash contains at least one element.

### A.8.3 Search Functions

Each of these functions searches a hash table for an element that compares equal to one provided. Based on the success of the search, they perform some action, such as inserting a new element into the hash table, or simply return the result of the search.

#### Function: struct hash\_elem \***hash\_insert** (struct hash \*hash, struct hash\_elem \*element)

Searches hash for an element equal to element. If none is found, inserts element into hash and returns a null pointer. If the table already contains an element equal to element, it is returned without modifying hash.

#### Function: struct hash\_elem \***hash\_replace** (struct hash \*hash, struct hash\_elem \*element)

Inserts element into hash. Any element equal to element already in hash is removed. Returns the element removed, or a null pointer if hash did not contain an element equal to element.

The caller is responsible for deallocating any resources associated with the returned element, as appropriate. For example, if the hash table elements are dynamically allocated using `malloc()`, then the caller must `free()` the element after it is no longer needed.

The element passed to the following functions is only used for hashing and comparison purposes. It is never actually inserted into the hash table. Thus, only key data in the element needs to be initialized, and other data in the element will not be used. It often makes sense to declare an instance of the element type as a local variable, initialize the key data, and then pass the address of its `struct hash_elem` to `hash_find()` or `hash_delete()`. See section [A.8.5 Hash Table Example](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC139), for an example. (Large structures should not be allocated as local variables. See section [A.2.1 `struct thread`](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC108), for more information.)

#### Function: struct hash\_elem \***hash\_find** (struct hash \*hash, struct hash\_elem \*element)

Searches hash for an element equal to element. Returns the element found, if any, or a null pointer otherwise.

#### Function: struct hash\_elem \***hash\_delete** (struct hash \*hash, struct hash\_elem \*element)

Searches hash for an element equal to element. If one is found, it is removed from hash and returned. Otherwise, a null pointer is returned and hash is unchanged.

The caller is responsible for deallocating any resources associated with the returned element, as appropriate. For example, if the hash table elements are dynamically allocated using `malloc()`, then the caller must `free()` the element after it is no longer needed.

### A.8.4 Iteration Functions

These functions allow iterating through the elements in a hash table. Two interfaces are supplied. The first requires writing and supplying a hash\_action\_func to act on each element (see section [A.8.1 Data Types](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC135)).

#### Function: void **hash\_apply** (struct hash \*hash, hash\_action\_func \*action)

Calls action once for each element in hash, in arbitrary order. action must not call any function that may modify the hash table, such as `hash_insert()` or `hash_delete()`. action must not modify key data in elements, although it may modify any other data.

The second interface is based on an "iterator" data type. Idiomatically, iterators are used as follows:

```
struct hash_iterator i;

hash_first (&i, h);
while (hash_next (&i))
  {
    struct foo *f = hash_entry (hash_cur (&i), struct foo, elem);
    ...do something with f...
  }
```

#### Type: **struct hash\_iterator**

Represents a position within a hash table. Calling any function that may modify a hash table, such as `hash_insert()` or `hash_delete()`, invalidates all iterators within that hash table.

Like `struct hash` and `struct hash_elem`, `struct hash_elem` is opaque.

#### Function: void **hash\_first** (struct hash\_iterator \*iterator, struct hash \*hash)

Initializes iterator to just before the first element in hash.

#### Function: struct hash\_elem \***hash\_next** (struct hash\_iterator \*iterator)

Advances iterator to the next element in hash, and returns that element. Returns a null pointer if no elements remain. After `hash_next()` returns null for iterator, calling it again yields undefined behavior.

#### Function: struct hash\_elem \***hash\_cur** (struct hash\_iterator \*iterator)

Returns the value most recently returned by `hash_next()` for iterator. Yields undefined behavior after `hash_first()` has been called on iterator but before `hash_next()` has been called for the first time.

### A.8.5 Hash Table Example

Suppose you have a structure, called `struct page`, that you want to put into a hash table. First, define `struct page` to include a `struct hash_elem` member:

```
struct page
  {
    struct hash_elem hash_elem; /* Hash table element. */
    void *addr;                 /* Virtual address. */
    /* ...other members... */
  };
```

We write a hash function and a comparison function using addr as the key. A pointer can be hashed based on its bytes, and the < operator works fine for comparing pointers:

```
/* Returns a hash value for page p. */
unsigned
page_hash (const struct hash_elem *p_, void *aux UNUSED)
{
  const struct page *p = hash_entry (p_, struct page, hash_elem);
  return hash_bytes (&p->addr, sizeof p->addr);
}

/* Returns true if page a precedes page b. */
bool
page_less (const struct hash_elem *a_, const struct hash_elem *b_,
           void *aux UNUSED)
{
  const struct page *a = hash_entry (a_, struct page, hash_elem);
  const struct page *b = hash_entry (b_, struct page, hash_elem);

  return a->addr < b->addr;
}
```

(The use of `UNUSED` in these functions' prototypes suppresses a warning that aux is unused. See section [E.3 Function and Parameter Attributes](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_11.html#SEC159), for information about `UNUSED`. See section [A.8.6 Auxiliary Data](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC140), for an explanation of aux.)

Then, we can create a hash table like this:

```
struct hash pages;

hash_init (&pages, page_hash, page_less, NULL);
```

Now we can manipulate the hash table we've created. If `p` is a pointer to a `struct page`, we can insert it into the hash table with:

```
hash_insert (&pages, &p->hash_elem);
```

If there's a chance that pages might already contain a page with the same addr, then we should check `hash_insert()`'s return value.

To search for an element in the hash table, use `hash_find()`. This takes a little setup, because `hash_find()` takes an element to compare against. Here's a function that will find and return a page based on a virtual address, assuming that pages is defined at file scope:

```
/* Returns the page containing the given virtual address,
   or a null pointer if no such page exists. */
struct page *
page_lookup (const void *address)
{
  struct page p;
  struct hash_elem *e;

  p.addr = address;
  e = hash_find (&pages, &p.hash_elem);
  return e != NULL ? hash_entry (e, struct page, hash_elem) : NULL;
}
```

`struct page` is allocated as a local variable here on the assumption that it is fairly small. Large structures should not be allocated as local variables. See section [A.2.1 `struct thread`](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC108), for more information.

A similar function could delete a page by address using `hash_delete()`.

### A.8.6 Auxiliary Data

In simple cases like the example above, there's no need for the aux parameters. In these cases, just pass a null pointer to `hash_init()` for aux and ignore the values passed to the hash function and comparison functions. (You'll get a compiler warning if you don't use the aux parameter, but you can turn that off with the `UNUSED` macro, as shown in the example, or you can just ignore it.)

aux is useful when you have some property of the data in the hash table is both constant and needed for hashing or comparison, but not stored in the data items themselves. For example, if the items in a hash table are fixed-length strings, but the items themselves don't indicate what that fixed length is, you could pass the length as an aux parameter.

### A.8.7 Synchronization

The hash table does not do any internal synchronization. It is the caller's responsibility to synchronize calls to hash table functions. In general, any number of functions that examine but do not modify the hash table, such as `hash_find()` or `hash_next()`, may execute simultaneously. However, these function cannot safely execute at the same time as any function that may modify a given hash table, such as `hash_insert()` or `hash_delete()`, nor may more than one function that can modify a given hash table execute safely at once.

It is also the caller's responsibility to synchronize access to data in hash table elements. How to synchronize access to this data depends on how it is designed and organized, as with any other data structure.
