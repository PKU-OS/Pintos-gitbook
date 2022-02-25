# Background

## Understanding Threads

**The first step is to read and understand the code for the initial thread system.**

* Pintos already implements **thread creation** and **thread completion**, **a simple scheduler** to switch between threads, and **synchronization primitives** **(semaphores, locks, condition variables, and optimization barriers)**.

Some of this code might seem slightly mysterious. You can read through parts of the source code to see what's going on.

* If you like, you can **add calls to `printf()`** almost anywhere, then recompile and run to see what happens and in what order.
* You can also **run the kernel in a debugger** and set breakpoints at interesting spots, single-step through code and examine data, and so on.

### Create a New Thread

**When a thread is created, you are creating a new context to be scheduled.** You provide a function to be run in this context as an argument to `thread_create()`.

* **The first time the thread is scheduled and runs, it starts from the beginning of that function and executes in that context.**
* **When the function returns, the thread terminates.** Each thread, therefore, acts like a mini-program running inside Pintos, with the function passed to `thread_create()` acting like `main()`.

**At any given time, exactly one thread runs and the rest, if any, become inactive.** The scheduler decides which thread to run next.

* If no thread is ready to run at any given time, then **the special "idle" thread**, implemented in `idle()`, runs.
* Synchronization primitives can force context switches when one thread needs to wait for another thread to do something.

### Context Switch

