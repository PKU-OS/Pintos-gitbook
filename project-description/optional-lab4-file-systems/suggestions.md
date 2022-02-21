# Suggestions

## Suggested Order of Implementation

To make your job easier, we suggest implementing the parts of this project in the following order:

1. Buffer cache (see section [3. Buffer Cache](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/project4.html#SEC95)). Implement the buffer cache and integrate it into the existing file system. At this point all the tests from project 2 (and project 3, if you're building on it) should still pass.
2. Extensible files (see section [1. Indexed and Extensible Files](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/project4.html#SEC93)). After this step, your project should pass the file growth tests.
3. Subdirectories (see section [2. Subdirectories](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/project4.html#SEC94)). Afterward, your project should pass the directory tests.
4. Remaining miscellaneous items.

You can implement extensible files and subdirectories in parallel if you temporarily make the number of entries in new directories fixed.

You should think about synchronization throughout.
