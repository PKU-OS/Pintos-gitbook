# Suggestions

## Suggested Order of Implementation

To make your job easier, we suggest implementing the parts of this project in the following order:

1. **Buffer cache** (see section [Buffer Cache](your-tasks.md#task-3-buffer-cache))**.**&#x20;
   * Implement the buffer cache and integrate it into the existing file system.&#x20;
   * At this point all the tests from project 2 (and project 3, if you're building on it) should still pass.
2. **Extensible files** (see section [Indexed and Extensible Files](your-tasks.md#task-1-indexed-and-extensible-files))**.**&#x20;
   * After this step, your project should pass the file growth tests.
3. **Subdirectories** (see section [Subdirectories](your-tasks.md#task-2-subdirectories))**.**&#x20;
   * Afterward, your project should pass the directory tests.
4. **Remaining miscellaneous items.**

You can implement extensible files and subdirectories in parallel if you temporarily make the number of entries in new directories fixed.

You should think about <mark style="color:red;">**synchronization**</mark> throughout.
