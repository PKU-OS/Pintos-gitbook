# Your Tasks

## Task 0: Design Document

* Download the [project 1 design document template](https://github.com/PKU-OS/pintos/blob/master/docs/p1.md). Read through it to motivate your design and fill it in after you finish the project.&#x20;
* We recommend that you read the design document template before you start working on the project.&#x20;
* See section [Project Documentation](../../appendix/project-documentation.md), for a sample design document that goes along with a fictitious project.

## Task 1: Alarm Clock

### Exercise 1.1

{% hint style="success" %}
<mark style="color:green;">**Exercise 1.1**</mark>

<mark style="color:green;">Reimplement</mark> <mark style="color:green;"></mark><mark style="color:green;">**`timer_sleep()`**</mark><mark style="color:green;">, defined in</mark> <mark style="color:green;"></mark><mark style="color:green;">`devices/timer.c`</mark><mark style="color:green;">.</mark>

* <mark style="color:green;">Although a working implementation is provided, it "busy waits," that is, it spins in a loop checking the current time and calling</mark> <mark style="color:green;"></mark><mark style="color:green;">`thread_yield()`</mark> <mark style="color:green;"></mark><mark style="color:green;">until enough time has gone by.</mark>&#x20;
* <mark style="color:green;">Reimplement it to</mark> <mark style="color:green;"></mark><mark style="color:green;">**avoid busy waiting**</mark><mark style="color:green;">.</mark>
{% endhint %}

* <mark style="color:blue;">**Function: void timer\_sleep (int64\_t ticks)**</mark>
  * **Suspends execution of the calling thread until time has advanced by at least **_**ticks**_** timer ticks.**&#x20;
  * Unless the system is otherwise idle, the thread need not wake up after exactly _ticks_. Just put it on the ready queue after they have waited for the right amount of time.
  * `timer_sleep()` is useful for threads that operate in real-time, e.g. for blinking the cursor once per second.
  * **The argument to `timer_sleep()` is expressed in **_**timer ticks**_**, not in milliseconds or any another unit.** There are `TIMER_FREQ` timer ticks per second, where `TIMER_FREQ` is a macro defined in `devices/timer.h`. The default value is 100. We don't recommend changing this value, because any change is likely to cause many of the tests to fail.
  * Separate functions **`timer_msleep()`, `timer_usleep()`**, and **`timer_nsleep()`** do exist for sleeping a specific number of milliseconds, microseconds, or nanoseconds, respectively, but **these will call `timer_sleep()`** automatically when necessary. You do not need to modify them.

If your delays seem too short or too long, reread the explanation of the `-r` option to `pintos` (see section [Debugging versus Testing](../../getting-started/debug-and-test/#debugging-versus-testing)).

{% hint style="info" %}
**Hint**

<mark style="color:red;">**You may find**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**`struct list`**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**and the provided functions to be useful for this exercise.**</mark>&#x20;

* Read the comments in `lib/kernel/list.h` carefully, since this list design/usage is different from the typical linked list you are familiar with (actually, Linux kernel [uses a similar design](https://0xax.gitbooks.io/linux-insides/content/DataStructures/linux-datastructures-1.html)).&#x20;
* Searching the Pintos codebase to see how `struct list` is used may also give you some inspiration.
{% endhint %}

{% hint style="info" %}
**Hint**

* You may want to leverage some synchronization primitive that provides some sort of thread _"waiting"_ functionality, e.g., semaphore.&#x20;
* **You do not have to wait for the Synchronization Lecture to be able to use these primitives.** Reading through section [Synchronization](../../appendix/reference-guide/synchronization.md) is sufficient.&#x20;
* In addition, when modifying some global variable, e.g., a global list, you will need to use some synchronization primitive as well to ensure it is not modified or read concurrently (e.g., a timer interrupt occurs during the modification and we switch to run another thread).
{% endhint %}

{% hint style="info" %}
**Hint**

**You need to decide where to check whether the elapsed time exceeded the sleep time.**&#x20;

* The original `timer_sleep` implementation calls `timer_ticks`, which returns **the current `ticks`**.&#x20;
* **Check where the static `ticks` variable is **_**updated**_**.** You can search with `grep` or `rg` to help you find this out (see [Development Tools](../../appendix/development-tools.md) for more details).
{% endhint %}

<mark style="color:red;">**The alarm clock implementation is**</mark><mark style="color:red;">** **</mark>_<mark style="color:red;">**not**</mark>_<mark style="color:red;">** **</mark><mark style="color:red;">**needed for later projects**</mark>, although it could be useful for project 4.

## Task 2: Priority Scheduling

### Exercise 2.1

{% hint style="success" %}
<mark style="color:green;">**Exercise 2.1**</mark>

<mark style="color:green;">**Implement**</mark><mark style="color:green;">** **</mark>_<mark style="color:green;">**priority scheduling**</mark>_<mark style="color:green;">** **</mark><mark style="color:green;">**in Pintos.**</mark> <mark style="color:green;"></mark><mark style="color:green;"></mark>&#x20;

* <mark style="color:green;">When a thread is added to the ready list that has a higher priority than the currently running thread, the current thread should</mark> <mark style="color:green;"></mark><mark style="color:green;">**immediately yield**</mark> <mark style="color:green;"></mark><mark style="color:green;">the processor to the new thread.</mark>&#x20;
* <mark style="color:green;">Similarly, when threads are waiting for a lock, semaphore, or condition variable,</mark> <mark style="color:green;"></mark><mark style="color:green;">**the highest priority waiting thread should be awakened first**</mark><mark style="color:green;">.</mark>&#x20;
* <mark style="color:green;">A thread may raise or lower its own priority at any time, but</mark> <mark style="color:green;"></mark><mark style="color:green;">**lowering its priority such that it no longer has the highest priority must cause it to immediately yield the CPU**</mark><mark style="color:green;">.</mark>
{% endhint %}

* **Thread priorities range from `PRI_MIN` (0) to `PRI_MAX` (63).** Lower numbers correspond to lower priorities, so that priority 0 is the lowest priority and priority 63 is the highest.&#x20;
* **The initial thread priority is passed as an argument to `thread_create()`**. If there's no reason to choose another priority, use **`PRI_DEFAULT` (31)**.&#x20;
* The `PRI_` macros are defined in `threads/thread.h`, and you should not change their values.

{% hint style="info" %}
**Hint**

For this exercise, **you need to consider **_**all the scenarios**_** where the priority must be enforced**.&#x20;

* For example, **when an alarm clock for a thread fires off**, that thread should be made ready again, which entails a priority check.&#x20;
* You can find some of these scenarios by **looking for places that modify `ready_list`** (directly and indirectly, rg can be helpful).
{% endhint %}

{% hint style="info" %}
**Hint**

* **To yield the CPU,** you can check the thread APIs in **`threads/thread.h`**.&#x20;
  * Read the comment and implementation of the corresponding thread function in `threads/thread.c`.&#x20;
  * **That function may not be used in interrupt context** (i.e., should not call it inside an interrupt handler).&#x20;
* **To yield the CPU **_<mark style="color:red;">**in the interrupt context**</mark>_, you can take a look at functions in **`threads/interrupt.c`**.
{% endhint %}

### Exercise 2.2.1

**One issue with priority scheduling is "priority inversion".**

* Consider high, medium, and low priority threads H, M, and L, respectively.&#x20;
* If H needs to wait for L (for instance, for a lock held by L), and M is on the ready list, then H will never get the CPU because the low priority thread will not get any CPU time.&#x20;
* **A partial fix for this problem is for H to "donate" its priority to L** while L is holding the lock, then **recall the donation** once L releases (and thus H acquires) the lock.

{% hint style="success" %}
<mark style="color:green;">**Exercise 2.2.1**</mark>

<mark style="color:green;">**Implement priority donation.**</mark>&#x20;

* <mark style="color:green;">You will need to</mark> <mark style="color:green;"></mark><mark style="color:green;">**account for all different situations in which priority donation is required**</mark><mark style="color:green;">.</mark>&#x20;
* <mark style="color:green;">You must implement priority donation</mark> <mark style="color:green;"></mark><mark style="color:green;">**for locks**</mark><mark style="color:green;">. You need</mark> <mark style="color:green;"></mark><mark style="color:green;">**not**</mark> <mark style="color:green;"></mark><mark style="color:green;">implement priority donation for the other Pintos synchronization constructs.</mark>&#x20;
* <mark style="color:green;">You do need to</mark> <mark style="color:green;"></mark><mark style="color:green;">**implement**</mark><mark style="color:green;">** **</mark>_<mark style="color:green;">**priority scheduling**</mark>_<mark style="color:green;">** **</mark><mark style="color:green;">**in all cases**</mark><mark style="color:green;">.</mark>&#x20;
* <mark style="color:green;">Be sure to</mark> <mark style="color:green;"></mark><mark style="color:green;">**handle multiple donations**</mark><mark style="color:green;">, in which multiple priorities are donated to a single thread</mark>.
{% endhint %}

### Exercise 2.2.2

{% hint style="success" %}
<mark style="color:green;">**Exercise 2.2.2**</mark>

<mark style="color:green;">**Support nested priority donation:**</mark>&#x20;

* <mark style="color:green;">if H is waiting on a lock that M holds and M is waiting on a lock that L holds, then both M and L should be boosted to H's priority.</mark>&#x20;
* <mark style="color:green;">If necessary, you may impose</mark> <mark style="color:green;"></mark><mark style="color:green;">**a reasonable limit**</mark> <mark style="color:green;"></mark><mark style="color:green;">on depth of nested priority donation, such as 8 levels.</mark>
{% endhint %}

**Note:** if you support nested priority donation, you need to pass the `priority-donate-nest` and `priority-donate-chain` tests.

### Exercise 2.3

{% hint style="success" %}
<mark style="color:green;">**Exercise 2.3**</mark>

<mark style="color:green;">**Implement the following functions that allow a thread to examine and modify its own priority.**</mark>&#x20;

<mark style="color:green;">Skeletons for these functions are provided in</mark> <mark style="color:green;"></mark><mark style="color:green;">**threads/thread.c**</mark><mark style="color:green;">.</mark>
{% endhint %}

* <mark style="color:blue;">**Function: void thread\_set\_priority (int new\_priority)**</mark>
  * **Sets the current thread's priority to **_**new\_priority**_**.**&#x20;
  * If the current thread no longer has the highest priority, **yields**.
* <mark style="color:blue;">**Function: int thread\_get\_priority (void)**</mark>
  * **Returns the current thread's priority.** In the presence of priority donation, returns the higher (donated) priority.

You need not provide any interface to allow a thread to directly modify other threads' priorities.

<mark style="color:red;">**The priority scheduler is not used in any later project.**</mark>

## Task3: Advanced Scheduler

### Exercise 3.1

{% hint style="success" %}
<mark style="color:green;">**Exercise 3.1**</mark>

<mark style="color:green;">**Implement a multilevel feedback queue scheduler**</mark> <mark style="color:green;"></mark><mark style="color:green;">similar to the 4.4BSD scheduler to reduce the average response time for running jobs on your system.</mark>&#x20;

<mark style="color:green;">See section</mark> [<mark style="color:green;">**4.4BSD Scheduler**</mark>](../../appendix/4.4bsd-scheduler.md)<mark style="color:green;">, for detailed requirements</mark>.
{% endhint %}

Like the priority scheduler, the advanced scheduler chooses the thread to run based on priorities. However, **the advanced scheduler does not do priority donation**.&#x20;

* Thus, we recommend that you have the priority scheduler working, except possibly for priority donation, before you start work on the advanced scheduler.

**You must write your code to allow us to choose a scheduling algorithm policy at Pintos startup time.**&#x20;

* **By default, the priority scheduler must be active, but we must be able to choose the 4.4BSD scheduler with the `-mlfqs` kernel option.** Passing this option sets **`thread_mlfqs`**, declared in `threads/thread.h`, to true when the options are parsed by `parse_options()`, which happens early in `pintos_init()`.

When the 4.4BSD scheduler is enabled, threads no longer directly control their own priorities.&#x20;

* **The priority argument to `thread_create()` should be ignored, as well as any calls to `thread_set_priority()`**,&#x20;
* and **`thread_get_priority()` should return the thread's current priority as set by the scheduler**.

{% hint style="info" %}
**Hint**

**Double check the implementations of your fixed-point arithmetic routines** (and ideally have some unit test for them).&#x20;

Some simple mistake in these routines could result in mysterious issues in your scheduler.
{% endhint %}

{% hint style="info" %}
**Hint**

<mark style="color:red;">**Efficiency matters a lot for the MLFQS exercise.**</mark>&#x20;

* An inefficient implementation can distort the system.&#x20;
* Read the comment in the test case **`mlfqs-load-avg.c`**.&#x20;
* In fact, the inefficiency in your alarm clock implementation can also influence your MLFQS behavior.
* So double-check if your implementation there can be optimized.
{% endhint %}

<mark style="color:red;">**The advanced scheduler is not used in any later project.**</mark>
