# Running Pintos

## The _"pintos"_ Utility

**We've supplied a program for conveniently running Pintos in a simulator (Qemu or Bochs), called `pintos`.** In the simplest case, you can invoke `pintos` as `pintos [argument...]`. Each argument is passed to the Pintos kernel for it to act on.

Try it out!&#x20;

* First `cd` into the newly created build directory.&#x20;
* Then issue the command `pintos -- run alarm-multiple`, which passes the arguments `run alarm-multiple` to the Pintos kernel.&#x20;
  * In these arguments, `run` instructs the kernel to run a test and `alarm-multiple` is the test to run.&#x20;
  * This command invokes Qemu. Then Pintos boots and runs the `alarm-multiple` test program, outputing a few lines of text. When it's done, you can close Qemu by <mark style="color:red;">`Ctrl+a+c`</mark> .
  * You can log the output to a file by redirecting at the command line, e.g. `pintos run alarm-multiple > logfile`.

### options

**The `pintos` program offers several options for configuring the simulator or the virtual hardware.** <mark style="color:red;">**If you specify any options, they must precede the arguments passed to the Pintos kernel and be separated from them by**</mark><mark style="color:red;">** **</mark><mark style="color:red;">**`--`**</mark>, so that the whole command looks like:

```
pintos option1 option2 ... -- arg1 arg2 ....
```

{% hint style="info" %}
You may be confused by the strange "--" at first, thus we will shed more light on it here.&#x20;

* The options specified **before** "--" are used to **config the pintos program**.&#x20;
* While the arguments that specified **after** "--" are **the real actions you expect pintos to execute**.
{% endhint %}

**You can invoke `pintos -h` to see a list of available options.**&#x20;

* Options can select a simulator to use: the default is Qemu, but `--bochs` selects Bochs.&#x20;
* You can run the simulator with a debugger. Just select `--gdb` option.
* You can set the amount of memory to give the VM with option `-m`.&#x20;
* Finally, you can select how you want VM output to be displayed: use `-v` to turn off the VGA display, `-t` to use your terminal window as the VGA display instead of opening a new window (Bochs only), or `-s` to suppress serial input from `stdin` and output to `stdout`.

{% hint style="success" %}
The pintos utility program is heavily used by our testing suites, so it is fully configurable and very flexible. You certainly do not need to remember all these options and you can always refer to it by running the command "pintos -h".
{% endhint %}
