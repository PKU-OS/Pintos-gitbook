# Synchronization

{% hint style="info" %}
**If sharing of resources between threads is not handled in a careful, controlled fashion, the result is usually a big mess.**&#x20;

* This is especially the case in operating system kernels, where faulty sharing can crash the entire machine.&#x20;
* Pintos provides several **synchronization primitives** to help out.
{% endhint %}

## Disabling Interrupts

**The crudest way to do synchronization is to disable interrupts, that is, to temporarily prevent the CPU from responding to interrupts.**&#x20;

* If interrupts are **off**, no other thread will preempt the running thread, because thread preemption is driven by the timer interrupt.&#x20;
* If interrupts are **on**, as they normally are, then the running thread may be **preempted** by another at any time, whether between two C statements or even within the execution of one.

Incidentally, this means that <mark style="color:red;">**Pintos is a "preemptible kernel,"**</mark> that is, kernel threads can be preempted at any time. **Traditional Unix systems are "nonpreemptible,"** that is, kernel threads can only be preempted at points where they explicitly call into the scheduler. (User programs can be preempted at any time in both models.) As you might imagine, preemptible kernels require more explicit synchronization.

### Types and Functions

Types and functions for disabling and enabling interrupts are in **`threads/interrupt.h`.**

<details>

<summary>Types and Functions for disabling/enabling interrupts</summary>

* <mark style="color:blue;">**Type: enum intr\_level**</mark>
  * One of **`INTR_OFF`** or **`INTR_ON`**, denoting that interrupts are disabled or enabled, respectively.
* <mark style="color:blue;">**Function: enum intr\_level**</mark> <mark style="color:blue;"></mark><mark style="color:blue;"></mark> <mark style="color:blue;"></mark><mark style="color:blue;">**intr\_get\_level (void)**</mark>
  * **Returns the current interrupt state.**
* <mark style="color:blue;">**Function: enum intr\_level**</mark> <mark style="color:blue;"></mark><mark style="color:blue;"></mark> <mark style="color:blue;"></mark><mark style="color:blue;">**intr\_set\_level**</mark> <mark style="color:blue;"></mark><mark style="color:blue;"></mark> <mark style="color:blue;"></mark><mark style="color:blue;">**(enum intr\_level level)**</mark>
  * **Turns interrupts on or off according to level.**&#x20;
  * **Returns the previous interrupt state.**
* <mark style="color:blue;">**Function: enum intr\_level**</mark> <mark style="color:blue;"></mark><mark style="color:blue;"></mark> <mark style="color:blue;"></mark><mark style="color:blue;">**intr\_enable**</mark> <mark style="color:blue;"></mark><mark style="color:blue;"></mark> <mark style="color:blue;"></mark><mark style="color:blue;">**(void)**</mark>
  * **Turns interrupts on.**&#x20;
  * **Returns the previous interrupt state.**
* <mark style="color:blue;">**Function: enum intr\_level intr\_disable (void)**</mark>
  * **Turns interrupts off.**&#x20;
  * **Returns the previous interrupt state.**

</details>

### Notes

**You should have **_**little**_** need to set the interrupt state directly.** Most of the time you should **use the other synchronization primitives** described in the following sections.&#x20;

