# Suggestions

## Suggested Order of Implementation

We suggest first implementing the following, which can happen in parallel:

* Argument passing. Every user program will page fault immediately until argument passing is implemented. If you implement it correctly, you should pass all the `args-xxx` test cases.
*
* User memory access (see section [3. Accessing User Memory](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/project2.html#SEC50)). All system calls need to read user memory. Few system calls need to write to user memory.
* System call infrastructure (see section [4. System Calls](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/project2.html#SEC56)). Implement enough code to read the system call number from the user stack and dispatch to a handler based on it.
* The `exit` system call. Every user program that finishes in the normal way calls `exit`. Even a program that returns from `main()` calls `exit` indirectly (see `_start()` in lib/user/entry.c).
* The `write` system call for writing to fd 1, the system console. All of our test programs write to the console (the user process version of `printf()` is implemented this way), so they will all malfunction until `write` is available.
* For now, change `process_wait()` to an infinite loop (one that waits forever). The provided implementation returns immediately, so Pintos will power off before any processes actually get to run. You will eventually need to provide a correct implementation.

After the above are implemented, user processes should work minimally. At the very least, they can write to the console and exit correctly. You can then refine your implementation so that some of the tests start to pass.
