# Interrupt Handling

**An&#x20;**_**interrupt**_**&#x20;notifies the CPU of some event.** Much of the work of an operating system relates to interrupts in one way or another. For our purposes, **we classify interrupts into two broad categories**:

1. _**Internal interrupts**_, that is, interrupts **caused** **directly by CPU instructions**.
   * System calls, attempts at invalid memory access (_page faults_), and attempts to divide by zero are some activities that cause internal interrupts.
   * Because they are caused by CPU instructions, internal interrupts are _**synchronous**_ or synchronized with CPU instructions. <mark style="color:red;">**`intr_disable()`**</mark> <mark style="color:red;">**does not disable internal interrupts.**</mark>
2. _**External interrupts**_, that is, interrupts **originating outside the CPU**.
   * These interrupts come from hardware devices such as the system timer, keyboard, serial ports, and disks.
   * External interrupts are _**asynchronous**_, meaning that their delivery is not synchronized with instruction execution. Handling of external interrupts can be postponed with **`intr_disable()`** and related functions (see section [Disabling Interrupts](synchronization.md#disabling-interrupts)).

**The CPU treats both classes of interrupts largely the same way, so Pintos has common infrastructure to handle both classes.**

* The following section describes this common **infrastructure**.
* The sections after that give the **specifics** of external and internal interrupts.

## Interrupt Infrastructure

**When an interrupt occurs, the CPU saves its most essential state on a stack and jumps to an interrupt handler routine.**

* The 80x86 architecture supports **256** interrupts, numbered 0 through 255, each with an independent handler defined in an array called the _interrupt descriptor table_ or **IDT**.

### **The Interrupt Handling Process**

1. **In Pintos, `intr_init()` in `threads/interrupt.c` sets up the IDT** so that each entry points to a unique entry point in `threads/intr-stubs.S` named `intrNN_stub()`, where NN is the interrupt number in hexadecimal.
2. Because the CPU doesn't give us any other way to find out the interrupt number, **this entry point pushes the interrupt number on the stack**. Then it **jumps to `intr_entry()`**, which **pushes all the registers** that the processor didn't already push for us, and then **calls `intr_handler()`**, which brings us back into C in `threads/interrupt.c`.
3. **The main job of `intr_handler()` is to call the function registered for handling the particular interrupt.** (If no function is registered, it dumps some information to the console and panics.) It also <mark style="color:red;">**does some extra processing for external interrupts**</mark> (see section [External Interrupt Handling](interrupt-handling.md#external-interrupt-handling)).
4. When `intr_handler()` returns, the assembly code in `threads/intr-stubs.S` **restores all the CPU registers saved earlier** and directs the CPU to **return from the interrupt**.

### Types and Functions

<details>

<summary>Types and Functions for Interrupt</summary>

* <mark style="color:blue;">**Type: void intr\_handler\_func (struct intr\_frame \*frame)**</mark>
  * **This is how an interrupt handler function must be declared.**
  * Its frame argument (see below) allows it to determine the **cause** of the interrupt and the **state** of the thread that was interrupted.
* <mark style="color:blue;">**Type: struct intr\_frame**</mark>
  * **The stack frame of an interrupt handler, as saved by the CPU, the interrupt stubs, and `intr_entry()`**. Its most interesting members are described below.
  * <mark style="color:orange;">**uint32\_t edi**</mark>
  * <mark style="color:orange;">**uint32\_t esi**</mark>
  * <mark style="color:orange;">**uint32\_t ebp**</mark>
  * <mark style="color:orange;">**uint32\_t esp\_dummy**</mark>
  * <mark style="color:orange;">**uint32\_t ebx**</mark>
  * <mark style="color:orange;">**uint32\_t edx**</mark>
  * <mark style="color:orange;">**uint32\_t ecx**</mark>
  * <mark style="color:orange;">**uint32\_t eax**</mark>
  * <mark style="color:orange;">**uint16\_t es**</mark>
  * <mark style="color:orange;">**uint16\_t ds**</mark>
    * **Register values in the interrupted thread**, pushed by `intr_entry()`. The `esp_dummy` value isn't actually used.
  * <mark style="color:orange;">**uint32\_t vec\_no**</mark>
    * **The interrupt vector number**, ranging from 0 to 255.
  * <mark style="color:orange;">**uint32\_t error\_code**</mark>
    * **The "error code"** pushed on the stack by the CPU for some internal interrupts.
  * <mark style="color:orange;">**void (\*eip) (void)**</mark>
    * **The address of&#x20;**<mark style="color:red;">**the next instruction to be executed**</mark>**&#x20;by the interrupted thread.**
  * <mark style="color:orange;">**void \*esp**</mark>
    * **The interrupted thread's stack pointer.**
* <mark style="color:blue;">**Function: const char \*intr\_name (uint8\_t vec)**</mark>
  * **Returns the name of the interrupt numbered vec, or `"unknown"` if the interrupt has no registered name.**

</details>

## Internal Interrupt Handling

**Internal interrupts are caused directly by CPU instructions executed by the running kernel thread or user process (from project 2 onward).** An internal interrupt is therefore said to arise in a <mark style="color:orange;">**"process context."**</mark>

* **In an internal interrupt's handler, it can make sense to examine the `struct intr_frame` passed to the interrupt handler, or even to modify it.** When the interrupt returns, modifications in `struct intr_frame` become changes to the calling thread or process's state. For example, the Pintos system call handler returns a value to the user program by modifying the saved EAX register.
* **There are no special restrictions on what an internal interrupt handler can or can't do.** <mark style="color:red;">**Generally they should run with interrupts enabled**</mark>, just like other code, and so they can be preempted by other kernel threads. Thus, they do need to **synchronize** with other threads on shared data and other resources (see section [Synchronization](synchronization.md)).
* <mark style="color:red;">**Internal interrupt handlers can be invoked recursively.**</mark> For example, the system call handler might cause a page fault while attempting to read user memory. Deep recursion would risk overflowing the limited kernel stack (see section [Struct thread](threads.md#struct-thread)), but should be unnecessary.

### Types and Functions

<details>

<summary>Types and Functions for Internal Interrupt Handling</summary>

* <mark style="color:blue;">**Function: void intr\_register\_int (uint8\_t vec, int dpl, enum intr\_level level, intr\_handler\_func \*handler, const char \*name)**</mark>
  * **Registers handler to be called when internal interrupt numbered vec is triggered. Names the interrupt name for debugging purposes.**
  * If level is **`INTR_ON`**, external interrupts will be processed normally during the interrupt handler's execution, which is normally desirable.
  * Specifying **`INTR_OFF`** will cause the CPU to disable external interrupts when it invokes the interrupt handler. **The effect is slightly different from calling `intr_disable()` inside the handler**, because that leaves a window of one or more CPU instructions in which external interrupts are still enabled. This is important for the page fault handler; refer to the comments in `userprog/exception.c` for details.
  * **dpl determines how the interrupt can be invoked.**
    * If dpl is **0**, then the interrupt can be invoked only by kernel threads. Otherwise dpl should be **3**, which allows user processes to invoke the interrupt with an explicit INT instruction.
    * **The value of dpl doesn't affect user processes' ability to invoke the interrupt indirectly**, e.g. an invalid memory reference will cause a page fault regardless of dpl.

</details>

## External Interrupt Handling

**External interrupts are caused by events outside the CPU.** They are **asynchronous**, so they can be invoked at any time that interrupts have not been disabled. We say that an external interrupt runs in an <mark style="color:orange;">**"interrupt context."**</mark>

* **In an external interrupt, the `struct intr_frame` passed to the handler is not very meaningful.** It describes the state of the thread or process that was interrupted, but there is no way to predict which one that is. **It is possible, although rarely useful, to examine it, but modifying it is a recipe for disaster.**
* <mark style="color:red;">**Only one external interrupt may be processed at a time.**</mark> Neither internal nor external interrupt may nest within an external interrupt handler. Thus, <mark style="color:red;">**an external interrupt's handler must run with interrupts disabled**</mark> (see section [Disabling Interrupts](synchronization.md#disabling-interrupts)).
* <mark style="color:red;">**An external interrupt handler must not sleep or yield**</mark>**, which rules out calling `lock_acquire()`, `thread_yield()`, and many other functions.** Sleeping in interrupt context would effectively put the interrupted thread to sleep, too, until the interrupt handler was again scheduled and returned. This would be unfair to the unlucky thread, and it would deadlock if the handler were waiting for the sleeping thread to, e.g., release a lock.
* **An external interrupt handler effectively monopolizes the machine and delays all other activities. Therefore, external interrupt handlers should complete as quickly as they can.** Anything that require much CPU time should instead run in a kernel thread, possibly one that the interrupt triggers using a synchronization primitive.
* **External interrupts are controlled by a pair of devices outside the CPU called&#x20;**_**programmable interrupt controllers**_**, or&#x20;**_**PICs**_**&#x20;for short.**
  * When **`intr_init()`** sets up the CPU's IDT, it also initializes the PICs for interrupt handling.
  * The PICs also must be "**acknowledged**" at the end of processing for each external interrupt. `intr_handler()` takes care of that by calling **`pic_end_of_interrupt()`**, which properly signals the PICs.

### Types and Functions

The following functions relate to external interrupts.

<details>

<summary>Types and Functions for External Interrupt Handling</summary>

* <mark style="color:blue;">**Function: void**</mark> <mark style="color:blue;">**intr\_register\_ext**</mark> <mark style="color:blue;">**(uint8\_t vec, intr\_handler\_func \*handler, const char \*name)**</mark>
  * **Registers handler to be called when external interrupt numbered vec is triggered. Names the interrupt name for debugging purposes.**
  * <mark style="color:red;">**The handler will run with interrupts disabled.**</mark>
* <mark style="color:blue;">**Function: bool**</mark> <mark style="color:blue;">**intr\_context**</mark> <mark style="color:blue;">**(void)**</mark>
  * **Returns true if we are running in an interrupt context, otherwise false.**
  *   Mainly used in functions that might sleep or that otherwise should not be called from interrupt context, in this form:

      ```
      ASSERT (!intr_context ());
      ```
* <mark style="color:blue;">**Function: void intr\_yield\_on\_return (void)**</mark>
  * **When called in an interrupt context, causes `thread_yield()` to be called just before the interrupt returns.**
  * Used in the timer interrupt handler when a thread's time slice expires, to cause a new thread to be scheduled.

</details>