* **The main reason to disable interrupts is to synchronize kernel threads with **_**external**_** interrupt handlers**, which cannot sleep and thus cannot use most other forms of synchronization (see section [External Interrupt Handling](interrupt-handling.md#external-interrupt-handling)).
* **Some external interrupts cannot be postponed, even by disabling interrupts.** These interrupts, called _non-maskable interrupts_ (NMIs), are supposed to be used only in emergencies, e.g. when the computer is on fire. **Pintos does not handle non-maskable interrupts.**

## Semaphores

Semaphores were invented by Edsger Dijkstra and first used in the `THE operating system`. **A **_**semaphore**_** is a nonnegative integer together with two operators that manipulate it atomically**, which are:

1. <mark style="color:blue;">**"Down" or "P"**</mark>
   * wait for the value to become positive, then decrement it.
2. <mark style="color:blue;">**"Up" or "V"**</mark>
   * increment the value (and wake up one waiting thread, if any).

**A semaphore initialized to 0 may be used to wait for an event that will happen exactly once.**

* For example, suppose thread A starts another thread B and wants to wait for B to signal that some activity is complete.&#x20;
* A can create a semaphore initialized to 0, pass it to B as it starts it, and then "down" the semaphore. When B finishes its activity, it "ups" the semaphore. This works regardless of whether A "downs" the semaphore or B "ups" it first.

**A semaphore initialized to 1 is typically used for controlling access to a resource.**&#x20;

* Before a block of code starts using the resource, it "downs" the semaphore, then after it is done with the resource it "ups" the resource.&#x20;
* In such a case **a lock**, described below, may be more appropriate.

**Semaphores can also be initialized to values larger than 1.** These are rarely used.

### Types and Functions

Pintos' semaphore type and operations are declared in **`threads/synch.h`**.

<details>

<summary>Types and Functions for semaphore</summary>

* <mark style="color:blue;">**Type: struct semaphore**</mark>
  * **Represents a semaphore.**
* <mark style="color:blue;">**Function: void sema\_init**</mark> <mark style="color:blue;"></mark><mark style="color:blue;"></mark> <mark style="color:blue;"></mark><mark style="color:blue;">**(struct semaphore \*sema, unsigned value)**</mark>
  * **Initializes sema as a new semaphore with the given initial value.**
* <mark style="color:blue;">**Function: void**</mark> <mark style="color:blue;"></mark><mark style="color:blue;"></mark> <mark style="color:blue;"></mark><mark style="color:blue;">**sema\_down**</mark> <mark style="color:blue;"></mark><mark style="color:blue;"></mark> <mark style="color:blue;"></mark><mark style="color:blue;">**(struct semaphore \*sema)**</mark>
  * **Executes the "down" or "P" operation on sema**, waiting for its value to become positive and then decrementing it by one.
* <mark style="color:blue;">**Function: bool**</mark> <mark style="color:blue;"></mark><mark style="color:blue;"></mark> <mark style="color:blue;"></mark><mark style="color:blue;">**sema\_try\_down**</mark> <mark style="color:blue;"></mark><mark style="color:blue;"></mark> <mark style="color:blue;"></mark><mark style="color:blue;">**(struct semaphore \*sema)**</mark>
  * **Tries to execute the "down" or "P" operation on sema, without waiting.**&#x20;
  * Returns true if sema was successfully decremented, or false if it was already zero and thus could not be decremented without waiting.&#x20;
  * Calling this function in a tight loop wastes CPU time, so use `sema_down()` or find a different approach instead.
* <mark style="color:blue;">**Function: void sema\_up**</mark> <mark style="color:blue;"></mark><mark style="color:blue;"></mark> <mark style="color:blue;"></mark><mark style="color:blue;">**(struct semaphore \*sema)**</mark>
  * **Executes the "up" or "V" operation on sema**, incrementing its value. If any threads are waiting on sema, wakes one of them up.
  * <mark style="color:red;">**Unlike most synchronization primitives,**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**`sema_up()`**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**may be called inside an external interrupt handler**</mark> (see section [External Interrupt Handling](interrupt-handling.md#external-interrupt-handling)).

</details>

### Notes

* **Semaphores are internally built out of disabling interrupt** (see section [Disabling Interrupts](synchronization.md#disabling-interrupts)) **and thread blocking and unblocking** (`thread_block()` and `thread_unblock()`).&#x20;
* Each semaphore maintains **a list of waiting threads**, using the linked list implementation in **`lib/kernel/list.c`**.

## Locks

**A **_**lock**_** is like a semaphore with an initial value of 1** (see section [Semaphores](synchronization.md#semaphores)). A lock's equivalent of "up" is called **"release"**, and the "down" operation is called **"acquire"**.

### Types and Functions

Lock types and functions are declared in **`threads/synch.h`**.

<details>

<summary>Types and Functions for locks</summary>

* <mark style="color:blue;">**Type:**</mark> <mark style="color:blue;"></mark><mark style="color:blue;"></mark> <mark style="color:blue;"></mark><mark style="color:blue;">**struct lock**</mark>
  * **Represents a lock.**
* <mark style="color:blue;">**Function: void lock\_init**</mark> <mark style="color:blue;"></mark><mark style="color:blue;"></mark> <mark style="color:blue;"></mark><mark style="color:blue;">**(struct lock \*lock)**</mark>
  * **Initializes lock as a new lock.** The lock is not initially owned by any thread.
* <mark style="color:blue;">**Function: void lock\_acquire (struct lock \*lock)**</mark>
  * **Acquires lock for the current thread**, first waiting for any current owner to release it if necessary.
* <mark style="color:blue;">**Function: bool lock\_try\_acquire (struct lock \*lock)**</mark>
  * **Tries to acquire lock for use by the current thread, without waiting.**
  * Returns true if successful, false if the lock is already owned.&#x20;
  * Calling this function in a tight loop is a bad idea because it wastes CPU time, so use `lock_acquire()` instead.
* <mark style="color:blue;">**Function: void**</mark> <mark style="color:blue;"></mark><mark style="color:blue;"></mark> <mark style="color:blue;"></mark><mark style="color:blue;">**lock\_release**</mark> <mark style="color:blue;"></mark><mark style="color:blue;"></mark> <mark style="color:blue;"></mark><mark style="color:blue;">**(struct lock \*lock)**</mark>
  * **Releases lock, which the current thread must own.**
* <mark style="color:blue;">**Function: bool lock\_held\_by\_current\_thread**</mark> <mark style="color:blue;"></mark><mark style="color:blue;"></mark> <mark style="color:blue;"></mark><mark style="color:blue;">**(const struct lock \*lock)**</mark>
  * **Returns true if the running thread owns lock, false otherwise.**&#x20;
  * There is no function to test whether an arbitrary thread owns a lock, because the answer could change before the caller could act on it.

</details>

### Notes

* Compared to a semaphore, a lock has one added restriction: **only the thread that acquires a lock, called the lock's "owner", is allowed to release it.** If this restriction is a problem, it's a good sign that a semaphore should be used, instead of a lock.
* <mark style="color:red;">**Locks in Pintos are not "recursive,"**</mark> that is, it is an error for the thread currently holding a lock to try to acquire that lock.

## Monitors

**A **_**monitor**_** is a higher-level form of synchronization than a semaphore or a lock.**&#x20;

* A monitor consists of **data** **being synchronized**, plus **a lock, called the **_**monitor lock**_, and **one or more **_**condition variables**_.&#x20;
* **Before it accesses the protected data, a thread first acquires the monitor lock.** It is then said to be "in the monitor".&#x20;
* While in the monitor, the thread has control over all the protected data, which it may freely examine or modify. **When access to the protected data is complete, it releases the monitor lock.**
* **Condition variables allow code in the monitor to wait for a condition to become true.** Each condition variable is associated with an abstract condition, e.g. "some data has arrived for processing" or "over 10 seconds has passed since the user's last keystroke".&#x20;
  * When code in the monitor needs to wait for a condition to become true, it **"waits"** on the associated condition variable, which releases the lock and waits for the condition to be signaled.&#x20;
  * If, on the other hand, it has caused one of these conditions to become true, it **"signals"** the condition to wake up one waiter, or **"broadcasts"** the condition to wake all of them.

### Types and Functions

Condition variable types and functions are declared in **`threads/synch.h`**.

<details>

<summary>Types and Functions for Condition Variable</summary>

* <mark style="color:blue;">**Type: struct condition**</mark>
  * **Represents a condition variable.**
* <mark style="color:blue;">**Function: void**</mark> <mark style="color:blue;"></mark><mark style="color:blue;"></mark> <mark style="color:blue;"></mark><mark style="color:blue;">**cond\_init**</mark> <mark style="color:blue;"></mark><mark style="color:blue;"></mark> <mark style="color:blue;"></mark><mark style="color:blue;">**(struct condition \*cond)**</mark>
  * **Initializes cond as a new condition variable.**
* <mark style="color:blue;">**Function: void**</mark> <mark style="color:blue;"></mark><mark style="color:blue;"></mark> <mark style="color:blue;"></mark><mark style="color:blue;">**cond\_wait**</mark> <mark style="color:blue;"></mark><mark style="color:blue;"></mark> <mark style="color:blue;"></mark><mark style="color:blue;">**(struct condition \*cond, struct lock \*lock)**</mark>
  * **Atomically releases lock (the monitor lock) and waits for cond to be signaled by some other piece of code.**&#x20;
  * **After cond is signaled, reacquires lock** **before returning**.
  * lock must be held before calling this function.
  * **Sending a signal and waking up from a wait are not an atomic operation.** Thus, typically `cond_wait()`'s caller must **recheck the condition** after the wait completes and, if necessary, wait again. See the next section for an example.
* <mark style="color:blue;">**Function: void cond\_signal**</mark> <mark style="color:blue;"></mark><mark style="color:blue;"></mark> <mark style="color:blue;"></mark><mark style="color:blue;">**(struct condition \*cond, struct lock \*lock)**</mark>
  * **If any threads are waiting on cond (protected by monitor lock lock), then this function wakes up one of them.** If no threads are waiting, returns without performing any action.&#x20;
  * lock must be held before calling this function.
* <mark style="color:blue;">**Function: void**</mark> <mark style="color:blue;"></mark><mark style="color:blue;"></mark> <mark style="color:blue;"></mark><mark style="color:blue;">**cond\_broadcast**</mark> <mark style="color:blue;"></mark><mark style="color:blue;"></mark> <mark style="color:blue;"></mark><mark style="color:blue;">**(struct condition \*cond, struct lock \*lock)**</mark>
  * **Wakes up all threads, if any, waiting on cond (protected by monitor lock lock).**&#x20;
  * lock must be held before calling this function.

</details>

### Notes

* The theoretical framework for monitors was laid out by **`C. A. R. Hoare`**.&#x20;
* Their practical usage was later elaborated in a paper on the **`Mesa`** operating system.

### Monitor example

**The classical example of a monitor is handling a buffer into which one or more "producer" threads write characters and out of which one or more "consumer" threads read characters.**&#x20;

To implement this we need, besides the monitor lock, two condition variables which we will call not\_full and not\_empty:

```cpp
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

* Note that `BUF_SIZE` must divide evenly into `SIZE_MAX + 1` for the above code to be completely correct. Otherwise, it will fail the first time `head` wraps around to 0.
* In practice, `BUF_SIZE` would ordinarily be a power of 2.

## Optimization Barrier

**An **_**optimization barrier**_** is a special statement that prevents the compiler from making assumptions about the state of memory across the barrier.**&#x20;

* The compiler will not reorder reads or writes of variables across the barrier or assume that a variable's value is unmodified across the barrier, except for local variables whose address is never taken.&#x20;
* In Pintos, `threads/synch.h` defines **the `barrier()` macro** as an optimization barrier.

**One reason to use an optimization barrier is when data can change asynchronously, without the compiler's knowledge**, e.g. by another thread or an interrupt handler.&#x20;

* The **`too_many_loops()`** function in `devices/timer.c` is an example. This function starts out by busy-waiting in a loop until a timer tick occurs:

```cpp
/* Wait for a timer tick. */
int64_t start = ticks;
while (ticks == start)
  barrier ();
```

* Without an optimization barrier in the loop, the compiler could conclude that the loop would never terminate, because `start` and `ticks` start out equal and the loop itself never changes them. It could then "optimize" the function into an infinite loop, which would definitely be undesirable.

**Optimization barriers can be used to avoid other compiler optimizations.**

* The **`busy_wait()`** function, also in `devices/timer.c`, is an example. It contains this loop:

```cpp
while (loops-- > 0)
  barrier ();
```

* The goal of this loop is to busy-wait by counting `loops` down from its original value to 0. Without the barrier, the compiler could delete the loop entirely, because it produces no useful output and has no side effects. The barrier forces the compiler to pretend that the loop body has an important effect.

**Finally, optimization barriers can be used to force the ordering of memory reads or writes.**

* For example, suppose we add a "feature" that, whenever a timer interrupt occurs, the character in the global variable `timer_put_char` is printed on the console, but only if the global Boolean variable `timer_do_put` is true. The best way to set up x to be printed is then to use an optimization barrier, like this:

```cpp
timer_put_char = 'x';
barrier ();
timer_do_put = true;
```

* Without the barrier, the code is buggy because the compiler is free to reorder operations when it doesn't see a reason to keep them in the same order. In this case, the compiler doesn't know that the order of assignments is important, so its optimizer is permitted to exchange their order. There's no telling whether it will actually do this, and it is possible that passing the compiler different optimization flags or using a different version of the compiler will produce different behavior.
* **Another solution is to disable interrupts around the assignments.** This does not prevent reordering, but it prevents the interrupt handler from intervening between the assignments. It also has the extra runtime cost of disabling and re-enabling interrupts:

```cpp
enum intr_level old_level = intr_disable ();
timer_put_char = 'x';
timer_do_put = true;
intr_set_level (old_level);
```

* **A second solution is to mark the declarations of `timer_put_char` and `timer_do_put` as volatile.** This keyword tells the compiler that the variables are **externally observable** and restricts its latitude for optimization. However, the semantics of volatile are not well-defined, so it is not a good general solution. **The base Pintos code does not use volatile at all.**
* The following is _not_ a solution, because locks neither prevent interrupts nor prevent the compiler from reordering the code within the region where the lock is held:

```cpp
lock_acquire (&timer_lock);     /* INCORRECT CODE */
timer_put_char = 'x';
timer_do_put = true;
lock_release (&timer_lock);
```

**The compiler treats invocation of any function defined externally, that is, in another source file, as a limited form of optimization barrier.** Specifically, the compiler assumes that any externally defined function may access any statically or dynamically allocated data and any local variable whose address is taken. This often means that explicit barriers can be omitted. It is one reason that Pintos contains few explicit barriers.

**A function defined in the same source file, or in a header included by the source file, cannot be relied upon as an optimization barrier.** This applies even to the invocation of a function before its definition, because the compiler may read and parse the entire source file before performing optimization.
