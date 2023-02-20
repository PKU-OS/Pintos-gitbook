# Suggestions

## Suggested Order of Implementation

We suggest first implementing the following, which can happen in parallel:

* **Argument passing.** Every user program will page fault immediately until argument passing is implemented. If you implement it correctly, you should pass all the `args-xxx` test cases.
* **`halt` system call.** This is the easiest system call to implement, but it is a big step to have a working system call infrastructure. You should implement enough code to read the system call number from the user stack and dispatch it to a handler based on it.
* **The `exit` system call.** Every user program that finishes in the normal way calls `exit`. Even a program that returns from `main()` calls `exit` indirectly (see `_start()` in `lib/user/entry.c`).
* **The `write` system call for writing to fd 1, the system console.** <mark style="color:red;">**All of our test programs write to the console**</mark> (the user process version of `printf()` is implemented this way), so they will all malfunction until `write` is available.
* <mark style="color:red;">**For now, change**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**`process_wait()`**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**to an infinite loop (one that waits forever).**</mark>** ** The provided implementation returns immediately, so Pintos will power off before any processes actually get to run. <mark style="color:red;">**You will eventually need to provide a correct implementation.**</mark>
* After the above are implemented, user processes should work minimally. At the very least, they can write to the console and exit correctly. **You can then refine your implementation so that some of the tests start to pass**.
