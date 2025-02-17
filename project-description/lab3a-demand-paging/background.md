# Background

## Source Files

You will work in the `vm/` directory for this project.

The vm directory contains only **Makefiles**. **The only change from userprog is that this new Makefile&#x20;**<mark style="color:red;">**turns on the setting -DVM**</mark>**.**

All code you write will be in new files or in files introduced in earlier projects.

You will probably be encountering just a few files for the first time:

* <mark style="color:blue;">**devices/block.h**</mark>
* <mark style="color:blue;">**devices/block.c**</mark>
  * **Provides sector-based read and write access to block device.**
* You will use this interface to access **the swap partition** as a block device.

## Memory Terminology

**Careful definitions are needed to keep discussion of virtual memory from being confusing.** Thus, we begin by presenting some terminology for memory and storage. Some of these terms should be familiar from project 2 (see section [Virtual Memory Layout](../lab2-user-programs/background.md#virtual-memory-layout)), but much of it is new.

### **Pages**

**A&#x20;**_**page**_**, sometimes called a** _**virtual page**_**, is a continuous region of virtual memory 4,096 bytes (the&#x20;**_**page size**_**) in length.**

* A page must be _**page-aligned**_, that is, start on a virtual address evenly divisible by the page size.
* Thus, a 32-bit virtual address can be divided into **a 20-bit&#x20;**_**page number**_ **and** **a 12-bit&#x20;**_**page offset**_ **(or just&#x20;**_**offset**_**)**, like this:

```
               31               12 11        0
              +-------------------+-----------+
              |    Page Number    |   Offset  |
              +-------------------+-----------+
                       Virtual Address
```

* **Each process has an independent set of&#x20;**_**user (virtual) pages**_, which are those pages **below virtual address `PHYS_BASE`**, typically **0xc0000000 (3 GB)**.
* **The set of&#x20;**_**kernel (virtual) pages**_**, on the other hand, is&#x20;**_**global**_, remaining the same regardless of what thread or process is active.
* The kernel may access both user and kernel pages, but a user process may access only its own user pages. See section [Virtual Memory Layout](../lab2-user-programs/background.md#virtual-memory-layout), for more information.

Pintos provides several useful functions for working with virtual addresses. See section [Virtual Addresses](../../appendix/reference-guide/virtual-addresses.md), for details.

### **Frames**

**A&#x20;**_**frame**_**, sometimes called a&#x20;**_**physical frame**_**&#x20;or a&#x20;**_**page frame**_**, is a continuous region of physical memory.**

* Like pages, frames must be _**page-size**_ and _**page-aligned**_.
* Thus, a 32-bit physical address can be divided into **a 20-bit&#x20;**_**frame number**_ and **a 12-bit&#x20;**_**frame offset**_ **(or just&#x20;**_**offset**_**)**, like this:

```
               31               12 11        0
              +-------------------+-----------+
              |    Frame Number   |   Offset  |
              +-------------------+-----------+
                       Physical Address
```

* **The 80x86 doesn't provide any way to directly access memory at a physical address. Pintos works around this by** <mark style="color:red;">**mapping kernel virtual memory directly to physical memory**</mark><mark style="color:red;">:</mark> the first page of kernel virtual memory is mapped to the first frame of physical memory, the second page to the second frame, and so on. Thus, **frames can be accessed through kernel virtual memory**.

Pintos provides functions for translating between physical addresses and kernel virtual addresses. See section [Virtual Addresses](../../appendix/reference-guide/virtual-addresses.md), for details.

### **Page Tables**

**In Pintos, a&#x20;**_**page table**_**&#x20;is a data structure that the CPU uses to&#x20;**_**translate**_ **a virtual address to a physical address, that is, from a page to a frame.**

* The page table format is dictated by the 80x86 architecture. Pintos provides page table management code in `pagedir.c` (see section [Page Table](../../appendix/reference-guide/page-table.md)).
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

**A&#x20;**_**swap slot**_ **is a continuous,&#x20;**_**page-size**_**&#x20;region of disk space in the swap partition.**

* Although hardware limitations dictating the placement of slots are looser than for pages and frames, swap slots should be _**page-aligned**_ because there is no downside in doing so.

## Resource Management Overview

You will need to design the following data structures:

### Supplemental page table

* Enables page fault handling by supplementing the hardware page table.
* See section [Managing the Supplemental Page Table](background.md#managing-the-supplemental-page-table).

### Frame table

* Allows efficient implementation of eviction policy.
* See section [Managing the Frame Table](background.md#managing-the-frame-table).

### Swap table

* Tracks usage of swap slots.
* See section [Managing the Swap Table](background.md#managing-the-swap-table).

### Some Notes

1. **You do not necessarily need to implement three completely distinct data structures:** it may be convenient to wholly or partially merge related resources into a unified data structure.
2. **For each data structure, you need to determine what information each element should contain.** You also need to decide on the data structure's scope, either local (per-process) or global (applying to the whole system), and how many instances are required within its scope.
3. **To simplify your design, you may store these data structures in non-pageable memory.** That means that you can be sure that pointers among them will remain valid.
4. **Possible choices of data structures include arrays, lists, bitmaps, and hash tables.**
   * An array is often the simplest approach, but a sparsely populated array wastes memory.
   * Lists are also simple, but traversing a long list to find a particular position wastes time.
   * Both arrays and lists can be resized, but lists more efficiently support insertion and deletion in the middle.
5. **Pintos includes a bitmap data structure in `lib/kernel/bitmap.c` and `lib/kernel/bitmap.h`.** A bitmap is an array of bits, each of which can be true or false. Bitmaps are typically used to **track usage in a set of (identical) resources**: if resource n is in use, then bit n of the bitmap is true. Pintos bitmaps are fixed in size, although you could extend their implementation to support resizing.
6. **Pintos also includes a hash table data structure** (see section [Hash Table](../../appendix/reference-guide/hash-table.md)). Pintos hash tables efficiently support insertions and deletions over a wide range of table sizes.
7. **Although more complex data structures may yield performance or other benefits, they may also needlessly complicate your implementation.** Thus, <mark style="color:red;">**we do not recommend implementing any advanced data structure**</mark> (e.g. a balanced binary tree) as part of your design.

## Managing the Supplemental Page Table

**The&#x20;**_**supplemental page table**_ **supplements the page table with additional data about each page.**

* It is needed because of the limitations imposed by the page table's format.
* Such a data structure is often called a "page table" also; we add the word "supplemental" to reduce confusion.

**The supplemental page table is used for at least two purposes.**

1. Most importantly, **on a page fault, the kernel looks up the virtual page that faulted in the supplemental page table to find out what data should be there.**
2. Second, **the kernel consults the supplemental page table when a process terminates, to decide what resources to free**.

You may organize the supplemental page table as you wish. **There are at least two basic approaches to its organization**: in terms of **segments** or in terms of **pages**.

* Optionally, you may **use the page table itself as an index to track the members of the supplemental page table**. You will have to modify the Pintos page table implementation in `pagedir.c` to do so. We recommend this approach for advanced students only. See section [Page Table Entry Format](../../appendix/reference-guide/page-table.md#page-table-entry-format), for more information.

<mark style="color:red;">**The most important user of the supplemental page table is the page fault handler.**</mark>

* In project 2, a page fault always indicated **a bug** in the kernel or a user program. In project 3, this is no longer true.
* **Now, a page fault might only indicate that the page must be brought in from a file or swap.** You will have to implement a more sophisticated page fault handler to handle these cases.

Your page fault handler, which you should implement by modifying **`page_fault()`** in `userprog/exception.c`, needs to do roughly the following:

1. **Locate the page that faulted in the supplemental page table.** If the memory reference is valid, use the supplemental page table entry to locate the data that goes in the page, which might be **in the file system**, or **in a swap slot**, or it might simply be **an all-zero page**.
   * If you implement **sharing**, the page's data might even already be in a page frame, but not in the page table.
   * **Invalid Accesses:** If the supplemental page table indicates that the user process should not expect any data at the address it was trying to access, or if the page lies within kernel virtual memory, or if the access is an attempt to write to a read-only page, then the access is invalid. Any invalid access **terminates the process** and thereby **frees all of its resources**.
2. **Obtain a frame to store the page.** See section [Managing the Frame Table](background.md#managing-the-frame-table), for details.
   1. If you implement **sharing**, the data you need may already be in a frame, in which case you must be able to locate that frame.
3. **Fetch the data into the frame, by reading it from the file system or swap, zeroing it, etc.**
   1. If you implement **sharing**, the page you need may already be in a frame, in which case no action is necessary in this step.
4. **Point the page table entry for the faulting virtual address to the physical page.** You can use the functions in `userprog/pagedir.c`.

## Managing the Frame Table

**The&#x20;**_**frame table**_ **contains one entry for each frame that contains a user page.**

* **Each entry in the frame table contains a pointer to the page**, if any, **that currently occupies it**, and other data of your choice.
* **The frame table allows Pintos to efficiently implement an eviction policy**, by choosing a page to evict when no frames are free.

<mark style="color:red;">**The frames used for user pages should be obtained from the "user pool," by calling**</mark> **`palloc_get_page(PAL_USER)`.**

* You must use `PAL_USER` to avoid allocating from the "kernel pool," which could cause some test cases to fail unexpectedly (see [Why PAL\_USER?](faq.md#why-should-i-use-pal_user-for-allocating-page-frames)).
* If you modify `palloc.c` as part of your frame table implementation, be sure to retain the distinction between the two pools.

**The most important operation on the frame table is obtaining an unused frame.**

* This is easy when a frame is free.
* When none is free, a frame must be made free by **evicting** some page from its frame.

**If no frame can be evicted without allocating a swap slot, but swap is full, panic the kernel.** Real OSes apply a wide range of policies to recover from or prevent such situations, but these policies are beyond the scope of this project.

**The process of eviction** comprises roughly the following steps:

1. **Choose a frame to evict, using your page replacement algorithm.** The "accessed" and "dirty" bits in the page table, described below, will come in handy.
2. **Remove references to the frame from any page table that refers to it.**
   * Unless you have implemented **sharing**, only a single page should refer to a frame at any given time.
3. **If necessary, write the page to the file system or to swap.**

The evicted frame may then be used to store a different page.

### **Accessed and Dirty Bits**

**80x86 hardware provides some assistance for implementing&#x20;**_**page replacement algorithms**_**, through a pair of bits in the page table entry (PTE) for each page.**

* On any **read or write** to a page, the CPU **sets the&#x20;**_**accessed bit**_ **to 1** in the page's PTE.
* And on any **write**, the CPU **sets the&#x20;**_**dirty bit**_**&#x20;to 1**.
* <mark style="color:red;">**The CPU never resets these bits to 0, but the OS may do so.**</mark>
* You need to be aware of _**aliases**_, that is, two (or more) pages that refer to the same frame. When an aliased frame is accessed, the accessed and dirty bits are updated in only one page table entry (the one for the page used for access). The accessed and dirty bits for the other aliases are not updated.
* <mark style="color:red;">**In Pintos, every user virtual page is aliased to its kernel virtual page.**</mark> You must manage these aliases somehow.
  * For example, your code could **check and update the accessed and dirty bits for both addresses**.
  * Alternatively, the kernel could avoid the problem by **only accessing user data through the user virtual address**.
* Other aliases should only arise if you implement **sharing** for extra credit (see [VM Extra Credit](faq.md#what-extra-credit-is-available)), or if there is **a bug** in your code.

See section [Accessed and Dirty Bits](../../appendix/reference-guide/page-table.md#accessed-and-dirty-bits), for details of the functions to work with accessed and dirty bits.

## Managing the Swap Table

**The swap table tracks in-use and free swap slots.**

* It should allow **picking an unused swap slot for evicting a page** from its frame to the swap partition.
* It should allow **freeing a swap slot when its page is read back or the process whose page was swapped is terminated**.

<mark style="color:red;">**You may use the**</mark> **`BLOCK_SWAP`**  _<mark style="color:red;">**block device**</mark>_ <mark style="color:red;">**for swapping**</mark>, obtaining the `struct block` that represents it by calling **`block_get_role()`**.

* From the `vm/build` directory, use the command **`pintos-mkdisk swap.dsk --swap-size=n`** to create an disk named swap.dsk that contains a n-MB swap partition.
* **Afterward, swap.dsk will&#x20;**_**automatically**_ **be attached as an extra disk when you run `pintos`.**
* Alternatively, you can tell `pintos` to use a temporary n-MB swap disk for a single run with **`--swap-size=n`**.

**Swap slots should be allocated lazily, that is, only when they are actually required by eviction.**

* Reading data pages from the executable and writing them to swap immediately at process startup is **not** lazy.
* Swap slots should **not** be reserved to store particular pages.
* Free a swap slot when its contents are read back into a frame.
