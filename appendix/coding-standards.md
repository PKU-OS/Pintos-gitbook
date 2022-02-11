# Coding Standards

Our standards for coding are most important for grading. We want to stress that aside from the fact that we are explicitly basing part of your grade on these things, good coding practices will improve the quality of your code. This makes it easier for your partners to interact with it, and ultimately, will improve your chances of having a good working program. That said once, the rest of this document will discuss only the ways in which our coding standards will affect our grading.

## Style

Style, for the purposes of our grading, refers to how readable your code is. At minimum, this means that your code is well formatted, your variable names are descriptive and your functions are decomposed and well commented. Any other factors which make it hard (or easy) for us to read or use your code will be reflected in your style grade.

The existing Pintos code is written in the GNU style and largely follows the [GNU Coding Standards](http://www.gnu.org/prep/standards\_toc.html). We encourage you to follow the applicable parts of them too, especially chapter 5, "Making the Best Use of C." Using a different style won't cause actual problems, but it's ugly to see gratuitous differences in style from one function to another. If your code is too ugly, it will cost you points.

Please limit C source file lines to at most 79 characters long.

Pintos comments sometimes refer to external standards or specifications by writing a name inside square brackets, like this: `[IA32-v3a]`. These names refer to the reference names used in this documentation (see section [Bibliography](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_14.html#SEC176)).

If you remove existing Pintos code, please delete it from your source file entirely. Don't just put it into a comment or a conditional compilation directive, because that makes the resulting code hard to read.

We're only going to do a compile in the directory for the project being submitted. You don't need to make sure that the previous projects also compile.

Project code should be written so that all of the subproblems for the project function together, that is, without the need to rebuild with different macros defined, etc. If you do extra credit work that changes normal Pintos behavior so as to interfere with grading, then you must implement it so that it only acts that way when given a special command-line option of the form -name, where name is a name of your choice. You can add such an option by modifying `parse_options()` in threads/init.c.

The introduction describes additional coding style requirements (see section [1.2.2 Design](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_1.html#SEC9)).

## C99

The Pintos source code uses a few features of the "C99" standard library that were not in the original 1989 standard for C. Many programmers are unaware of these feature, so we will describe them. The new features used in Pintos are mostly in new headers:

### **\<stdbool.h>**

Defines macros `bool`, a 1-bit type that takes on only the values 0 and 1, `true`, which expands to 1, and `false`, which expands to 0.

### \<stdint.h>

On systems that support them, this header defines types `intn_t` and `uintn_t` for n = 8, 16, 32, 64, and possibly other values. These are 2's complement signed and unsigned types, respectively, with the given number of bits.

On systems where it is possible, this header also defines types `intptr_t` and `uintptr_t`, which are integer types big enough to hold a pointer.

On all systems, this header defines types `intmax_t` and `uintmax_t`, which are the system's signed and unsigned integer types with the widest ranges.

For every signed integer type `type_t` defined here, as well as for `ptrdiff_t` defined in \<stddef.h>, this header also defines macros `TYPE_MAX` and `TYPE_MIN` that give the type's range. Similarly, for every unsigned integer type `type_t` defined here, as well as for `size_t` defined in \<stddef.h>, this header defines a `TYPE_MAX` macro giving its maximum value.

### \<inttypes.h>

\<stdint.h> provides no straightforward way to format the types it defines with `printf()` and related functions. This header provides macros to help with that. For every `intn_t` defined by \<stdint.h>, it provides macros `PRIdn` and `PRIin` for formatting values of that type with `"%d"` and `"%i"`. Similarly, for every `uintn_t`, it provides `PRIon`, `PRIun`, `PRIux`, and `PRIuX`.

You use these something like this, taking advantage of the fact that the C compiler concatenates adjacent string literals:

```
#include <inttypes.h>
...
int32_t value = ...;
printf ("value=%08"PRId32"\n", value);
```

The % is not supplied by the `PRI` macros. As shown above, you supply it yourself and follow it by any flags, field width, etc.

### \<stdio.h>

The `printf()` function has some new type modifiers for printing standard types:

#### j

For `intmax_t` (e.g. %jd) or `uintmax_t` (e.g. %ju).

#### z

For `size_t` (e.g. %zu).

#### t

For `ptrdiff_t` (e.g. %td).

Pintos `printf()` also implements a nonstandard ' flag that groups large numbers with commas to make them easier to read.

## Unsafe String Functions

A few of the string functions declared in the standard \<string.h> and \<stdio.h> headers are notoriously unsafe. The worst offenders are intentionally not included in the Pintos C library:

### `strcpy`

When used carelessly this function can overflow the buffer reserved for its output string. Use `strlcpy()` instead. Refer to comments in its source code in `lib/string.c` for documentation.

### `strncpy`

This function can leave its destination buffer without a null string terminator. It also has performance problems. Again, use `strlcpy()`.

### `strcat`

Same issue as `strcpy()`. Use `strlcat()` instead. Again, refer to comments in its source code in `lib/string.c` for documentation.

### `strncat`

The meaning of its buffer size argument is surprising. Again, use `strlcat()`.

### `strtok`

Uses global data, so it is unsafe in threaded programs such as kernels. Use `strtok_r()` instead, and see its source code in `lib/string.c` for documentation and an example.

### `sprintf`

Same issue as `strcpy()`. Use `snprintf()` instead. Refer to comments in `lib/stdio.h` for documentation.

### `vsprintf`

Same issue as `strcpy()`. Use `vsnprintf()` instead.

If you try to use any of these functions, the error message will give you a hint by referring to an identifier like `dont_use_sprintf_use_snprintf`.
