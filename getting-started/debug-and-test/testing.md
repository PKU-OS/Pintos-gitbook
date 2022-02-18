# Testing

{% hint style="warning" %}
When working on a big project, it's a bad idea to write 1000 LOC blindly and then start to compile, hoping your code will pass all the tests magically (If you really make it, please let us know, you definitely should be the next Linus Torvalds!). Instead, test-driven development should be your choice. **We will elaborate on this during the TA tutorial section.**
{% endhint %}

## File structure

Under `src/tests`, each lab (except for lab 0) has its corresponding test directory:

* <mark style="color:blue;">**"src/tests/threads"**</mark>
* <mark style="color:blue;">**"src/tests/userprog"**</mark>
* <mark style="color:blue;">**"src/tests/vm"**</mark>
* <mark style="color:blue;">**"src/tests/filesys"**</mark>

{% hint style="info" %}
Some labs (like lab3: virtual memory) may contain other labs' test cases, you can see the TEST\_SUBDIRS variable defined in its corresponding Make.vars file for the details (e.g. for lab3, the file is src/vm/Make.vars).
{% endhint %}

**Under each lab's test directory, the files can be classified as follows:**

* <mark style="color:blue;">**"foo.c"**</mark>
  * **C source code** for `foo` test case
* <mark style="color:blue;">**"foo.ck"**</mark>
  * Perl script for checking your output with the reference, you can find **the desired output** for `foo` test case in it.
* <mark style="color:blue;">**"Rubric.bar"**</mark>
  * `bar` is the name of the testing **set**, and the Rubric file defines which tests this set includes and their assigned scores.&#x20;
  * For example, `src/tests/threads/Rubric.alam` defines the `alarm` testing set which contains all the tests for "Functionality and robustness of alarm clock" :

```
Functionality and robustness of alarm clock:
4	alarm-single
4	alarm-multiple
4	alarm-simultaneous
4	alarm-priority

1	alarm-zero
1	alarm-negative
```

* <mark style="color:blue;">**"Grading"**</mark>
  * Defines the proportion of testing points of each testing set. For example, `src/tests/threads/Grading` :

```
# Percentage of the testing point total designated for each set of
# tests.

20.0%	tests/threads/Rubric.alarm
40.0%	tests/threads/Rubric.priority
40.0%	tests/threads/Rubric.mlfqs
```

See more in section[ Grading](../grading.md).

## Run all tests

**Each project has several tests.** **To completely test your implementation, invoke `make check` from the project build directory.**&#x20;

* This will build and run each test and **print a "pass" or "fail" message** for each one.&#x20;
* When a test fails, `make check` also **prints some details of the reason for failure**.&#x20;
* After running all the tests, `make check` also **prints a summary of the test results**.

{% hint style="info" %}
For **project 1**, the tests will probably run faster in **Bochs**.&#x20;

For **the rest of the projects**, they will run much faster in **QEMU**.

* `make check` will select the faster simulator by default, but you can override its choice by specifying `SIMULATOR=--bochs` or `SIMULATOR=--qemu` on the `make check` command line.
{% endhint %}

## Run an individual test

**You can also run an individual test at a time.**&#x20;

* A given test _foo_ writes its output to `foo.output`, then a script scores the output as "pass" or "fail" and writes the result to `foo.result`.

**To run and grade a single test, `make` the `.result` file explicitly **<mark style="color:red;">**from the build directory**</mark>**.**

* e.g. Type `make tests/threads/alarm-multiple.result` in the build directory to test the `alarm-multiple` case.&#x20;
* If `make` says that **the test result is up-to-date**, but you want to re-run it anyway
  * either delete the `.output` file by hand, e.g. `rm tests/threads/alarm-multiple.output`
  * or run `make clean` to delete all build and test output files.

## Special options

* By default, each test provides feedback only at completion, not during its run.  ****  If you prefer, you can **observe the progress of each test** by specifying `VERBOSE=1` on the `make` command line, as in `make check VERBOSE=1`.&#x20;
* You can also **provide arbitrary options** to the `pintos` run by the tests with `PINTOSOPTS='...'`
  * e.g. `make check PINTOSOPTS='-j 1'` to select a jitter value of 1 (see section [Debugging versus Testing](./#debugging-versus-testing)).

{% hint style="info" %}
**All of the tests and related files are in pintos/src/tests.**&#x20;

<mark style="color:red;">**Before we test your submission, we will replace the contents of that directory with a pristine, unmodified copy, to ensure that the correct tests are used.**</mark>&#x20;

Thus, you can modify some of the tests if that helps in debugging, but we will run and grade with the original.
{% endhint %}

{% hint style="info" %}
**All software has bugs, so some of our tests may be flawed.**&#x20;

If you think a test failure comes down to a bug in the test, rather than one in your code, please point it out. We will look at it and fix it if necessary. However, there is a much higher possibility that the test failure is caused by a bug (or many bugs) in your code.
{% endhint %}

{% hint style="warning" %}
**Please don't try to take advantage of our generosity in giving out the whole test suite.**

Your code has to work properly in the general case, not just for the test cases we supply. For example, it would be unacceptable to explicitly base the kernel's behavior on the name of the running test case. Such attempts to sidestep the test cases will receive no credit. If you think your solution may be in a gray area here, please ask us about it.
{% endhint %}
