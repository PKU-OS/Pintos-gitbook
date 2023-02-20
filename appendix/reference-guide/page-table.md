# Page Table

**The code in `pagedir.c` is an abstract interface to the 80x86 hardware page table, also called a "page directory" by Intel processor documentation.**

* The page table interface uses a **`uint32_t *`** to represent a page table because this is convenient for accessing their internal structure.
* The sections below describe **the page table interface** and **internals**.

## Creation, Destruction, and Activation

**These functions create, destroy, and activate page tables.** The base Pintos code already calls these functions where necessary, so **it should not be necessary to call them yourself**.

* <mark style="color:blue;">**Function: uint32\_t \*pagedir\_create (void)**</mark>
  * **Creates and returns a new page table.**
  * <mark style="color:red;">**The new page table contains Pintos's normal kernel virtual page mappings, but no user virtual mappings.**</mark>
  * Returns a null pointer if memory cannot be obtained.
* <mark style="color:blue;">**Function: void pagedir\_destroy (uint32\_t \*pd)**</mark>
  * **Frees all of the resources held by pd, including the page table itself and the frames that it maps.**
* <mark style="color:blue;">**Function: void pagedir\_activate (uint32\_t \*pd)**</mark>
  * **Activates pd.**
  * The active page table is the one used by the CPU to translate memory references.

## Inspection and Updates

**These functions examine or update the mappings from pages to frames encapsulated by a page table.** They work on both active and inactive page tables (that is, those for running and suspended processes), <mark style="color:red;">**flushing the TLB**</mark> as necessary.

* <mark style="color:blue;">**Function: bool pagedir\_set\_page (uint32\_t \*pd, void \*upage, void \*kpage, bool writable)**</mark>
  * **Adds to pd** _a mapping from user page_ **upage** _to the frame identified by kernel virtual address_ **kpage**_**.** If_ **writable** is true, the page is mapped read/write; otherwise, it is mapped read-only.
  * User page _upage_ must not already be mapped in _pd_.
  * **Kernel page kpage** should be a kernel virtual address <mark style="color:red;">**obtained from the user pool**</mark> with `palloc_get_page(PAL_USER)`.
  * Returns true if successful, false on failure. Failure will occur if additional memory required for the page table cannot be obtained.
* <mark style="color:blue;">**Function: void \*pagedir\_get\_page (uint32\_t \*pd, const void \*uaddr)**</mark>
  * **Looks up the frame mapped to uaddr** _in_ **pd.**
  * Returns the kernel virtual address for that frame, if _uaddr_ is mapped, or a null pointer if it is not.
* <mark style="color:blue;">**Function: void pagedir\_clear\_page (uint32\_t \*pd, void \*page)**</mark>
  * **Marks page** _"not present" in_ **pd.** Later accesses to the _page_ will fault.
  * **Other bits in the page table for page** are preserved, permitting the accessed and dirty bits (see the next section) to be checked.
  * This function has no effect if _page_ is not mapped.

## Accessed and Dirty Bits

**80x86 hardware provides some assistance for implementing page replacement algorithms, through a pair of bits in the page table entry (PTE) for each page.**

* On **any read or write** to a page, the CPU **sets the accessed bit** _to 1 in the page's PTE, and on **any write**, the CPU sets the_ **dirty bit** to 1.
* <mark style="color:red;">**The CPU never resets these bits to 0, but the OS may do so.**</mark>
* Proper interpretation of these bits requires understanding of _**aliases**_, that is, two (or more) pages that refer to the same frame. When an aliased frame is accessed, the accessed and dirty bits are updated in only one page table entry (the one for the page used for access). The accessed and dirty bits for the other aliases are not updated.
* In project 3, you will apply these bits in implementing **page replacement algorithms**.

The followings are some functions related to accessed and dirty bits:

* <mark style="color:blue;">**Function: bool pagedir\_is\_dirty (uint32\_t \*pd, const void \*page)**</mark>
* <mark style="color:blue;">**Function: bool pagedir\_is\_accessed (uint32\_t \*pd, const void \*page)**</mark>
  * Returns true if page directory _pd_ contains a page table entry for _page_ that is marked dirty (or accessed). Otherwise, returns false.
