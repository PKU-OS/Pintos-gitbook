# FAQ

## General FAQ

#### **How much code will I need to write?**

Here's a summary of our reference solution, produced by the `diffstat` program. The final row gives total lines inserted and deleted; a changed line counts as both an insertion and a deletion.

This summary is relative to the Pintos base code, but the reference solution for project 4 is based on the reference solution to project 3. Thus, the reference solution runs with virtual memory enabled. See section FAQ, for the summary of project 3.

The reference solution represents just one possible solution. Many other solutions are also possible and many of those differ greatly from the reference solution. Some excellent solutions may not modify all the files modified by the reference solution, and some may modify files not modified by the reference solution.

```
 Makefile.build       |    5 
 devices/timer.c      |   42 ++
 filesys/Make.vars    |    6 
 filesys/cache.c      |  473 +++++++++++++++++++++++++
 filesys/cache.h      |   23 +
 filesys/directory.c  |   99 ++++-
 filesys/directory.h  |    3 
 filesys/file.c       |    4 
 filesys/filesys.c    |  194 +++++++++-
 filesys/filesys.h    |    5 
 filesys/free-map.c   |   45 +-
 filesys/free-map.h   |    4 
 filesys/fsutil.c     |    8 
 filesys/inode.c      |  444 ++++++++++++++++++-----
 filesys/inode.h      |   11 
 threads/init.c       |    5 
 threads/interrupt.c  |    2 
 threads/thread.c     |   32 +
 threads/thread.h     |   38 +-
 userprog/exception.c |   12 
 userprog/pagedir.c   |   10 
 userprog/process.c   |  332 +++++++++++++----
 userprog/syscall.c   |  582 ++++++++++++++++++++++++++++++-
 userprog/syscall.h   |    1 
 vm/frame.c           |  161 ++++++++
 vm/frame.h           |   23 +
 vm/page.c            |  297 +++++++++++++++
 vm/page.h            |   50 ++
 vm/swap.c            |   85 ++++
 vm/swap.h            |   11 
 30 files changed, 2721 insertions(+), 286 deletions(-)
```

#### **Can `BLOCK_SECTOR_SIZE` change?**

No, `BLOCK_SECTOR_SIZE` is fixed at 512. For IDE disks, this value is a fixed property of the hardware. Other disks do not necessarily have a 512-byte sector, but for simplicity Pintos only supports those that do.

### 4.1 Indexed Files FAQ

#### **What is the largest file size that we are supposed to support?**

The file system partition we create will be 8 MB or smaller. However, individual files will have to be smaller than the partition to accommodate the metadata. You'll need to consider this when deciding your inode organization.

### 4.2 Subdirectories FAQ

#### **How should a file name like a//b be interpreted?**

Multiple consecutive slashes are equivalent to a single slash, so this file name is the same as a/b.

#### **How about a file name like /../x?**

The root directory is its own parent, so it is equivalent to /x.

#### **How should a file name that ends in / be treated?**

Most Unix systems allow a slash at the end of the name for a directory, and reject other names that end in slashes. We will allow this behavior, as well as simply rejecting a name that ends in a slash.

### 4.3 Buffer Cache FAQ

#### **Can we keep a `struct inode_disk` inside `struct inode`?**

The goal of the 64-block limit is to bound the amount of cached file system data. If you keep a block of disk data--whether file data or metadata--anywhere in kernel memory then you have to count it against the 64-block limit. The same rule applies to anything that's "similar" to a block of disk data, such as a `struct inode_disk` without the `length` or `sector_cnt` members.

That means you'll have to change the way the inode implementation accesses its corresponding on-disk inode right now, since it currently just embeds a `struct inode_disk` in `struct inode` and reads the corresponding sector from disk when it's created. Keeping extra copies of inodes would subvert the 64-block limitation that we place on your cache.

You can store a pointer to inode data in `struct inode`, but if you do so you should carefully make sure that this does not limit your OS to 64 simultaneously open files. You can also store other information to help you find the inode when you need it. Similarly, you may store some metadata along each of your 64 cache entries.

You can keep a cached copy of the free map permanently in memory if you like. It doesn't have to count against the cache size.

`byte_to_sector()` in filesys/inode.c uses the `struct inode_disk` directly, without first reading that sector from wherever it was in the storage hierarchy. This will no longer work. You will need to change `inode_byte_to_sector()` to obtain the `struct inode_disk` from the cache before using it.
