# Your Tasks

## Task 0: Design Document

* Download the [project 4 design document template](https://github.com/PKU-OS/pintos/blob/master/docs/p4.md). Read through it to motivate your design and fill it in after you finish the project.&#x20;
* We recommend that you read the design document template before you start working on the project.&#x20;
* See section [Project Documentation](../../appendix/project-documentation.md), for a sample design document that goes along with a fictitious project.

## Task 1: Indexed and Extensible Files

### **Exercise 1.1**

**The basic file system allocates files as a single extent**, making it vulnerable to **external fragmentation**, that is, it is possible that an n-block file cannot be allocated even though n blocks are free.&#x20;

**Eliminate this problem by modifying the on-disk inode structure.** In practice, this probably means <mark style="color:red;">**using an index structure with direct, indirect, and doubly indirect blocks**</mark>.&#x20;

* You are welcome to choose a different scheme as long as you explain the rationale for it in your design documentation, and as long as it does not suffer from external fragmentation (as does the extent-based file system we provide).

{% hint style="success" %}
<mark style="color:green;">**Exercise 1.1**</mark>

<mark style="color:green;">**Modify inode structure to accommodate external fragmentation.**</mark>
{% endhint %}

#### Some Important Notes

* **You can assume that the file system partition will not be larger than 8 MB. You must support files as large as the partition (minus metadata).**&#x20;
  * **Each inode is stored in one disk sector**, limiting the number of block pointers that it can contain.&#x20;
  * Supporting 8 MB files will require you to implement doubly-indirect blocks.
* **An extent-based file can only grow if it is followed by empty space, but indexed inodes make file growth possible whenever free space is available.** Implement file growth.&#x20;
  * In the basic file system, the file size is specified when the file is created.&#x20;
  * In most modern file systems, a file is initially created with size 0 and is then <mark style="color:red;">**expanded every time a write is made off the end of the file**</mark>**. **<mark style="color:red;">**Your file system must allow this.**</mark>
* **There should be no predetermined limit on the size of a file, except that a file cannot exceed the size of the file system (minus metadata).**&#x20;
  * This also applies to the root directory file, which should now be allowed to expand beyond its initial limit of 16 files.
* **User programs are allowed to seek beyond the current end-of-file (EOF).**&#x20;
  * The seek itself does **not** extend the file.&#x20;
  * **Writing at a position past EOF extends the file to the position being written**, and any gap between the previous EOF and the start of the write must be filled with zeros.&#x20;
  * A read starting from a position past EOF returns no bytes.
* **Writing far beyond EOF can cause many blocks to be entirely zero.**&#x20;
  * Some file systems **allocate and write** real data blocks for these implicitly zeroed blocks.&#x20;
  * Other file systems **do not allocate these blocks at all until they are explicitly written**. The latter file systems are said to support "**sparse files.**"&#x20;
  * You may adopt **either** allocation strategy in your file system.

## Task 2: Subdirectories

### **Exercise 2.1**

**Implement a hierarchical name space.**&#x20;

* In the basic file system, all files live in a **single** directory.&#x20;
* Modify this to allow directory entries to point to files or to other directories.

{% hint style="success" %}
<mark style="color:green;">**Exercise 2.1**</mark>

<mark style="color:green;">**Implement a hierarchical name space.**</mark>&#x20;

* <mark style="color:green;">In the basic file system, all files live in a</mark> <mark style="color:green;"></mark><mark style="color:green;">**single**</mark> <mark style="color:green;"></mark><mark style="color:green;">directory.</mark>&#x20;
* <mark style="color:green;">Modify this to allow directory entries to point to files or to other directories.</mark>
{% endhint %}

#### Some Important Notes

* **Make sure that directories can expand beyond their original size just as any other file can.**
* **The basic file system has a 14-character limit on file names.**&#x20;
  * You may retain this limit for **individual** **file** **name** components, or may extend it, at your option.&#x20;
  * You must allow **full** **path** **names** to be much longer than 14 characters.

### **Exercise 2.2**

**Maintain a separate current directory for each process.**&#x20;

* At startup, set the _**root**_ as **the initial process's current directory**.&#x20;
* When one process starts another with the **`exec`** system call, **the child process inherits its parent's current directory**. After that, the two processes' current directories are **independent**, so that either changing its own current directory has no effect on the other. (This is why, under Unix, the `cd` command is a shell built-in, not an external program.)

{% hint style="success" %}
<mark style="color:green;">**Exercise 2.2**</mark>

<mark style="color:green;">**Update the system calls to support hierarchical file names.**</mark>
{% endhint %}

#### Some Important Notes

* **Update the existing system calls so that:**
  * Anywhere a file name is provided by the caller, an **absolute** or **relative** path name may be used.&#x20;
  * The directory separator character is **forward slash (/)**.&#x20;
  * You must also support special file names **`.`** and **`..`**, which have the same meanings as they do in Unix.
* **Update the open system call so that it can also open directories.**&#x20;
  * Of the existing system calls, only `close` needs to accept a file descriptor for a directory.
* **Update the remove system call so that it can delete empty directories (other than the root) in addition to regular files.**&#x20;
  * Directories may only be deleted if they **do not contain any files or subdirectories** (other than `.` and `..`).&#x20;
  * You may decide whether to allow deletion of a directory that is open by a process or in use as a process's current working directory. If it is allowed, then attempts to open files (including `.` and `..`) or create new files in a deleted directory must be disallowed.

### **Exercise 2.3**

{% hint style="success" %}
<mark style="color:green;">**Exercise 2.3**</mark>

<mark style="color:green;">**Implement the following new system calls:**</mark>
{% endhint %}

* <mark style="color:blue;">**System Call: bool chdir (const char \*dir)**</mark>
  * **Changes the current working directory of the process to dir**, which may be relative or absolute.&#x20;
  * Returns true if successful, false on failure.
* <mark style="color:blue;">**System Call: bool mkdir (const char \*dir)**</mark>
  * **Creates the directory named dir**, which may be relative or absolute.&#x20;
  * Returns true if successful, false on failure.&#x20;
  * **Fails if dir already exists or if any directory name in dir, besides the last, does not already exist.** That is, `mkdir("/a/b/c")` succeeds only if /a/b already exists and /a/b/c does not.
* <mark style="color:blue;">**System Call: bool readdir (int fd, char \*name)**</mark>
  * **Reads a directory entry from file descriptor fd**, which must represent a directory.&#x20;
  * If successful, stores the null-terminated file name in name, which must have room for `READDIR_MAX_LEN + 1` bytes, and returns true. If no entries are left in the directory, returns false.
  * **`.` and `..` should not be returned by `readdir`.**
  * **If the directory changes while it is open, then it is acceptable for some entries not to be read at all or to be read multiple times.** Otherwise, each directory entry should be read once, **in any order**.
  * `READDIR_MAX_LEN` is defined in `lib/user/syscall.h`. If your file system supports longer file names than the basic file system, you should increase this value from the default of 14.
* <mark style="color:blue;">**System Call: bool isdir (int fd)**</mark>
  * **Returns true if fd represents a directory, false if it represents an ordinary file.**
* <mark style="color:blue;">**System Call: int inumber (int fd)**</mark>
  * **Returns the **_**inode number**_** of the inode associated with fd**, which may represent an ordinary file or a directory.
  * **An inode number persistently identifies a file or directory.** It is **unique** during the file's existence. In Pintos, the sector number of the inode is suitable for use as an inode number.

We have provided **`ls`** and **`mkdir`** user programs, which are straightforward once the above syscalls are implemented. We have also provided **`pwd`**, which is not so straightforward. The **`shell`** program implements **`cd`** internally.

The **`pintos`** extract and append commands should now accept full path names, assuming that the directories used in the paths have already been created. This should not require any significant extra effort on your part.

## Task 3: Buffer Cache

### **Exercise 3.1**

**Modify the file system to keep a cache of file blocks.**&#x20;

* When a request is made to read or write a block, **check to see if it is in the cache**, and if so, use the cached data without going to disk.&#x20;
* Otherwise, **fetch the block from disk into the cache**, evicting an older entry if necessary.&#x20;
* **You are limited to a cache **<mark style="color:red;">**no greater than 64 sectors in size**</mark>**.**

{% hint style="success" %}
<mark style="color:green;">**Exercise 3.1**</mark>

<mark style="color:green;">**Implement buffer cache and cache replacement algorithm.**</mark>
{% endhint %}

#### Some Important Notes

* **You must implement a cache replacement algorithm that is **<mark style="color:red;">**at least as good as the "clock" algorithm**</mark>**.**&#x20;
  * We encourage you to account for the generally greater value of **metadata** compared to data.
  * Experiment to see what combination of accessed, dirty, and other information results in the best performance, as measured by the number of disk accesses.
* **You can keep a cached copy of the free map permanently in memory if you like.** It doesn't have to count against the cache size.
* **The provided inode code uses a "bounce buffer" allocated with `malloc()` to translate the disk's sector-by-sector interface into the system call interface's byte-by-byte interface.**&#x20;
  * You should <mark style="color:red;">**get rid of these bounce buffers**</mark>.&#x20;
  * Instead, copy data into and out of sectors in the **buffer cache** directly.
* **Your cache should be **_<mark style="color:red;">**write-behind**</mark>_**, that is, keep **_**dirty**_** blocks in the cache, instead of immediately writing modified data to disk.**&#x20;
  * Write dirty blocks to disk whenever they are **evicted**.&#x20;
  * Because write-behind makes your file system more fragile in the face of crashes, <mark style="color:red;">**in addition you should periodically write all dirty, cached blocks back to disk**</mark>.&#x20;
  * **The cache should also be written back to disk in `filesys_done()`**, so that halting Pintos flushes the cache.
  * **If you have `timer_sleep()` from the first project working, write-behind is an excellent application.** Otherwise, you may implement a less general facility, but make sure that it does not exhibit busy-waiting.
* **You should also implement **_<mark style="color:red;">**read-ahead**</mark>_**, that is, automatically fetch the next block of a file into the cache when one block of a file is read, in case that block is about to be read.**&#x20;
  * Read-ahead is only really useful when done **asynchronously**. That means, if a process requests disk block 1 from the file, it should block until disk block 1 is read in, but once that read is complete, control should return to the process immediately.&#x20;
  * The read-ahead request for disk block 2 should be handled **asynchronously**, **in the background**.

{% hint style="info" %}
**Tips:**&#x20;

**We recommend integrating the cache into your design **_**early**_**.**&#x20;

* In the past, many groups have tried to tack the cache onto a design late in the design process. This is very difficult. These groups have often turned in projects that failed most or all of the tests.
{% endhint %}

## Task 4: Synchronization

### **Exercise 4.1**

**The provided file system requires external synchronization, that is, callers must ensure that **_**only**_**  **_**one**_** thread can be running in the file system code at once.**&#x20;

* Your submission must adopt a **finer-grained** synchronization strategy that **does not require external synchronization**.&#x20;
* To the extent possible, operations on independent entities should be independent, so that they do not need to wait on each other.

{% hint style="success" %}
<mark style="color:green;">**Exercise 4.1**</mark>

<mark style="color:green;">**Support finer-grained synchronization of file system.**</mark>
{% endhint %}

#### Some Important Notes

* **Operations on different cache blocks must be independent.** In particular, when I/O is required on a particular block, operations on other blocks that do not require I/O should proceed without having to wait for the I/O to complete.
* **Multiple processes must be able to access a single file at once.**&#x20;
  * Multiple reads of a single file must be able to complete without waiting for one another.&#x20;
  * When writing to a file **does not extend the file**, multiple processes should also be able to write a single file at once.&#x20;
  * A read of a file by one process when the file is being written by another process is allowed to show that none, all, or part of the write has completed. (However, after the `write` system call returns to its caller, all subsequent readers must see the change.)&#x20;
  * Similarly, when two processes simultaneously write to the same part of a file, their data may be interleaved.
* **On the other hand, extending a file and writing data into **_**the new section**_** must be atomic.**
  * Suppose processes A and B both have a given file open and both are positioned at end-of-file. If A reads and B writes the file at the same time, A may read all, part, or none of what B writes. However, A may not read data other than what B writes, e.g. if B's data is all nonzero bytes, A is not allowed to see any zeros.
* **Operations on different directories should take place concurrently.** **Operations on the same directory may wait for one another.**
* Keep in mind that **only data shared by multiple threads needs to be synchronized.**&#x20;
  * In the base file system, `struct file` and `struct dir` are accessed only by a single thread.
