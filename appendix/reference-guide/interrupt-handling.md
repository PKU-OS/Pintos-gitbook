# Interrupt Handling

An _interrupt_ notifies the CPU of some event. Much of the work of an operating system relates to interrupts in one way or another. For our purposes, we classify interrupts into two broad categories:

* _Internal interrupts_, that is, interrupts caused directly by CPU instructions. System calls, attempts at invalid memory access (_page faults_), and attempts to divide by zero are some activities that cause internal interrupts. Because they are caused by CPU instructions, internal interrupts are _synchronous_ or synchronized with CPU instructions. `intr_disable()` does not disable internal interrupts.
* _External interrupts_, that is, interrupts originating outside the CPU. These interrupts come from hardware devices such as the system timer, keyboard, serial ports, and disks. External interrupts are _asynchronous_, meaning that their delivery is not synchronized with instruction execution. Handling of external interrupts can be postponed with `intr_disable()` and related functions (see section [Disabling Interrupts](synchronization.md#disabling-interrupts)).

The CPU treats both classes of interrupts largely the same way, so Pintos has common infrastructure to handle both classes. The following section describes this common infrastructure. The sections after that give the specifics of external and internal interrupts.

## Interrupt Infrastructure

When an interrupt occurs, the CPU saves its most essential state on a stack and jumps to an interrupt handler routine. The 80x86 architecture supports 256 interrupts, numbered 0 through 255, each with an independent handler defined in an array called the _interrupt descriptor table_ or IDT.

In Pintos, `intr_init()` in `threads/interrupt.c` sets up the IDT so that each entry points to a unique entry point in `threads/intr-stubs.S` named `intrNN_stub()`, where NN is the interrupt number in hexadecimal. Because the CPU doesn't give us any other way to find out the interrupt number, this entry point pushes the interrupt number on the stack. Then it jumps to `intr_entry()`, which pushes all the registers that the processor didn't already push for us, and then calls `intr_handler()`, which brings us back into C in `threads/interrupt.c`.

The main job of `intr_handler()` is to call the function registered for handling the particular interrupt. (If no function is registered, it dumps some information to the console and panics.) It also does some extra processing for external interrupts (see section [External Interrupt Handling](interrupt-handling.md#external-interrupt-handling)).

When `intr_handler()` returns, the assembly code in `threads/intr-stubs.S` restores all the CPU registers saved earlier and directs the CPU to return from the interrupt.

<details>

<summary>Types and Functions for interrupt</summary>

#### Type: **void intr\_handler\_func (struct intr\_frame \*frame)**

This is how an interrupt handler function must be declared. Its frame argument (see below) allows it to determine the cause of the interrupt and the state of the thread that was interrupted.

#### Type: **struct intr\_frame**

The stack frame of an interrupt handler, as saved by the CPU, the interrupt stubs, and `intr_entry()`. Its most interesting members are described below.

#### Member of `struct intr_frame`: uint32\_t **edi**

#### Member of `struct intr_frame`: uint32\_t **esi**

#### Member of `struct intr_frame`: uint32\_t **ebp**

#### Member of `struct intr_frame`: uint32\_t **esp\_dummy**

#### Member of `struct intr_frame`: uint32\_t **ebx**

#### Member of `struct intr_frame`: uint32\_t **edx**

#### Member of `struct intr_frame`: uint32\_t **ecx**

#### Member of `struct intr_frame`: uint32\_t **eax**

#### Member of `struct intr_frame`: uint16\_t **es**

#### Member of `struct intr_frame`: uint16\_t **ds**

Register values in the interrupted thread, pushed by `intr_entry()`. The `esp_dummy` value isn't actually used.

#### Member of `struct intr_frame`: uint32\_t **vec\_no**

The interrupt vector number, ranging from 0 to 255.

#### Member of `struct intr_frame`: uint32\_t **error\_code**

The "error code" pushed on the stack by the CPU for some internal interrupts.

#### Member of `struct intr_frame`: void **(\*eip)** (void)

The address of the next instruction to be executed by the interrupted thread.

#### Member of `struct intr_frame`: void \***esp**

The interrupted thread's stack pointer.

#### Function: const char \***intr\_name** (uint8\_t vec)

Returns the name of the interrupt numbered vec, or `"unknown"` if the interrupt has no registered name.

</details>

## Internal Interrupt Handling

Internal interrupts are caused directly by CPU instructions executed by the running kernel thread or user process (from project 2 onward). An internal interrupt is therefore said to arise in a "process context."

In an internal interrupt's handler, it can make sense to examine the `struct intr_frame` passed to the interrupt handler, or even to modify it. When the interrupt returns, modifications in `struct intr_frame` become changes to the calling thread or process's state. For example, the Pintos system call handler returns a value to the user program by modifying the saved EAX register.

There are no special restrictions on what an internal interrupt handler can or can't do. Generally they should run with interrupts enabled, just like other code, and so they can be preempted by other kernel threads. Thus, they do need to synchronize with other threads on shared data and other resources (see section [Synchronization](synchronization.md)).

Internal interrupt handlers can be invoked recursively. For example, the system call handler might cause a page fault while attempting to read user memory. Deep recursion would risk overflowing the limited kernel stack (see section [`struct thread`](threads.md#struct-thread)), but should be unnecessary.

#### Function: void **intr\_register\_int** (uint8\_t vec, int dpl, enum intr\_level level, intr\_handler\_func \*handler, const char \*name)

Registers handler to be called when internal interrupt numbered vec is triggered. Names the interrupt name for debugging purposes.

If level is `INTR_ON`, external interrupts will be processed normally during the interrupt handler's execution, which is normally desirable. Specifying `INTR_OFF` will cause the CPU to disable external interrupts when it invokes the interrupt handler. The effect is slightly different from calling `intr_disable()` inside the handler, because that leaves a window of one or more CPU instructions in which external interrupts are still enabled. This is important for the page fault handler; refer to the comments in `userprog/exception.c` for details.

dpl determines how the interrupt can be invoked. If dpl is 0, then the interrupt can be invoked only by kernel threads. Otherwise dpl should be 3, which allows user processes to invoke the interrupt with an explicit INT instruction. The value of dpl doesn't affect user processes' ability to invoke the interrupt indirectly, e.g. an invalid memory reference will cause a page fault regardless of dpl.

## External Interrupt Handling

External interrupts are caused by events outside the CPU. They are asynchronous, so they can be invoked at any time that interrupts have not been disabled. We say that an external interrupt runs in an "interrupt context."

In an external interrupt, the `struct intr_frame` passed to the handler is not very meaningful. It describes the state of the thread or process that was interrupted, but there is no way to predict which one that is. It is possible, although rarely useful, to examine it, but modifying it is a recipe for disaster.

Only one external interrupt may be processed at a time. Neither internal nor external interrupt may nest within an external interrupt handler. Thus, an external interrupt's handler must run with interrupts disabled (see section [Disabling Interrupts](synchronization.md#disabling-interrupts)).

An external interrupt handler must not sleep or yield, which rules out calling `lock_acquire()`, `thread_yield()`, and many other functions. Sleeping in interrupt context would effectively put the interrupted thread to sleep, too, until the interrupt handler was again scheduled and returned. This would be unfair to the unlucky thread, and it would deadlock if the handler were waiting for the sleeping thread to, e.g., release a lock.

An external interrupt handler effectively monopolizes the machine and delays all other activities. Therefore, external interrupt handlers should complete as quickly as they can. Anything that require much CPU time should instead run in a kernel thread, possibly one that the interrupt triggers using a synchronization primitive.

External interrupts are controlled by a pair of devices outside the CPU called _programmable interrupt controllers_, _PICs_ for short. When `intr_init()` sets up the CPU's IDT, it also initializes the PICs for interrupt handling. The PICs also must be "acknowledged" at the end of processing for each external interrupt. `intr_handler()` takes care of that by calling `pic_end_of_interrupt()`, which properly signals the PICs.

The following functions relate to external interrupts.

#### Function: void **intr\_register\_ext** (uint8\_t vec, intr\_handler\_func \*handler, const char \*name)

Registers handler to be called when external interrupt numbered vec is triggered. Names the interrupt name for debugging purposes. The handler will run with interrupts disabled.

#### Function: bool **intr\_context** (void)

Returns true if we are running in an interrupt context, otherwise false. Mainly used in functions that might sleep or that otherwise should not be called from interrupt context, in this form:

```
ASSERT (!intr_context ());
```

#### Function: void **intr\_yield\_on\_return** (void)

When called in an interrupt context, causes `thread_yield()` to be called just before the interrupt returns. Used in the timer interrupt handler when a thread's time slice expires, to cause a new thread to be scheduled.



