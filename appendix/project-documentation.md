# Project Documentation

{% hint style="info" %}
This chapter presents a sample assignment and a filled-in design document for one possible implementation. Its purpose is to give you an idea of what we expect to see in your own design documents.
{% endhint %}

## Sample Assignment

{% hint style="success" %}
Implement `thread_join()`.
{% endhint %}

#### Function: void **thread\_join** (tid\_t tid)

Blocks the current thread until thread tid exits. If A is the running thread and B is the argument, then we say that "A joins B."

Incidentally, the argument is a thread id, instead of a thread pointer, because a thread pointer is not unique over time. That is, when a thread dies, its memory may be, whether immediately or much later, reused for another thread. If thread A over time had two children B and C that were stored at the same address, then `thread_join(B)` and `thread_join(C)` would be ambiguous.

A thread may only join its immediate children. Calling `thread_join()` on a thread that is not the caller's child should cause the caller to return immediately. Children are not "inherited," that is, if A has child B and B has child C, then A always returns immediately should it try to join C, even if B is dead.

A thread need not ever be joined. Your solution should properly free all of a thread's resources, including its `struct thread`, whether it is ever joined or not, and regardless of whether the child exits before or after its parent. That is, a thread should be freed exactly once in all cases.

Joining a given thread is idempotent. That is, joining a thread multiple times is equivalent to joining it once, because it has already exited at the time of the later joins. Thus, joins on a given thread after the first should return immediately.

You must handle all the ways a join can occur: nested joins (A joins B, then B joins C), multiple joins (A joins B, then A joins C), and so on.

## Sample Design Document

```
                         +-----------------+
                         |      CS 140     |
                         |  SAMPLE PROJECT |
                         | DESIGN DOCUMENT |
                         +-----------------+

---- GROUP ----

Ben Pfaff <blp@stanford.edu>

---- PRELIMINARIES ----

>> If you have any preliminary comments on your submission, notes for
>> the TAs, or extra credit, please give them here.

(This is a sample design document.)

>> Please cite any offline or online sources you consulted while
>> preparing your submission, other than the Pintos documentation,
>> course text, and lecture notes.

None.

                                 JOIN
                                 ====

---- DATA STRUCTURES ----

>> Copy here the declaration of each new or changed `struct' or `struct'
>> member, global or static variable, `typedef', or enumeration.
>> Identify the purpose of each in 25 words or less.

A "latch" is a new synchronization primitive.  Acquires block
until the first release.  Afterward, all ongoing and future
acquires pass immediately.

    /* Latch. */
    struct latch 
      {
        bool released;              /* Released yet? */
        struct lock monitor_lock;   /* Monitor lock. */
        struct condition rel_cond;  /* Signaled when released. */
      };

Added to struct thread:

    /* Members for implementing thread_join(). */
    struct latch ready_to_die;   /* Release when thread about to die. */
    struct semaphore can_die;    /* Up when thread allowed to die. */
    struct list children;        /* List of child threads. */
    list_elem children_elem;     /* Element of `children' list. */

---- ALGORITHMS ----

>> Briefly describe your implementation of thread_join() and how it
>> interacts with thread termination.

thread_join() finds the joined child on the thread's list of
children and waits for the child to exit by acquiring the child's
ready_to_die latch.  When thread_exit() is called, the thread
releases its ready_to_die latch, allowing the parent to continue.

---- SYNCHRONIZATION ----

>> Consider parent thread P with child thread C.  How do you ensure
>> proper synchronization and avoid race conditions when P calls wait(C)
>> before C exits?  After C exits?  How do you ensure that all resources
>> are freed in each case?  How about when P terminates without waiting,
>> before C exits?  After C exits?  Are there any special cases?

C waits in thread_exit() for P to die before it finishes its own
exit, using the can_die semaphore "down"ed by C and "up"ed by P as
it exits.  Regardless of whether whether C has terminated, there
is no race on wait(C), because C waits for P's permission before
it frees itself.

Regardless of whether P waits for C, P still "up"s C's can_die
semaphore when P dies, so C will always be freed.  (However,
freeing C's resources is delayed until P's death.)

The initial thread is a special case because it has no parent to
wait for it or to "up" its can_die semaphore.  Therefore, its
can_die semaphore is initialized to 1.

---- RATIONALE ----

>> Critique your design, pointing out advantages and disadvantages in
>> your design choices.

This design has the advantage of simplicity.  Encapsulating most
of the synchronization logic into a new "latch" structure
abstracts what little complexity there is into a separate layer,
making the design easier to reason about.  Also, all the new data
members are in `struct thread', with no need for any extra dynamic
allocation, etc., that would require extra management code.

On the other hand, this design is wasteful in that a child thread
cannot free itself before its parent has terminated.  A parent
thread that creates a large number of short-lived child threads
could unnecessarily exhaust kernel memory.  This is probably
acceptable for implementing kernel threads, but it may be a bad
idea for use with user processes because of the larger number of
resources that user processes tend to own.
```
