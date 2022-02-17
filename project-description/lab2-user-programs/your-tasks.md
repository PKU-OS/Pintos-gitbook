# Your Tasks

## Task 0: Design Document

Download [the project 2 design document template](https://github.com/PKU-OS/pintos/blob/master/docs/p2.md). Read through it to motivate your design and fill it in after you finish the project. We recommend that you read the design document template before you start working on the project. See section [Project Documentation](../../appendix/project-documentation.md), for a sample design document that goes along with a fictitious project.

## Task 1: Process Termination Messages

Whenever a user process terminates, because it called `exit` or for any other reason, print the process's name and exit code, formatted as if printed by `printf ("%s: exit(%d)\n", ...);`. The name printed should be the full name passed to `process_execute()`, omitting command-line arguments. Do not print these messages when a kernel thread that is not a user process terminates, or when the `halt` system call is invoked. The message is optional when a process fails to load.

{% hint style="success" %}
**Exercise 1.1**

Print exit message formatted as `"%s: exit(%d)\n"` with process name and exit status when process is terminated.
{% endhint %}

Aside from this, don't print any other messages that Pintos as provided doesn't already print. You may find extra messages useful during debugging, but they will confuse the grading scripts and thus lower your score.

## Task 2: Argument Passing

Currently, `process_execute()` does not support passing arguments to new processes. Implement this functionality, by extending `process_execute()` so that instead of simply taking a program file name as its argument, it divides it into words at spaces. The first word is the program name, the second word is the first argument, and so on. That is, `process_execute("grep foo bar")` should run `grep` passing two arguments `foo` and `bar`.

{% hint style="success" %}
**Exercise 2.1**

Add argument passing support for `process_execute()`.
{% endhint %}

Within a command line, multiple spaces are equivalent to a single space, so that `process_execute("grep foo bar")` is equivalent to our original example. You can impose a reasonable limit on the length of the command line arguments. For example, you could limit the arguments to those that will fit in a single page (4 kB). (There is an unrelated limit of 128 bytes on command-line arguments that the `pintos` utility can pass to the kernel.)

{% hint style="info" %}
**Hint**

You can parse argument strings any way you like. If you're lost, look at`strtok_r()`, prototyped in `lib/string.h` and implemented with thorough comments in `lib/string.c`. You can find more about it by looking at the man page (run `man strtok_r` at the prompt).
{% endhint %}

See section [Program Startup Details](background.md#program-startup-details), for information on exactly how you need to set up the stack.

## Task 3: Accessing User Memory

As part of a system call, the kernel must often access memory through pointers provided by a user program. The kernel must be very careful about doing so, because the user can pass a null pointer, a pointer to unmapped virtual memory, or a pointer to kernel virtual address space (above `PHYS_BASE`). All of these types of invalid pointers must be rejected without harm to the kernel or other running processes, by terminating the offending process and freeing its resources.

{% hint style="success" %}
**Exercise 3.1**

Support reading from and writing to user memory for system calls.
{% endhint %}

There are at least two reasonable ways to do this correctly. The first method is to verify the validity of a user-provided pointer, then dereference it. If you choose this route, you'll want to look at the functions in `userprog/pagedir.c` and in `threads/vaddr.h`. This is the simplest way to handle user memory access.

The second method is to check only that a user pointer points below `PHYS_BASE`, then dereference it. An invalid user pointer will cause a "page fault" that you can handle by modifying the code for `page_fault()` in `userprog/exception.c`. This technique is normally faster because it takes advantage of the processor's MMU, so it tends to be used in real kernels (including Linux).

In either case, you need to make sure not to "leak" resources. For example, suppose that your system call has acquired a lock or allocated memory with `malloc()`. If you encounter an invalid user pointer afterward, you must still be sure to release the lock or free the page of memory. If you choose to verify user pointers before dereferencing them, this should be straightforward. It's more difficult to handle if an invalid pointer causes a page fault, because there's no way to return an error code from a memory access. Therefore, for those who want to try the latter technique, we'll provide a little bit of helpful code:

```c
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

Each of these functions assumes that the user address has already been verified to be below `PHYS_BASE`. They also assume that you've modified `page_fault()` so that a page fault in the kernel merely sets `eax` to `0xffffffff` (as the return value `-1`) and copies its former value into `eip`.

## Task 4: System Calls

{% hint style="success" %}
**Exercise 4.1**

Implement the system call handler in `userprog/syscall.c`. The skeleton implementation we provide "handles" system calls by terminating the process. It will need to retrieve the system call number, then any system call arguments, and carry out appropriate actions.
{% endhint %}

{% hint style="success" %}
**Exercise 4.2**

Implement the following system calls. The prototypes listed are those seen by a user program that includes `lib/user/syscall.h`. (This header, and all others in `lib/user`, are for use by user programs only.) System call numbers for each system call are defined in `lib/syscall-nr.h`:
{% endhint %}

#### System Call: void **halt** (void)

Terminates Pintos by calling `shutdown_power_off()` (declared in `devices/shutdown.h`). This should be seldom used, because you lose some information about possible deadlock situations, etc.

#### System Call: void **exit** (int status)

Terminates the current user program, returning _status_ to the kernel. If the process's parent `wait`s for it (see below), this is the _status_ that will be returned. Conventionally, a _status_ of 0 indicates success and nonzero values indicate errors.

#### System Call: pid\_t **exec** (const char \*cmd\_line)

Runs the executable whose name is given in _cmd\_line_, passing any given arguments, and returns the new process's program id (pid). Must return pid `-1`, which otherwise should not be a valid pid, if the program cannot load or run for any reason. Thus, the parent process cannot return from the `exec` until it knows whether the child process successfully loaded its executable. You must use appropriate synchronization to ensure this.

#### System Call: int **wait** (pid\_t pid)

Waits for a child process _pid_ and retrieves the child's exit status.

If _pid_ is still alive, waits until it terminates. Then, returns the status that _pid_ passed to `exit`. If pid did not call `exit()`, but was terminated by the kernel (e.g. killed due to an exception), `wait(pid)` must return -1. It is perfectly legal for a parent process to wait for child processes that have already terminated by the time the parent calls `wait`, but the kernel must still allow the parent to retrieve its child's exit status, or learn that the child was terminated by the kernel.

`wait` must fail and return -1 immediately if any of the following conditions is true:

*   _pid_ does not refer to a direct child of the calling process. _pid_ is a direct child of the calling process if and only if the calling process received pid as a return value from a successful call to `exec`.

    Note that children are not inherited: if A spawns child B and B spawns child process C, then A cannot wait for C, even if B is dead. A call to `wait(C)` by process A must fail. Similarly, orphaned processes are not assigned to a new parent if their parent process exits before they do.
* The process that calls `wait` has already called `wait` on _pid_. That is, a process may wait for any given child at most once.

Processes may spawn any number of children, wait for them in any order, and may even exit without having waited for some or all of their children. Your design should consider all the ways in which waits can occur. All of a process's resources, including its `struct thread`, must be freed whether its parent ever waits for it or not, and regardless of whether the child exits before or after its parent.

You must ensure that Pintos does not terminate until the initial process exits. The supplied Pintos code tries to do this by calling `process_wait()` (in `userprog/process.c`) from `pintos_init()` (in `threads/init.c`). We suggest that you implement `process_wait()` according to the comment at the top of the function and then implement the `wait` system call in terms of `process_wait()`.

{% hint style="info" %}
**Hint 1**

You need to store several pieces of execution information such as exit status for each process in case its parent _may_ call `wait`. This information should be accessible to the parent process even after this process dies. You should consider storing it in the heap (e.g. using malloc()).
{% endhint %}

{% hint style="info" %}
**Hint 2**

Choosing where to initialize the execution information is important because the new process thread may finish before `process_execute` returns. If some parts of the execution information are designated for the child process, you may need to initialize them at a point where you are certain the newly created thread has not finished executing yet. You may also want to read the `thread_create` function definition again, especially the last argument, which can help you pass argument from the parent thread to the child thread.
{% endhint %}

{% hint style="info" %}
**Hint 3**

Choosing when and where to free the execution information is also important. The parent process could die before the child process and vice versa. You would want whichever process that dies last to free the information.
{% endhint %}

#### System Call: bool **create** (const char \*file, unsigned initial\_size)

Creates a new file called _file_ initially _initial\_size_ bytes in size. Returns true if successful, false otherwise. Creating a new file does not open it: opening the new file is a separate operation which would require a `open` system call.

#### System Call: bool **remove** (const char \*file)

Deletes the file called _file_. Returns true if successful, false otherwise. A file may be removed regardless of whether it is open or closed, and removing an open file does not close it. See [Removing an Open File](faq.md#what-happens-when-an-open-file-is-removed), for details.

#### System Call: int **open** (const char \*file)

Opens the file called _file_. Returns a nonnegative integer handle called a "file descriptor" (fd), or -1 if the file could not be opened.

File descriptors numbered 0 and 1 are reserved for the console: fd 0 (`STDIN_FILENO`) is standard input, fd 1 (`STDOUT_FILENO`) is standard output. The `open` system call will never return either of these file descriptors, which are valid as system call arguments only as explicitly described below.

Each process has an independent set of file descriptors. File descriptors are _**not**_ inherited by child processes (different from Unix semantics).

When a single file is opened more than once, whether by a single process or different processes, each `open` returns a new file descriptor. Different file descriptors for a single file are closed independently in separate calls to `close` and they do not share a file position.

#### System Call: int **filesize** (int fd)

Returns the size, in bytes, of the file open as _fd_.

#### System Call: int **read** (int fd, void \*buffer, unsigned size)

Reads _size_ bytes from the file open as _fd_ into _buffer_. Returns the number of bytes actually read (0 at end of file), or -1 if the file could not be read (due to a condition other than end of file). Fd 0 reads from the keyboard using `input_getc()`.

#### System Call: int **write** (int fd, const void \*buffer, unsigned size)

Writes size bytes from _buffer_ to the open file _fd_. Returns the number of bytes actually written, which may be less than _size_ if some bytes could not be written.

Writing past end-of-file would normally extend the file, but file growth is not implemented by the basic file system. The expected behavior is to write as many bytes as possible up to end-of-file and return the actual number written, or 0 if no bytes could be written at all.

Fd 1 writes to the console. Your code to write to the console should write all of _buffer_ in one call to `putbuf()`, at least as long as _size_ is not bigger than a few hundred bytes. (It is reasonable to break up larger buffers.) Otherwise, lines of text output by different processes may end up interleaved on the console, confusing both human readers and our grading scripts.

#### System Call: void **seek** (int fd, unsigned position)

Changes the next byte to be read or written in open file _fd_ to _position_, expressed in bytes from the beginning of the file. (Thus, a position of 0 is the file's start.)

A seek past the current end of a file is not an error. A later read obtains 0 bytes, indicating end of file. A later write extends the file, filling any unwritten gap with zeros. (However, in Pintos files have a fixed length until project 4 is complete, so writes past end of file will return an error.) These semantics are implemented in the file system and do not require any special effort in system call implementation.

#### System Call: unsigned **tell** (int fd)

Returns the position of the next byte to be read or written in open file _fd_, expressed in bytes from the beginning of the file.

#### System Call: void **close** (int fd)

Closes file descriptor _fd_. Exiting or terminating a process implicitly closes all its open file descriptors, as if by calling this function for each one.

The file defines other syscalls. Ignore them for now. You will implement some of them in project 3 and the rest in project 4, so be sure to design your system with extensibility in mind.

To implement syscalls, you need to provide ways to read and write data in user virtual address space. You need this ability before you can even obtain the system call number, because the system call number is on the user's stack in the user's virtual address space. This can be a bit tricky: what if the user provides an invalid pointer, a pointer into kernel memory, or a block partially in one of those regions? You should handle these cases by terminating the user process. We recommend writing and testing this code before implementing any other system call functionality. See section [Accessing User Memory](your-tasks.md#3.-accessing-user-memory), for more information.

You must synchronize system calls so that any number of user processes can make them at once. In particular, it is not safe to call into the file system code provided in the `filesys/` directory from multiple threads at once. Your system call implementation must treat the file system code as a critical section. Don't forget that `process_execute()` also accesses files. For now, we recommend against modifying code in the `filesys/` directory.

We have provided you a user-level function for each system call in `lib/user/syscall.c`. These provide a way for user processes to invoke each system call from a C program. Each uses a little inline assembly code to invoke the system call and (if appropriate) returns the system call's return value.

When you're done with this part, and forevermore, Pintos should be bulletproof. Nothing that a user program can do should ever cause the OS to crash, panic, fail an assertion, or otherwise malfunction. It is important to emphasize this point: our tests will try to break your system calls in many, many ways. You need to think of all the corner cases and handle them. The sole way a user program should be able to cause the OS to halt is by invoking the `halt` system call.

If a system call is passed an invalid argument, acceptable options include returning an error value (for those calls that return a value), returning an undefined value, or terminating the process.

See section [System Call Details](background.md#system-call-details), for details on how system calls work.

## Task 5: Denying Writes to Executables

{% hint style="success" %}
**Exercise 5.1**

Add code to deny writes to files in use as executables. Many OSes do this because of the unpredictable results if a process tried to run code that was in the midst of being changed on disk. This is especially important once virtual memory is implemented in project 3, but it can't hurt even now.
{% endhint %}

You can use `file_deny_write()` to prevent writes to an open file. Calling `file_allow_write()` on the file will re-enable them (unless the file is denied writes by another opener). Closing a file will also re-enable writes. Thus, to deny writes to a process's executable, you must keep it open as long as the process is still running.

##
