# Background

### New Code

Here are some files that are probably new to you. These are in the filesys directory except where indicated:

fsutil.cSimple utilities for the file system that are accessible from the kernel command line.filesys.hfilesys.cTop-level interface to the file system. See section [4.1.2 Using the File System](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/project2.html#SEC46), for an introduction.directory.hdirectory.cTranslates file names to inodes. The directory data structure is stored as a file.inode.hinode.cManages the data structure representing the layout of a file's data on disk.file.hfile.cTranslates file reads and writes to disk sector reads and writes.lib/kernel/bitmap.hlib/kernel/bitmap.cA bitmap data structure along with routines for reading and writing the bitmap to disk files.

Our file system has a Unix-like interface, so you may also wish to read the Unix man pages for `creat`, `open`, `close`, `read`, `write`, `lseek`, and `unlink`. Our file system has calls that are similar, but not identical, to these. The file system translates these calls into disk operations.

All the basic functionality is there in the code above, so that the file system is usable from the start, as you've seen in the previous two projects. However, it has severe limitations which you will remove.

While most of your work will be in filesys, you should be prepared for interactions with all previous parts.

### Testing File System Persistence

By now, you should be familiar with the basic process of running the Pintos tests. See section [1.2.1 Testing](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_1.html#SEC8), for review, if necessary.

Until now, each test invoked Pintos just once. However, an important purpose of a file system is to ensure that data remains accessible from one boot to another. Thus, the tests that are part of the file system project invoke Pintos a second time. The second run combines all the files and directories in the file system into a single file, then copies that file out of the Pintos file system into the host (Unix) file system.

The grading scripts check the file system's correctness based on the contents of the file copied out in the second run. This means that your project will not pass any of the extended file system tests until the file system is implemented well enough to support `tar`, the Pintos user program that produces the file that is copied out. The `tar` program is fairly demanding (it requires both extensible file and subdirectory support), so this will take some work. Until then, you can ignore errors from `make check` regarding the extracted file system.

Incidentally, as you may have surmised, the file format used for copying out the file system contents is the standard Unix "tar" format. You can use the Unix `tar` program to examine them. The tar file for test t is named t.tar.
