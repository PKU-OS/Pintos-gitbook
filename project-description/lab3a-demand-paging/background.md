# Background

## Source Files

You will work in the `vm/` directory for this project.&#x20;

The vm directory contains only **Makefiles**. **The only change from userprog is that this new Makefile **<mark style="color:red;">**turns on the setting -DVM**</mark>**.**&#x20;

All code you write will be in new files or in files introduced in earlier projects.

You will probably be encountering just a few files for the first time:

* <mark style="color:blue;">**devices/block.h**</mark>
* <mark style="color:blue;">**devices/block.c**</mark>
  * **Provides sector-based read and write access to block device.**&#x20;
* You will use this interface to access **the swap partition** as a block device.

## Memory Terminology

**Careful definitions are needed to keep discussion of virtual memory from being confusing.** Thus, we begin by presenting some terminology for memory and storage. Some of these terms should be familiar from project 2 (see section [Virtual Memory Layout](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_4.html#SEC48)), but much of it is new.

### **Pages**

**A **_**page**_**, sometimes called a **_**virtual page**_**, is a continuous region of virtual memory 4,096 bytes (the **_**page size**_**) in length.**&#x20;

* A page must be _**page-aligned**_, that is, start on a virtual address evenly divisible by the page size.&#x20;
* Thus, a 32-bit virtual address can be divided into **a 20-bit **_**page number**_ and **a 12-bit **_**page offset**_** (or just **_**offset**_**)**, like this:

```
               31               12 11        0
              +-------------------+-----------+
              |    Page Number    |   Offset  |
              +-------------------+-----------+
                       Virtual Address
```

* **Each process has an independent set of **_**user (virtual) pages**_, which are those pages **below virtual address `PHYS_BASE`**, typically **0xc0000000 (3 GB)**.&#x20;
* **The set of **_**kernel (virtual) pages**_, on the other hand, **is **_**global**_, remaining the same regardless of what thread or process is active.&#x20;
* The kernel may access both user and kernel pages, but a user process may access only its own user pages. See section [4.1.4 Virtual Memory Layout](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_4.html#SEC48), for more information.

Pintos provides several useful functions for working with virtual addresses. See section [A.6 Virtual Addresses](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC125), for details.

### **Frames**

**A **_**frame**_**, sometimes called a **_**physical frame**_** or a **_**page frame**_**, is a continuous region of physical memory.**&#x20;

* Like pages, frames must be _**page-size**_ and _**page-aligned**_.&#x20;
* Thus, a 32-bit physical address can be divided into **a 20-bit **_**frame number**_ and **a 12-bit **_**frame offset**_** (or just **_**offset**_**)**, like this:

```
               31               12 11        0
              +-------------------+-----------+
              |    Frame Number   |   Offset  |
              +-------------------+-----------+
                       Physical Address
```

* **The 80x86 doesn't provide any way to directly access memory at a physical address. Pintos works around this by **<mark style="color:red;">**mapping kernel virtual memory directly to physical memory**</mark><mark style="color:red;">:</mark> the first page of kernel virtual memory is mapped to the first frame of physical memory, the second page to the second frame, and so on. Thus, **frames can be accessed through kernel virtual memory**.

Pintos provides functions for translating between physical addresses and kernel virtual addresses. See section [A.6 Virtual Addresses](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC125), for details.

### **Page Tables**

**In Pintos, a **_**page table**_** is a data structure that the CPU uses to **_**translate**_** a virtual address to a physical address, that is, from a page to a frame.**&#x20;

* The page table format is dictated by the 80x86 architecture. Pintos provides page table management code in `pagedir.c` (see section [A.7 Page Table](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC126)).
* **The diagram below illustrates the relationship between pages and frames.** The virtual address, on the left, consists of a page number and an offset. The page table translates the page number into a frame number, which is combined with the unmodified offset to obtain the physical address, on the right.

```
                          +----------+
         .--------------->|Page Table|---------.
        /                 +----------+          |
   31   |   12 11    0                    31    V   12 11    0
  +-----------+-------+                  +------------+-------+
  |  Page Nr  |  Ofs  |                  |  Frame Nr  |  Ofs  |
  +-----------+-------+                  +------------+-------+
   Virt Addr      |                       Phys Addr       ^
                   \_____________________________________/
```

### **Swap Slots**

**A **_**swap slot**_** is a continuous, **_**page-size**_** region of disk space in the swap partition.**&#x20;

* Although hardware limitations dictating the placement of slots are looser than for pages and frames, swap slots should be _**page-aligned**_ because there is no downside in doing so.

## Resource Management Overview

You will need to design the following data structures:

### Supplemental page table

Enables page fault handling by supplementing the hadrware page table. See section [B.4 Managing the Supplemental Page Table](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_5.html#SEC73).

### Frame table

Allows efficient implementation of eviction policy. See section [B.5 Managing the Frame Table](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_5.html#SEC74).

### Swap table

Tracks usage of swap slots. See section [B.6 Managing the Swap Table](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_5.html#SEC76).

You do not necessarily need to implement three completely distinct data structures: it may be convenient to wholly or partially merge related resources into a unified data structure.

For each data structure, you need to determine what information each element should contain. You also need to decide on the data structure's scope, either local (per-process) or global (applying to the whole system), and how many instances are required within its scope.

To simplify your design, you may store these data structures in non-pageable memory. That means that you can be sure that pointers among them will remain valid.

Possible choices of data structures include arrays, lists, bitmaps, and hash tables. An array is often the simplest approach, but a sparsely populated array wastes memory. Lists are also simple, but traversing a long list to find a particular position wastes time. Both arrays and lists can be resized, but lists more efficiently support insertion and deletion in the middle.

Pintos includes a bitmap data structure in lib/kernel/bitmap.c and lib/kernel/bitmap.h. A bitmap is an array of bits, each of which can be true or false. Bitmaps are typically used to track usage in a set of (identical) resources: if resource n is in use, then bit n of the bitmap is true. Pintos bitmaps are fixed in size, although you could extend their implementation to support resizing.

Pintos also includes a hash table data structure (see section [A.8 Hash Table](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC134)). Pintos hash tables efficiently support insertions and deletions over a wide range of table sizes.

Although more complex data structures may yield performance or other benefits, they may also needlessly complicate your implementation. Thus, we do not recommend implementing any advanced data structure (e.g. a balanced binary tree) as part of your design.

### Managing the Supplemental Page Table

The _supplemental page table_ supplements the page table with additional data about each page. It is needed because of the limitations imposed by the page table's format. Such a data structure is often called a "page table" also; we add the word "supplemental" to reduce confusion.

The supplemental page table is used for at least two purposes. Most importantly, on a page fault, the kernel looks up the virtual page that faulted in the supplemental page table to find out what data should be there. Second, the kernel consults the supplemental page table when a process terminates, to decide what resources to free.

You may organize the supplemental page table as you wish. There are at least two basic approaches to its organization: in terms of segments or in terms of pages. Optionally, you may use the page table itself as an index to track the members of the supplemental page table. You will have to modify the Pintos page table implementation in pagedir.c to do so. We recommend this approach for advanced students only. See section [A.7.4.2 Page Table Entry Format](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC132), for more information.

The most important user of the supplemental page table is the page fault handler. In project 2, a page fault always indicated a bug in the kernel or a user program. In project 3, this is no longer true. Now, a page fault might only indicate that the page must be brought in from a file or swap. You will have to implement a more sophisticated page fault handler to handle these cases. Your page fault handler, which you should implement by modifying `page_fault()` in userprog/exception.c, needs to do roughly the following:

1.  Locate the page that faulted in the supplemental page table. If the memory reference is valid, use the supplemental page table entry to locate the data that goes in the page, which might be in the file system, or in a swap slot, or it might simply be an all-zero page. If you implement sharing, the page's data might even already be in a page frame, but not in the page table.

    If the supplemental page table indicates that the user process should not expect any data at the address it was trying to access, or if the page lies within kernel virtual memory, or if the access is an attempt to write to a read-only page, then the access is invalid. Any invalid access terminates the process and thereby frees all of its resources.
2.  Obtain a frame to store the page. See section [B.5 Managing the Frame Table](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_5.html#SEC74), for details.

    If you implement sharing, the data you need may already be in a frame, in which case you must be able to locate that frame.
3.  Fetch the data into the frame, by reading it from the file system or swap, zeroing it, etc.

    If you implement sharing, the page you need may already be in a frame, in which case no action is necessary in this step.
4. Point the page table entry for the faulting virtual address to the physical page. You can use the functions in userprog/pagedir.c.

### Managing the Frame Table

The _frame table_ contains one entry for each frame that contains a user page. Each entry in the frame table contains a pointer to the page, if any, that currently occupies it, and other data of your choice. The frame table allows Pintos to efficiently implement an eviction policy, by choosing a page to evict when no frames are free.

The frames used for user pages should be obtained from the "user pool," by calling `palloc_get_page(PAL_USER)`. You must use `PAL_USER` to avoid allocating from the "kernel pool," which could cause some test cases to fail unexpectedly (see [Why PAL\_USER?](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_5.html#Why%20PAL\_USER?)). If you modify palloc.c as part of your frame table implementation, be sure to retain the distinction between the two pools.

The most important operation on the frame table is obtaining an unused frame. This is easy when a frame is free. When none is free, a frame must be made free by evicting some page from its frame.

If no frame can be evicted without allocating a swap slot, but swap is full, panic the kernel. Real OSes apply a wide range of policies to recover from or prevent such situations, but these policies are beyond the scope of this project.

The process of eviction comprises roughly the following steps:

1. Choose a frame to evict, using your page replacement algorithm. The "accessed" and "dirty" bits in the page table, described below, will come in handy.
2.  Remove references to the frame from any page table that refers to it.

    Unless you have implemented sharing, only a single page should refer to a frame at any given time.
3. If necessary, write the page to the file system or to swap.

The evicted frame may then be used to store a different page.

#### **Accessed and Dirty Bits**

80x86 hardware provides some assistance for implementing page replacement algorithms, through a pair of bits in the page table entry (PTE) for each page. On any read or write to a page, the CPU sets the _accessed bit_ to 1 in the page's PTE, and on any write, the CPU sets the _dirty bit_ to 1. The CPU never resets these bits to 0, but the OS may do so.

You need to be aware of _aliases_, that is, two (or more) pages that refer to the same frame. When an aliased frame is accessed, the accessed and dirty bits are updated in only one page table entry (the one for the page used for access). The accessed and dirty bits for the other aliases are not updated.

In Pintos, every user virtual page is aliased to its kernel virtual page. You must manage these aliases somehow. For example, your code could check and update the accessed and dirty bits for both addresses. Alternatively, the kernel could avoid the problem by only accessing user data through the user virtual address.

Other aliases should only arise if you implement sharing for extra credit (see [VM Extra Credit](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_5.html#VM%20Extra%20Credit)), or if there is a bug in your code.

See section [A.7.3 Accessed and Dirty Bits](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC129), for details of the functions to work with accessed and dirty bits.

### Managing the Swap Table

The swap table tracks in-use and free swap slots. It should allow picking an unused swap slot for evicting a page from its frame to the swap partition. It should allow freeing a swap slot when its page is read back or the process whose page was swapped is terminated.

You may use the `BLOCK_SWAP` block device for swapping, obtaining the `struct block` that represents it by calling `block_get_role()`. From the vm/build directory, use the command `pintos-mkdisk swap.dsk --swap-size=n` to create an disk named swap.dsk that contains a n-MB swap partition. Afterward, swap.dsk will automatically be attached as an extra disk when you run `pintos`. Alternatively, you can tell `pintos` to use a temporary n-MB swap disk for a single run with --swap-size=n.

Swap slots should be allocated lazily, that is, only when they are actually required by eviction. Reading data pages from the executable and writing them to swap immediately at process startup is not lazy. Swap slots should not be reserved to store particular pages.

Free a swap slot when its contents are read back into a frame.
