# Background

**Up to now, all of the code you have run under Pintos has been part of the operating system kernel.**

* This means, for example, that **all the test code from the last assignment ran as part of the kernel**, with full access to privileged parts of the system.
* Once we start running user programs on top of the operating system, this is no longer true. This project deals with the consequences.

<mark style="color:red;">**We allow more than one process to run at a time. Each process has one thread (multithreaded processes are not supported in Pintos).**</mark>

**User programs are written under the illusion that they have the entire machine.** This means that when you load and run multiple processes at a time, you must manage memory, scheduling, and other states correctly to maintain this illusion.

In the previous project, we compiled our test code directly into your kernel, so we had to require certain specific function interfaces within the kernel. **From now on, we will test your operating system by running user programs.**

* This gives you much greater freedom.
* You must make sure that the user program interface meets the specifications described here, but given that constraint, you are free to restructure or rewrite kernel code however you wish.

## Source Files

**The easiest way to get an overview of the programming you will be doing is to simply go over each part you'll be working with.** In `userprog/`, you'll find a small number of files, but here is where the bulk of your work will be:

* <mark style="color:blue;">**process.c**</mark>
* <mark style="color:blue;">**process.h**</mark>
  * **Loads ELF binaries and starts processes.**
* <mark style="color:blue;">**pagedir.c**</mark>
* <mark style="color:blue;">**pagedir.h**</mark>
  * **A simple manager for 80x86 hardware page tables.**
  * Although you probably won't want to modify this code for this project, <mark style="color:red;">**you may want to call some of its functions**</mark>.
  * See section [Page Table](../../appendix/reference-guide/page-table.md), for more information.
* <mark style="color:blue;">**syscall.c**</mark>
* <mark style="color:blue;">**syscall.h**</mark>
  * **Whenever a user process wants to access some kernel functionality, it invokes a system call.** This is a skeleton system call handler.
  * Currently, it just prints a message and terminates the user process. In part 2 of this project you will add code to do everything else needed by system calls.
* <mark style="color:blue;">**exception.c**</mark>
* <mark style="color:blue;">**exception.h**</mark>
  * **When a user process performs a privileged or prohibited operation, it traps into the kernel as an "exception" or "fault."** These files handle exceptions.
  * Currently all exceptions simply print a message and terminate the process. Some, but not all, solutions to project 2 require modifying `page_fault()` in this file.
* <mark style="color:blue;">**gdt.c**</mark>
* <mark style="color:blue;">**gdt.h**</mark>
  * **The 80x86 is a segmented architecture.** The Global Descriptor Table (GDT) is a table that describes the segments in use. These files set up the GDT.
  * You should not need to modify these files for any of the projects. You can read the code if you're interested in how the GDT works.
* <mark style="color:blue;">**tss.c**</mark>
* <mark style="color:blue;">**tss.h**</mark>
  * **The Task-State Segment (TSS) is used for 80x86 architectural task switching.**
  * Pintos uses the TSS **only** for switching stacks when a user process enters an interrupt handler, as does Linux.
  * You should not need to modify these files for any of the projects. You can read the code if you're interested in how the TSS works.

## Using the File System

<mark style="color:red;">**You will need to interface to the file system code for this project**</mark>, because user programs are loaded from the file system and many of the system calls you must implement deal with the file system.

* However, the focus of this project is not the file system, so we have provided a simple but complete file system in the `filesys/` directory.
* You will want to look over the `filesys.h` and `file.h` interfaces to understand how to use the file system, and especially its many limitations.
* **There is no need to modify the file system code for this project**, and so we recommend that you **do not**. Working on the file system is likely to distract you from this project's focus.

### Limitations

Proper use of the file system routines now will make life much easier for project 4, when you improve the file system implementation. **Until then, you will have to tolerate the following limitations:**

