# Pintos World

{% hint style="info" %}
Let's dive into the Pintos World!
{% endhint %}

## Code Guidance

Now you've had a taste of the booting process of Pintos. Let's take a closer look at what's inside. Here's the directory structure that you should see in pintos/src:

* **"threads/":** Source code for the base kernel, which you will modify starting in project 1.
* **"userprog/":** Source code for the user program loader, which you will modify starting with project 2.
* **"vm/":** An almost empty directory. You will implement virtual memory here in project 3.
* **"filesys/":** Source code for a basic file system. You will use this file system starting with project 2, but you will not modify it until project 4.
* **"devices/":** Source code for I/O device interfacing: keyboard, timer, disk, etc. You will modify the timer implementation in project 1. Otherwise, you should have no need to change this code.
* **"lib/":** An implementation of a subset of the standard C library. The code in this directory is compiled into both the Pintos kernel and, starting from project 2, user programs that run under it. In both kernel code and user programs, headers in this directory can be included using the `#include <...>` notation. You should have little need to modify this code.
* **"lib/kernel/":** Parts of the C library that are included only in the Pintos kernel. This also includes implementations of some data types that you are free to use in your kernel code: `bitmaps`, `doubly linked lists`, and `hash tables`. In the kernel, headers in this directory can be included using the `#include <...>` notation.
* **"lib/user/":** Parts of the C library that are included only in Pintos user programs. In user programs, headers in this directory can be included using the `#include <...>` notation.
* **"tests/":** Tests for each project. You can modify this code if it helps you test your submission, but we will replace it with the originals before we run the tests.
* **"examples/":** Example user programs for use starting with project 2.
* **"misc/" and "utils/"**: These files may come in handy if you decide to try working with Pintos on your own machine. Otherwise, you can ignore them.

{% hint style="info" %}
You may feel overwhelmed by so many source files and have no idea where to start. Don't worry, now you only need to have a big picture in mind and your kind TA will guide you through the source files in each Lab section.
{% endhint %}

If you want to understand the dependencies of these modules and functions in these modules, you can generate the code documentation with Doxygen. We provide an online code browser based on Doxygen at [https://pku-os.github.io/pintos-doxygen/html/](https://pku-os.github.io/pintos-doxygen/html/).

## Building Pintos

{% hint style="info" %}
There is one Makefile under each lab directory (threads/, userprog/, vm/, filesys/).  When you are working on a specific lab, you should build Pintos under its corresponding directory.&#x20;

Lab0 will use the same directory as Lab1, i.e., threads/. &#x20;
{% endhint %}

{% hint style="info" %}
If you have a look at the content of these Makefiles, you may be surprised that they are the same! So what's the difference when you run `make` under different lab directories? \
(hint: Make.vars)
{% endhint %}

Now let's build the source code supplied for the first project.&#x20;

{% hint style="info" %}
If you set up your environment using docker, we assume that you are always in the container environment with the Pintos directory mounted from now on.
{% endhint %}

First, `cd` into the `threads/` directory. Then, issue the `make` command. This will create a build directory under `threads/`, populate it with a Makefile and a few subdirectories, and then build the kernel inside. The entire build should take less than 30 seconds.

After building, the following are the interesting files in the build directory:

* **"Makefile":** A copy of `pintos/src/Makefile.build`. It describes how to build the kernel.
* **"kernel.o":** Object file for the entire kernel. This is the result of linking object files compiled from each individual kernel source file into a single object file. It contains debug information, so you can run GDB or `backtrace` (see section [Debug and Test](debug-and-test/)) on it.
* **"kernel.bin":** Memory image of the kernel, that is, the exact bytes loaded into memory to run the Pintos kernel. This is just `kernel.o` with debug information stripped out, which saves a lot of space, which in turn keeps the kernel from bumping up against a 512 kB size limit imposed by the kernel loader's design.
* **"loader.bin":** Memory image for the kernel loader, a small chunk of code written in assembly language that reads the kernel from disk into memory and starts it up. It is exactly 512 bytes long, a size fixed by the PC BIOS.

Subdirectories of build contain object files (.o) and dependency files (.d), both produced by the compiler. The dependency files tell `make` which source files need to be recompiled when other source or header files are changed.

## Running Pintos

### _"pintos"_ utility

We've supplied a program for conveniently running Pintos in a simulator (Qemu or Bochs), called `pintos`. In the simplest case, you can invoke `pintos` as `pintos argument...`. Each argument is passed to the Pintos kernel for it to act on.

Try it out! First `cd` into the newly created build directory. Then issue the command `pintos -- run alarm-multiple`, which passes the arguments `run alarm-multiple` to the Pintos kernel. In these arguments, `run` instructs the kernel to run a test and `alarm-multiple` is the test to run.

This command invokes Qemu. Then Pintos boots and runs the `alarm-multiple` test program, which outputs a few lines of text. When it's done, you can close Qemu by <mark style="color:red;">`Ctrl+a+C`</mark> .

You can log the output to a file by redirecting at the command line, e.g. `pintos run alarm-multiple > logfile`.

### options&#x20;

The `pintos` program offers several options for configuring the simulator or the virtual hardware. If you specify any options, they must precede the arguments passed to the Pintos kernel and be separated from them by `--`, so that the whole command looks like:

`pintos option1 option2 ... -- arg1 arg2 ...`.&#x20;

{% hint style="info" %}
You may be confused by the strange "--" at first. So I will explain it to you again. The options specified before "--" are used to config the pintos program. While the arguments that specified after "--" are the real actions you expect pintos to execute.
{% endhint %}

You can invoke `pintos -h` to see a list of available options. Options can select a simulator to use: the default is Qemu, but `--bochs` selects Bochs. You can run the simulator with a debugger. You can set the amount of memory to give the VM. Finally, you can select how you want VM output to be displayed: use -v to turn off the VGA display, -t to use your terminal window as the VGA display instead of opening a new window (Bochs only), or -s to suppress serial input from `stdin` and output to `stdout`.

{% hint style="success" %}
The pintos program is heavily used by our testing suites, so it is fully configurable and very flexible. You certainly do not need to remember all these options and you can always refer to it by running "pintos -h".
{% endhint %}
