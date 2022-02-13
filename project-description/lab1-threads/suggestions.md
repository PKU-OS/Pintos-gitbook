# Suggestions

## Development Suggestions

Your TAs find some tools very useful during the development, see [Development Tools](../../appendix/development-tools.md) for details.

You should expect to (always) run into bugs that you simply don't understand while working on this and subsequent projects. When you do, reread the appendix on debugging tools, which is filled with useful debugging tips that should help you to get back up to speed (see section [Debugging Tools](../../getting-started/debug-and-test/debugging.md)). Be sure to read the section on backtraces (see section [Backtraces](../../getting-started/debug-and-test/debugging.md#backtraces)), which will help you to get the most out of every kernel panic or assertion failure.

{% hint style="info" %}
**Tips**

We also suggest you read the [Your Tasks](your-tasks.md) as well as the [FAQs](faq.md) carefully---perhaps multiple times. A lot of the confusion comes from not reading them carefully.
{% endhint %}

{% hint style="info" %}
**Tips**

`make check` runs all tests. If you are debugging one test, you can run just that test with `make tests/threads/<test_case>.result`. Read the [Testing](../../getting-started/debug-and-test/testing.md) part for more details.
{% endhint %}

## Suggested Order of Implementation

We suggest first implementing the following, which can happen in parallel:

* Optimized `timer_sleep()` in Exercise 1.1.
* Basic support for priority scheduling in Exercise 2.1 when a new thread is created in `thread_create()`.
* Implement fixed-point real arithmetic routines ([Fixed-Point Real Arithmetic](../../appendix/4.4bsd-scheduler.md#fixed-point-real-arithmetic)) that are needed for MLFQS in exercise 3.1.

Then you can add full support for priority scheduling in Exercise 2.1 by considering all other possible scenarios and the synchronization primitives.

Then you can tackle either Exercise 3.1 first or Exercise 2.2 first.
