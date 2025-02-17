# Background

## New Code

Here are some files that are probably new to you. These are **in the filesys directory** except where indicated:

* <mark style="color:blue;">**fsutil.c**</mark>
  * **Simple utilities** for the file system that are accessible from the kernel command line.
* <mark style="color:blue;">**filesys.h**</mark>
* <mark style="color:blue;">**filesys.c**</mark>
  * **Top-level interface to the file system.**
  * See section [Using the File System](../lab2-user-programs/background.md#using-the-file-system), for an introduction.
* <mark style="color:blue;">**directory.h**</mark>
* <mark style="color:blue;">**directory.c**</mark>
  * **Translates file names to inodes.**
  * The directory data structure is stored as a file.
* <mark style="color:blue;">**inode.h**</mark>
* <mark style="color:blue;">**inode.c**</mark>
  * **Manages the data structure representing the layout of a file's data on disk.**
* <mark style="color:blue;">**file.h**</mark>
* <mark style="color:blue;">**file.c**</mark>
  * **Translates file reads and writes to disk sector reads and writes.**
* <mark style="color:blue;">**lib/kernel/bitmap.h**</mark>
* <mark style="color:blue;">**lib/kernel/bitmap.c**</mark>
  * **A bitmap data structure** along with **routines for reading and writing the bitmap to disk files.**

**Our file system has a&#x20;**_**Unix-like**_ **interface, so you may also wish to read the Unix man pages for `creat`,  `open`,  `close`,  `read`, `write`, `lseek`, and `unlink`.**

* Our file system has calls that are <mark style="color:red;">**similar, but not identical**</mark>, to these.
* The file system translates these calls into **disk operations**.

**All the&#x20;**_**basic**_ **functionality is there in the code above**, so that the file system is usable from the start, as you've seen in the previous two projects. However, it **has severe limitations** which you will remove.

While most of your work will be in `filesys/`, you should be prepared for interactions with all previous parts.

## Testing File System Persistence

**By now, you should be familiar with the basic process of running the Pintos tests.** See section [Testing](../../getting-started/debug-and-test/testing.md), for review, if necessary.

**Until now, each test invoked Pintos just&#x20;**_**once**_**.** However, <mark style="color:red;">**an important purpose of a file system is to ensure that data remains accessible from one boot to another**</mark>.

* Thus, the tests that are part of the file system project **invoke Pintos a second** **time**.
* The second run combines all the files and directories in the file system into a single file, then copies that file out of the Pintos file system into the host (Unix) file system.

**The grading scripts check the file system's correctness based on the contents of the file copied out in the second run.**

* This means that your project will not pass any of the extended file system tests until the file system is implemented well enough to support **`tar`**, the Pintos user program that produces the file that is copied out.
* The **`tar`** program is fairly demanding (it requires both extensible file and subdirectory support), so this will take some work. Until then, you can ignore errors from `make check` regarding the extracted file system.
* Incidentally, as you may have surmised, **the file format used for copying out the file system contents is the standard Unix "tar" format**. You can use the Unix `tar` program to examine them. The tar file for test _t_ is named _t.tar_.
