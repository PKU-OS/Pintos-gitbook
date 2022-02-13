# Your Tasks

## Task 0: Design Document

Download the [project 1 design document template](https://github.com/PKU-OS/sp22/blob/main/assets/docs-template/p1.md). Read through it to motivate your design and fill it in after you finish the project. We recommend that you read the design document template before you start working on the project. See section [Project Documentation](../../appendix/project-documentation.md), for a sample design document that goes along with a fictitious project.

## Task 1: Alarm Clock

{% hint style="success" %}
**Exercise 1.1**

Reimplement `timer_sleep()`, defined in devices/timer.c. Although a working implementation is provided, it "busy waits," that is, it spins in a loop checking the current time and calling `thread_yield()` until enough time has gone by. Reimplement it to avoid busy waiting.
{% endhint %}

#### Function: void **timer\_sleep** (int64\_t ticks)

Suspends execution of the calling thread until time has advanced by at least _ticks_ timer ticks. Unless the system is otherwise idle, the thread need not wake up after exactly _ticks_. Just put it on the ready queue after they have waited for the right amount of time.

`timer_sleep()` is useful for threads that operate in real-time, e.g. for blinking the cursor once per second.

The argument to `timer_sleep()` is expressed in timer ticks, not in milliseconds or any another unit. There are `TIMER_FREQ` timer ticks per second, where `TIMER_FREQ` is a macro defined in `devices/timer.h`. The default value is 100. We don't recommend changing this value, because any change is likely to cause many of the tests to fail.

Separate functions `timer_msleep()`, `timer_usleep()`, and `timer_nsleep()` do exist for sleeping a specific number of milliseconds, microseconds, or nanoseconds, respectively, but these will call `timer_sleep()` automatically when necessary. You do not need to modify them.

If your delays seem too short or too long, reread the explanation of the `-r` option to `pintos` (see section [Debugging versus Testing](../../getting-started/debug-and-test/#debugging-versus-testing)).

{% hint style="info" %}
**Hint**

You may find `struct list` and the provided functions to be useful for this exercise. Read the comments in `lib/kernel/list.h` carefully, since this list design/usage is different from the typical linked list you are familiar with (actually, Linux kernel [uses a similar design](https://0xax.gitbooks.io/linux-insides/content/DataStructures/linux-datastructures-1.html)). Searching the Pintos codebase to see how `struct list` is used may also give you some inspiration.
{% endhint %}

{% hint style="info" %}
**Hint**

You may want to leverage some synchronization primitive that provides some sort of thread _"waiting"_ functionality, e.g., semaphore. You do not have to wait for the Synchronization Lecture to be able to use these primitives. Reading through section [Synchronization](../../appendix/reference-guide/synchronization.md) is sufficient. In addition, when modifying some global variable, e.g., a global list, you will need to use some synchronization primitive as well to ensure it is not modified or read concurrently (e.g., a timer interrupt occurs during the modification and we switch to run another thread).
{% endhint %}

{% hint style="info" %}
**Hint**

You need to decide where to check whether the elapsed time exceeded the sleep time. The original `timer_sleep` implementation calls `timer_ticks`, which returns the current `ticks`. Check where the static `ticks` variable is _updated_. You can search with `grep` or `rg` to help you find this out (see [Development Tools](../../appendix/development-tools.md) for more details).
{% endhint %}

The alarm clock implementation is _not_ needed for later projects, although it could be useful for project 4.

## Task 2: Priority Scheduling

{% hint style="success" %}
**Exercise 2.1**

Implement priority scheduling in Pintos. When a thread is added to the ready list that has a higher priority than the currently running thread, the current thread should immediately yield the processor to the new thread. Similarly, when threads are waiting for a lock, semaphore, or condition variable, the highest priority waiting thread should be awakened first. A thread may raise or lower its own priority at any time, but lowering its priority such that it no longer has the highest priority must cause it to immediately yield the CPU.
{% endhint %}

Thread priorities range from `PRI_MIN` (0) to `PRI_MAX` (63). Lower numbers correspond to lower priorities, so that priority 0 is the lowest priority and priority 63 is the highest. The initial thread priority is passed as an argument to `thread_create()`. If there's no reason to choose another priority, use `PRI_DEFAULT` (31). The `PRI_` macros are defined in `threads/thread.h`, and you should not change their values.

{% hint style="info" %}
**Hint**

For this exercise, you need to consider all the scenarios where the priority must be enforced. For example, when an alarm clock for a thread fires off, that thread should be made ready again, which entails a priority check. You can find some of these scenarios by looking for places that modify `ready_list` (directly and indirectly, rg can be helpful).
{% endhint %}

{% hint style="info" %}
**Hint**

To yield the CPU, you can check the thread APIs in `threads/thread.h`. Read the comment and implementation of the corresponding thread function in `threads/thread.c`. That function may not be used in interrupt context (i.e., should not call it inside an interrupt handler). To yield the CPU in the interrupt context, you can take a look at functions in `threads/interrupt.c`.&#x20;
{% endhint %}

One issue with priority scheduling is "priority inversion". Consider high, medium, and low priority threads H, M, and L, respectively. If H needs to wait for L (for instance, for a lock held by L), and M is on the ready list, then H will never get the CPU because the low priority thread will not get any CPU time. A partial fix for this problem is for H to "donate" its priority to L while L is holding the lock, then recall the donation once L releases (and thus H acquires) the lock.

{% hint style="success" %}
**Exercise 2.2.1**

Implement priority donation. You will need to account for all different situations in which priority donation is required. You must implement priority donation for locks. You need not implement priority donation for the other Pintos synchronization constructs. You do need to implement priority scheduling in all cases. Be sure to handle multiple donations, in which multiple priorities are donated to a single thread.
{% endhint %}

{% hint style="success" %}
**Exercise 2.2.2**

Support nested priority donation: if H is waiting on a lock that M holds and M is waiting on a lock that L holds, then both M and L should be boosted to H's priority. If necessary, you may impose a reasonable limit on depth of nested priority donation, such as 8 levels.
{% endhint %}

Note: if you support nested priority donation, you need to pass the `priority-donate-nest` and `priority-donate-chain` tests.

{% hint style="success" %}
**Exercise 2.3**

Implement the following functions that allow a thread to examine and modify its own priority. Skeletons for these functions are provided in threads/thread.c.
{% endhint %}

#### Function: void **thread\_set\_priority** (int new\_priority)

Sets the current thread's priority to _new\_priority_. If the current thread no longer has the highest priority, yields.

#### Function: int **thread\_get\_priority** (void)

Returns the current thread's priority. In the presence of priority donation, returns the higher (donated) priority.

You need not provide any interface to allow a thread to directly modify other threads' priorities.

The priority scheduler is not used in any later project.

## Task3: Advanced Scheduler

{% hint style="success" %}
**Exercise 3.1**

Implement a multilevel feedback queue scheduler similar to the 4.4BSD scheduler to reduce the average response time for running jobs on your system. See section [4.4BSD Scheduler](../../appendix/4.4bsd-scheduler.md), for detailed requirements.
{% endhint %}

Like the priority scheduler, the advanced scheduler chooses the thread to run based on priorities. However, the advanced scheduler does not do priority donation. Thus, we recommend that you have the priority scheduler working, except possibly for priority donation, before you start work on the advanced scheduler.

You must write your code to allow us to choose a scheduling algorithm policy at Pintos startup time. By default, the priority scheduler must be active, but we must be able to choose the 4.4BSD scheduler with the `-mlfqs` kernel option. Passing this option sets `thread_mlfqs`, declared in `threads/thread.h`, to true when the options are parsed by `parse_options()`, which happens early in `pintos_init()`.

When the 4.4BSD scheduler is enabled, threads no longer directly control their own priorities. The priority argument to `thread_create()` should be ignored, as well as any calls to `thread_set_priority()`, and `thread_get_priority()` should return the thread's current priority as set by the scheduler.

{% hint style="info" %}
**Hint**

Double check the implementations of your fixed-point arithmetic routines (and ideally have some unit test for them). Some simple mistake in these routines could result in mysterious issues in your scheduler.
{% endhint %}

{% hint style="info" %}
**Hint**

Efficiency matters a lot for the MLFQS exercise. An inefficient implementation can distort the system. Read the comment in the test case `mlfqs-load-avg.c`. In fact, the inefficiency in your alarm clock implementation can also influence your MLFQS behavior. So double-check if your implementation there can be optimized.
{% endhint %}

The advanced scheduler is not used in any later project.
