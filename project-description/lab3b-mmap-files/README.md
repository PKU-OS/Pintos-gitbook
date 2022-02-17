# Lab3b: Mmap Files

## Requirements

### 0. Design Document

Before you turn in your project, you must copy [the project 3b design document template](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/vm\_part2.tmpl) into your source tree under the name pintos/src/vm/PART2\_DESIGNDOC and fill it in. We recommend that you read the design document template before you start working on the project. See section [D. Project Documentation](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_10.html#SEC153), for a sample design document that goes along with a fictitious project.

### 1. Stack Growth

{% hint style="success" %}
**Exercise 1.1**

Implement stack growth. In project 2, the stack was a single page at the top of the user virtual address space, and programs were limited to that much stack. Now, if the stack grows past its current size, allocate additional pages as necessary.
{% endhint %}

Allocate additional pages only if they "appear" to be stack accesses. Devise a heuristic that attempts to distinguish stack accesses from other accesses.

User programs are buggy if they write to the stack below the stack pointer, because typical real OSes may interrupt a process at any time to deliver a "signal," which pushes data on the stack.However, the 80x86 `PUSH` instruction checks access permissions before it adjusts the stack pointer, so it may cause a page fault 4 bytes below the stack pointer. (Otherwise, `PUSH` would not be restartable in a straightforward fashion.) Similarly, the `PUSHA` instruction pushes 32 bytes at once, so it can fault 32 bytes below the stack pointer.

You will need to be able to obtain the current value of the user program's stack pointer. Within a system call or a page fault generated by a user program, you can retrieve it from the `esp` member of the `struct intr_frame` passed to `syscall_handler()` or `page_fault()`, respectively. If you verify user pointers before accessing them (see section 3 [Accessing User Memory](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/project2.html#SEC50) in project 2), these are the only cases you need to handle. On the other hand, if you depend on page faults to detect invalid memory access, you will need to handle another case, where a page fault occurs in the kernel. Since the processor only saves the stack pointer when an exception causes a switch from user to kernel mode, reading `esp` out of the `struct intr_frame` passed to `page_fault()` would yield an undefined value, not the user stack pointer. You will need to arrange another way, such as saving `esp` into `struct thread` on the initial transition from user to kernel mode.

You should impose some absolute limit on stack size, as do most OSes. Some OSes make the limit user-adjustable, e.g. with the `ulimit` command on many Unix systems. On many GNU/Linux systems, the default limit is 8 MB.

The first stack page need not be allocated lazily. You can allocate and initialize it with the command line arguments at load time, with no need to wait for it to be faulted in.

All stack pages should be candidates for eviction. An evicted stack page should be written to swap.

### 2. Memory Mapped Files

The file system is most commonly accessed with `read` and `write` system calls. A secondary interface is to "map" the file into virtual pages, using the `mmap` system call. The program can then use memory instructions directly on the file data.

Suppose file foo is 0x1000 bytes (4 kB, or one page) long. If foo is mapped into memory starting at address 0x5000, then any memory accesses to locations 0x5000...0x5fff will access the corresponding bytes of foo.

Here's a program that uses `mmap` to print a file to the console. It opens the file specified on the command line, maps it at virtual address 0x10000000, writes the mapped data to the console (fd 1), and unmaps the file.

```
#include <stdio.h>
#include <syscall.h>
int main (int argc UNUSED, char *argv[]) 
{
  void *data = (void *) 0x10000000;     /* Address at which to map. */

  int fd = open (argv[1]);              /* Open file. */
  mapid_t map = mmap (fd, data);        /* Map file. */
  write (1, data, filesize (fd));       /* Write file to console. */
  munmap (map);                         /* Unmap file (optional). */
  return 0;
}
```

A similar program with full error handling is included as mcat.c in the examples directory, which also contains mcp.c as a second example of `mmap`.

Your submission must be able to track what memory is used by memory mapped files. You need a table of file mappings to track which files are mapped into which pages. This is necessary to properly handle page faults in the mapped regions and to ensure that mapped files do not overlap any other segments within the process.

{% hint style="success" %}
**Exercise 2.1**

Implement memory mapped files, including the following system calls.

* mapid\_t **mmap** (int fd, void \*addr)
* void **munmap** (mapid\_t mapping)
{% endhint %}

#### System Call: mapid\_t **mmap** (int fd, void \*addr)

Maps the file open as fd into the process's virtual address space. The entire file is mapped into consecutive virtual pages starting at addr.

Your VM system must lazily load pages in `mmap` regions and use the `mmap`ed file itself as backing store for the mapping. That is, evicting a page mapped by `mmap` writes it back to the file it was mapped from.

If the file's length is not a multiple of `PGSIZE`, then some bytes in the final mapped page "stick out" beyond the end of the file. Set these bytes to zero when the page is faulted in from the file system, and discard them when the page is written back to disk.

If successful, this function returns a "mapping ID" that uniquely identifies the mapping within the process. On failure, it must return -1, which otherwise should not be a valid mapping id, and the process's mappings must be unchanged.

A call to `mmap` may fail if the file open as fd has a length of zero bytes. It must fail if addr is not page-aligned or if the range of pages mapped overlaps any existing set of mapped pages, including the stack or pages mapped at executable load time. It must also fail if addr is 0, because some Pintos code assumes virtual page 0 is not mapped. Finally, file descriptors 0 and 1, representing console input and output, are not mappable.

#### System Call: void **munmap** (mapid\_t mapping)

Unmaps the mapping designated by mapping, which must be a mapping ID returned by a previous call to `mmap` by the same process that has not yet been unmapped.

All mappings are implicitly unmapped when a process exits, whether via `exit` or by any other means. When a mapping is unmapped, whether implicitly or explicitly, all pages written to by the process are written back to the file, and pages not written must not be. The pages are then removed from the process's list of virtual pages.

Closing or removing a file does not unmap any of its mappings. Once created, a mapping is valid until `munmap` is called or the process exits, following the Unix convention. See [Removing an Open File](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_4.html#Removing%20an%20Open%20File), for more information. You should use the `file_reopen` function to obtain a separate and independent reference to the file for each of its mappings.

If two or more processes map the same file, there is no requirement that they see consistent data. Unix handles this by making the two mappings share the same physical page, but the `mmap` system call also has an argument allowing the client to specify whether the page is shared or private (i.e. copy-on-write).

## Submission Instruction

{% hint style="info" %}
**Git Branch**

We will collect your solution automatically through GitHub by taking a snapshot by the deadline. Thus, be sure to commit your changes and do a `git push` to GitHub, especially in the last few minutes! Your submission must reside in a branch called `lab3b-handin`. If you are developing in other branches, in the end, don't forget to merge changes from that branch to the `lab3b-handin` branch.
{% endhint %}

{% hint style="info" %}
**Late Hours**

If you decide to use the late hour tokens, fill out [this form](https://forms.office.com/r/PV6czESZpS) **before the deadline**, so that we won't be collecting and grading your solution immediately. **When you finish** (within the token limit), fill out [this form](https://forms.office.com/r/PV6czESZpS) again to indicate you are done. Don't forget to fill out the form for the second time to avoid leaking your late tokens.
{% endhint %}

## FAQ

#### **How much code will I need to write?**

Here's a summary of our reference solution, produced by the `diffstat` program. The final row gives total lines inserted and deleted; a changed line counts as both an insertion and a deletion.

This summary is relative to the Pintos base code, but the reference solution for project 3 starts from the reference solution to project 2. See section [4.4 FAQ](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_4.html#SEC58), for the summary of project 2.

The reference solution represents just one possible solution. Many other solutions are also possible and many of those differ greatly from the reference solution. Some excellent solutions may not modify all the files modified by the reference solution, and some may modify files not modified by the reference solution.

```
 Makefile.build       |    4
 devices/timer.c      |   42 ++
 threads/init.c       |    5
 threads/interrupt.c  |    2
 threads/thread.c     |   31 +
 threads/thread.h     |   37 +-
 userprog/exception.c |   12
 userprog/pagedir.c   |   10
 userprog/process.c   |  319 +++++++++++++-----
 userprog/syscall.c   |  545 ++++++++++++++++++++++++++++++-
 userprog/syscall.h   |    1
 vm/frame.c           |  162 +++++++++
 vm/frame.h           |   23 +
 vm/page.c            |  297 ++++++++++++++++
 vm/page.h            |   50 ++
 vm/swap.c            |   85 ++++
 vm/swap.h            |   11
 17 files changed, 1532 insertions(+), 104 deletions(-)
```

#### **Do we need a working Project 2 to implement Project 3?**

Yes.

#### **Why do user processes sometimes fault above the stack pointer?**

You might notice that, in the stack growth tests, the user program faults on an address that is above the user program's current stack pointer, even though the `PUSH` and `PUSHA` instructions would cause faults 4 and 32 bytes below the current stack pointer.

This is not unusual. The `PUSH` and `PUSHA` instructions are not the only instructions that can trigger user stack growth. For instance, a user program may allocate stack space by decrementing the stack pointer using a `SUB $n, %esp` instruction, and then use a `MOV ..., m(%esp)` instruction to write to a stack location within the allocated space that is m bytes above the current stack pointer. Such accesses are perfectly valid, and your kernel must grow the user program's stack to allow those accesses to succeed.

#### **Does the virtual memory system need to support data segment growth?**

No. The size of the data segment is determined by the linker. We still have no dynamic allocation in Pintos (although it is possible to "fake" it at the user level by using memory-mapped files). Supporting data segment growth should add little additional complexity to a well-designed system.