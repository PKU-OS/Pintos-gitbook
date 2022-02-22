# C Standards

## C99

**The Pintos source code uses a few features of the "C99" standard library that were not in the original 1989 standard for C.** Many programmers are unaware of these features, so we will describe them. **The new features used in Pintos are mostly in new headers:**

* <mark style="color:blue;">**\<stdbool.h>**</mark>
  * **Defines macros `bool`**, a 1-bit type that takes on only the values 0 and 1, `true`, which expands to 1, and `false`, which expands to 0.
* <mark style="color:blue;">**\<stdint.h>**</mark>
  * On systems that support them, **this header defines types `intn_t` and `uintn_t` for n = 8, 16, 32, 64, and possibly other values.** These are 2's complement signed and unsigned types, respectively, with the given number of bits.
  * On systems where it is possible, **this header also defines types `intptr_t` and `uintptr_t`**, which are integer types **big enough** to hold a pointer.
  * On all systems, **this header defines types `intmax_t` and `uintmax_t`**, which are the system's signed and unsigned integer types with the **widest** ranges.
  * For every **signed** integer type `type_t` defined here, as well as for `ptrdiff_t` defined in `<stddef.h>`, **this header also defines macros `TYPE_MAX` and `TYPE_MIN` that give the type's range**.&#x20;
  * Similarly, for every **unsigned** integer type `type_t` defined here, as well as for `size_t` defined in `<stddef.h>`, **this header defines a `TYPE_MAX` macro giving its maximum value**.
* <mark style="color:blue;">**\<inttypes.h>**</mark>
  * **`<stdint.h>` provides no straightforward way to format the types it defines with `printf()` and related functions. This header provides macros to help with that.**&#x20;
  * For every **`intn_t`** defined by `<stdint.h>`, it provides macros **`PRIdn`** and **`PRIin`** for formatting values of that type with **`"%d"`** and **`"%i"`**.&#x20;
  * Similarly, for every **`uintn_t`**, it provides **`PRIon`**, **`PRIun`**, **`PRIux`**, and **`PRIuX`**.

You use these something like this, taking advantage of the fact that the C compiler concatenates adjacent string literals:

```
#include <inttypes.h>
...
int32_t value = ...;
printf ("value=%08"PRId32"\n", value);
```

The % is not supplied by the `PRI` macros. As shown above, you supply it yourself and follow it by any flags, field width, etc.

* <mark style="color:blue;">**\<stdio.h>**</mark>
  * **The `printf()` function has some new type modifiers for printing standard types:**
    * **j**
      * For `intmax_t` (e.g. %jd) or `uintmax_t` (e.g. %ju).
    * **z**
      * For `size_t` (e.g. %zu).
    * **t**
      * For `ptrdiff_t` (e.g. %td).
  * **Pintos `printf()` also implements a nonstandard " ' " flag** that groups large numbers with commas to make them easier to read.

## Unsafe String Functions

**A few of the string functions declared in the standard `<string.h>` and `<stdio.h>` headers are notoriously unsafe.** The worst offenders are intentionally not included in the Pintos C library (this is completely different from the standard C library, you can see this [FAQ](../project-description/lab0-booting/faq.md#kernel-monitor-faq) for details):

* <mark style="color:blue;">**strcpy**</mark>
  * When used carelessly this function **can overflow the buffer** reserved for its output string.
  * Use **`strlcpy()`** instead. Refer to comments in its source code in `lib/string.c` for documentation.
* <mark style="color:blue;">**strncpy**</mark>
  * This function **can leave its destination buffer without a null string terminator**. It also has **performance problems**.&#x20;
  * Again, use **`strlcpy()`**.
* <mark style="color:blue;">**strcat**</mark>
  * **Same issue as `strcpy()`**.&#x20;
  * Use **`strlcat()`** instead. Again, refer to comments in its source code in `lib/string.c` for documentation.
* <mark style="color:blue;">**strncat**</mark>
  * The meaning of its buffer size argument is surprising.&#x20;
  * Again, use **`strlcat()`**.
* <mark style="color:blue;">**strtok**</mark>
  * Uses global data, so it is **unsafe in threaded programs** such as kernels.&#x20;
  * Use **`strtok_r()`** instead, and see its source code in `lib/string.c` for documentation and an example.
* <mark style="color:blue;">**sprintf**</mark>
  * **Same issue as `strcpy()`**.&#x20;
  * Use **`snprintf()`** instead. Refer to comments in `lib/stdio.h` for documentation.
* <mark style="color:blue;">**vsprintf**</mark>
  * **Same issue as `strcpy()`**.&#x20;
  * Use **`vsnprintf()`** instead.

If you try to use any of these functions, **the error message will give you a hint** by referring to an identifier like `dont_use_sprintf_use_snprintf`.