**The mechanics of a context switch are `in threads/switch.S`**, which is 80x86 assembly code. (You don't have to understand it.)

* It **saves** the state of the currently running thread and **restores** the state of the thread we're switching to.

Using the GDB debugger, slowly trace through a context switch to see what happens (see section [GDB](../../getting-started/debug-and-test/debugging.md#gdb)).

* You can **set a breakpoint on `schedule()`** to start out, and then **single-step from there**.
* Be sure to keep track of each thread's address and state, and what procedures are on the call stack for each thread.
* <mark style="color:red;"><mark style="color:red;">**You will notice that when one thread calls**<mark style="color:red;"></mark><mark style="color:red;">** **</mark><mark style="color:red;">**`switch_threads()`**</mark><mark style="color:red;">**, another thread starts running, and the first thing the new thread does is to return from**</mark><mark style="color:red;">** **</mark><mark style="color:red;"><mark style="color:red;">****<mark style="color:red;"></mark><mark style="color:red;">** **</mark><mark style="color:red;">**`switch_threads()`**</mark><mark style="color:red;">**.**</mark>
* You will understand the thread system once you understand why and how the `switch_threads()` that gets called is different from the `switch_threads()` that returns. See section [Thread Switching](../../appendix/reference-guide/threads.md#thread-switching), for more information.

{% hint style="warning" %}
**Warning**

<mark style="color:red;">**In Pintos, each thread is assigned a small, fixed-size execution stack just under 4 kB in size.**</mark>

* The kernel tries to detect stack overflow, but it cannot do so perfectly. You may cause bizarre problems, such as mysterious kernel panics, if you declare large data structures as non-static local variables, e.g. int buf\[1000];.
* Alternatives to stack allocation include the page allocator and the block allocator (see section [Memory Allocation](../../appendix/reference-guide/memory-allocation.md)).
{% endhint %}

## Source Files

**This part provides a brief overview of the source files related to lab1.** You will not need to modify most of this code, but the hope is that presenting this overview will give you a start on what code to look at.

<details>

<summary>source files in threads/</summary>

* <mark style="color:blue;">**loader.S**</mark>
* <mark style="color:blue;">**loader.h**</mark>
  * **The kernel loader.** Assembles to 512 bytes of code and data that the PC BIOS loads into memory and which in turn finds the kernel on disk, loads it into memory, and jumps to `start()` in start.S.
  * See section [The Loader](../../appendix/reference-guide/loading.md#the-loader), for details. You should not need to look at this code or modify it.
* <mark style="color:blue;">**start.S**</mark>
  * **Does basic setup needed for memory protection and 32-bit operation on 80x86 CPUs.** Unlike the loader, this code is actually **part of the kernel**.
  * See section [Low-Level Kernel Initialization](../../appendix/reference-guide/loading.md#low-level-kernel-initialization), for details.
* <mark style="color:blue;">**kernel.lds.S**</mark>
  * **The linker script used to link the kernel.** Sets the load address of the kernel and arranges for `start.S` to be near the beginning of the kernel image.
  * See section The [Loader](../../appendix/reference-guide/loading.md#the-loader), for details. Again, you should not need to look at this code or modify it, but it's here in case you're curious.
* <mark style="color:blue;">**init.c**</mark>
* <mark style="color:blue;">**init.h**</mark>
  * **Kernel initialization**, including `pintos_init()`, the kernel's "main program."
  * **You should look over `pintos_init()` at least to see what gets initialized.** You might want to add your own initialization code here.
  * See section [High-Level Kernel Initialization](../../appendix/reference-guide/loading.md#high-level-kernel-initialization), for details.
* <mark style="color:blue;">**thread.c**</mark>
* <mark style="color:blue;">**thread.h**</mark>
  * **Basic thread support.** Much of your work will take place in these files.
  * `thread.h` defines `struct thread`, which you are likely to modify in all four projects.
  * See [`struct thread`](../../appendix/reference-guide/threads.md#struct-thread) and [Threads](../../appendix/reference-guide/threads.md) for more information.
* <mark style="color:blue;">**switch.S**</mark>
* <mark style="color:blue;">**switch.h**</mark>
  * **Assembly language routine for switching threads.** Already discussed above.
  * See section [Thread Functions](../../appendix/reference-guide/threads.md#thread-functions), for more information.
* <mark style="color:blue;">**palloc.c**</mark>
* <mark style="color:blue;">**palloc.h**</mark>
  * **Page allocator**, which hands out system memory in multiples of 4 kB pages.
  * See section [Page Allocator](../../appendix/reference-guide/memory-allocation.md#page-allocator), for more information.
* <mark style="color:blue;">**malloc.c**</mark>
* <mark style="color:blue;">**malloc.h**</mark>
  * **A simple implementation of `malloc()` and `free()` for the kernel.**
  * See section [Block Allocator](../../appendix/reference-guide/memory-allocation.md#block-allocator), for more information.
* <mark style="color:blue;">**interrupt.c**</mark>
* <mark style="color:blue;">**interrupt.h**</mark>
  * **Basic interrupt handling and functions for turning interrupts on and off.**
  * See section [Interrupt Handling](../../appendix/reference-guide/interrupt-handling.md), for more information.
* <mark style="color:blue;">**intr-stubs.S**</mark>
* <mark style="color:blue;">**intr-stubs.h**</mark>
  * **Assembly code for low-level interrupt handling.**
  * See section [Interrupt Infrastructure](../../appendix/reference-guide/interrupt-handling.md#interrupt-infrastructure), for more information.
* <mark style="color:blue;">**synch.c**</mark>
* <mark style="color:blue;">**synch.h**</mark>
  * **Basic synchronization primitives: semaphores, locks, condition variables, and optimization barriers.** You will need to use these for synchronization in all four projects.
  * See section [Synchronization](../../appendix/reference-guide/synchronization.md), for more information.
* <mark style="color:blue;">**io.h**</mark>
  * **Functions for I/O port access.**
  * This is mostly used by source code in the devices directory that you won't have to touch.
* <mark style="color:blue;">**vaddr.h**</mark>
* <mark style="color:blue;">**pte.h**</mark>
  * **Functions and macros for working with virtual addresses and page table entries.**
  * These will be more important to you in project 3. For now, you can ignore them.
* <mark style="color:blue;">**flags.h**</mark>
  * **Macros that define a few bits in the 80x86 "flags" register.** Probably of no interest.

</details>

<details>

<summary>device files</summary>

The basic threaded kernel also includes these files in the `devices` directory:

* <mark style="color:blue;">**timer.c**</mark>
* <mark style="color:blue;">**timer.h**</mark>
  * **System timer that ticks, by default, 100 times per second.**
  * <mark style="color:red;">**You will modify this code in this project.**</mark>
* <mark style="color:blue;">**vga.c**</mark>
* <mark style="color:blue;">**vga.h**</mark>
  * **VGA display driver.** Responsible for writing text to the screen.
  * You should have no need to look at this code. `printf()` calls into the VGA display driver for you, so there's little reason to call this code yourself.
* <mark style="color:blue;">**serial.c**</mark>
* <mark style="color:blue;">**serial.h**</mark>
  * **Serial port driver.**
  * Again, `printf()` calls this code for you, so you don't need to do so yourself. It handles serial input by passing it to the input layer (see below).
* <mark style="color:blue;">**block.c**</mark>
* <mark style="color:blue;">**block.h**</mark>
  * **An abstraction layer for **_**block devices**_, that is, random-access, disk-like devices that are organized as arrays of fixed-size blocks.
  * Out of the box, Pintos supports two types of block devices: **IDE disks** and **partitions**.
  * Block devices, regardless of type, won't actually be used until project 2.
* <mark style="color:blue;">**ide.c**</mark>
* <mark style="color:blue;">**ide.h**</mark>
  * **Supports reading and writing sectors on up to 4 IDE disks.**
* <mark style="color:blue;">**partition.c**</mark>
* <mark style="color:blue;">**partition.h**</mark>
  * **Understands the structure of partitions on disks**, allowing a single disk to be carved up into multiple regions (partitions) for independent use.
* <mark style="color:blue;">**kbd.c**</mark>
* <mark style="color:blue;">**kbd.h**</mark>
  * **Keyboard driver.** Handles keystrokes passing them to the input layer (see below).
* <mark style="color:blue;">**input.c**</mark>
* <mark style="color:blue;">**input.h**</mark>
  * **Input layer.** Queues input characters passed along by the keyboard or serial drivers.
* <mark style="color:blue;">**intq.c**</mark>
* <mark style="color:blue;">**intq.h**</mark>
  * **Interrupt queue**, for managing a circular queue that both kernel threads and interrupt handlers want to access. Used by the keyboard and serial drivers.
* <mark style="color:blue;">**rtc.c**</mark>
* <mark style="color:blue;">**rtc.h**</mark>
  * **Real-time clock driver**, to enable the kernel to determine the current date and time.
  * By default, this is only used by `thread/init.c` to choose an initial seed for the random number generator.
* <mark style="color:blue;">**speaker.c**</mark>
* <mark style="color:blue;">**speaker.h**</mark>
  * **Driver that can produce tones on the PC speaker.**
* <mark style="color:blue;">**pit.c**</mark>
* <mark style="color:blue;">**pit.h**</mark>
  * **Code to configure the 8254 Programmable Interrupt Timer.**
  * This code is used by both `devices/timer.c` and `devices/speaker.c` because each device uses one of the PIT's output channel.

</details>

<details>

<summary>lib files</summary>

Finally, `lib` and `lib/kernel` contain useful library routines. (`lib/user` will be used by user programs, starting in project 2, but it is not part of the kernel.)

Here's a few more details:

* <mark style="color:blue;">**ctype.h**</mark>
* <mark style="color:blue;">**inttypes.h**</mark>
* <mark style="color:blue;">**limits.h**</mark>
* <mark style="color:blue;">**stdarg.h**</mark>
* <mark style="color:blue;">**stdbool.h**</mark>
* <mark style="color:blue;">**stddef.h**</mark>
* <mark style="color:blue;">**stdint.h**</mark>
* <mark style="color:blue;">**stdio.c**</mark>
* <mark style="color:blue;">**stdio.h**</mark>
* <mark style="color:blue;">**stdlib.c**</mark>
* <mark style="color:blue;">**stdlib.h**</mark>
* <mark style="color:blue;">**string.c**</mark>
* <mark style="color:blue;">**string.h**</mark>
  * **A subset of the standard C library.**
  * See section [C99](../../appendix/coding-standards.md#c99), for information on a few recently introduced pieces of the C library that you might not have encountered before.
  * See section [Unsafe String Functions](../../appendix/coding-standards.md#unsafe-string-functions), for information on what's been intentionally left out for safety.
* <mark style="color:blue;">**debug.c**</mark>
* <mark style="color:blue;">**debug.h**</mark>
  * **Functions and macros to aid debugging.**
  * See section [Debugging Tools](../../getting-started/debug-and-test/debugging.md), for more information.
* <mark style="color:blue;">**random.c**</mark>
* <mark style="color:blue;">**random.h**</mark>
  * **Pseudo-random number generator.**
  * **The actual sequence of random values will not vary from one Pintos run to another**, unless you do one of three things:
    1. specify a new random seed value on the `-rs` kernel command-line option on each run,
    2. or use a simulator other than Bochs,
    3. or specify the `-r` option to `pintos`.
* <mark style="color:blue;">**round.h**</mark>
  * **Macros for rounding.**
* <mark style="color:blue;">**syscall-nr.h**</mark>
  * **System call numbers.**
  * Not used until project 2.
* <mark style="color:blue;">**kernel/list.c**</mark>
* <mark style="color:blue;">**kernel/list.h**</mark>
  * **Doubly linked list implementation.**
  * <mark style="color:red;">**Used all over the Pintos code, and you'll probably want to use it a few places yourself in project 1.**</mark>
* <mark style="color:blue;">**kernel/bitmap.c**</mark>
* <mark style="color:blue;">**kernel/bitmap.h**</mark>
  * **Bitmap implementation.**
  * You can use this in your code if you like, but you probably won't have any need for it in project 1.
* <mark style="color:blue;">**kernel/hash.c**</mark>
* <mark style="color:blue;">**kernel/hash.h**</mark>
  * **Hash table implementation.**
  * Likely to come in handy for project 3.
* <mark style="color:blue;">**kernel/console.c**</mark>
* <mark style="color:blue;">**kernel/console.h**</mark>
* <mark style="color:blue;">**kernel/stdio.h**</mark>
  * **Implements `printf()` and a few other functions.**

</details>

## Synchronization

<mark style="color:red;">**Proper synchronization is an important part of the solutions to these problems.**</mark>

* Any synchronization problem can be easily solved by **turning interrupts off**: while interrupts are off, there is no concurrency, so there's no possibility for race conditions. Therefore, it's tempting to solve all synchronization problems this way, but <mark style="color:red;">**don't**</mark>.
* Instead, **use semaphores, locks, and condition variables** to solve the bulk of your synchronization problems.
* Read the tour section on synchronization (see section [Synchronization](../../appendix/reference-guide/synchronization.md)) or the comments in `threads/synch.c` if you're unsure what synchronization primitives may be used in what situations.

In the Pintos projects, **the only class of problem best solved by disabling interrupts is coordinating data **_**shared between a kernel thread and an interrupt handler**_**.**

* Because **interrupt handlers can't sleep, they can't acquire locks**. This means that data shared between kernel threads and an interrupt handler must be protected within a kernel thread by turning off interrupts.

**This project only requires accessing a little bit of thread state from interrupt handlers.**

* For the alarm clock, the timer interrupt needs to wake up sleeping threads.
* In the advanced scheduler, the timer interrupt needs to access a few global and per-thread variables. When you access these variables from kernel threads, you will need to disable interrupts to prevent the timer interrupt from interfering.

<mark style="color:red;">**When you do turn off interrupts, take care to do so for the least amount of code possible.**</mark>

* Otherwise you can end up **losing important things** such as timer ticks or input events.
* Turning off interrupts also **increases the interrupt handling latency**, which can make a machine feel sluggish if taken too far.

**The synchronization primitives themselves in `synch.c` are implemented by disabling interrupts.**

* You may need to increase the amount of code that runs with interrupts disabled here, but you should still try to keep it to a minimum.

**Disabling interrupts can be useful for debugging**, if you want to make sure that a section of code is not interrupted.

* You should remove debugging code before turning in your project. (Don't just comment it out, because that can make the code difficult to read.)

<mark style="color:red;">**There should be no busy waiting in your submission.**</mark>

* A tight loop that calls `thread_yield()` is one form of busy waiting.
