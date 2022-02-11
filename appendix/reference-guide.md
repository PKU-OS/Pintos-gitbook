# Reference Guide

OS programming requires looking up documentations of various sorts, e.g., ISA, hardware interfaces, compilers, architecture. Often times, the lengthy manual is the only reference that is available. Pintos is a small and simple OS. But in doing the projects, you will run into many situations where you need to look up some documentation. This course will be one of the few courses where you will be reading a lot of documentations, which hopefully helps you become efficient at it and write better documentations later. For the listed references, you don’t need to go over them from beginning to end at once unless it’s recommended so.

## Pintos <a href="#pintos" id="pintos"></a>

* You should read everything below **before attempting any of the projects**:

{% content-ref url="broken-reference" %}
[Broken link](broken-reference)
{% endcontent-ref %}

* [Project Guide](broken-reference)
* [Setup Instructions](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/setup.html)
* [Getting Started](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_1.html)
* [Coding Standards](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_9.html)
* [Design Document](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_10.html)
* You’ll want to read these once you start work on the projects. **Their advice can save you a lot of time**:
  * [Debugging Tools](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_11.html)
  * [Development Tools](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_12.html)
* You need not read the reference guide, but you may find the information in it valuable from time to time:
  * [Reference Guide](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_7.html)
* Bracketed notations in Pintos source code comments can be looked up in the [Bibliography](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_14.html). The corresponding manual (e.g., ELF specification) will be fount in the link of the Bibliography.
* [Code Document](https://jhu-cs318.github.io/pintos-doxygen/html/index.html)

***

#### OS development <a href="#os-development" id="os-development"></a>

* [OS Dev wiki](https://wiki.osdev.org/Main\_Page): a great resource for OS development in general, lots of good references.

***

#### C and x86 assembly <a href="#c-and-x86-assembly" id="c-and-x86-assembly"></a>

* If your C or assembly are rusty, it is useful to recap the basics before attempting lab 1:
  * [The C Programming Language](https://catalyst.library.jhu.edu/catalog/bib\_651579) book by Brian Kernighan and Dennis Ritchie (also known as ‘K\&R’). Best reference for C language.
  * [A tutorial on C pointers](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/project/specs/pointers.pdf)
  * [C programming wikibook](https://en.wikibooks.org/wiki/C\_Programming)
  * [AT\&T assembly syntax](https://en.wikibooks.org/wiki/X86\_Assembly/GAS\_Syntax)
  * [Inline assembly](https://wiki.osdev.org/Inline\_Assembly)

***

#### x86 Emulator <a href="#x86-emulator" id="x86-emulator"></a>

* [QEMU](http://www.qemu.org): a popular, fast CPU emulator and virtualizer.
  * [Documentation](http://wiki.qemu.org/Qemu-doc.html)
* [Bochs](http://bochs.sourceforge.net): another popular x86 emulator. slower than QEMU, but more mature and faithful.
  * [Documentation](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/references.html)
