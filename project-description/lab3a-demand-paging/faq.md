# FAQ

## General FAQ

#### <mark style="color:red;background-color:red;">**Do we need a working Project 2 to implement Project 3?**</mark>

<mark style="color:red;background-color:red;">**Yes.**</mark>

#### **What extra credit is available?**

You may implement sharing: when multiple processes are created that use the same executable file, share read-only pages among those processes instead of creating separate copies of read-only segments for each process. If you carefully designed your data structures, sharing of read-only pages should not make this part significantly harder.

#### **How do we resume a process after we have handled a page fault?**

Returning from `page_fault()` resumes the current user process (see section [Internal Interrupt Handling](../../appendix/reference-guide/interrupt-handling.md#internal-interrupt-handling)). It will then retry the instruction to which the instruction pointer points.

#### **Does the virtual memory system need to support data segment growth?**

No. The size of the data segment is determined by the linker. We still have no dynamic allocation in Pintos (although it is possible to "fake" it at the user level by using memory-mapped files). Supporting data segment growth should add little additional complexity to a well-designed system.

#### **Why should I use `PAL_USER` for allocating page frames?**

Passing `PAL_USER` to `palloc_get_page()` causes it to allocate memory from the user pool, instead of the main kernel pool. Running out of pages in the user pool just causes user programs to page, but running out of pages in the kernel pool will cause many failures because so many kernel functions need to obtain memory. You can layer some other allocator on top of `palloc_get_page()` if you like, but it should be the underlying mechanism.
