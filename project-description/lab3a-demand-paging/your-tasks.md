# Your Tasks

**This assignment is an open-ended design problem.**

* We are going to say as little as possible about how to do things. Instead we will focus on what functionality we require your OS to support.
* We will expect you to come up with a design that makes sense. You will have the freedom to choose how to handle page faults, how to organize the swap partition, how to implement paging, etc.

{% hint style="success" %}
<mark style="color:green;">**Here are all the tests you need to pass to get a full score in Lab3a.**</mark>

1. <mark style="color:green;">**All tests in**</mark> `tests/userprog`
2. <mark style="color:green;">**All tests in**</mark> `tests/filesys/base`
3. <mark style="color:green;">**Part of the tests in**</mark> `tests/vm`
   * page-linear
   * page-parallel
   * page-shuffle
   * page-merge-seq
   * page-merge-par
   * pt-bad-addr
   * pt-bad-read
   * pt-write-code
   * pt-write-code2
   * pt-grow-bad
{% endhint %}

## Task 0: Design Document

* Download the [project 3a design document template](https://github.com/PKU-OS/pintos/blob/master/docs/p3a.md). Read through it to motivate your design and fill it in after you finish the project.
* We recommend that you read the design document template before you start working on the project.
* See section [Project Documentation](../../appendix/project-documentation.md), for a sample design document that goes along with a fictitious project.

## Task 1: Paging

### **Exercise 1.1**

{% hint style="success" %}
<mark style="color:green;">**Exercise 1.1**</mark>

<mark style="color:green;">**Implement paging for segments loaded from executables.**</mark>

* <mark style="color:green;">All of these pages should be loaded</mark> <mark style="color:green;">**lazily**</mark><mark style="color:green;">, that is, only as the kernel intercepts page faults for them.</mark>
* <mark style="color:green;">Upon eviction:</mark>
  * <mark style="color:green;">Pages</mark> <mark style="color:green;">**modified**</mark> <mark style="color:green;">since load (e.g. as indicated by the "dirty bit") should</mark> <mark style="color:green;">**be written to swap**</mark><mark style="color:green;">.</mark>
  * <mark style="color:green;">**Unmodified**</mark> <mark style="color:green;">pages, including read-only pages, should</mark> <mark style="color:green;">**never be written to swap**</mark> <mark style="color:green;">because they can always be read back from the executable.</mark>
{% endhint %}

### **Exercise 1.2**

{% hint style="success" %}
<mark style="color:green;">**Exercise 1.2**</mark>

<mark style="color:green;">**Implement a global page replacement algorithm that approximates LRU.**</mark>

* <mark style="color:green;">Your algorithm should perform</mark> <mark style="color:green;">**at least as well as**</mark> <mark style="color:green;">the simple variant of the "second chance" or "clock" algorithm.</mark>
{% endhint %}

**Your design should allow for parallelism.** If one page fault requires I/O, in the meantime processes that do not fault should continue executing and other page faults that do not require I/O should be able to complete. This will require some **synchronization** effort.

**You'll need to modify the core of the program loader**, which is the loop in **`load_segment()`** in `userprog/process.c`.

* Each time around the loop, **`page_read_bytes`** receives the number of bytes to read from the executable file and **`page_zero_bytes`** receives the number of bytes to initialize to zero following the bytes read. **The two always sum to `PGSIZE` (4,096).** The handling of a page depends on these variables' values:
* If `page_read_bytes` equals `PGSIZE`, the page should be demand paged from the underlying file on its first access.
* If `page_zero_bytes` equals `PGSIZE`, the page does not need to be read from disk at all because it is all zeroes. You should handle such pages by **creating a new page consisting of all zeroes at the first page fault**.
* Otherwise, neither `page_read_bytes` nor `page_zero_bytes` equals `PGSIZE`. In this case, an initial part of the page is to be read from the underlying file and the remainder zeroed.

{% hint style="info" %}
**Hint**

<mark style="color:red;">**In order for demand paging to work, you need to record metadata for each lazily-loaded page**</mark>, which allows you to know what location to read its content from disk later.

* In particular, if before demand paging a page's content comes from reading offset `X` of the executable file at loading time, after demand paging, you should still read the content from offset `X` of the executable file during page fault handling.
* <mark style="color:red;">**The supplementary page table keeps track of relationship of memory pages and their backing store locations.**</mark> You should consider filling in the supplementary page table in `load_segment`.
{% endhint %}

{% hint style="info" %}
**Tips**

If you would like to retain the previous file-reading code in `load_segment`, you can use macros like this to **select the behavior of `load_segment` at compilation time**:

```
static bool load_segment(...)
  {
  #ifndef VM
    file_seek (file, ofs);
  ...
  #else
  ... // fill in code for demand paging behavior in lab 3.
  #endif
  }
  
```

If you compile Pintos under lab 1 (`threads` directory) or lab 2 (`userprog` directory), the `#ifndef VM` section will be selected. If you compile Pintos under lab 3 or lab 4, the `#else` section will be selected.
{% endhint %}

{% hint style="info" %}
**Tips**

**You can use the `-ul`** **kernel command-line option to limit the size of the user pool, which makes it easy to test your VM implementation with various user memory sizes.**

* For example, `pintos --swap-size=2 --filesys-size=2 -p ../../examples/echo -a echo -- -ul=4 -f -q run 'echo hello world'` will test Pintos with 4 page frames for user program.
{% endhint %}

{% hint style="info" %}
**Debugging Tips**

<mark style="color:red;">**Debugging in this lab can be challenging since the root cause (bug) can be far away from the symptom point**</mark>, sometimes even many page faults away that make it difficult to track it down using backtrace.

**So when you encounter some strange error** (e.g., kernel panic due to dereferencing some invalid pointer), **do not limit yourself to just inspect the failure code region** (e.g., the invalid mem access per se).

* For example, the bug could be because of a one-off bug when you calculate the swap address, which much later cause a incorrect page content to be fetched in; the garbage content may be interpreted as a code page and the CPU will execute invalid instructions.
* Sometimes, it could be even caused by one boolean flag in the some page struct being incorrectly set (the course instructor was bitten by such a bug!).

<mark style="color:red;">**One gdb command that is particularly useful for this lab is**</mark> _<mark style="color:red;">**watchpoint**</mark>_.

* Different from breakpoint, which has to be set to a particular code location, watchpoint allows you to **pause the execution whenever the value of a specified expression changes** without being bound to a specific location.
* In other words, if there are N places that can possibly change the value an expression, you will be able to track down who made the change without setting breakpoints everywhere. See [this reference](https://sourceware.org/gdb/download/onlinedocs/gdb/Set-Watchpoints.html) for how to use watchpoint.

<mark style="color:red;">**Besides using gdb to debug, we also suggest a "pair-programming" style debugging**</mark>: explain the core code logic line by line to your teammate and pay special attention to logic such as calculating the address, setting flags/enum statuses, resetting offsets, locking, etc.

* It may take much shorter time to localize the bugs compared to directly debugging the symptom head-on.
{% endhint %}

## Task 2: Accessing User Memory

### **Exercise 2.1**

{% hint style="success" %}
<mark style="color:green;">**Exercise 2.1**</mark>

<mark style="color:green;">**Adjust user memory access code in**</mark> _<mark style="color:green;">**system call handling**</mark>_ <mark style="color:green;">**to deal with potential page faults.**</mark>
{% endhint %}

**You will need to adapt your code to access user memory** (see section 3 [Accessing User Memory](../lab2-user-programs/your-tasks.md#task-3-accessing-user-memory) in project 2) **while handling a system call.**

* Just as user processes may access pages whose content is currently in a file or in swap space, so can they pass addresses that refer to such non-resident pages to system calls.
* Moreover, unless your kernel takes measures to prevent this, a page may be evicted from its frame even while it is being accessed by kernel code. If kernel code accesses such non-resident user pages, a page fault will result.

**While accessing user memory, your kernel must either be prepared to handle such page faults, or it must prevent them from occurring.**

* The kernel must prevent such page faults while it is **holding resources** it would need to acquire to handle these faults.
* In Pintos, **such resources include locks** **acquired by the device driver(s)** that control the device(s) containing the file system and swap space.
* As a concrete example, you must not allow page faults to occur while a device driver accesses a user buffer passed to `file_read`, because you would not be able to invoke the driver while handling such faults.

**Preventing such page faults requires cooperation between the code within which the access occurs and your page eviction code.**

* For instance, you could extend your frame table to record when a page contained in a frame must not be evicted. (This is also referred to as "**pinning**" or "**locking**" the page in its frame.)
* Pinning restricts your page replacement algorithm's choices when looking for pages to evict, so be sure to pin pages no longer than necessary, and avoid pinning pages when it is not necessary.