* <mark style="color:blue;">**Function: void pagedir\_set\_dirty (uint32\_t \*pd, const void \*page, bool value)**</mark>
* <mark style="color:blue;">**Function: void pagedir\_set\_accessed (uint32\_t \*pd, const void \*page, bool value)**</mark>
  * If page directory _pd_ has a page table entry for _page_, then its dirty (or accessed) bit is set to _value_.

## Page Table Details

**The functions provided with Pintos are sufficient to implement the projects.** However, you may still find it worthwhile to understand the hardware page table format, so we'll go into a little detail in this section.

### **Structure**

* The top-level paging data structure is a page called the **"page directory" (PD)** arranged as **an array of 1,024 32-bit page directory entries (PDEs)**, each of which represents **4 MB** of virtual memory.
* Each PDE may point to the physical address of another page called a **"page table" (PT)** arranged, similarly, as **an array of 1,024 32-bit page table entries (PTEs)**, each of which translates a single **4 kB** virtual page to a physical page.

### Address Translation

Translation of a virtual address into a physical address follows **the three-step process** illustrated in the diagram below:

{% hint style="info" %}
Actually, **virtual to physical translation on the 80x86 architecture occurs via an intermediate "linear address,"** but Pintos (and most modern 80x86 OSes) set up the CPU so that linear and virtual addresses are one and the same. Thus, you can effectively ignore this CPU feature.
{% endhint %}

1. **The most-significant 10 bits of the virtual address (bits 22...31) index the page directory.**
   * If the PDE is marked **"present,"** the physical address of a page table is read from the PDE thus obtained.
   * If the PDE is marked **"not present,"** then a page fault occurs.
2. **The next 10 bits of the virtual address (bits 12...21) index the page table.**
   * If the PTE is marked **"present,"** the physical address of a data page is read from the PTE thus obtained.
   * If the PTE is marked **"not present,"** then a page fault occurs.
3. **The least-significant 12 bits of the virtual address (bits 0...11) are added to the data page's physical base address, yielding the final physical address.**

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

<details>

<summary>Macros and Functions for Page Tables</summary>

* <mark style="color:blue;">**Macro: PTSHIFT**</mark>
* <mark style="color:blue;">**Macro: PTBITS**</mark>
  * **The starting bit index (12) and number of bits (10), respectively, in a page table index.**
* <mark style="color:blue;">**Macro: PTMASK**</mark>
  * **A bit mask with the bits in the page table index set to 1 and the rest set to 0 (0x3ff000)**.
* <mark style="color:blue;">**Macro: PTSPAN**</mark>
  * **The number of bytes of virtual address space that a single page table page covers (4,194,304 bytes, or 4 MB).**
* <mark style="color:blue;">**Macro: PDSHIFT**</mark>
* <mark style="color:blue;">**Macro: PDBITS**</mark>
  * **The starting bit index (22) and number of bits (10), respectively, in a page directory index.**
* <mark style="color:blue;">**Macro: PDMASK**</mark>
  * **A bit mask with the bits in the page directory index set to 1 and other bits set to 0 (0xffc00000).**
* <mark style="color:blue;">**Function: uintptr\_t pd\_no (const void \*va)**</mark>
* <mark style="color:blue;">**Function: uintptr\_t pt\_no (const void \*va)**</mark>
  * **Returns the page directory index or page table index, respectively, for virtual address \_va**\_**.** These functions are defined in **`threads/pte.h`**.
* <mark style="color:blue;">**Function: unsigned pg\_ofs (const void \*va)**</mark>
  * **Returns the page offset for virtual address \_va**\_**.** This function is defined in **`threads/vaddr.h`**.

</details>

### **Page Table Entry Format**

You do not need to understand the PTE format to do the Pintos projects, unless you wish to incorporate the page table into your supplemental page table in project 3.

