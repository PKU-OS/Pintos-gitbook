# Your Tasks

## Task 0: Design Document

* Download the [project 2 design document template](https://github.com/PKU-OS/pintos/blob/master/docs/p2.md). Read through it to motivate your design and fill it in after you finish the project.&#x20;
* We recommend that you read the design document template before you start working on the project.&#x20;
* See section [Project Documentation](../../appendix/project-documentation.md), for a sample design document that goes along with a fictitious project.

## Task 1: Process Termination Messages

### **Exercise 1.1**

**Whenever **<mark style="color:red;">**a user process**</mark>** terminates**, because it called `exit` or for any other reason, **print the process's name and exit code**, formatted as if printed by `printf ("%s: exit(%d)\n", ...);`.&#x20;

* The name printed should be **the full name** passed to `process_execute()`, omitting command-line arguments.&#x20;
* <mark style="color:red;">**Do not print these messages when a kernel thread that is not a user process terminates, or when the**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**`halt`**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**system call is invoked.**</mark> The message is optional when a process fails to load.
* Aside from this, **don't print any other messages that Pintos as provided doesn't already print**. You may find extra messages useful during debugging, but they will confuse the grading scripts and thus lower your score.

{% hint style="success" %}
<mark style="color:green;">**Exercise 1.1**</mark>

<mark style="color:green;">**Print exit message**</mark> <mark style="color:green;"></mark><mark style="color:green;">formatted as</mark> <mark style="color:green;"></mark><mark style="color:green;">`"%s: exit(%d)\n"`</mark> <mark style="color:green;"></mark><mark style="color:green;">with</mark> <mark style="color:green;"></mark><mark style="color:green;">**process name**</mark> <mark style="color:green;"></mark><mark style="color:green;">and</mark> <mark style="color:green;"></mark><mark style="color:green;">**exit status**</mark> <mark style="color:green;"></mark><mark style="color:green;">when process is terminated.</mark>
{% endhint %}

## Task 2: Argument Passing

### **Exercise 2.1**

**Currently, `process_execute()` does not support passing arguments to new processes. You need to implement it in this task.**

* Implement this functionality, by extending `process_execute()` so that instead of simply taking a program file name as its argument, it divides it into words at spaces.&#x20;
* The first word is the program name, the second word is the first argument, and so on. That is, `process_execute("grep foo bar")` should run `grep` passing two arguments `foo` and `bar`.
* Within a command line, **multiple spaces are equivalent to a single space**, so that `process_execute("grep foo bar")` is equivalent to our original example.&#x20;
* **You can impose a reasonable limit on the length of the command line arguments.** For example, you could limit the arguments to those that will fit in a single page (4 kB). (There is an unrelated limit of 128 bytes on command-line arguments that the `pintos` utility can pass to the kernel.)

{% hint style="success" %}
<mark style="color:green;">**Exercise 2.1**</mark>

<mark style="color:green;">**Add argument passing support**</mark> <mark style="color:green;"></mark><mark style="color:green;">for</mark> <mark style="color:green;"></mark><mark style="color:green;">`process_execute()`</mark><mark style="color:green;">.</mark>
{% endhint %}

{% hint style="info" %}
**Hint**

**You can parse argument strings any way you like.**&#x20;

* If you're lost, look at**`strtok_r()`**, prototyped in `lib/string.h` and implemented with thorough comments in `lib/string.c`. You can find more about it by looking at the man page (run `man strtok_r` at the prompt).
{% endhint %}

See section [Program Startup Details](background.md#program-startup-details), for information on exactly how you need to set up the stack.

## Task 3: Accessing User Memory

### **Exercise 3.1**

**As part of a system call, the kernel must often access memory through pointers provided by a user program.**&#x20;

* <mark style="color:red;">**The kernel must be very careful about doing so**</mark>, because the user can pass a null pointer, a pointer to unmapped virtual memory, or a pointer to kernel virtual address space (above `PHYS_BASE`).&#x20;
* All of these types of invalid pointers must be rejected without harm to the kernel or other running processes, by <mark style="color:red;">**terminating the offending process and freeing its resources**</mark>.

{% hint style="success" %}
<mark style="color:green;">**Exercise 3.1**</mark>

<mark style="color:green;">Support</mark> <mark style="color:green;"></mark><mark style="color:green;">**reading from**</mark> <mark style="color:green;"></mark><mark style="color:green;">and</mark> <mark style="color:green;"></mark><mark style="color:green;">**writing to**</mark> <mark style="color:green;"></mark><mark style="color:green;"></mark> <mark style="color:green;"></mark><mark style="color:green;">**user memory for system calls**</mark><mark style="color:green;">.</mark>
{% endhint %}

**There are at least two reasonable ways to do this correctly.**&#x20;

1. **The first method is to verify the validity of a user-provided pointer, then dereference it.**&#x20;
   * If you choose this route, you'll want to look at the functions in `userprog/pagedir.c` and in `threads/vaddr.h`.&#x20;
   * This is the **simplest** way to handle user memory access.
2. **The second method is to check only that a user pointer points below `PHYS_BASE`, then dereference it.**&#x20;
   * <mark style="color:red;">**An invalid user pointer will cause a**</mark><mark style="color:red;">** **</mark>_<mark style="color:red;">**"page fault"**</mark>_ that you can handle by modifying the code for `page_fault()` in `userprog/exception.c`.&#x20;
   * This technique is normally **faster** because it takes advantage of the processor's MMU, so it tends to be used in real kernels (including Linux).

**In either case, you need to make sure not to "leak" resources.**&#x20;

* For example, suppose that your system call has acquired a lock or allocated memory with `malloc()`.&#x20;
* If you encounter an invalid user pointer afterward, you must still be sure to release the lock or free the page of memory.&#x20;
* If you choose to verify user pointers before dereferencing them, this should be straightforward.&#x20;
* It's more difficult to handle if an invalid pointer causes a page fault, because **there's no way to return an error code from a memory access**.&#x20;

<mark style="color:red;">**Therefore, for those who want to try the latter technique, we'll provide a little bit of helpful code, which utilizes**</mark><mark style="color:red;">** **</mark>_<mark style="color:red;">**assembly instructions**</mark>_<mark style="color:red;">** **</mark><mark style="color:red;">**and**</mark><mark style="color:red;">** **</mark>_<mark style="color:red;">**page fault handler**</mark>_<mark style="color:red;">** **</mark><mark style="color:red;">**to return an error code from dereferencing an invalid pointer.**</mark> You may need to read some material related to assembly code yourself before you can fully understand it.

* <mark style="color:blue;">**Helpful read/write Interfaces:**</mark> Each of these functions **assumes that the user address has already been verified to be below `PHYS_BASE`.**&#x20;

```cpp
/* Reads a byte at user virtual address UADDR.
   UADDR must be below PHYS_BASE.
   Returns the byte value if successful, -1 if a segfault
   occurred. */
static int
get_user (const uint8_t *uaddr)
{
  int result;
  asm ("movl $1f, %0; movzbl %1, %0; 1:"
       : "=&a" (result) : "m" (*uaddr));
  return result;
}

/* Writes BYTE to user address UDST.
   UDST must be below PHYS_BASE.
   Returns true if successful, false if a segfault occurred. */
static bool
put_user (uint8_t *udst, uint8_t byte)
{
  int error_code;
  asm ("movl $1f, %0; movb %b2, %1; 1:"
       : "=&a" (error_code), "=m" (*udst) : "q" (byte));
  return error_code != -1;
}
```

* <mark style="color:blue;">**You need to modify**</mark><mark style="color:blue;">** **</mark><mark style="color:blue;">**`page_fault()`**</mark><mark style="color:blue;">** **</mark><mark style="color:blue;">**meanwhile:**</mark> If the page fault is triggered by **a bad reference from a system call**, merely sets `eax` to `0xffffffff` (as the return value `-1`) and copies its former value into `eip`.

```c
/* Page fault handler needs to be modified. 
   The following code is absolutely INCOMPLETE!!! */
static void
page_fault (struct intr_frame *f) 
{
   ...
   if (page_fault_triggered_by_a_bad_reference_from_a_system_call) 
   {
      f->eip = (void (*) (void)) f->eax;
      f->eax = -1;
      return;
   }
   ...
}
```

## Task 4: System Calls

### **Exercise 4.1**

{% hint style="success" %}
<mark style="color:green;">**Exercise 4.1**</mark>

<mark style="color:green;">**Implement the system call handler in**</mark><mark style="color:green;">** **</mark><mark style="color:green;">**`userprog/syscall.c`**</mark><mark style="color:green;">**.**</mark> <mark style="color:green;"></mark><mark style="color:green;"></mark>&#x20;

* <mark style="color:green;">The skeleton implementation we provide "handles" system calls by terminating the process.</mark>&#x20;
* <mark style="color:green;">It will need to</mark> <mark style="color:green;"></mark><mark style="color:green;">**retrieve the system call number**</mark><mark style="color:green;">, then</mark> <mark style="color:green;"></mark><mark style="color:green;">**any system call arguments**</mark><mark style="color:green;">, and</mark> <mark style="color:green;"></mark><mark style="color:green;">**carry out appropriate actions**</mark><mark style="color:green;">.</mark>
{% endhint %}

### **Exercise 4.2**

{% hint style="success" %}
<mark style="color:green;">**Exercise 4.2**</mark>

<mark style="color:green;">**Implement the following system calls. (13 in all for this lab)**</mark>

* <mark style="color:green;">The prototypes listed are those</mark> <mark style="color:green;"></mark><mark style="color:green;">**seen by a user program**</mark> <mark style="color:green;"></mark><mark style="color:green;">that includes</mark> <mark style="color:green;"></mark><mark style="color:green;">`lib/user/syscall.h`</mark><mark style="color:green;">. (This header, and all others in</mark> <mark style="color:green;"></mark><mark style="color:green;">`lib/user`</mark><mark style="color:green;">, are for use by user programs only.)</mark>&#x20;
* <mark style="color:green;">**System call numbers**</mark> <mark style="color:green;"></mark><mark style="color:green;">for each system call are defined in</mark> <mark style="color:green;"></mark><mark style="color:green;">**`lib/syscall-nr.h`**</mark><mark style="color:green;">:</mark>
{% endhint %}

* <mark style="color:blue;">**System Call: void halt (void)**</mark>
  * **Terminates Pintos by calling `shutdown_power_off()`** (declared in `devices/shutdown.h`).&#x20;
  * This should be seldom used, because you lose some information about possible deadlock situations, etc.
* <mark style="color:blue;">**System Call: void exit (int status)**</mark>
  * **Terminates the current user program, returning **_**status**_** to the kernel.**&#x20;
  * <mark style="color:red;">**If the process's parent**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**`wait`**</mark><mark style="color:red;">**s for it (see below), this is the**</mark><mark style="color:red;">** **</mark>_<mark style="color:red;">**status**</mark>_<mark style="color:red;">** **</mark><mark style="color:red;">**that will be returned.**</mark> Conventionally, a _status_ of 0 indicates success and nonzero values indicate errors.
* <mark style="color:blue;">**System Call: pid\_t exec (const char \*cmd\_line)**</mark>
  * **Runs the executable whose name is given in **_**cmd\_line**_**, passing any given arguments, and returns the new process's program id (pid).**&#x20;
  * **If the program cannot load or run for any reason, must return pid `-1`**, which otherwise should not be a valid pid.
  * Thus, <mark style="color:red;">**the parent process cannot return from the**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**`exec`**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**until it knows whether the child process successfully loaded its executable**</mark>**.** You must use appropriate **synchronization** to ensure this.
* <mark style="color:blue;">**System Call: int wait (pid\_t pid)**</mark>
  * **Waits for a child process **_**pid**_** and retrieves the child's exit status.**
  * <mark style="color:red;">**If**</mark><mark style="color:red;">** **</mark>_<mark style="color:red;">**pid**</mark>_<mark style="color:red;">** **</mark><mark style="color:red;">**is still alive:**</mark>
    * **Wait until it terminates.** Then, returns the **status** that _pid_ passed to `exit`.&#x20;
    * If **pid did not call `exit()`**, but was terminated by the kernel (e.g. killed due to an exception), `wait(pid)` must return **-1**.&#x20;
  * <mark style="color:red;">**It is perfectly legal for a parent process to wait for child processes that have already terminated by the time the parent calls**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**`wait`**</mark>,&#x20;
    * but the kernel must still **allow the parent to retrieve its child's exit status**,&#x20;
    * or **learn that the child was terminated by the kernel**.
  * <mark style="color:red;">**`wait`**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**must fail and return -1 immediately if any of the following conditions is true:**</mark>
    1.  _**pid**_** does not refer to a **_**direct**_** child of the calling process.** _pid_ is a direct child of the calling process if and only if the calling process received pid as a return value from a successful call to `exec`.

        Note that **children are not inherited**: if A spawns child B and B spawns child process C, then A cannot wait for C, even if B is dead. A call to `wait(C)` by process A must fail.&#x20;

        Similarly, <mark style="color:red;">**orphaned processes are not assigned to a new parent**</mark> if their parent process exits before they do.
    2. **The process that calls `wait` has already called `wait` on **_**pid**_**.** That is, a process may wait for any given child at most once.
  * **Processes may spawn any number of children, wait for them in any order, and may even exit without having waited for some or all of their children.**&#x20;
    * Your design should consider all the ways in which waits can occur.&#x20;
    * <mark style="color:red;">**All of a process's resources, including its**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**`struct thread`**</mark><mark style="color:red;">**, must be freed**</mark> **whether its parent ever waits for it or not, and regardless of whether the child exits before or after its parent.**
  * <mark style="color:red;">**You must ensure that Pintos does not terminate until the initial process exits.**</mark>&#x20;
    * The supplied Pintos code tries to do this by calling `process_wait()` (in `userprog/process.c`) from `pintos_init()` (in `threads/init.c`).&#x20;
    * We suggest that you **implement `process_wait()` according to the comment at the top of the function** and then **implement the `wait` system call in terms of `process_wait()`**.

{% hint style="info" %}
**Hint 1**

**You need to store several pieces of execution information such as **_**exit status**_** for each process in case its parent **_**may**_** call `wait`.**&#x20;

* This information should be accessible to the parent process even after this process dies.&#x20;
* You should consider storing it in the heap (e.g. using malloc()).
{% endhint %}

{% hint style="info" %}
**Hint 2**

**Choosing where to initialize the execution information is important because the new process thread may finish before `process_execute` returns.**&#x20;

* If some parts of the execution information are designated for the child process, you may need to initialize them at a point where you are certain the newly created thread has not finished executing yet.&#x20;
* You may also want to read the **`thread_create`** function definition again, especially the last argument, which can help you pass argument from the parent thread to the child thread.
{% endhint %}

{% hint style="info" %}
**Hint 3**

**Choosing when and where to free the execution information is also important.**&#x20;

* The parent process could die before the child process and vice versa. You would want **whichever process that dies last to free the information**.
{% endhint %}

* <mark style="color:blue;">**System Call: bool create (const char \*file, unsigned initial\_size)**</mark>
  * **Creates a new file called **_**file**_** initially **_**initial\_size**_** bytes in size.** Returns true if successful, false otherwise.&#x20;
  * **Creating a new file does not open it**: opening the new file is a separate operation which would require a `open` system call.
* <mark style="color:blue;">**System Call: bool remove (const char \*file)**</mark>
  * **Deletes the file called **_**file**_**.** Returns true if successful, false otherwise.&#x20;
  * **A file may be removed regardless of whether it is open or closed, and removing an open file does not close it.** See [Removing an Open File](faq.md#what-happens-when-an-open-file-is-removed), for details.
* <mark style="color:blue;">**System Call: int open (const char \*file)**</mark>
  * **Opens the file called **_**file**_**.** Returns a **nonnegative integer handle** called a "file descriptor" (fd), or **-1** if the file could not be opened.
  * **File descriptors numbered 0 and 1 are reserved for the console**: fd 0 (`STDIN_FILENO`) is standard input, fd 1 (`STDOUT_FILENO`) is standard output. <mark style="color:red;">**The**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**`open`**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**system call will never return either of these file descriptors**</mark>, which are valid as system call arguments only as explicitly described below.
  * **Each process has an independent set of file descriptors.** <mark style="color:red;">**File descriptors are**</mark><mark style="color:red;">** **</mark>_<mark style="color:red;">**not**</mark>_<mark style="color:red;">** **</mark><mark style="color:red;">**inherited by child processes (different from Unix semantics)!!!**</mark>
  * **When a single file is opened more than once, whether by a single process or different processes, each `open` returns a new file descriptor.** Different file descriptors for a single file are closed independently in separate calls to `close` and they do not share a file position.
* <mark style="color:blue;">**System Call: int filesize (int fd)**</mark>
  * **Returns the size, in bytes, of the file open as **_**fd**_**.**
* <mark style="color:blue;">**System Call: int read (int fd, void \*buffer, unsigned size)**</mark>
  * **Reads **_**size**_** bytes from the file open as **_**fd**_** into **_**buffer**_**.** Returns **the number of bytes actually read (0 at end of file)**, or **-1** if the file could not be read (due to a condition other than end of file).&#x20;
  * <mark style="color:red;">**Fd 0 reads from the keyboard using**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**`input_getc()`**</mark><mark style="color:red;">**.**</mark>
* <mark style="color:blue;">**System Call: int write (int fd, const void \*buffer, unsigned size)**</mark>
  * **Writes size bytes from **_**buffer**_** to the open file **_**fd**_**.** Returns **the number of bytes actually written**, which may be less than _size_ if some bytes could not be written.
  * **Writing past end-of-file would normally extend the file, but file growth is not implemented by the basic file system.** The expected behavior is to **write as many bytes as possible** up to end-of-file and return the actual number written, or 0 if no bytes could be written at all.
  * <mark style="color:red;">**Fd 1 writes to the console.**</mark> <mark style="color:red;">**Your code to write to the console should write all of**</mark><mark style="color:red;">** **</mark>_<mark style="color:red;">**buffer**</mark>_<mark style="color:red;">** **</mark><mark style="color:red;">**in**</mark><mark style="color:red;">** **</mark>_<mark style="color:red;">**one call**</mark>_<mark style="color:red;">** **</mark><mark style="color:red;">**to**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**`putbuf()`**</mark>, at least as long as _size_ is not bigger than a few hundred bytes. (It is reasonable to break up larger buffers.) **Otherwise, lines of text output by different processes may end up interleaved on the console**, confusing both human readers and our grading scripts.
* <mark style="color:blue;">**System Call: void seek (int fd, unsigned position)**</mark>
  * **Changes the next byte to be read or written in open file **_**fd**_** to **_**position**_**, expressed in bytes from the beginning of the file.** (Thus, a position of 0 is the file's start.)
  * **A seek past the current end of a file is **_**not**_** an error.**&#x20;
    * A later read obtains 0 bytes, indicating end of file.&#x20;
    * A later write extends the file, filling any unwritten gap with zeros. (However, in Pintos files have a fixed length until project 4 is complete, so writes past end of file will return an error.)&#x20;
    * These semantics are implemented in the file system and <mark style="color:red;">**do not require any special effort**</mark> in system call implementation.
* <mark style="color:blue;">**System Call: unsigned tell (int fd)**</mark>
  * **Returns the position of the next byte to be read or written in open file **_**fd**_**, expressed in bytes from the beginning of the file.**
* <mark style="color:blue;">**System Call: void close (int fd)**</mark>
  * **Closes file descriptor **_**fd**_**.**&#x20;
  * <mark style="color:red;">**Exiting or terminating a process implicitly closes all its open file descriptors, as if by calling this function for each one.**</mark>

**The file defines other syscalls. Ignore them for now.** You will implement some of them in project 3 and the rest in project 4, so <mark style="color:red;">**be sure to design your system with extensibility in mind**</mark>.

#### <mark style="color:red;">Some Important Notes</mark>

**To implement syscalls, you need to provide ways to read and write data in **_**user virtual address space**_**.**&#x20;

* **You need this ability before you can even obtain the system call number**, because **the system call number is on the user's stack in the user's virtual address space**.&#x20;
* This can be a bit tricky: what if the user provides an invalid pointer, a pointer into kernel memory, or a block partially in one of those regions? **You should handle these cases by terminating the user process.** We recommend writing and testing this code before implementing any other system call functionality. See section [Accessing User Memory](your-tasks.md#task-3-accessing-user-memory), for more information.

**You must synchronize system calls so that any number of user processes can make them at once.**&#x20;

* In particular, it is not safe to call into the file system code provided in the `filesys/` directory from multiple threads at once. <mark style="color:red;">**Your system call implementation must treat the file system code as a critical section.**</mark> **Don't forget that `process_execute()` also accesses files.** For now, we recommend **against** modifying code in the `filesys/` directory.

We have provided you a **user-level function** for each system call in **`lib/user/syscall.c`**. These provide a way for user processes to invoke each system call from a C program. Each uses a little **inline assembly code** to invoke the system call and (if appropriate) returns the system call's return value.

**When you're done with this part, and forevermore, Pintos should be bulletproof.** Nothing that a user program can do should ever cause the OS to crash, panic, fail an assertion, or otherwise malfunction.&#x20;

* <mark style="color:red;">**It is important to emphasize this point: our tests will try to break your system calls in many, many ways. You need to think of all the corner cases and handle them.**</mark>&#x20;
* The **sole** way a user program should be able to cause the OS to halt is by invoking the `halt` system call.
* If a system call is passed **an invalid argument**, acceptable options include **returning an error value (for those calls that return a value)**, **returning an undefined value**, or **terminating the process**.

See section [System Call Details](background.md#system-call-details), for details on how system calls work.

## Task 5: Denying Writes to Executables

### **Exercise 5.1**

{% hint style="success" %}
<mark style="color:green;">**Exercise 5.1**</mark>

<mark style="color:green;">**Add code to deny writes to files in use as executables.**</mark> <mark style="color:green;"></mark><mark style="color:green;"></mark>&#x20;

* <mark style="color:green;">Many OSes do this because of the unpredictable results if a process tried to run code that was in the midst of being changed on disk.</mark>&#x20;
* <mark style="color:green;">This is especially important once virtual memory is implemented in project 3, but it can't hurt even now.</mark>
{% endhint %}

**You can use `file_deny_write()` to prevent writes to an open file.**&#x20;

* Calling **`file_allow_write()`** on the file will **re-enable** them (unless the file is denied writes by another opener).&#x20;
* **Closing a file will also re-enable writes.** Thus, to deny writes to a process's executable, you must keep it open as long as the process is still running.
