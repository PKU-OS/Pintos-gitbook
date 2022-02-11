# Development Tools

Here are some tools that you might find useful while developing code.

## F.1 Tags

Tags are an index to the functions and global variables declared in a program. Many editors, including Emacs and `vi`, can use them. The Makefile in pintos/src produces Emacs-style tags with the command `make TAGS` or `vi`-style tags with `make tags`.

In Emacs, use M-. to follow a tag in the current window, C-x 4 . in a new window, or C-x 5 . in a new frame. If your cursor is on a symbol name for any of those commands, it becomes the default target. If a tag name has multiple definitions, M-0 M-. jumps to the next one. To jump back to where you were before you followed the last tag, use M-\*.

## F.2 cscope

The `cscope` program also provides an index to functions and variables declared in a program. It has some features that tag facilities lack. Most notably, it can find all the points in a program at which a given function is called.

The Makefile in pintos/src produces `cscope` indexes when it is invoked as `make cscope`. Once the index has been generated, run `cscope` from a shell command line; no command-line arguments are normally necessary. Then use the arrow keys to choose one of the search criteria listed near the bottom of the terminal, type in an identifier, and hit Enter. `cscope` will then display the matches in the upper part of the terminal. You may use the arrow keys to choose a particular match; if you then hit Enter, `cscope` will invoke the default system editor[(9)](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/pintos\_fot.html#FOOT9) and position the cursor on that match. To start a new search, type Tab. To exit `cscope`, type Ctrl-d.

Emacs and some versions of `vi` have their own interfaces to `cscope`. For information on how to use these interface, visit [http://cscope.sourceforge.net, the `cscope` home page](http://cscope.sourceforge.net%2C%20the%20%3Ccode%3Ecscope%3C/CODE%3E%20homepage).

## F.3 Git

It's crucial that you use a source code control system to manage your Pintos code. This will allow you to keep track of your changes and coordinate changes made by different people in the project. For this class we recommend that you use Git; if you followed the instructions on getting started, a Git repository will already have been created for you. If you don't already know how to use Git, we recommend that you read the [Pro Git](http://git-scm.com/book) book online.

## F.4 VNC

VNC stands for Virtual Network Computing. It is, in essence, a remote display system which allows you to view a computing "desktop" environment not only on the machine where it is running, but from anywhere on the Internet and from a wide variety of machine architectures. It is already installed on the lab machines. For more information, look at the [VNC Home Page](http://www.realvnc.com).
