# (Optional) Lab4: File Systems

**Note: Lab 4 is not** **required all students**. Students who are interested in understanding the internals of file systems in depth can optionally complete this lab and receive extra credits (a max of 30% of the project grade).

In the previous two assignments, you made extensive use of a file system without actually worrying about how it was implemented underneath. For this last assignment, you will improve the implementation of the file system. You will be working primarily in the `filesys` directory.

You may build project 4 on top of project 2 or project 3. In either case, all of the functionality needed for project 2 must work in your filesys submission. If you build on project 3, then all of the project 3 functionality must work also, and you will need to edit `filesys/Make.vars` to enable VM functionality. You can receive up to 10% extra credit if you do enable VM. We ask that you hand in your code for this lab in a branch called lab4-handin. Depending on which project you plan to build on, create this branch with `git checkout -b lab4-handin lab2-handin` or `git checkout -b lab4-handin lab3-handin`.

##
