# Threads

## Struct _**thread**_

**The main Pintos data structure for threads is `struct thread`, declared in `threads/thread.h`.**

* <mark style="color:blue;">**Structure: struct thread**</mark>
  * **Represents **<mark style="color:red;">**a thread**</mark>** or **<mark style="color:red;">**a user process**</mark>**.**&#x20;
  * In the projects, you will have to add your own members to `struct thread`. You may also change or delete the definitions of existing members.

<details>

<summary>Members of <code>struct thread</code></summary>

* <mark style="color:blue;">**tid\_t tid**</mark>
  * **The thread's thread identifier or **_**tid**_**.** Every thread must have a tid that is unique over the entire lifetime of the kernel.&#x20;
  * By default, `tid_t` is a `typedef` for `int` and each new thread receives the numerically next higher tid, **starting from 1 for the initial process**. You can change the type and the numbering scheme if you like.
* <mark style="color:blue;">**enum thread\_status status**</mark>
  * **The thread's state, one of the following:**
  * <mark style="color:orange;">**`THREAD_RUNNING`**</mark>_**:**_&#x20;
    * The thread is **running**. Exactly one thread is running at a given time. `thread_current()` returns the running thread.
  * <mark style="color:orange;">**`THREAD_READY`**</mark>_**:**_&#x20;
    * The thread is **ready to run**, but it's not running right now.&#x20;
    * The thread could be selected to run the next time the scheduler is invoked.&#x20;
    * Ready threads are kept in a doubly linked list called **`ready_list`**.
  * <mark style="color:orange;">**`THREAD_BLOCKED`**</mark>_**:**_&#x20;
    * The thread is **waiting for something**, e.g. a lock to become available, an interrupt to be invoked.&#x20;
    * The thread won't be scheduled again until it transitions to the `THREAD_READY` state with a call to **`thread_unblock()`**.&#x20;
    * **This is most conveniently done indirectly**, using one of the Pintos synchronization primitives that block and unblock threads automatically (see section [Synchronization](synchronization.md)).
    * **There is no **_**a priori**_** way to tell what a blocked thread is waiting for**, but a backtrace can help (see section [Backtraces](../../getting-started/debug-and-test/debugging.md#assert)).
  * <mark style="color:orange;">**`THREAD_DYING`**</mark>**:**&#x20;
    * The thread **will be destroyed** by the scheduler **after switching to the next thread**.
* <mark style="color:blue;">**char name\[16]**</mark>
  * **The thread's name** as a string, or at least the first few characters of it.
* <mark style="color:blue;">**uint8\_t \*stack**</mark>
  * **Every thread has its own stack to keep track of its state.** When the thread is running, the CPU's stack pointer register tracks the top of the stack and this member is unused. But when the CPU switches to another thread, this member **saves the thread's stack pointer**. No other members are needed to save the thread's registers, because the other registers that must be saved are saved on the stack.
  * When an interrupt occurs, whether in the kernel or a user program, an **`struct intr_frame`** is pushed onto the stack. When the interrupt occurs in a user program, the **`struct intr_frame`** is always at the very top of the page. See section [Interrupt Handling](interrupt-handling.md), for more information.
* <mark style="color:blue;">**int priority**</mark>
  * **A thread priority, ranging from `PRI_MIN` (0) to `PRI_MAX` (63).**&#x20;
  * **Lower** numbers correspond to **lower** priorities, so that priority 0 is the lowest priority and priority 63 is the highest.&#x20;
  * Pintos as provided ignores thread priorities, but you will implement priority scheduling in project 1.
* <mark style="color:blue;">**struct list\_elem allelem**</mark>
  * **This "list element" is used to link the thread into the list of all threads.** Each thread is inserted into this list when it is created and removed when it exits.&#x20;
  * The **`thread_foreach()`** function should be used to iterate over all threads.
* <mark style="color:blue;">**struct list\_elem elem**</mark>
  * **A "list element" used to put the thread into doubly linked lists**, either **`ready_list`** (the list of threads ready to run) or **a list of threads waiting** on a semaphore in `sema_down()`. It can do double duty because a thread waiting on a semaphore is not ready, and vice versa.
* <mark style="color:blue;">**uint32\_t \*pagedir**</mark>
  * Only present in project 2 and later. See section [Page Table](page-table.md).
* <mark style="color:blue;">**unsigned magic**</mark>
  * **Always set to `THREAD_MAGIC`**, which is just an arbitrary number defined in threads/thread.c, and **used to detect stack overflow**.&#x20;
  * **`thread_current()`** checks that the `magic` member of the running thread's `struct thread` is set to `THREAD_MAGIC`.&#x20;
  * Stack overflow tends to change this value, triggering the assertion. <mark style="color:red;">**For greatest benefit, as you add members to**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**`struct thread`**</mark><mark style="color:red;">**, leave**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**`magic`**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**at the end**</mark>.

</details>

### Notes

* <mark style="color:red;">**Every**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**`struct thread`**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**occupies the beginning of its own page of memory.**</mark> **The rest of the page is used for the thread's stack**, which grows downward from the end of the page. It looks like this:

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

* **This has two consequences.**&#x20;
  1. First, **`struct thread` must not be allowed to grow too big.** If it does, then there will not be enough room for the kernel stack. The base `struct thread` is only a few bytes in size. It probably should stay well under 1 kB.
  2. Second, **kernel stacks must not be allowed to grow too large.** If a stack overflows, it will corrupt the thread state. Thus, **kernel functions should not allocate large structures or arrays as non-static local variables**. Use dynamic allocation with `malloc()` or `palloc_get_page()` instead (see section [Memory Allocation](memory-allocation.md)).

## Thread Functions

**`threads/thread.c` implements several public functions for thread support.** Let's take a look at the most useful:

<details>

<summary>Functions of <code>struct thread</code></summary>

* <mark style="color:blue;">**Function: void thread\_init (void)**</mark>
  * **Called by `pintos_init()` to initialize the thread system.** Its main purpose is to **create a `struct thread` for Pintos's initial thread**. This is possible because the Pintos loader puts the initial thread's stack at the top of a page, in the same position as any other Pintos thread.
  * **Before `thread_init()` runs, `thread_current()` will fail because the running thread's `magic` value is incorrect.** Lots of functions call `thread_current()` directly or indirectly, including `lock_acquire()` for locking a lock, so `thread_init()` is called early in Pintos initialization.
* <mark style="color:blue;">**Function: void thread\_start (void)**</mark>
  * **Called by `pintos_init()` to start the scheduler.** **Creates the idle thread**, that is, the thread that is scheduled when no other thread is ready.&#x20;
  * Then <mark style="color:red;">**enables interrupts**</mark>**,** which as a side effect enables the scheduler because the scheduler runs on return from the timer interrupt, using **`intr_yield_on_return()`**.
* <mark style="color:blue;">**Function: void thread\_tick (void)**</mark>
  * **Called by the timer interrupt at each timer tick.**&#x20;
  * It keeps track of thread statistics and triggers the scheduler when a time slice expires.
* <mark style="color:blue;">**Function: void thread\_print\_stats (void)**</mark>
  * **Called during Pintos shutdown to print thread statistics.**
* <mark style="color:blue;">**Function: tid\_t thread\_create (const char \*name, int priority, thread\_func \*func, void \*aux)**</mark>
  * **Creates and starts a new thread named name with the given priority, returning the new thread's tid.** **The thread executes func, passing aux as the function's single argument.**
  * `thread_create()` **allocates a page** for the thread's `struct thread` and stack and **initializes** its members, then it **sets up a set of fake stack frames** for it (see section [Thread Switching](threads.md#thread-switching)).&#x20;
  * **The thread is initialized in the **_**blocked**_** state**, then unblocked just before returning, which allows the new thread to be scheduled (see **Thread States**).
* <mark style="color:blue;">**Type: void thread\_func (void \*aux)**</mark>
  * This is **the type of the function** passed to `thread_create()`, whose aux argument is passed along as the function's argument.
* <mark style="color:blue;">**Function: void thread\_block (void)**</mark>
  * **Transitions the running thread from the running state to the blocked state** (see **Thread States**). The thread will not run again until **`thread_unblock()`** is called on it, so you'd better have some way arranged for that to happen.&#x20;
  * Because `thread_block()` is **so low-level**, you should <mark style="color:red;">**prefer to use one of the synchronization primitives**</mark> <mark style="color:red;">**instead**</mark> (see section [Synchronization](synchronization.md)).
* <mark style="color:blue;">**Function: void thread\_unblock (struct thread \*thread)**</mark>
  * **Transitions thread, which must be in the blocked state, to the ready state**, allowing it to resume running (see **Thread States**).&#x20;
  * This is called when the event that the thread is waiting for occurs, e.g. when the lock that the thread is waiting on becomes available.
* <mark style="color:blue;">**Function: struct thread \*thread\_current (void)**</mark>
  * **Returns the running thread.**
* <mark style="color:blue;">**Function: tid\_t thread\_tid (void)**</mark>
  * **Returns the running thread's thread id.**&#x20;
  * Equivalent to `thread_current ()->tid`.
* <mark style="color:blue;">**Function: const char \*thread\_name (void)**</mark>
  * **Returns the running thread's name.**&#x20;
  * Equivalent to `thread_current ()->name`.
* <mark style="color:blue;">**Function: void thread\_exit (void)**</mark><mark style="color:blue;">** **</mark><mark style="color:blue;">**`NO_RETURN`**</mark>
  * **Causes the current thread to exit.**
  * **Never returns**, hence `NO_RETURN` (see section [Function and Parameter Attributes](../../getting-started/debug-and-test/debugging.md#function-and-parameter-attributes)).
* <mark style="color:blue;">**Function: void thread\_yield (void)**</mark>
  * **Yields the CPU to the scheduler, which picks a new thread to run.**&#x20;
  * **The new thread might be the current thread**, so you can't depend on this function to keep this thread from running for any particular length of time.
* <mark style="color:blue;">**Function: void thread\_foreach (thread\_action\_func \*action, void \*aux)**</mark>
  * **Iterates over all threads **_**t**_** and invokes `action(t, aux)` on each.**&#x20;
  * _action_ must refer to a function that matches the signature given by **`thread_action_func()`**:
* <mark style="color:blue;">**Type: void thread\_action\_func (struct thread \*thread, void \*aux)**</mark>
  * **Performs some action on a thread, given aux.**

<!---->

* <mark style="color:blue;">**Function: int thread\_get\_priority (void)**</mark>
* <mark style="color:blue;">**Function: void thread\_set\_priority (int new\_priority)**</mark>
  * Stub to set and get thread priority. Used in project 1.

<!---->

* <mark style="color:blue;">**Function: int thread\_get\_nice (void)**</mark>
* <mark style="color:blue;">**Function: void thread\_set\_nice (int new\_nice)**</mark>
* <mark style="color:blue;">**Function: int thread\_get\_recent\_cpu (void)**</mark>
* <mark style="color:blue;">**Function: int thread\_get\_load\_avg (void)**</mark>
  * Stubs for the advanced scheduler. See section [4.4BSD Scheduler](../4.4bsd-scheduler.md).

</details>

## Thread Switching

**`schedule()` is responsible for switching threads.**&#x20;

* It is internal to `threads/thread.c` and <mark style="color:red;">**called only by the three public thread functions that need to switch threads**</mark>: **`thread_block()`**, **`thread_exit()`**, and **`thread_yield()`**.&#x20;
* Before any of these functions call `schedule()`, they <mark style="color:red;">**disable interrupts**</mark> (or ensure that they are already disabled) and then change the running thread's state to something other than running.

### Schedule()

**`schedule()` is short but tricky.**&#x20;

1. It **records the current thread** in local variable `cur`,&#x20;
2. **determines the next thread to run** as local variable next (by calling `next_thread_to_run()`),&#x20;
3. and then **calls `switch_threads()` to do the actual thread switch**.&#x20;
   * <mark style="color:red;background-color:red;">**The thread we switched to was also running inside**</mark><mark style="color:red;background-color:red;">** **</mark><mark style="color:red;background-color:red;">**`switch_threads()`**</mark><mark style="color:red;background-color:red;">**, as are all the threads not currently running, so the new thread now returns out of**</mark><mark style="color:red;background-color:red;">** **</mark><mark style="color:red;background-color:red;">**`switch_threads()`**</mark><mark style="color:red;background-color:red;">**, returning the previously running thread.**</mark>
   * **`switch_threads()` is an assembly language routine in `threads/switch.S`.**&#x20;
     * It **saves** registers on the stack, **saves** the CPU's current stack pointer in the current `struct thread`'s `stack` member.
     * It **restores** the new thread's `stack` into the CPU's stack pointer, **restores** registers from the stack, and returns.
4. **The rest of the scheduler is implemented in `thread_schedule_tail()`.**&#x20;
   * It **marks the new thread as **_**running**_.&#x20;
   * If the thread we just switched from is **in the dying state**, then it also **frees the page** that contained the dying thread's `struct thread` and stack.&#x20;
   * These couldn't be freed prior to the thread switch because the switch needed to use it.

### Run a Thread for the First Time

**Running a thread for the first time is a special case.**&#x20;

When **`thread_create()`** creates a new thread, it goes through a fair amount of trouble to get it started properly. In particular, the new thread hasn't started running yet, so there's no way for it to be running inside `switch_threads()` as the scheduler expects. To solve the problem, **`thread_create()`** **creates some **_**fake stack frames**_ **in the new thread's stack**:

1. **The topmost fake stack frame is for `switch_threads()`, represented by `struct switch_threads_frame`.** The important part of this frame is its `eip` member, the return address. We point `eip` to **`switch_entry()`**, indicating it to be the function that called `switch_threads()`.
2. **The next fake stack frame is for `switch_entry()`**, an assembly language routine in threads/switch.S that adjusts the stack pointer, **calls `thread_schedule_tail()`** (this special case is why `thread_schedule_tail()` is separate from `schedule()`), and returns. We fill in its stack frame so that it **returns into `kernel_thread()`**, a function in `threads/thread.c`.
3. **The final stack frame is for `kernel_thread()`**, which **enables interrupts** and **calls the thread's function** (the function passed to `thread_create()`). If the thread's function returns, it calls **`thread_exit()`** to terminate the thread.

<mark style="color:red;">**The following is the stack page layout of a thread created by**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**`thread_create()`**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**and scheduled for the first time.**</mark> If you find some problems, feel free to contact us.

```
                  4 kB +---------------------------------+
                       |               aux               |
                       |            function             |
                       |            eip (NULL)           | kernel_thread_frame
                       +---------------------------------+
                       |      eip (to kernel_thread)     | switch_entry_frame
                       +---------------------------------+
                       |               next              | 
                       |               cur               |
                       |      eip (to switch_entry)      | 
                       |               ebx               | 
                       |               ebp               | 
                       |               esi               | 
                       |               edi               | switch_threads_frame
                       +---------------------------------+
                       |          kernel stack           |
                       |                |                |
                       |                |                |
                       |                V                |
                       |         grows downward          | 
sizeof (struct thread) +---------------------------------+
                       |              magic              |
                       |                :                |
                       |                :                |
                       |               name              |
                       |              status             |
                  0 kB +---------------------------------+
```
