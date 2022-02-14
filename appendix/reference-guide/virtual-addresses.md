# Virtual Addresses

A 32-bit virtual address can be divided into a 20-bit _page number_ and a 12-bit _page offset_ (or just _offset_), like this:

```
               31               12 11        0
              +-------------------+-----------+
              |    Page Number    |   Offset  |
              +-------------------+-----------+
                       Virtual Address
```

Header `threads/vaddr.h` defines these functions and macros for working with virtual addresses:

#### Macro: **PGSHIFT**

#### Macro: **PGBITS**

The bit index (0) and number of bits (12) of the offset part of a virtual address, respectively.

#### Macro: **PGMASK**

A bit mask with the bits in the page offset set to 1, the rest set to 0 (0xfff).

#### Macro: **PGSIZE**

The page size in bytes (4,096).

#### Function: unsigned **pg\_ofs** (const void \*va)

Extracts and returns the page offset in virtual address _va_.

#### Function: uintptr\_t **pg\_no** (const void \*va)

Extracts and returns the page number in virtual address _va_.

#### Function: void \***pg\_round\_down** (const void \*va)

Returns the start of the virtual page that _va_ points within, that is, _va_ with the page offset set to 0.

#### Function: void \***pg\_round\_up** (const void \*va)

Returns _va_ rounded up to the nearest page boundary.

Virtual memory in Pintos is divided into two regions: user virtual memory and kernel virtual memory. The boundary between them is `PHYS_BASE`:

#### Macro: **PHYS\_BASE**

Base address of kernel virtual memory. It defaults to 0xc0000000 (3 GB), but it may be changed to any multiple of 0x10000000 from 0x80000000 to 0xf0000000.

User virtual memory ranges from virtual address 0 up to `PHYS_BASE`. Kernel virtual memory occupies the rest of the virtual address space, from `PHYS_BASE` up to 4 GB.

#### Function: bool **is\_user\_vaddr** (const void \*va)

#### Function: bool **is\_kernel\_vaddr** (const void \*va)

Returns true if _va_ is a user or kernel virtual address, respectively, false otherwise.

The 80x86 doesn't provide any way to directly access memory given a physical address. This ability is often necessary in an operating system kernel, so Pintos works around it by mapping kernel virtual memory one-to-one to physical memory. That is, virtual address `PHYS_BASE` accesses physical address 0, virtual address `PHYS_BASE` + 0x1234 accesses physical address 0x1234, and so on up to the size of the machine's physical memory. Thus, adding `PHYS_BASE` to a physical address obtains a kernel virtual address that accesses that address; conversely, subtracting `PHYS_BASE` from a kernel virtual address obtains the corresponding physical address. Header threads/vaddr.h provides a pair of functions to do these translations:

#### Function: void \***ptov** (uintptr\_t pa)

Returns the kernel virtual address corresponding to physical address _pa_, which should be between 0 and the number of bytes of physical memory.

#### Function: uintptr\_t **vtop** (void \*va)

Returns the physical address corresponding to _va_, which must be a kernel virtual address.

##
