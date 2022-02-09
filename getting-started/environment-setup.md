# Environment Setup

{% hint style="info" %}
To develop the Pintos projects, you’ll need two essential sets of tools:

* 80x86 cross-compiler toolchain for 32-bit architecture including a C compiler, assembler, linker, and debugger.
* x86 emulator: QEMU or Bochs
{% endhint %}

## Option A(recommended): Docker

{% hint style="success" %}
The easiest way to set up the development environment is Docker. Your kind TA has built a docker image for you in advance that contains all the toolchains to compile, run and debug Pintos. This docker image has been tested on Mac (intel chip), Mac (apple chip), Windows, and Linux.
{% endhint %}

### Download

First, you need to install docker on your laptop. Go to the [docker download page](https://www.docker.com/get-started) for help.&#x20;

Then pull the docker image and run it, just type the command below into your favorite shell (you can run `docker run --help` to find out what this command means in detail):

```
docker run -it pkuflyingpig/pintos bash
```

This image is about 3GB (it contains a full Ubuntu18.04), so it may take some time at its first run.

If everything goes well, you will enter a bash shell.&#x20;

Type `pwd` you will find your home directory is under `/home/PKUOS` .

Type `ls` you will find that there is a `toolchain` directory that contains all the dependencies.&#x20;

Now you own a tiny Ubuntu OS inside your host computer, and you can shut it down easily by `Ctrl+d`. You can check that it has exited by running `docker ps -a`.

### Boot Pintos

Now go back to your host machine, git clone the Pintos skeleton code repository by running:

```
git clone git@github.com:PKU-OS/pintos.git
```

{% hint style="warning" %}
_Note: we have made some customization to the official Pintos distribution. So you should be only getting the source code from the above channels. In other words, do not download from other websites._
{% endhint %}

Then run the docker image again but this time mount your `path/to/pintos` into the container.

{% hint style="warning" %}
_Note: Do not just copy and paste! You need to replace the absolute path to your pintos directory in the command below!_
{% endhint %}

```
docker run -it --rm --name pintos --mount type=bind,source=absolute/path/to/pintos/on/your/host/machine,target=/home/PKUOS/pintos pkuflyingpig/pintos bash
```

{% hint style="info" %}
`Docker notes:`

* `-rm` tells docker to delete the container after running
* `--name pintos` names the container as `pintos`, this will be helpful in the debugging part.
{% endhint %}

{% hint style="info" %}
You may use this command throughout this semester, and it is tedious to remember and type again and again. You can use the `alias` utility to save this command as `pintos-up` for example.&#x20;
{% endhint %}

Now when you `ls`, you will find there is a new directory called `pintos` under your home directory in the container, it is shared by the container and your host laptop.

Now here comes the exciting part, type the following commands in turn:

```
cd pintos/src/threads/
make
cd build
pintos --
```

Bomb! The last command will trigger Qemu to simulate a 32-bit x86 machine and boot your Pintos on it. If you see something like:

```
Pintos hda1
Loading............
Kernel command line:
Pintos booting with 3,968 kB RAM...
367 pages available in kernel pool.
367 pages available in user pool.
Calibrating timer...  32,716,800 loops/s.
Boot complete.
```

Your Pintos has been booted successfully, congratulations :)

You can shut down the Qemu by `Ctrl+a+x` or `Ctrl+c` if the previous one does not work.

{% hint style="success" %}
Now Pintos will exit immediately after booting, so you may not see much useful information in the output. Don't be disappointed.
{% endhint %}

{% hint style="info" %}
If you want to understand the internal details of the docker image, you can look through the [dockerfile](https://github.com/PKU-OS/Pintos-dockerfile/blob/main/dockerfile) on Github.
{% endhint %}

### What's the magic?

Now Let's conclude what you have done. First, You used docker to run a Ubuntu container that functions as a full-edged Linux OS inside your host OS. Then you used Qemu to simulate a 32-bit x86 computer inside your container. Finally, you boot a tiny toy OS -- Pintos on the computer which Qemu simulates. Wow, virtualization is amazing, right?

Throughout this semester, you can modify your Pintos source code in your host machine with your favorite IDE but compile/run/debug/test your Pintos in the container. You can leave the container running when you are modifying Pintos, because your modification will be visible in the container immediately.

## Option B: Virtual Machine

{% hint style="warning" %}
This method is not fully tested by your TAs, so we may not provide much help if you encounter some strange errors.
{% endhint %}

If you would like to use a VM for development, there is a [VirtualBox](https://www.virtualbox.org) VM provided by Johns Hopkins Univerisity that runs Ubuntu 18.04 with the necessary toolchain installed. You can download the image [here](https://bit.ly/3j9Elp4). The image is large (2.5 GB), so the download can take a while. The md5sum for the VM image is `69c89938d4b768bdcca4362fd39f06e4`. The initial login password is `jhucs318`.

When you enter the VM successfully, you can follow the instruction [here](environment-setup.md#boot-pintos) to boot Pintos. You should ignore all the instructions related to Docker.
