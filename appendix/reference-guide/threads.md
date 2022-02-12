# Threads

## `struct thread`

The main Pintos data structure for threads is `struct thread`, declared in `threads/thread.h`.

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

Second, kernel stacks must not be allowed to grow too large. If a stack overflows, it will corrupt the thread state. Thus, kernel functions should not allocate large structures or arrays as non-static local variables. Use dynamic allocation with `malloc()` or `palloc_get_page()` instead (see section [Memory Allocation](memory-allocation.md)).

<details>

<summary>Members of <code>struct thread</code></summary>

#### tid\_t **tid**

The thread's thread identifier or _tid_. Every thread must have a tid that is unique over the entire lifetime of the kernel. By default, `tid_t` is a `typedef` for `int` and each new thread receives the numerically next higher tid, starting from 1 for the initial process. You can change the type and the numbering scheme if you like.

#### enum thread\_status **status**

The thread's state, one of the following:

#### _**`THREAD_RUNNING`**_

The thread is running. Exactly one thread is running at a given time. `thread_current()` returns the running thread.

#### _**`THREAD_READY`**_

The thread is ready to run, but it's not running right now. The thread could be selected to run the next time the scheduler is invoked. Ready threads are kept in a doubly linked list called `ready_list`.

#### _**`THREAD_BLOCKED`**_

The thread is waiting for something, e.g. a lock to become available, an interrupt to be invoked. The thread won't be scheduled again until it transitions to the `THREAD_READY` state with a call to `thread_unblock()`. This is most conveniently done indirectly, using one of the Pintos synchronization primitives that block and unblock threads automatically (see section [Synchronization](synchronization.md)).

