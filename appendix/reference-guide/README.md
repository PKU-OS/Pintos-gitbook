# Code Guide

This chapter is a reference for the Pintos code. The reference guide does not cover all of the code in Pintos, but it does cover those pieces that students most often find troublesome. You may find that you want to read each part of the reference guide as you work on the project where it becomes important.

Here are the sections in the chapter:

{% content-ref url="loading.md" %}
[loading.md](loading.md)
{% endcontent-ref %}

{% content-ref url="threads.md" %}
[threads.md](threads.md)
{% endcontent-ref %}

{% content-ref url="synchronization.md" %}
[synchronization.md](synchronization.md)
{% endcontent-ref %}

{% content-ref url="interrupt-handling.md" %}
[interrupt-handling.md](interrupt-handling.md)
{% endcontent-ref %}

{% content-ref url="memory-allocation.md" %}
[memory-allocation.md](memory-allocation.md)
{% endcontent-ref %}

{% content-ref url="virtual-addresses.md" %}
[virtual-addresses.md](virtual-addresses.md)
{% endcontent-ref %}

{% content-ref url="page-table.md" %}
[page-table.md](page-table.md)
{% endcontent-ref %}

{% content-ref url="hash-table.md" %}
[hash-table.md](hash-table.md)
{% endcontent-ref %}

## A.5 Memory Allocation

Pintos contains two memory allocators, one that allocates memory in units of a page, and one that can allocate blocks of any size.

### A.5.1 Page Allocator

The page allocator declared in threads/palloc.h allocates memory in units of a page. It is most often used to allocate memory one page at a time, but it can also allocate multiple contiguous pages at once.

