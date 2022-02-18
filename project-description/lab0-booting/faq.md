# FAQ

## Kernel Monitor FAQ

**I've added `#include<stdlib.h>` in the source file, but why the compiler still gives me warning when using `malloc`?**

That is the typical first "culture shock" doing kernel programming—Welcome to The Matrix!

Basically, you cannot use functions like `gets` from what you would use in your regular C program. Pintos make it less shocking (and your life easier) by providing a set of routines that mimics common standard C library functions like `printf`. But their implementations are not really exactly the same because standard C library works in the user mode and usually leverages syscalls to implement the functions while the lib in pintos is at kernel-level. Also, this is a Pintos-only choice. In other kernels or real OSes, you wouldn't even be able to see function names like `printf`. Instead, they are named differently to avoid confusion. For example, in Linux kernel, to print something, you will need to use `printk` because `printf` is not available in kernel.

For `malloc`, it is the same story. Pintos does provide the `malloc` routine. But its implementation is different from the `malloc` in standard C library. It's just named so to make you feel more comfortable. The `malloc` kernel routine is defined in `threads/malloc.h`. So you will have to include it to use this header file instead.

**Wait, then how does the compiler know it should include the `stdlib.h` in the Pintos codebase (`pintos/src/lib`), instead of the standard one at `/usr/include/stdlib.h`?**

Good question. A `#include<XXX.h>` directive will indeed search for system headers instead of a local header file. However, as explained earlier: (1) we should not use the `/usr/include/stdlib.h` because later the kernel and standard C library are not operating in the same level and linking them together later won't work; (2) we must use the `pintos/src/lib/stdlib.h`. How to achieve this? The magic happens in the GCC flags.

For (1), we need to use a flag called `-nostdinc` to tell GCC to not use the standard C headers in its header search path. For (2), we need to use the `-I/path/to/my/headers` flag to tell GCC to treat `/path/to/my/headers` in its header file search path. Once both are done, you can use `#include<stdlib.h>` in a pintos source file.

You can find this trick is done for compiling all pintos source files in the `CPPFLAGS` in `Make.config`. Again, this system header include format is just the pintos author's attempt to make programming pintos as familiar to regular C programming as possible for you. You will need to be conscious that there is some good-will smoke and mirror here :)

**I tried to use `scanf` but got a ``undefined reference to `scanf' error``.**

This is for the same reason explained in the `malloc` question. Pintos only implements a subset of C standard library functions. You can check the header files in `src/lib` to see what are these available functions. If a standard function is not listed there, you cannot use it (and will have to implement it yourself if you really need it).