There is no _a priori_ way to tell what a blocked thread is waiting for, but a backtrace can help (see section [Backtraces](../../getting-started/debug-and-test/debugging.md#backtraces)).

#### **`THREAD_DYING`**

The thread will be destroyed by the scheduler after switching to the next thread.

#### char **name\[16]**

The thread's name as a string, or at least the first few characters of it.

#### uint8\_t \***stack**

Every thread has its own stack to keep track of its state. When the thread is running, the CPU's stack pointer register tracks the top of the stack and this member is unused. But when the CPU switches to another thread, this member saves the thread's stack pointer. No other members are needed to save the thread's registers, because the other registers that must be saved are saved on the stack.

When an interrupt occurs, whether in the kernel or a user program, an `struct intr_frame` is pushed onto the stack. When the interrupt occurs in a user program, the `struct intr_frame` is always at the very top of the page. See section [Interrupt Handling](interrupt-handling.md), for more information.

#### int **priority**

A thread priority, ranging from `PRI_MIN` (0) to `PRI_MAX` (63). Lower numbers correspond to lower priorities, so that priority 0 is the lowest priority and priority 63 is the highest. Pintos as provided ignores thread priorities, but you will implement priority scheduling in project 1.

#### `struct list_elem` **allelem**

This "list element" is used to link the thread into the list of all threads. Each thread is inserted into this list when it is created and removed when it exits. The `thread_foreach()` function should be used to iterate over all threads.

#### `struct thread`: `struct list_elem` **elem**

A "list element" used to put the thread into doubly linked lists, either `ready_list` (the list of threads ready to run) or a list of threads waiting on a semaphore in `sema_down()`. It can do double duty because a thread waiting on a semaphore is not ready, and vice versa.

#### `struct thread`: uint32\_t \***pagedir**

Only present in project 2 and later. See section [Page Table](page-table.md).

#### `struct thread`: unsigned **magic**

Always set to `THREAD_MAGIC`, which is just an arbitrary number defined in threads/thread.c, and used to detect stack overflow. `thread_current()` checks that the `magic` member of the running thread's `struct thread` is set to `THREAD_MAGIC`. Stack overflow tends to change this value, triggering the assertion. For greatest benefit, as you add members to `struct thread`, leave `magic` at the end.

</details>

## Thread Functions

`threads/thread.c` implements several public functions for thread support. Let's take a look at the most useful:

<details>

<summary>Functions of <code>struct thread</code></summary>

#### Function: void **thread\_init** (void)

Called by `pintos_init()` to initialize the thread system. Its main purpose is to create a `struct thread` for Pintos's initial thread. This is possible because the Pintos loader puts the initial thread's stack at the top of a page, in the same position as any other Pintos thread.

Before `thread_init()` runs, `thread_current()` will fail because the running thread's `magic` value is incorrect. Lots of functions call `thread_current()` directly or indirectly, including `lock_acquire()` for locking a lock, so `thread_init()` is called early in Pintos initialization.

#### Function: void **thread\_start** (void)

Called by `pintos_init()` to start the scheduler. Creates the idle thread, that is, the thread that is scheduled when no other thread is ready. Then enables interrupts, which as a side effect enables the scheduler because the scheduler runs on return from the timer interrupt, using `intr_yield_on_return()`.

#### Function: void **thread\_tick** (void)

Called by the timer interrupt at each timer tick. It keeps track of thread statistics and triggers the scheduler when a time slice expires.

#### Function: void **thread\_print\_stats** (void)

Called during Pintos shutdown to print thread statistics.

#### Function: tid\_t **thread\_create** (const char \*name, int priority, thread\_func \*func, void \*aux)

Creates and starts a new thread named name with the given priority, returning the new thread's tid. The thread executes func, passing aux as the function's single argument.

`thread_create()` allocates a page for the thread's `struct thread` and stack and initializes its members, then it sets up a set of fake stack frames for it (see section [Thread Switching](threads.md#thread-switching)). The thread is initialized in the blocked state, then unblocked just before returning, which allows the new thread to be scheduled (see Thread States).

#### Type: **void thread\_func (void \*aux)**

This is the type of the function passed to `thread_create()`, whose aux argument is passed along as the function's argument.

#### Function: void **thread\_block** (void)

Transitions the running thread from the running state to the blocked state (see Thread States). The thread will not run again until `thread_unblock()` is called on it, so you'd better have some way arranged for that to happen. Because `thread_block()` is so low-level, you should prefer to use one of the synchronization primitives instead (see section [Synchronization](synchronization.md)).

#### Function: void **thread\_unblock** (struct thread \*thread)

Transitions thread, which must be in the blocked state, to the ready state, allowing it to resume running (see Thread States). This is called when the event that the thread is waiting for occurs, e.g. when the lock that the thread is waiting on becomes available.

#### Function: struct thread \***thread\_current** (void)

Returns the running thread.

#### Function: tid\_t **thread\_tid** (void)

Returns the running thread's thread id. Equivalent to `thread_current ()->tid`.

#### Function: const char \***thread\_name** (void)

Returns the running thread's name. Equivalent to `thread_current ()->name`.

#### Function: void **thread\_exit** (void) `NO_RETURN`

Causes the current thread to exit. Never returns, hence `NO_RETURN` (see section [Function and Parameter Attributes](../../getting-started/debug-and-test/debugging.md#function-and-parameter-attributes)).

#### Function: void **thread\_yield** (void)

Yields the CPU to the scheduler, which picks a new thread to run. The new thread might be the current thread, so you can't depend on this function to keep this thread from running for any particular length of time.

#### Function: void **thread\_foreach** (thread\_action\_func \*action, void \*aux)

Iterates over all threads t and invokes `action(t, aux)` on each. action must refer to a function that matches the signature given by `thread_action_func()`:

#### Type: **void thread\_action\_func (struct thread \*thread, void \*aux)**

Performs some action on a thread, given aux.

#### Function: int **thread\_get\_priority** (void)

#### Function: void **thread\_set\_priority** (int new\_priority)

Stub to set and get thread priority. Used in project 1.

#### Function: int **thread\_get\_nice** (void)

#### Function: void **thread\_set\_nice** (int new\_nice)

#### Function: int **thread\_get\_recent\_cpu** (void)

#### Function: int **thread\_get\_load\_avg** (void)

#### Stubs for the advanced scheduler. See section [4.4BSD Scheduler](../4.4bsd-scheduler.md).

</details>

## Thread Switching

`schedule()` is responsible for switching threads. It is internal to `threads/thread.c` and called only by the three public thread functions that need to switch threads: `thread_block()`, `thread_exit()`, and `thread_yield()`. Before any of these functions call `schedule()`, they disable interrupts (or ensure that they are already disabled) and then change the running thread's state to something other than running.

`schedule()` is short but tricky. It records the current thread in local variable `cur`, determines the next thread to run as local variable next (by calling `next_thread_to_run()`), and then calls `switch_threads()` to do the actual thread switch. The thread we switched to was also running inside `switch_threads()`, as are all the threads not currently running, so the new thread now returns out of `switch_threads()`, returning the previously running thread.

`switch_threads()` is an assembly language routine in `threads/switch.S`. It saves registers on the stack, saves the CPU's current stack pointer in the current `struct thread`'s `stack` member, restores the new thread's `stack` into the CPU's stack pointer, restores registers from the stack, and returns.

The rest of the scheduler is implemented in `thread_schedule_tail()`. It marks the new thread as running. If the thread we just switched from is in the dying state, then it also frees the page that contained the dying thread's `struct thread` and stack. These couldn't be freed prior to the thread switch because the switch needed to use it.

Running a thread for the first time is a special case. When `thread_create()` creates a new thread, it goes through a fair amount of trouble to get it started properly. In particular, the new thread hasn't started running yet, so there's no way for it to be running inside `switch_threads()` as the scheduler expects. To solve the problem, `thread_create()` creates some fake stack frames in the new thread's stack:

* The topmost fake stack frame is for `switch_threads()`, represented by `struct switch_threads_frame`. The important part of this frame is its `eip` member, the return address. We point `eip` to `switch_entry()`, indicating it to be the function that called `switch_entry()`.
* The next fake stack frame is for `switch_entry()`, an assembly language routine in threads/switch.S that adjusts the stack pointer, calls `thread_schedule_tail()` (this special case is why `thread_schedule_tail()` is separate from `schedule()`), and returns. We fill in its stack frame so that it returns into `kernel_thread()`, a function in `threads/thread.c`.
* The final stack frame is for `kernel_thread()`, which enables interrupts and calls the thread's function (the function passed to `thread_create()`). If the thread's function returns, it calls `thread_exit()` to terminate the thread.
