# Suggestions

## Suggested Order of Implementation

We suggest the following initial order of implementation:

* **Frame table** (see section [B.5 Managing the Frame Table](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/project3a.html#SEC74)).&#x20;
  * Change `process.c` to use your frame table allocator.&#x20;
  * **Do not implement swapping yet.** If you run out of frames, fail the allocator or panic the kernel.&#x20;
  * After this step, your kernel should still pass all the project 2 test cases.
* **Supplemental page table and page fault handler** (see section [B.4 Managing the Supplemental Page Table](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/project3a.html#SEC73)).&#x20;
  * Change `process.c` to record the necessary information in the supplemental page table when loading an executable and setting up its stack.&#x20;
  * Implement loading of code and data segments in the page fault handler.&#x20;
  * For now, consider only valid accesses.
  * After this step, your kernel should pass all of the project 2 functionality test cases, but only some of the robustness tests.
* From here, you can **implement page reclamation on process exit**.
* The next step is to **implement eviction** (see section [B.5 Managing the Frame Table](https://www.cs.jhu.edu/\~huang/cs318/fall21/project/project3a.html#SEC74)).&#x20;
  * Initially you could choose the page to evict randomly.&#x20;
  * At this point, you need to consider how to manage accessed and dirty bits and aliasing of user and kernel pages.&#x20;
  * **Synchronization** is also a concern: how do you deal with it if process A faults on a page whose frame process B is in the process of evicting?
  * Finally, implement a eviction strategy such as the clock algorithm.