**The actual format of a page table entry is summarized below.** For complete information, refer to section 3.7, "Page Translation Using 32-Bit Physical Addressing," in \[[IA32-v3a](../bibliography.md#hardware-references)].

```
 31                                   12 11 9      6 5     2 1 0
+---------------------------------------+----+----+-+-+---+-+-+-+
|           Physical Address            | AVL|    |D|A|   |U|W|P|
+---------------------------------------+----+----+-+-+---+-+-+-+
```

Some more information on each bit is given below. The names are **`threads/pte.h`** macros that represent the bits' values:

<details>

<summary>Macros and Functions for Page Table Entry</summary>

* <mark style="color:blue;">**Macro: PTE\_P**</mark>
  * **Bit 0, the "present" bit.**
  * When this bit is 1, the other bits are interpreted as described below. When this bit is 0, any attempt to access the page will page fault. The remaining bits are then not used by the CPU and may be used by the OS for any purpose.
* <mark style="color:blue;">**Macro: PTE\_W**</mark>
  * **Bit 1, the "read/write" bit.**
  * When it is 1, the page is writable. When it is 0, write attempts will page fault.
* <mark style="color:blue;">**Macro: PTE\_U**</mark>
  * **Bit 2, the "user/supervisor" bit.**
  * When it is 1, user processes may access the page. When it is 0, only the kernel may access the page (user accesses will page fault).
  * Pintos clears this bit in PTEs for kernel virtual memory, to prevent user processes from accessing them.
* <mark style="color:blue;">**Macro: PTE\_A**</mark>
  * **Bit 5, the "accessed" bit.**
  * See section [Accessed and Dirty Bits](page-table.md#3.-accessed-and-dirty-bits).
* <mark style="color:blue;">**Macro: PTE\_D**</mark>
  * **Bit 6, the "dirty" bit.**
  * See section [Accessed and Dirty Bits](page-table.md#3.-accessed-and-dirty-bits).
* <mark style="color:blue;">**Macro: PTE\_AVL**</mark>
  * **Bits 9...11, available for operating system use.**
  * Pintos, as provided, does not use them and sets them to 0.
* <mark style="color:blue;">**Macro: PTE\_ADDR**</mark>
  * **Bits 12...31, the top 20 bits of the physical address of a frame.** The low 12 bits of the frame's address are always 0.
* <mark style="color:orange;">**Other bits are either reserved or uninteresting in a Pintos context and should be set to 0.**</mark>

Header **`threads/pte.h`** defines three functions for working with page table entries:

* <mark style="color:blue;">**Function: uint32\_t pte\_create\_kernel (uint32\_t \*page, bool writable)**</mark>
  * **Returns a page table entry that points to page, which should be a kernel virtual address.**
  * The PTE's present bit will be set. It will be marked for kernel-only access.
  * If _writable_ is true, the PTE will also be marked read/write; otherwise, it will be read-only.
* <mark style="color:blue;">**Function: uint32\_t pte\_create\_user (uint32\_t \*page, bool writable)**</mark>
  * **Returns a page table entry that points to page, which should be the kernel virtual address of a frame in the user pool.**
  * The PTE's present bit will be set and it will be marked to allow user-mode access.
  * If writable is true, the PTE will also be marked read/write; otherwise, it will be read-only.
* <mark style="color:blue;">**Function: void \*pte\_get\_page (uint32\_t pte)**</mark>
  * **Returns the kernel virtual** _address for the frame that_ **pte** points to.
  * The _pte_ may be present or not-present; if it is not-present then the pointer returned is only meaningful if the address bits in the PTE actually represent a physical address.

</details>

### **Page Directory Entry Format**

**Page directory entries have the same format as PTEs, except that the physical address points to a page table page instead of a frame.** Header **`threads/pte.h`** defines two functions for working with page directory entries:

* <mark style="color:blue;">**Function: uint32\_t pde\_create (uint32\_t \*pt)**</mark>
  * **Returns a page directory that points to pt, which should be the kernel virtual address of a page table page.**
  * The PDE's present bit will be set, it will be marked to allow user-mode access, and it will be marked read/write.
* <mark style="color:blue;">**Function: uint32\_t \*pde\_get\_pt (uint32\_t pde)**</mark>
  * **Returns the kernel virtual address for the page table page that pde, which must be marked present, points to.**