The page allocator divides the memory it allocates into two pools, called the kernel and user pools. By default, each pool gets half of system memory above 1 MB, but the division can be changed with the -ul kernel command line option (see [Why PAL\_USER?](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_5.html#Why%20PAL\_USER?)). An allocation request draws from one pool or the other. If one pool becomes empty, the other may still have free pages. The user pool should be used for allocating memory for user processes and the kernel pool for all other allocations. This will only become important starting with project 3. Until then, all allocations should be made from the kernel pool.

Each pool's usage is tracked with a bitmap, one bit per page in the pool. A request to allocate n pages scans the bitmap for n consecutive bits set to false, indicating that those pages are free, and then sets those bits to true to mark them as used. This is a "first fit" allocation strategy (see [Wilson](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_14.html#Wilson)).

The page allocator is subject to fragmentation. That is, it may not be possible to allocate n contiguous pages even though n or more pages are free, because the free pages are separated by used pages. In fact, in pathological cases it may be impossible to allocate 2 contiguous pages even though half of the pool's pages are free. Single-page requests can't fail due to fragmentation, so requests for multiple contiguous pages should be limited as much as possible.

Pages may not be allocated from interrupt context, but they may be freed.

When a page is freed, all of its bytes are cleared to 0xcc, as a debugging aid (see section [E.8 Tips](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_11.html#SEC168)).

Page allocator types and functions are described below.

#### Function: void \***palloc\_get\_page** (enum palloc\_flags flags)

#### Function: void \***palloc\_get\_multiple** (enum palloc\_flags flags, size\_t page\_cnt)

Obtains and returns one page, or page\_cnt contiguous pages, respectively. Returns a null pointer if the pages cannot be allocated.

The flags argument may be any combination of the following flags:

#### Page Allocator Flag: **`PAL_ASSERT`**

If the pages cannot be allocated, panic the kernel. This is only appropriate during kernel initialization. User processes should never be permitted to panic the kernel.

#### Page Allocator Flag: **`PAL_ZERO`**

Zero all the bytes in the allocated pages before returning them. If not set, the contents of newly allocated pages are unpredictable.

#### Page Allocator Flag: **`PAL_USER`**

Obtain the pages from the user pool. If not set, pages are allocated from the kernel pool.

#### Function: void **palloc\_free\_page** (void \*page)

#### Function: void **palloc\_free\_multiple** (void \*pages, size\_t page\_cnt)

Frees one page, or page\_cnt contiguous pages, respectively, starting at pages. All of the pages must have been obtained using `palloc_get_page()` or `palloc_get_multiple()`.

### A.5.2 Block Allocator

The block allocator, declared in threads/malloc.h, can allocate blocks of any size. It is layered on top of the page allocator described in the previous section. Blocks returned by the block allocator are obtained from the kernel pool.

The block allocator uses two different strategies for allocating memory. The first strategy applies to blocks that are 1 kB or smaller (one-fourth of the page size). These allocations are rounded up to the nearest power of 2, or 16 bytes, whichever is larger. Then they are grouped into a page used only for allocations of that size.

The second strategy applies to blocks larger than 1 kB. These allocations (plus a small amount of overhead) are rounded up to the nearest page in size, and then the block allocator requests that number of contiguous pages from the page allocator.

In either case, the difference between the allocation requested size and the actual block size is wasted. A real operating system would carefully tune its allocator to minimize this waste, but this is unimportant in an instructional system like Pintos.

As long as a page can be obtained from the page allocator, small allocations always succeed. Most small allocations do not require a new page from the page allocator at all, because they are satisfied using part of a page already allocated. However, large allocations always require calling into the page allocator, and any allocation that needs more than one contiguous page can fail due to fragmentation, as already discussed in the previous section. Thus, you should minimize the number of large allocations in your code, especially those over approximately 4 kB each.

When a block is freed, all of its bytes are cleared to 0xcc, as a debugging aid (see section [E.8 Tips](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_11.html#SEC168)).

The block allocator may not be called from interrupt context.

The block allocator functions are described below. Their interfaces are the same as the standard C library functions of the same names.

#### Function: void \***malloc** (size\_t size)

Obtains and returns a new block, from the kernel pool, at least size bytes long. Returns a null pointer if size is zero or if memory is not available.

#### Function: void \***calloc** (size\_t a, size\_t b)

Obtains a returns a new block, from the kernel pool, at least `a * b` bytes long. The block's contents will be cleared to zeros. Returns a null pointer if a or b is zero or if insufficient memory is available.

#### Function: void \***realloc** (void \*block, size\_t new\_size)

Attempts to resize block to new\_size bytes, possibly moving it in the process. If successful, returns the new block, in which case the old block must no longer be accessed. On failure, returns a null pointer, and the old block remains valid.

A call with block null is equivalent to `malloc()`. A call with new\_size zero is equivalent to `free()`.

#### Function: void **free** (void \*block)

Frees block, which must have been previously returned by `malloc()`, `calloc()`, or `realloc()` (and not yet freed).

## A.6 Virtual Addresses

A 32-bit virtual address can be divided into a 20-bit _page number_ and a 12-bit _page offset_ (or just _offset_), like this:

```
               31               12 11        0
              +-------------------+-----------+
              |    Page Number    |   Offset  |
              +-------------------+-----------+
                       Virtual Address
```

Header threads/vaddr.h defines these functions and macros for working with virtual addresses:

#### Macro: **PGSHIFT**

#### Macro: **PGBITS**

The bit index (0) and number of bits (12) of the offset part of a virtual address, respectively.

#### Macro: **PGMASK**

A bit mask with the bits in the page offset set to 1, the rest set to 0 (0xfff).

#### Macro: **PGSIZE**

The page size in bytes (4,096).

#### Function: unsigned **pg\_ofs** (const void \*va)

Extracts and returns the page offset in virtual address va.

#### Function: uintptr\_t **pg\_no** (const void \*va)

Extracts and returns the page number in virtual address va.

#### Function: void \***pg\_round\_down** (const void \*va)

Returns the start of the virtual page that va points within, that is, va with the page offset set to 0.

#### Function: void \***pg\_round\_up** (const void \*va)

Returns va rounded up to the nearest page boundary.

Virtual memory in Pintos is divided into two regions: user virtual memory and kernel virtual memory (see section [4.1.4 Virtual Memory Layout](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_4.html#SEC48)). The boundary between them is `PHYS_BASE`:

#### Macro: **PHYS\_BASE**

Base address of kernel virtual memory. It defaults to 0xc0000000 (3 GB), but it may be changed to any multiple of 0x10000000 from 0x80000000 to 0xf0000000.

User virtual memory ranges from virtual address 0 up to `PHYS_BASE`. Kernel virtual memory occupies the rest of the virtual address space, from `PHYS_BASE` up to 4 GB.

#### Function: bool **is\_user\_vaddr** (const void \*va)

#### Function: bool **is\_kernel\_vaddr** (const void \*va)

Returns true if va is a user or kernel virtual address, respectively, false otherwise.

The 80x86 doesn't provide any way to directly access memory given a physical address. This ability is often necessary in an operating system kernel, so Pintos works around it by mapping kernel virtual memory one-to-one to physical memory. That is, virtual address `PHYS_BASE` accesses physical address 0, virtual address `PHYS_BASE` + 0x1234 accesses physical address 0x1234, and so on up to the size of the machine's physical memory. Thus, adding `PHYS_BASE` to a physical address obtains a kernel virtual address that accesses that address; conversely, subtracting `PHYS_BASE` from a kernel virtual address obtains the corresponding physical address. Header threads/vaddr.h provides a pair of functions to do these translations:

#### Function: void \***ptov** (uintptr\_t pa)

Returns the kernel virtual address corresponding to physical address pa, which should be between 0 and the number of bytes of physical memory.

#### Function: uintptr\_t **vtop** (void \*va)

Returns the physical address corresponding to va, which must be a kernel virtual address.

## A.7 Page Table

The code in pagedir.c is an abstract interface to the 80x86 hardware page table, also called a "page directory" by Intel processor documentation. The page table interface uses a `uint32_t *` to represent a page table because this is convenient for accessing their internal structure.

The sections below describe the page table interface and internals.

### A.7.1 Creation, Destruction, and Activation

These functions create, destroy, and activate page tables. The base Pintos code already calls these functions where necessary, so it should not be necessary to call them yourself.

#### Function: uint32\_t \***pagedir\_create** (void)

Creates and returns a new page table. The new page table contains Pintos's normal kernel virtual page mappings, but no user virtual mappings.

Returns a null pointer if memory cannot be obtained.

#### Function: void **pagedir\_destroy** (uint32\_t \*pd)

Frees all of the resources held by pd, including the page table itself and the frames that it maps.

#### Function: void **pagedir\_activate** (uint32\_t \*pd)

Activates pd. The active page table is the one used by the CPU to translate memory references.

### A.7.2 Inspection and Updates

These functions examine or update the mappings from pages to frames encapsulated by a page table. They work on both active and inactive page tables (that is, those for running and suspended processes), flushing the TLB as necessary.

#### Function: bool **pagedir\_set\_page** (uint32\_t \*pd, void \*upage, void \*kpage, bool writable)

Adds to pd a mapping from user page upage to the frame identified by kernel virtual address kpage. If writable is true, the page is mapped read/write; otherwise, it is mapped read-only.

User page upage must not already be mapped in pd.

Kernel page kpage should be a kernel virtual address obtained from the user pool with `palloc_get_page(PAL_USER)` (see [Why PAL\_USER?](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_5.html#Why%20PAL\_USER?)).

Returns true if successful, false on failure. Failure will occur if additional memory required for the page table cannot be obtained.

#### Function: void \***pagedir\_get\_page** (uint32\_t \*pd, const void \*uaddr)

Looks up the frame mapped to uaddr in pd. Returns the kernel virtual address for that frame, if uaddr is mapped, or a null pointer if it is not.

#### Function: void **pagedir\_clear\_page** (uint32\_t \*pd, void \*page)

Marks page "not present" in pd. Later accesses to the page will fault.

Other bits in the page table for page are preserved, permitting the accessed and dirty bits (see the next section) to be checked.

This function has no effect if page is not mapped.

### A.7.3 Accessed and Dirty Bits

80x86 hardware provides some assistance for implementing page replacement algorithms, through a pair of bits in the page table entry (PTE) for each page. On any read or write to a page, the CPU sets the _accessed bit_ to 1 in the page's PTE, and on any write, the CPU sets the _dirty bit_ to 1. The CPU never resets these bits to 0, but the OS may do so.

Proper interpretation of these bits requires understanding of _aliases_, that is, two (or more) pages that refer to the same frame. When an aliased frame is accessed, the accessed and dirty bits are updated in only one page table entry (the one for the page used for access). The accessed and dirty bits for the other aliases are not updated.

See section [5.1.5.1 Accessed and Dirty Bits](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_5.html#SEC75), on applying these bits in implementing page replacement algorithms.

#### Function: bool **pagedir\_is\_dirty** (uint32\_t \*pd, const void \*page)

#### Function: bool **pagedir\_is\_accessed** (uint32\_t \*pd, const void \*page)

Returns true if page directory pd contains a page table entry for page that is marked dirty (or accessed). Otherwise, returns false.

#### Function: void **pagedir\_set\_dirty** (uint32\_t \*pd, const void \*page, bool value)

#### Function: void **pagedir\_set\_accessed** (uint32\_t \*pd, const void \*page, bool value)

If page directory pd has a page table entry for page, then its dirty (or accessed) bit is set to value.

### A.7.4 Page Table Details

The functions provided with Pintos are sufficient to implement the projects. However, you may still find it worthwhile to understand the hardware page table format, so we'll go into a little detail in this section.

#### **A.7.4.1 Structure**

The top-level paging data structure is a page called the "page directory" (PD) arranged as an array of 1,024 32-bit page directory entries (PDEs), each of which represents 4 MB of virtual memory. Each PDE may point to the physical address of another page called a "page table" (PT) arranged, similarly, as an array of 1,024 32-bit page table entries (PTEs), each of which translates a single 4 kB virtual page to a physical page.

Translation of a virtual address into a physical address follows the three-step process illustrated in the diagram below:[(5)](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_fot.html#FOOT5)

1. The most-significant 10 bits of the virtual address (bits 22...31) index the page directory. If the PDE is marked "present," the physical address of a page table is read from the PDE thus obtained. If the PDE is marked "not present" then a page fault occurs.
2. The next 10 bits of the virtual address (bits 12...21) index the page table. If the PTE is marked "present," the physical address of a data page is read from the PTE thus obtained. If the PTE is marked "not present" then a page fault occurs.
3. The least-significant 12 bits of the virtual address (bits 0...11) are added to the data page's physical base address, yielding the final physical address.

```
 31                  22 21                  12 11                   0
+----------------------+----------------------+----------------------+
| Page Directory Index |   Page Table Index   |    Page Offset       |
+----------------------+----------------------+----------------------+
             |                    |                     |
     _______/             _______/                _____/
    /                    /                       /
   /    Page Directory  /      Page Table       /    Data Page
  /     .____________. /     .____________.    /   .____________.
  |1,023|____________| |1,023|____________|    |   |____________|
  |1,022|____________| |1,022|____________|    |   |____________|
  |1,021|____________| |1,021|____________|    \__\|____________|
  |1,020|____________| |1,020|____________|       /|____________|
  |     |            | |     |            |        |            |
  |     |            | \____\|            |_       |            |
  |     |      .     |      /|      .     | \      |      .     |
  \____\|      .     |_      |      .     |  |     |      .     |
       /|      .     | \     |      .     |  |     |      .     |
        |      .     |  |    |      .     |  |     |      .     |
        |            |  |    |            |  |     |            |
        |____________|  |    |____________|  |     |____________|
       4|____________|  |   4|____________|  |     |____________|
       3|____________|  |   3|____________|  |     |____________|
       2|____________|  |   2|____________|  |     |____________|
       1|____________|  |   1|____________|  |     |____________|
       0|____________|  \__\0|____________|  \____\|____________|
                           /                      /
```

Pintos provides some macros and functions that are useful for working with raw page tables:

#### Macro: **PTSHIFT**

#### Macro: **PTBITS**

The starting bit index (12) and number of bits (10), respectively, in a page table index.

#### Macro: **PTMASK**

A bit mask with the bits in the page table index set to 1 and the rest set to 0 (0x3ff000).

#### Macro: **PTSPAN**

The number of bytes of virtual address space that a single page table page covers (4,194,304 bytes, or 4 MB).

#### Macro: **PDSHIFT**

#### Macro: **PDBITS**

The starting bit index (22) and number of bits (10), respectively, in a page directory index.

#### Macro: **PDMASK**

A bit mask with the bits in the page directory index set to 1 and other bits set to 0 (0xffc00000).

#### Function: uintptr\_t **pd\_no** (const void \*va)

#### Function: uintptr\_t **pt\_no** (const void \*va)

Returns the page directory index or page table index, respectively, for virtual address va. These functions are defined in threads/pte.h.

#### Function: unsigned **pg\_ofs** (const void \*va)

Returns the page offset for virtual address va. This function is defined in threads/vaddr.h.

#### **A.7.4.2 Page Table Entry Format**

You do not need to understand the PTE format to do the Pintos projects, unless you wish to incorporate the page table into your supplemental page table (see section [5.1.4 Managing the Supplemental Page Table](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_5.html#SEC73)).

The actual format of a page table entry is summarized below. For complete information, refer to section 3.7, "Page Translation Using 32-Bit Physical Addressing," in \[ [IA32-v3a](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_14.html#IA32-v3a)].

```
 31                                   12 11 9      6 5     2 1 0
+---------------------------------------+----+----+-+-+---+-+-+-+
|           Physical Address            | AVL|    |D|A|   |U|W|P|
+---------------------------------------+----+----+-+-+---+-+-+-+
```

Some more information on each bit is given below. The names are threads/pte.h macros that represent the bits' values:

#### Macro: **PTE\_P**

Bit 0, the "present" bit. When this bit is 1, the other bits are interpreted as described below. When this bit is 0, any attempt to access the page will page fault. The remaining bits are then not used by the CPU and may be used by the OS for any purpose.

#### Macro: **PTE\_W**

Bit 1, the "read/write" bit. When it is 1, the page is writable. When it is 0, write attempts will page fault.

#### Macro: **PTE\_U**

Bit 2, the "user/supervisor" bit. When it is 1, user processes may access the page. When it is 0, only the kernel may access the page (user accesses will page fault).

Pintos clears this bit in PTEs for kernel virtual memory, to prevent user processes from accessing them.

#### Macro: **PTE\_A**

Bit 5, the "accessed" bit. See section [A.7.3 Accessed and Dirty Bits](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC129).

#### Macro: **PTE\_D**

Bit 6, the "dirty" bit. See section [A.7.3 Accessed and Dirty Bits](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC129).

#### Macro: **PTE\_AVL**

Bits 9...11, available for operating system use. Pintos, as provided, does not use them and sets them to 0.

#### Macro: **PTE\_ADDR**

Bits 12...31, the top 20 bits of the physical address of a frame. The low 12 bits of the frame's address are always 0.

Other bits are either reserved or uninteresting in a Pintos context and should be set to@tie{}0.

Header threads/pte.h defines three functions for working with page table entries:

#### Function: uint32\_t **pte\_create\_kernel** (uint32\_t \*page, bool writable)

Returns a page table entry that points to page, which should be a kernel virtual address. The PTE's present bit will be set. It will be marked for kernel-only access. If writable is true, the PTE will also be marked read/write; otherwise, it will be read-only.

#### Function: uint32\_t **pte\_create\_user** (uint32\_t \*page, bool writable)

Returns a page table entry that points to page, which should be the kernel virtual address of a frame in the user pool (see [Why PAL\_USER?](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_5.html#Why%20PAL\_USER?)). The PTE's present bit will be set and it will be marked to allow user-mode access. If writable is true, the PTE will also be marked read/write; otherwise, it will be read-only.

#### Function: void \***pte\_get\_page** (uint32\_t pte)

Returns the kernel virtual address for the frame that pte points to. The pte may be present or not-present; if it is not-present then the pointer returned is only meaningful if the address bits in the PTE actually represent a physical address.

#### **A.7.4.3 Page Directory Entry Format**

Page directory entries have the same format as PTEs, except that the physical address points to a page table page instead of a frame. Header threads/pte.h defines two functions for working with page directory entries:

#### Function: uint32\_t **pde\_create** (uint32\_t \*pt)

Returns a page directory that points to page, which should be the kernel virtual address of a page table page. The PDE's present bit will be set, it will be marked to allow user-mode access, and it will be marked read/write.

#### Function: uint32\_t \***pde\_get\_pt** (uint32\_t pde)

Returns the kernel virtual address for the page table page that pde, which must be marked present, points to.

## A.8 Hash Table

Pintos provides a hash table data structure in lib/kernel/hash.c. To use it you will need to include its header file, lib/kernel/hash.h, with `#include <hash.h>`. No code provided with Pintos uses the hash table, which means that you are free to use it as is, modify its implementation for your own purposes, or ignore it, as you wish.

Most implementations of the virtual memory project use a hash table to translate pages to frames. You may find other uses for hash tables as well.

### A.8.1 Data Types

A hash table is represented by `struct hash`.

#### Type: **struct hash**

Represents an entire hash table. The actual members of `struct hash` are "opaque." That is, code that uses a hash table should not access `struct hash` members directly, nor should it need to. Instead, use hash table functions and macros.

The hash table operates on elements of type `struct hash_elem`.

#### Type: **struct hash\_elem**

Embed a `struct hash_elem` member in the structure you want to include in a hash table. Like `struct hash`, `struct hash_elem` is opaque. All functions for operating on hash table elements actually take and return pointers to `struct hash_elem`, not pointers to your hash table's real element type.

You will often need to obtain a `struct hash_elem` given a real element of the hash table, and vice versa. Given a real element of the hash table, you may use the & operator to obtain a pointer to its `struct hash_elem`. Use the `hash_entry()` macro to go the other direction.

#### Macro: type \***hash\_entry** (struct hash\_elem \*elem, type, member)

Returns a pointer to the structure that elem, a pointer to a `struct hash_elem`, is embedded within. You must provide type, the name of the structure that elem is inside, and member, the name of the member in type that elem points to.

For example, suppose `h` is a `struct hash_elem *` variable that points to a `struct thread` member (of type `struct hash_elem`) named `h_elem`. Then, `hash_entry@tie{`(h, struct thread, h\_elem)} yields the address of the `struct thread` that `h` points within.

See section [A.8.5 Hash Table Example](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC139), for an example.

Each hash table element must contain a key, that is, data that identifies and distinguishes elements, which must be unique among elements in the hash table. (Elements may also contain non-key data that need not be unique.) While an element is in a hash table, its key data must not be changed. Instead, if need be, remove the element from the hash table, modify its key, then reinsert the element.

For each hash table, you must write two functions that act on keys: a hash function and a comparison function. These functions must match the following prototypes:

#### Type: **unsigned hash\_hash\_func (const struct hash\_elem \*element, void \*aux)**

Returns a hash of element's data, as a value anywhere in the range of `unsigned int`. The hash of an element should be a pseudo-random function of the element's key. It must not depend on non-key data in the element or on any non-constant data other than the key. Pintos provides the following functions as a suitable basis for hash functions.

#### Function: unsigned **hash\_bytes** (const void \*buf, size\_t \*size)

Returns a hash of the size bytes starting at buf. The implementation is the general-purpose [Fowler-Noll-Vo hash](http://en.wikipedia.org/wiki/Fowler\_Noll\_Vo\_hash) for 32-bit words.

#### Function: unsigned **hash\_string** (const char \*s)

Returns a hash of null-terminated string s.

#### Function: unsigned **hash\_int** (int i)

Returns a hash of integer i.

If your key is a single piece of data of an appropriate type, it is sensible for your hash function to directly return the output of one of these functions. For multiple pieces of data, you may wish to combine the output of more than one call to them using, e.g., the ^ (exclusive or) operator. Finally, you may entirely ignore these functions and write your own hash function from scratch, but remember that your goal is to build an operating system kernel, not to design a hash function.

See section [A.8.6 Auxiliary Data](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC140), for an explanation of aux.

#### Type: **bool hash\_less\_func (const struct hash\_elem \*a, const struct hash\_elem \*b, void \*aux)**

Compares the keys stored in elements a and b. Returns true if a is less than b, false if a is greater than or equal to b.

If two elements compare equal, then they must hash to equal values.

See section [A.8.6 Auxiliary Data](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC140), for an explanation of aux.

See section [A.8.5 Hash Table Example](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC139), for hash and comparison function examples.

A few functions accept a pointer to a third kind of function as an argument:

#### Type: **void hash\_action\_func (struct hash\_elem \*element, void \*aux)**

Performs some kind of action, chosen by the caller, on element.

See section [A.8.6 Auxiliary Data](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC140), for an explanation of aux.

### A.8.2 Basic Functions

These functions create, destroy, and inspect hash tables.

#### Function: bool **hash\_init** (struct hash \*hash, hash\_hash\_func \*hash\_func, hash\_less\_func \*less\_func, void \*aux)

Initializes hash as a hash table with hash\_func as hash function, less\_func as comparison function, and aux as auxiliary data. Returns true if successful, false on failure. `hash_init()` calls `malloc()` and fails if memory cannot be allocated.

See section [A.8.6 Auxiliary Data](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC140), for an explanation of aux, which is most often a null pointer.

#### Function: void **hash\_clear** (struct hash \*hash, hash\_action\_func \*action)

Removes all the elements from hash, which must have been previously initialized with `hash_init()`.

If action is non-null, then it is called once for each element in the hash table, which gives the caller an opportunity to deallocate any memory or other resources used by the element. For example, if the hash table elements are dynamically allocated using `malloc()`, then action could `free()` the element. This is safe because `hash_clear()` will not access the memory in a given hash element after calling action on it. However, action must not call any function that may modify the hash table, such as `hash_insert()` or `hash_delete()`.

#### Function: void **hash\_destroy** (struct hash \*hash, hash\_action\_func \*action)

If action is non-null, calls it for each element in the hash, with the same semantics as a call to `hash_clear()`. Then, frees the memory held by hash. Afterward, hash must not be passed to any hash table function, absent an intervening call to `hash_init()`.

#### Function: size\_t **hash\_size** (struct hash \*hash)

Returns the number of elements currently stored in hash.

#### Function: bool **hash\_empty** (struct hash \*hash)

Returns true if hash currently contains no elements, false if hash contains at least one element.

### A.8.3 Search Functions

Each of these functions searches a hash table for an element that compares equal to one provided. Based on the success of the search, they perform some action, such as inserting a new element into the hash table, or simply return the result of the search.

#### Function: struct hash\_elem \***hash\_insert** (struct hash \*hash, struct hash\_elem \*element)

Searches hash for an element equal to element. If none is found, inserts element into hash and returns a null pointer. If the table already contains an element equal to element, it is returned without modifying hash.

#### Function: struct hash\_elem \***hash\_replace** (struct hash \*hash, struct hash\_elem \*element)

Inserts element into hash. Any element equal to element already in hash is removed. Returns the element removed, or a null pointer if hash did not contain an element equal to element.

The caller is responsible for deallocating any resources associated with the returned element, as appropriate. For example, if the hash table elements are dynamically allocated using `malloc()`, then the caller must `free()` the element after it is no longer needed.

The element passed to the following functions is only used for hashing and comparison purposes. It is never actually inserted into the hash table. Thus, only key data in the element needs to be initialized, and other data in the element will not be used. It often makes sense to declare an instance of the element type as a local variable, initialize the key data, and then pass the address of its `struct hash_elem` to `hash_find()` or `hash_delete()`. See section [A.8.5 Hash Table Example](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC139), for an example. (Large structures should not be allocated as local variables. See section [A.2.1 `struct thread`](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC108), for more information.)

#### Function: struct hash\_elem \***hash\_find** (struct hash \*hash, struct hash\_elem \*element)

Searches hash for an element equal to element. Returns the element found, if any, or a null pointer otherwise.

#### Function: struct hash\_elem \***hash\_delete** (struct hash \*hash, struct hash\_elem \*element)

Searches hash for an element equal to element. If one is found, it is removed from hash and returned. Otherwise, a null pointer is returned and hash is unchanged.

The caller is responsible for deallocating any resources associated with the returned element, as appropriate. For example, if the hash table elements are dynamically allocated using `malloc()`, then the caller must `free()` the element after it is no longer needed.

### A.8.4 Iteration Functions

These functions allow iterating through the elements in a hash table. Two interfaces are supplied. The first requires writing and supplying a hash\_action\_func to act on each element (see section [A.8.1 Data Types](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC135)).

#### Function: void **hash\_apply** (struct hash \*hash, hash\_action\_func \*action)

Calls action once for each element in hash, in arbitrary order. action must not call any function that may modify the hash table, such as `hash_insert()` or `hash_delete()`. action must not modify key data in elements, although it may modify any other data.

The second interface is based on an "iterator" data type. Idiomatically, iterators are used as follows:

```
struct hash_iterator i;

hash_first (&i, h);
while (hash_next (&i))
  {
    struct foo *f = hash_entry (hash_cur (&i), struct foo, elem);
    ...do something with f...
  }
```

#### Type: **struct hash\_iterator**

Represents a position within a hash table. Calling any function that may modify a hash table, such as `hash_insert()` or `hash_delete()`, invalidates all iterators within that hash table.

Like `struct hash` and `struct hash_elem`, `struct hash_elem` is opaque.

#### Function: void **hash\_first** (struct hash\_iterator \*iterator, struct hash \*hash)

Initializes iterator to just before the first element in hash.

#### Function: struct hash\_elem \***hash\_next** (struct hash\_iterator \*iterator)

Advances iterator to the next element in hash, and returns that element. Returns a null pointer if no elements remain. After `hash_next()` returns null for iterator, calling it again yields undefined behavior.

#### Function: struct hash\_elem \***hash\_cur** (struct hash\_iterator \*iterator)

Returns the value most recently returned by `hash_next()` for iterator. Yields undefined behavior after `hash_first()` has been called on iterator but before `hash_next()` has been called for the first time.

### A.8.5 Hash Table Example

Suppose you have a structure, called `struct page`, that you want to put into a hash table. First, define `struct page` to include a `struct hash_elem` member:

```
struct page
  {
    struct hash_elem hash_elem; /* Hash table element. */
    void *addr;                 /* Virtual address. */
    /* ...other members... */
  };
```

We write a hash function and a comparison function using addr as the key. A pointer can be hashed based on its bytes, and the < operator works fine for comparing pointers:

```
/* Returns a hash value for page p. */
unsigned
page_hash (const struct hash_elem *p_, void *aux UNUSED)
{
  const struct page *p = hash_entry (p_, struct page, hash_elem);
  return hash_bytes (&p->addr, sizeof p->addr);
}

/* Returns true if page a precedes page b. */
bool
page_less (const struct hash_elem *a_, const struct hash_elem *b_,
           void *aux UNUSED)
{
  const struct page *a = hash_entry (a_, struct page, hash_elem);
  const struct page *b = hash_entry (b_, struct page, hash_elem);

  return a->addr < b->addr;
}
```

(The use of `UNUSED` in these functions' prototypes suppresses a warning that aux is unused. See section [E.3 Function and Parameter Attributes](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_11.html#SEC159), for information about `UNUSED`. See section [A.8.6 Auxiliary Data](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC140), for an explanation of aux.)

Then, we can create a hash table like this:

```
struct hash pages;

hash_init (&pages, page_hash, page_less, NULL);
```

Now we can manipulate the hash table we've created. If `p` is a pointer to a `struct page`, we can insert it into the hash table with:

```
hash_insert (&pages, &p->hash_elem);
```

If there's a chance that pages might already contain a page with the same addr, then we should check `hash_insert()`'s return value.

To search for an element in the hash table, use `hash_find()`. This takes a little setup, because `hash_find()` takes an element to compare against. Here's a function that will find and return a page based on a virtual address, assuming that pages is defined at file scope:

```
/* Returns the page containing the given virtual address,
   or a null pointer if no such page exists. */
struct page *
page_lookup (const void *address)
{
  struct page p;
  struct hash_elem *e;

  p.addr = address;
  e = hash_find (&pages, &p.hash_elem);
  return e != NULL ? hash_entry (e, struct page, hash_elem) : NULL;
}
```

`struct page` is allocated as a local variable here on the assumption that it is fairly small. Large structures should not be allocated as local variables. See section [A.2.1 `struct thread`](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html#SEC108), for more information.

A similar function could delete a page by address using `hash_delete()`.

### A.8.6 Auxiliary Data

In simple cases like the example above, there's no need for the aux parameters. In these cases, just pass a null pointer to `hash_init()` for aux and ignore the values passed to the hash function and comparison functions. (You'll get a compiler warning if you don't use the aux parameter, but you can turn that off with the `UNUSED` macro, as shown in the example, or you can just ignore it.)

aux is useful when you have some property of the data in the hash table is both constant and needed for hashing or comparison, but not stored in the data items themselves. For example, if the items in a hash table are fixed-length strings, but the items themselves don't indicate what that fixed length is, you could pass the length as an aux parameter.

### A.8.7 Synchronization

The hash table does not do any internal synchronization. It is the caller's responsibility to synchronize calls to hash table functions. In general, any number of functions that examine but do not modify the hash table, such as `hash_find()` or `hash_next()`, may execute simultaneously. However, these function cannot safely execute at the same time as any function that may modify a given hash table, such as `hash_insert()` or `hash_delete()`, nor may more than one function that can modify a given hash table execute safely at once.

It is also the caller's responsibility to synchronize access to data in hash table elements. How to synchronize access to this data depends on how it is designed and organized, as with any other data structure.