* **No internal synchronization.** Concurrent accesses will interfere with one another. <mark style="color:red;">**You should use synchronization to ensure that only one process at a time is executing file system code.**</mark>
* **File size is fixed at creation time.** The root directory is represented as a file, so the number of files that may be created is also limited.
* **File data is allocated as a single extent, that is, data in a single file must occupy a contiguous range of sectors on disk.** _External fragmentation_ can therefore become a serious problem as a file system is used over time.
* **No subdirectories.**
* **File names are limited to 14 characters.**
* **A system crash mid-operation may corrupt the disk in a way that cannot be repaired automatically.** There is no file system repair tool anyway.

### Features

**One important feature is included:**

* **Unix-like semantics for `filesys_remove()` are implemented.** That is, if a file is open when it is removed, its blocks are not deallocated and it may still be accessed by any threads that have it open, until the last one closes it.

### Create your simulated disk/file system

**You need to be able to create a simulated disk with a file system partition.** The `pintos-mkdisk` program provides this functionality.

* From the `userprog/build` directory, execute `pintos-mkdisk filesys.dsk --filesys-size=2`. This command **creates a simulated disk** named `filesys.dsk` that contains a 2 MB Pintos file system partition.
* Then **format the file system partition** by passing `-f` `-q` on the kernel's command line: `pintos -- -f -q`. The -f option causes the file system to be formatted, and `-q` causes Pintos to exit as soon as the format is done.

**You'll need a way to copy files in and out of the simulated file system.** The `pintos -p` ("put") and `-g` ("get") options do this.

* To **copy file into** **the Pintos file system**, use the command `pintos -p file -- -q`. (The -- is needed because -p is for the `pintos` script, not for the simulated kernel.)
* To copy it to the Pintos file system under the name `newname`, add `-a newname`: `pintos -p file -a newname -- -q`.
* The commands for **copying files out of a VM** are similar, but substitute `-g` for `-p`.
* Incidentally, these commands work by passing **special commands `extract` and `append`** on the kernel's command line and copying to and from a special simulated "scratch" partition. If you're very curious, you can look at the `pintos` script as well as `filesys/fsutil.c` to learn the implementation details.

<mark style="color:red;">**Here's a summary**</mark> of how to **create a disk** with a file system partition, **format** the file system, **copy** the `echo` program into the new disk, and then **run** `echo`, passing argument `PKUOS`. (Argument passing won't work until you implemented it.)

* It assumes that you've already built the examples in `examples/` and that **the current directory is `userprog/build`**:

```
$ pintos-mkdisk filesys.dsk --filesys-size=2
$ pintos -- -f -q
$ pintos -p ../../examples/echo -a echo -- -q
$ pintos -- -q run 'echo PKUOS'
```

* **The three final steps can actually be combined into a single command:**

```
$ pintos-mkdisk filesys.dsk --filesys-size=2
$ pintos -p ../../examples/echo -a echo -- -f -q run 'echo PKUOS'
```

* **If you don't want to keep the file system disk around for later use or inspection, you can even combine all four steps into a single command.** The `--filesys-size=n` option creates a temporary file system partition approximately `n` megabytes in size just for the duration of the `pintos` run. The Pintos automatic test suite makes extensive use of this syntax:

```
$ pintos --filesys-size=2 -p ../../examples/echo -a echo -- -f -q run 'echo PKUOS'
```

* You can **delete a file** from the Pintos file system using the **`rm file`** kernel action. e.g. `pintos -q rm file`.
* Also, **`ls` lists the files** in the file system and **`cat file` prints a file's contents** to the display.

## How User Programs Work

**Pintos can run normal C programs, as long as they fit into memory and use only the system calls you implement**.

* Notably, <mark style="color:red;">**`malloc()`**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**cannot be implemented**</mark> because none of the system calls required for this project allow for memory allocation.
* <mark style="color:red;">**Pintos also can't run programs that use floating point operations**</mark>, since the kernel doesn't save and restore the processor's floating-point unit when switching threads.

**The `src/examples` directory contains a few sample user programs.**

* The **Makefile** in this directory compiles the provided examples, and you can edit it to compile your own programs as well.
* Some of the example programs will only work once projects 3 or 4 have been implemented.

