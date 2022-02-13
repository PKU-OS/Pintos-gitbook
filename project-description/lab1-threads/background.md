# Background

## Understanding Threads

The first step is to read and understand the code for the initial thread system. Pintos already implements thread creation and thread completion, a simple scheduler to switch between threads, and synchronization primitives (semaphores, locks, condition variables, and optimization barriers).

Some of this code might seem slightly mysterious. You can read through parts of the source code to see what's going on. If you like, you can add calls to `printf()` almost anywhere, then recompile and run to see what happens and in what order. You can also run the kernel in a debugger and set breakpoints at interesting spots, single-step through code and examine data, and so on.

When a thread is created, you are creating a new context to be scheduled. You provide a function to be run in this context as an argument to `thread_create()`. The first time the thread is scheduled and runs, it starts from the beginning of that function and executes in that context. When the function returns, the thread terminates. Each thread, therefore, acts like a mini-program running inside Pintos, with the function passed to `thread_create()` acting like `main()`.

At any given time, exactly one thread runs and the rest, if any, become inactive. The scheduler decides which thread to run next. (If no thread is ready to run at any given time, then the special "idle" thread, implemented in `idle()`, runs.) Synchronization primitives can force context switches when one thread needs to wait for another thread to do something.

The mechanics of a context switch are `in threads/switch.S`, which is 80x86 assembly code. (You don't have to understand it.) It saves the state of the currently running thread and restores the state of the thread we're switching to.

Using the GDB debugger, slowly trace through a context switch to see what happens (see section [GDB](../../getting-started/debug-and-test/debugging.md#gdb)). You can set a breakpoint on `schedule()` to start out, and then single-step from there. Be sure to keep track of each thread's address and state, and what procedures are on the call stack for each thread. You will notice that when one thread calls `switch_threads()`, another thread starts running, and the first thing the new thread does is to return from `switch_threads()`. You will understand the thread system once you understand why and how the `switch_threads()` that gets called is different from the `switch_threads()` that returns. See section [Thread Switching](../../appendix/reference-guide/threads.md#thread-switching), for more information.

{% hint style="warning" %}
**Warning**

In Pintos, each thread is assigned a small, fixed-size execution stack just under 4 kB in size. The kernel tries to detect stack overflow, but it cannot do so perfectly. You may cause bizarre problems, such as mysterious kernel panics, if you declare large data structures as non-static local variables, e.g. int buf\[1000];. Alternatives to stack allocation include the page allocator and the block allocator (see section [Memory Allocation](../../appendix/reference-guide/memory-allocation.md)).
{% endhint %}

### Source Files

This part provides a brief overview of the source files related to lab1. You will not need to modify most of this code, but the hope is that presenting this overview will give you a start on what code to look at.

<details>

<summary>source files in threads/</summary>

**loader.S**

**loader.h**

The kernel loader. Assembles to 512 bytes of code and data that the PC BIOS loads into memory and which in turn finds the kernel on disk, loads it into memory, and jumps to `start()` in start.S. See section [The Loader](../../appendix/reference-guide/loading.md#the-loader), for details. You should not need to look at this code or modify it.

**start.S**

Does basic setup needed for memory protection and 32-bit operation on 80x86 CPUs. Unlike the loader, this code is actually part of the kernel. See section [Low-Level Kernel Initialization](../../appendix/reference-guide/loading.md#low-level-kernel-initialization), for details.

**kernel.lds.S**

The linker script used to link the kernel. Sets the load address of the kernel and arranges for `start.S` to be near the beginning of the kernel image. See section The [Loader](../../appendix/reference-guide/loading.md#the-loader), for details. Again, you should not need to look at this code or modify it, but it's here in case you're curious.

**init.c**

**init.h**

Kernel initialization, including `pintos_init()`, the kernel's "main program." You should look over `pintos_init()` at least to see what gets initialized. You might want to add your own initialization code here. See section [High-Level Kernel Initialization](../../appendix/reference-guide/loading.md#high-level-kernel-initialization), for details.

**thread.c**

**thread.h**

Basic thread support. Much of your work will take place in these files. `thread.h` defines `struct thread`, which you are likely to modify in all four projects. See [`struct thread`](../../appendix/reference-guide/threads.md#struct-thread) and [Threads](../../appendix/reference-guide/threads.md) for more information.

**switch.S**

**switch.h**

Assembly language routine for switching threads. Already discussed above. See section [Thread Functions](../../appendix/reference-guide/threads.md#thread-functions), for more information.

**palloc.c**

**palloc.h**

Page allocator, which hands out system memory in multiples of 4 kB pages. See section [Page Allocator](../../appendix/reference-guide/memory-allocation.md#page-allocator), for more information.

**malloc.c**

**malloc.h**

A simple implementation of `malloc()` and `free()` for the kernel. See section [Block Allocator](../../appendix/reference-guide/memory-allocation.md#block-allocator), for more information.

**interrupt.c**

**interrupt.h**

Basic interrupt handling and functions for turning interrupts on and off. See section [Interrupt Handling](../../appendix/reference-guide/interrupt-handling.md), for more information.

**intr-stubs.S**

**intr-stubs.h**

Assembly code for low-level interrupt handling. See section [Interrupt Infrastructure](../../appendix/reference-guide/interrupt-handling.md#interrupt-infrastructure), for more information.

**synch.c**

**synch.h**

Basic synchronization primitives: semaphores, locks, condition variables, and optimization barriers. You will need to use these for synchronization in all four projects. See section [Synchronization](../../appendix/reference-guide/synchronization.md), for more information.

**io.h**

Functions for I/O port access. This is mostly used by source code in the devices directory that you won't have to touch.

**vaddr.h**

**pte.h**

Functions and macros for working with virtual addresses and page table entries. These will be more important to you in project 3. For now, you can ignore them.

**flags.h**

Macros that define a few bits in the 80x86 "flags" register. Probably of no interest.

</details>

<details>

<summary>device files</summary>

The basic threaded kernel also includes these files in the devices directory:

**timer.c**

**timer.h**

System timer that ticks, by default, 100 times per second. You will modify this code in this project.

**vga.c**

**vga.h**

VGA display driver. Responsible for writing text to the screen. You should have no need to look at this code. `printf()` calls into the VGA display driver for you, so there's little reason to call this code yourself.

**serial.c**

**serial.h**

Serial port driver. Again, `printf()` calls this code for you, so you don't need to do so yourself. It handles serial input by passing it to the input layer (see below).

**block.c**

**block.h**

An abstraction layer for _block devices_, that is, random-access, disk-like devices that are organized as arrays of fixed-size blocks. Out of the box, Pintos supports two types of block devices: IDE disks and partitions. Block devices, regardless of type, won't actually be used until project 2.

**ide.c**

**ide.h**

Supports reading and writing sectors on up to 4 IDE disks.

**partition.c**

**partition.h**

Understands the structure of partitions on disks, allowing a single disk to be carved up into multiple regions (partitions) for independent use.

**kbd.c**

**kbd.h**

Keyboard driver. Handles keystrokes passing them to the input layer (see below).

**input.c**

**input.h**

Input layer. Queues input characters passed along by the keyboard or serial drivers.

**intq.c**

**intq.h**

Interrupt queue, for managing a circular queue that both kernel threads and interrupt handlers want to access. Used by the keyboard and serial drivers.

**rtc.c**

**rtc.h**

Real-time clock driver, to enable the kernel to determine the current date and time. By default, this is only used by `thread/init.c` to choose an initial seed for the random number generator.

**speaker.c**

**speaker.h**

Driver that can produce tones on the PC speaker.

**pit.c**

**pit.h**

Code to configure the 8254 Programmable Interrupt Timer. This code is used by both `devices/timer.c` and `devices/speaker.c` because each device uses one of the PIT's output channel.

</details>

<details>

<summary>lib files</summary>

Finally, lib and lib/kernel contain useful library routines. (`lib/user` will be used by user programs, starting in project 2, but it is not part of the kernel.) Here's a few more details:

ctype.h

inttypes.h

limits.h

stdarg.h

stdbool.h

stddef.h

stdint.h

stdio.c

stdio.h

stdlib.c

stdlib.h

string.c

string.h

A subset of the standard C library. See section [C99](../../appendix/coding-standards.md#c99), for information on a few recently introduced pieces of the C library that you might not have encountered before. See section [C.3 Unsafe String Functions](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_9.html#SEC152), for information on what's been intentionally left out for safety.debug.cdebug.hFunctions and macros to aid debugging. See section [E. Debugging Tools](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_11.html#SEC156), for more information.random.crandom.hPseudo-random number generator. The actual sequence of random values will not vary from one Pintos run to another, unless you do one of three things: specify a new random seed value on the -rs kernel command-line option on each run, or use a simulator other than Bochs, or specify the -r option to `pintos`.round.hMacros for rounding.syscall-nr.hSystem call numbers. Not used until project 2.kernel/list.ckernel/list.hDoubly linked list implementation. Used all over the Pintos code, and you'll probably want to use it a few places yourself in project 1.kernel/bitmap.ckernel/bitmap.hBitmap implementation. You can use this in your code if you like, but you probably won't have any need for it in project 1.kernel/hash.ckernel/hash.hHash table implementation. Likely to come in handy for project 3.kernel/console.ckernel/console.hkernel/stdio.hImplements `printf()` and a few other functions.

</details>

### Synchronization

Proper synchronization is an important part of the solutions to these problems. Any synchronization problem can be easily solved by turning interrupts off: while interrupts are off, there is no concurrency, so there's no possibility for race conditions. Therefore, it's tempting to solve all synchronization problems this way, but **don't**. Instead, use semaphores, locks, and condition variables to solve the bulk of your synchronization problems. Read the tour section on synchronization (see section [A.3 Synchronization](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC111)) or the comments in threads/synch.c if you're unsure what synchronization primitives may be used in what situations.

In the Pintos projects, the only class of problem best solved by disabling interrupts is coordinating data shared between a kernel thread and an interrupt handler. Because interrupt handlers can't sleep, they can't acquire locks. This means that data shared between kernel threads and an interrupt handler must be protected within a kernel thread by turning off interrupts.

This project only requires accessing a little bit of thread state from interrupt handlers. For the alarm clock, the timer interrupt needs to wake up sleeping threads. In the advanced scheduler, the timer interrupt needs to access a few global and per-thread variables. When you access these variables from kernel threads, you will need to disable interrupts to prevent the timer interrupt from interfering.

When you do turn off interrupts, take care to do so for the least amount of code possible, or you can end up losing important things such as timer ticks or input events. Turning off interrupts also increases the interrupt handling latency, which can make a machine feel sluggish if taken too far.

The synchronization primitives themselves in synch.c are implemented by disabling interrupts. You may need to increase the amount of code that runs with interrupts disabled here, but you should still try to keep it to a minimum.

Disabling interrupts can be useful for debugging, if you want to make sure that a section of code is not interrupted. You should remove debugging code before turning in your project. (Don't just comment it out, because that can make the code difficult to read.)

There should be no busy waiting in your submission. A tight loop that calls `thread_yield()` is one form of busy waiting.

###
