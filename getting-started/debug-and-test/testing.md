# Testing

## Run all tests

Each project has several tests. To completely test your submission, invoke `make check` from the project build directory. This will build and run each test and print a "pass" or "fail" message for each one. When a test fails, `make check` also prints some details of the reason for failure. After running all the tests, `make check` also prints a summary of the test results.

For project 1, the tests will probably run faster in Bochs. For the rest of the projects, they will run much faster in QEMU. `make check` will select the faster simulator by default, but you can override its choice by specifying `SIMULATOR=--bochs` or `SIMULATOR=--qemu` on the `make check` command line.

## Run individual test

You can also run individual tests one at a time. A given test **t** writes its output to `t.output`, then a script scores the output as "pass" or "fail" and writes the result to `t.result`.&#x20;

To run and grade a single test, `make` the `.result` file explicitly from the build directory, e.g. `make tests/threads/alarm-multiple.result`. If `make` says that the test result is up-to-date, but you want to re-run it anyway, either delete the `.output` file by hand, e.g., `rm tests/threads/alarm-multiple.output`, or run `make clean` to delete all build and test output files.

## Special options

By default, each test provides feedback only at completion, not during its run. If you prefer, you can observe the progress of each test by specifying VERBOSE=1 on the `make` command line, as in `make check VERBOSE=1`. You can also provide arbitrary options to the `pintos` run by the tests with PINTOSOPTS='...', e.g. `make check PINTOSOPTS='-j 1'` to select a jitter value of 1 (see section [Debugging versus Testing](testing.md#debugging-versus-testing)).

{% hint style="info" %}
All of the tests and related files are in pintos/src/tests. Before we test your submission, we will replace the contents of that directory by a pristine, unmodified copy, to ensure that the correct tests are used. Thus, you can modify some of the tests if that helps in debugging, but we will run the originals.
{% endhint %}

{% hint style="info" %}
All software has bugs, so some of our tests may be flawed. If you think a test failure is a bug in the test, not a bug in your code, please point it out. We will look at it and fix it if necessary. However, there is a much higher possibility that the test failure is caused by the bug in your code.
{% endhint %}

{% hint style="warning" %}
Please don't try to take advantage of our generosity in giving out our test suite. Your code has to work properly in the general case, not just for the test cases we supply. For example, it would be unacceptable to explicitly base the kernel's behavior on the name of the running test case. Such attempts to side-step the test cases will receive no credit. If you think your solution may be in a gray area here, please ask us about it.
{% endhint %}

##
