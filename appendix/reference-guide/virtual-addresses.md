# Virtual Addresses

**A 32-bit virtual address can be divided into a 20-bit&#x20;**_**page number**_**&#x20;and a 12-bit&#x20;**_**page offset**_**&#x20;(or just&#x20;**_**offset**_**), like this:**

```
               31               12 11        0
              +-------------------+-----------+
              |    Page Number    |   Offset  |
              +-------------------+-----------+
                       Virtual Address
```

## Macros and Functions

### Work with Virtual Addresses

Header **`threads/vaddr.h`** defines these functions and macros for working with virtual addresses:

* <mark style="color:blue;">**Macro: PGSHIFT**</mark>
* <mark style="color:blue;">**Macro: PGBITS**</mark>
  * **The bit index (0) and number of bits (12) of the offset part of a virtual address, respectively.**
* <mark style="color:blue;">**Macro: PGMASK**</mark>
  * **A bit mask with the bits in the page offset set to 1, the rest set to 0 (0xfff).**
* <mark style="color:blue;">**Macro: PGSIZE**</mark>
  * **The page size in bytes (4,096).**
* <mark style="color:blue;">**Function: unsigned pg\_ofs (const void \*va)**</mark>
  * **Extracts and returns the page offset in virtual address&#x20;**_**va**_**.**
* <mark style="color:blue;">**Function: uintptr\_t pg\_no (const void \*va)**</mark>
  * **Extracts and returns the page number in virtual address&#x20;**_**va**_**.**
* <mark style="color:blue;">**Function: void \*pg\_round\_down (const void \*va)**</mark>
  * **Returns the start of the virtual page that&#x20;**_**va**_**&#x20;points within**, that is, _va_ with the page offset set to 0.
* <mark style="color:blue;">**Function: void \*pg\_round\_up (const void \*va)**</mark>
  * **Returns&#x20;**_**va**_ **rounded up to the nearest page boundary.**

### Work with User/Kernel Virtual Memory Boundary

Virtual memory in Pintos is divided into **two regions**: **user virtual memory** and **kernel virtual memory.** The boundary between them is **`PHYS_BASE`**:

* <mark style="color:blue;">**Macro: PHYS\_BASE**</mark>
  * **Base address of kernel virtual memory.**
  * It **defaults to 0xc0000000 (3 GB)**, but it may be changed to any multiple of 0x10000000 from 0x80000000 to 0xf0000000.
  * User virtual memory ranges from virtual address 0 up to `PHYS_BASE`. Kernel virtual memory occupies the rest of the virtual address space, from `PHYS_BASE` up to 4 GB.
* <mark style="color:blue;">**Function: bool is\_user\_vaddr (const void \*va)**</mark>
* <mark style="color:blue;">**Function: bool is\_kernel\_vaddr (const void \*va)**</mark>
  * **Returns true if&#x20;**_**va**_ **is a user or kernel virtual address, respectively, false otherwise.**

### Work with Mapping Kernel VM One-to-One to PM

**The 80x86 doesn't provide any way to directly access memory given a physical address.** This ability is often necessary in an operating system kernel, so **Pintos works around it by mapping kernel virtual memory one-to-one to physical memory**.

* That is, virtual address `PHYS_BASE` accesses physical address 0, virtual address `PHYS_BASE` + 0x1234 accesses physical address 0x1234, and so on up to the size of the machine's physical memory.
* Thus, <mark style="color:red;">**adding**</mark><mark style="color:red;">\*\*</mark> `PHYS_BASE` <mark style="color:red;">**to a physical address obtains a kernel virtual address that accesses that address**</mark>**; conversely,&#x20;**<mark style="color:red;">**subtracting**</mark> `PHYS_BASE` \*\*<mark style="color:red;">**from a kernel virtual address obtains the corresponding physical address**</mark>.

Header **`threads/vaddr.h`** provides a pair of functions to do these translations:

* <mark style="color:blue;">**Function: void \*ptov (uintptr\_t pa)**</mark>
  * **Returns the kernel virtual address corresponding to physical address&#x20;**_**pa**_, which should be between 0 and the number of bytes of physical memory.
* <mark style="color:blue;">**Function: uintptr\_t vtop (void \*va)**</mark>
  * **Returns the physical address corresponding to&#x20;**_**va**_, which must be a kernel virtual address.