**Pintos can load ELF** executables with the loader provided for you in `userprog/process.c`.

* ELF is a file format used by Linux, Solaris, and many other operating systems for object files, shared libraries, and executables.
* You can actually use any compiler and linker that output 80x86 ELF executables to produce programs for Pintos. (We've provided compilers and linkers that should do just fine.)

You should realize immediately that, **until you copy a test program to the simulated file system, Pintos will be unable to do useful work.**

* You won't be able to do interesting things until you copy a variety of programs to the file system.
* You might want to create a clean reference file system disk and copy that over whenever you trash your `filesys.dsk` beyond a useful state, which may happen occasionally while debugging.

## Virtual Memory Layout

Virtual memory in Pintos is divided into two regions: <mark style="color:red;">**user virtual memory**</mark> and <mark style="color:red;">**kernel virtual memory**</mark>.

* **User virtual memory ranges from virtual address 0 up to `PHYS_BASE`**, which is defined in `threads/vaddr.h` and defaults to 0xc0000000 (3 GB).
* **Kernel virtual memory occupies the rest of the virtual address space**, from `PHYS_BASE` up to 4 GB.

**User virtual memory is per-process.**

* When the kernel switches from one process to another, it also switches user virtual address spaces by changing the processor's page directory base register (see `pagedir_activate()` in `userprog/pagedir.c`).
* **`struct thread` contains a pointer to a process's page table.**

**Kernel virtual memory is global.**

* It is always mapped the same way, regardless of what user process or kernel thread is running.
* **In Pintos, kernel virtual memory is mapped one-to-one to physical memory, starting at `PHYS_BASE`.** That is, virtual address `PHYS_BASE` accesses physical address 0, virtual address `PHYS_BASE` + 0x1234 accesses physical address 0x1234, and so on **up to the size of the machine's physical memory**.

**A user program can only access its own user virtual memory.**

* An attempt to access kernel virtual memory causes a _**page fault**_, handled by `page_fault()` in `userprog/exception.c`, and the process will be terminated.

**Kernel threads can access both** **kernel virtual memory and, if a user process is running, the user virtual memory of the running process.**

* However, even in the kernel, an attempt to access memory at an unmapped user virtual address will cause a _**page fault**_.

### **Typical Memory Layout**

**Conceptually, each process is free to lay out its own user virtual memory however it chooses.** In practice, user virtual memory is laid out like this:

```
   PHYS_BASE +----------------------------------+
             |            user stack            |
             |                 |                |
             |                 |                |
             |                 V                |
             |          grows downward          |
             |                                  |
             |                                  |
             |                                  |
             |                                  |
             |           grows upward           |
             |                 ^                |
             |                 |                |
             |                 |                |
             +----------------------------------+
             | uninitialized data segment (BSS) |
             +----------------------------------+
             |     initialized data segment     |
             +----------------------------------+
             |           code segment           |
  0x08048000 +----------------------------------+
             |                                  |
             |                                  |
             |                                  |
             |                                  |
             |                                  |
           0 +----------------------------------+
```

* <mark style="color:red;">**In this project, the user stack is fixed in size**</mark>, but in project 3 it will be allowed to grow. Traditionally, the size of the uninitialized data segment can be adjusted with a system call, but you will not have to implement this.
* **The code segment** in Pintos starts at user virtual address 0x08048000, approximately 128 MB from the bottom of the address space. This value is specified in \[[SysV-i386](../../appendix/bibliography.md#software-references)] and has no deep significance.
* **The linker sets the layout of a user program in memory, as directed by a "linker script" that tells it the names and locations of the various program segments.** You can learn more about linker scripts by reading the "Scripts" chapter in the linker manual, accessible via `info ld`.
* To view the layout of a particular executable, run `objdump` (80x86) or `i386-elf-objdump` (SPARC) with the `-p` option.

## 80x86 Calling Convention

**This section summarizes important points of the convention used for normal function calls on 32-bit 80x86 implementations of Unix.** Some details are omitted for brevity. If you do want all the details, refer to \[[SysV-i386](../../appendix/bibliography.md#software-references)].

**The calling convention works like this:**

1.  **The caller pushes each of the function's arguments on the stack one by one, normally using the `PUSH` assembly language instruction.** Arguments are pushed in **right-to-left** order.

    **The stack grows downward:** each push decrements the stack pointer, then stores into the location it now points to, like the C expression `*--sp = value`.
2. **The caller pushes the address of its next instruction (the \_return address**\_**) on the stack and jumps to the first instruction of the callee.** A single 80x86 instruction, `CALL`, does both.
3. **The callee executes.** When it takes control, the stack pointer points to the return address, the first argument is just above it, the second argument is just above the first argument, and so on.
4. **If the callee has a return value, it stores it into register `EAX`.**
5. **The callee returns by popping the return address from the stack and jumping to the location it specifies, using the 80x86 `RET` instruction.**
6. **The caller pops the arguments off the stack.**

Consider a function `f()` that takes three `int` arguments. This diagram shows a sample stack frame as seen by the callee at the beginning of step 3 above, supposing that `f()` is invoked as `f(1, 2, 3)`. The initial stack address is arbitrary:

```
                             +----------------+
                  0xbffffe7c |        3       |
                  0xbffffe78 |        2       |
                  0xbffffe74 |        1       |
stack pointer --> 0xbffffe70 | return address |
                             +----------------+
```

## Program Startup Details

### The Entry Point

**The Pintos C library for user programs designates `_start()`, in `lib/user/entry.c`, as **<mark style="color:red;">**the**</mark>**  **<mark style="color:red;">**entry point for user programs**</mark>**.** This function is a wrapper around `main()` that <mark style="color:red;">**calls**</mark> <mark style="color:red;"></mark><mark style="color:red;"></mark> `exit()` <mark style="color:red;">**if**</mark> `main()` <mark style="color:red;">**returns**</mark>:

```c
void
_start (int argc, char *argv[])
{
  exit (main (argc, argv));
}
```

### **Argument Passing**

**The kernel must put the arguments for the initial function on the stack before it allows the user program to begin executing.**

* The arguments are passed in the same way as **the normal calling convention** (see section [80x86 Calling Convention](background.md#80x86-calling-convention)), and you will implement this in Pintos kernel (see section [Argument Passing](your-tasks.md#task-2-argument-passing) of Your Tasks).

Consider **how to handle arguments** for the following example command: `/bin/ls -l foo bar`.

1. First, **break the command into words**: `/bin/ls`, `-l`, `foo`, `bar`.
2. **Place the words at the top of the stack.** Order doesn't matter, because they will be referenced through pointers.
3. **Then, push the address of each string plus \_a null pointer sentinel**\_**, on the stack, in right-to-left order.** These are the elements of `argv`. The null pointer sentinel ensures that `argv[argc]` is a null pointer, as required by the C standard. The order ensures that `argv[0]` is at the lowest virtual address.
4. Word-aligned accesses are faster than unaligned accesses, so **for best performance, round the stack pointer down to a multiple of 4 before the first push**.
5. **Then, push `argv` (the address of `argv[0]`) and `argc`, in that order.**
6. **Finally, push a fake "return address"**: although the entry function will never return, its stack frame must have the same structure as any other.

The table below shows the state of the stack right before the beginning of the user program, assuming `PHYS_BASE` is 0xc0000000:

| Address    | Name           | Data       | Type          |
| ---------- | -------------- | ---------- | ------------- |
| 0xbffffffc | `argv[3][...]` | bar\0      | `char[4]`     |
| 0xbffffff8 | `argv[2][...]` | foo\0      | `char[4]`     |
| 0xbffffff5 | `argv[1][...]` | -l\0       | `char[3]`     |
| 0xbfffffed | `argv[0][...]` | /bin/ls\0  | `char[8]`     |
| 0xbfffffec | word-align     | 0          | `uint8_t`     |
| 0xbfffffe8 | `argv[4]`      | 0          | `char *`      |
| 0xbfffffe4 | `argv[3]`      | 0xbffffffc | `char *`      |
| 0xbfffffe0 | `argv[2]`      | 0xbffffff8 | `char *`      |
| 0xbfffffdc | `argv[1]`      | 0xbffffff5 | `char *`      |
| 0xbfffffd8 | `argv[0]`      | 0xbfffffed | `char *`      |
| 0xbfffffd4 | `argv`         | 0xbfffffd8 | `char **`     |
| 0xbfffffd0 | `argc`         | 4          | `int`         |
| 0xbfffffcc | return address | 0          | `void (*) ()` |

In this example, the stack pointer would be initialized to 0xbfffffcc.

<mark style="color:red;">**As shown above, your code should start the stack**</mark> <mark style="color:red;"></mark><mark style="color:red;"></mark> <mark style="color:red;"></mark>_<mark style="color:red;">**at the very top of the user virtual address space**</mark>_, in the page just below virtual address <mark style="color:red;"></mark> `PHYS_BASE` (defined in <mark style="color:red;"></mark> `threads/vaddr.h`).

You may find the non-standard `hex_dump()` function, declared in `<stdio.h>`, useful for **debugging your argument passing code**. Here's what it would show in the above example:

```
bfffffc0                                      00 00 00 00 |            ....|
bfffffd0  04 00 00 00 d8 ff ff bf-ed ff ff bf f5 ff ff bf |................|
bfffffe0  f8 ff ff bf fc ff ff bf-00 00 00 00 00 2f 62 69 |............./bi|
bffffff0  6e 2f 6c 73 00 2d 6c 00-66 6f 6f 00 62 61 72 00 |n/ls.-l.foo.bar.|
```

## System Call Details

**The first project already dealt with one way that the operating system can regain control from a user program: interrupts from timers and I/O devices.**

* These are **"external" interrupts**, because they are caused by entities outside the CPU (see section [External Interrupt Handling](../../appendix/reference-guide/interrupt-handling.md#external-interrupt-handling)).

**The operating system also deals with software exceptions, which are events that occur in program code (see section** [**Internal Interrupt Handling**](../../appendix/reference-guide/interrupt-handling.md#internal-interrupt-handling)**).**

* These can be **errors** such as a page fault or division by zero.
* Exceptions are also the means by which a user program can request services (**"system calls"**) from the operating system.

**In the 80x86 architecture, **<mark style="color:red;">**the**</mark>**  `int`** <mark style="color:red;"></mark> <mark style="color:red;"></mark><mark style="color:red;">**instruction**</mark> is the most commonly used means for invoking system calls.

* This instruction is handled in the same way as other software exceptions.
* In Pintos, user programs invoke `int $0x30` to make a system call. **The system call number and any additional arguments are expected to be pushed on the stack** in the normal fashion before invoking the interrupt (see section [80x86 Calling Convention](background.md#80x86-calling-convention)).
* Thus, when **the system call handler `syscall_handler()`** gets control, the system call number is in the 32-bit word at the caller's stack pointer, the first argument is in the 32-bit word at the next higher address, and so on. The caller's stack pointer is accessible to `syscall_handler()` as the esp member of the **`struct intr_frame`** passed to it. (`struct intr_frame` is on the kernel stack.)

**The 80x86 convention for function return values is to place them in the `EAX` register.** System calls that return a value can do so by modifying the _eax_ member of **`struct intr_frame`**.

<mark style="color:red;">**You should try to avoid writing large amounts of repetitive code for implementing system calls (this will cause a deduction on your code quality score, see section**</mark> [<mark style="color:red;">**Code Style**</mark>](../../getting-started/grading.md#code-style)<mark style="color:red;">**).**</mark>

* **Each system call argument, whether an integer or a pointer, takes up 4 bytes on the stack.** You should be able to take advantage of this to avoid writing much near-identical code for retrieving each system call's arguments from the stack.
