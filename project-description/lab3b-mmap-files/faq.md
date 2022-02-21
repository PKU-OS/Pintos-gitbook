# FAQ

## General FAQ

#### **How much code will I need to write?**

Here's a summary of our reference solution, produced by the `diffstat` program. The final row gives total lines inserted and deleted; a changed line counts as both an insertion and a deletion.

This summary is relative to the Pintos base code, but the reference solution for project 3 starts from the reference solution to project 2. See section FAQ, for the summary of project 2.

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
