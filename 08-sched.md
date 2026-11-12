## 第八章：调度

任何操作系统的进程数量都可能超过计算机的 CPU 数量，因此需要制定一个方案，在各进程之间分时共享 CPU。理想情况下，这种共享对用户进程是透明的。一种常见的方法是通过把进程**多路复用（multiplexing）**到硬件 CPU 上，给每个进程提供它拥有自己的虚拟 CPU 的假象。本章解释 xv6 如何实现这种多路复用。

在继续阅读本章之前，请先阅读 [`kernel/proc.h`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.h)、[`kernel/swtch.S`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/swtch.S)，以及 [`kernel/proc.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c) 中的 `yield()`、`sched()` 和 `scheduler()`。

### 8.1 多路复用

xv6 通过在两种情况下把每个 CPU 从一个进程切换到另一个进程来实现多路复用。第一种情况是，当进程发起一个会阻塞（必须等待）的系统调用时，例如 `read` 或 `wait`，xv6 会进行切换。第二种情况是，xv6 周期性地强制切换，以应对长时间计算而不阻塞的进程。前者称为**自愿切换（voluntary switch）**，后者称为**非自愿切换（involuntary switch）**。

实现多路复用会带来一些挑战。首先，如何从一个进程切换到另一个进程？基本思路是保存和恢复 CPU 寄存器，但这一过程无法用 C 语言表达，因此相当棘手。第二，如何以对用户进程透明的方式强制切换？xv6 采用标准技术：由硬件定时器的中断驱动上下文切换。第三，所有 CPU 都在同一组进程之间切换，因此需要一套加锁方案，以避免诸如两个 CPU 同时决定运行同一个进程之类的错误。第四，当进程退出时，必须释放该进程的内存和其他资源，但进程自身无法完成所有这些工作。第五，多核机器的每个 CPU 必须记住它正在执行哪个进程，这样系统调用才能影响正确进程的内核状态。

### 8.2 上下文切换概述

术语“上下文切换”（context switch）指的是 CPU 停止执行一个内核线程（通常是为了以后恢复执行）、转而恢复执行另一个内核线程所涉及的步骤；这种切换是多路复用的核心。xv6 并不直接从一个进程的内核线程上下文切换到另一个进程的内核线程；相反，一个内核线程通过上下文切换到该 CPU 的“调度器线程”来放弃 CPU，调度器线程则挑选另一个进程的内核线程来运行，并上下文切换到该线程。

![Figure 8.1](fig/switch.svg)

*图 8.1：从一个用户进程切换到另一个用户进程。在这个例子中，xv6 运行在一个 CPU 上（因此只有一个调度器线程）。*

从更宏观的角度看，从一个用户进程切换到另一个用户进程所涉及的步骤如图 8.1 所示：从旧进程的用户空间发生一次陷阱（系统调用或中断）并进入它的内核线程，上下文切换到当前 CPU 的调度器线程，上下文切换到新进程的内核线程，最后陷阱返回到用户级进程。

### 8.3 代码：上下文切换

`kernel/swtch.S` 中的 `swtch()` 函数包含了线程上下文切换的核心：它保存被切出线程的 CPU 寄存器，并恢复被切入线程此前保存的寄存器。这样做之所以足够，根本原因在于线程的状态由内存中的数据（例如它的栈）加上它的 CPU 寄存器构成；线程的内存不需要保存和恢复，因为不同线程把各自的数据保存在 RAM 的不同区域；但 CPU 只有一组寄存器，因此必须在线程之间进行切换（保存和恢复）。

每个线程的 `struct proc` 都包含一个 `struct context`，用于在该线程不运行时保存它的寄存器。CPU 的调度器线程的 `struct context` 位于该 CPU 的 `struct cpu` 中。当线程 X 想要切换到线程 Y 时，线程 X 调用 `swtch(&X's context, &Y's context)`。`swtch()` 把当前 CPU 寄存器保存到 X 的 context 中，然后把 Y 的 context 的内容加载到 CPU 寄存器中，最后返回。

下面是 `swtch` 的一个节选副本：

```asm
swtch:
        sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        ...
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ...
        ld s11, 104(a1)

        ret
```

`a0` 存放第一个函数参数，`a1` 存放第二个；在这里，这两个参数都是 `struct context` 指针。`16(a0)` 指的是 `a0` 所指向的内存中偏移 16 字节处；对照 [`kernel/proc.h`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.h#L2) 中 `struct context` 的定义，这就是名为 `s0` 的结构体字段。

`swtch` 的 `ret` 返回到哪里？它返回到 `ra` 寄存器所指向的指令。在线程 X 调用 `swtch()` 切换到 Y 的例子中，当 `ret` 执行时，`ra` 刚刚从 Y 的 `struct context` 中加载。而 Y 的 `struct context` 中的 `ra`，最初是 Y 在过去放弃 CPU 时调用 `swtch` 所保存的。因此 `ret` 返回到 Y 调用 `swtch()` 之后的指令；也就是说，X 对 `swtch()` 的调用就像是从 Y 原来那次 `swtch()` 调用返回一样。而 `sp` 将是 Y 的栈指针，因为 `swtch` 从 Y 的 `struct context` 中加载了 `sp`；因此返回之后，Y 将在自己的栈上执行。`swtch()` 不必直接保存或恢复程序计数器；保存和恢复 `ra` 就足够了。

[`swtch`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/swtch.S#L9) 保存被调用者保存寄存器（callee-saved register，即 `ra,sp,s0..s11`），但不保存调用者保存寄存器（caller-saved register）。RISC-V 调用约定要求：如果代码需要在函数调用前后保留某个调用者保存寄存器中的值，编译器必须生成指令，在函数调用之前把该寄存器保存到栈上，并在函数返回时从栈上恢复。因此，`swtch` 可以依赖调用它的函数已经保存了调用者保存寄存器（或者调用函数根本不需要这些寄存器中的值）。

### 8.4 代码：调度

上一节考察了 `swtch` 的内部细节；现在我们假定 `swtch` 已经可用，来考察如何从一个进程的内核线程经由调度器切换到另一个进程。调度器以每个 CPU 一个特殊线程的形式存在，每个线程都运行 `scheduler` 函数。这个函数负责选择下一个要运行哪个进程。每个 CPU 都有自己的调度器线程，因为在任意时刻可能有不止一个 CPU 在寻找可运行的对象。进程切换总是经由调度器线程进行，而不是直接从一个进程切换到另一个进程，以避免出现没有栈可供调度器执行的情况（例如旧进程已经退出，或者当前没有其他想运行的进程）。

一个想要放弃 CPU 的进程必须获取自己的进程锁 `p->lock`，释放它持有的其他所有锁，更新自己的状态（`p->state`），然后调用 `sched`。你可以在 `yield`、`sleep` 和 `kexit` 中看到这个序列。`sched` 调用 `swtch`，把当前上下文保存到 `p->context` 中，并切换到 `cpu->context` 中的调度器上下文。`swtch` 在调度器的栈上返回，就好像 `scheduler` 的 `swtch` 返回了一样 [(proc.c:453)](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L453)。

[`scheduler`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L429) 运行一个循环：找到一个要运行的进程，用 `swtch()` 切换到它，它最终会用 `swtch()` 切换回调度器，调度器则继续它的循环。调度器遍历进程表，寻找一个可运行的进程，即其 `p->state` 为 `RUNNABLE` 的进程。一旦找到这样的进程，它就设置每个 CPU 的当前进程变量 `c->proc`，把该进程标记为 `RUNNING`，然后调用 `swtch` 开始运行它 [(proc.c:448-453)](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L448-L453)。在过去的某个时刻，目标进程一定调用过 `swtch()`；调度器对 `swtch()` 的调用实际上就是从那次更早的调用返回。图 8.2 说明了这种模式。

![Figure 8.2](fig/swtch.svg)

*图 8.2：`swtch()` 总是以调度器线程为起点或终点，并且相关的 `p->lock` 始终被持有。*

xv6 在调用 `swtch` 的整个过程中持有 `p->lock`：`swtch` 的调用者获取该锁，但锁会在 `swtch` 返回之后由被切换到的目标线程释放。这种安排并不寻常：更常见的做法是，获取锁的线程同时也负责释放锁。xv6 的上下文切换之所以打破这一惯例，是因为 `p->state` 和 `p->context` 必须原子地一起更新。例如，如果在调用 `swtch` 之前就释放了 `p->lock`，另一个 CPU `c` 可能因为该进程的状态是 `RUNNABLE` 而决定运行它。CPU `c` 会调用 `swtch`，从 `p->context` 中恢复寄存器，而此时原来的 CPU 仍在向 `p->context` 中保存寄存器。结果就是，该进程在 CPU `c` 上会带着只保存了一半的寄存器被恢复，并且两个 CPU 会使用同一个栈，从而造成混乱。一旦 `yield` 开始修改一个正在运行的进程的状态、使其变为 `RUNNABLE`，`p->lock` 就必须一直保持持有，直到该进程保存完所有寄存器、并且调度器已经在自己的栈上运行。最早正确的释放时机是在 `scheduler`（在自己栈上运行）清除 `c->proc` 之后。同样，一旦 `scheduler` 开始把一个 `RUNNABLE` 进程转换为 `RUNNING`，锁就不能释放，直到该进程的内核线程完全运行起来（即在 `swtch` 之后，例如在 `yield` 中）。

有一种情况下，调度器对 `swtch` 的调用最终不会进入 `sched`。`allocproc` 会把新进程的 context 中的 `ra` 寄存器设置为 [`forkret`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L513)，这样该进程的第一次 `swtch` 就会“返回”到该函数的开头。`forkret` 的作用是释放 `p->lock`，并设置一些返回用户空间所需的控制寄存器和 trapframe 字段。最后，`forkret` 模拟从系统调用正常返回用户空间的路径。

### 8.5 代码：mycpu 与 myproc

xv6 经常需要指向当前进程的 `proc` 结构体的指针。在单处理器上，可以用一个全局变量指向当前的 `proc`。这在多核机器上行不通，因为每个 CPU 执行的是不同的进程。解决这个问题的办法是利用每个 CPU 都有自己的一组寄存器这一事实。

当某个 CPU 在内核中执行时，xv6 确保该 CPU 的 `tp` 寄存器始终保存着该 CPU 的 hartid。RISC-V 对它的 CPU 进行编号，每个 CPU 都有一个唯一的 *hartid*。[`mycpu`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L74) 使用 `tp` 作为索引来访问一个 `cpu` 结构体数组，返回当前 CPU 对应的那一项。一个 [`struct cpu`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.h#L22) 中保存着指向当前在该 CPU 上运行的进程的 `proc` 结构体的指针（如果有的话）、为该 CPU 的调度器线程保存的寄存器，以及管理中断禁用所需的嵌套自旋锁计数。

要确保一个 CPU 的 `tp` 中保存着该 CPU 的 hartid，需要费一点周折，因为用户代码可以随意修改 `tp`。`start` 在 CPU 启动过程的早期、还处于机器模式时就设置 `tp` 寄存器 [(start.c:48)](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/start.c#L48)。在准备返回用户空间时，`prepare_return` 会把 `tp` 保存到 trampoline 页中，以防用户代码修改它。最后，`uservec` 在从用户空间进入内核时恢复保存的 `tp` [(trampoline.S:78)](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trampoline.S#L78)。编译器保证在内核代码中永远不修改 `tp`。如果 xv6 能在需要时向 RISC-V 硬件询问当前的 hartid，那会更方便，但 RISC-V 只允许在机器模式下这样做，在监管模式（supervisor mode）下不允许。

`cpuid` 和 `mycpu` 的返回值是不可靠的：如果定时器中断导致线程让出 CPU，随后又在另一个 CPU 上恢复执行，那么之前返回的值就不再正确。为了避免这个问题，xv6 要求代码在调用 `cpuid()` 或 `mycpu()` 之前禁用中断，并且只有在用完返回值之后才启用中断。

[`myproc`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L83) 函数返回当前 CPU 上正在运行的进程的 `struct proc` 指针。`myproc` 禁用中断，调用 `mycpu`，从 `struct cpu` 中取出当前进程指针（`c->proc`），然后启用中断。即使启用了中断，`myproc` 的返回值也可以安全使用：如果定时器中断把调用进程移到了另一个 CPU 上，它的 `struct proc` 指针仍然保持不变。

### 8.6 现实世界

xv6 调度器实现了一个简单的调度策略，依次运行每个进程。这种策略称为*轮询调度（round robin）*。真实的操作系统实现了更复杂的策略，例如允许进程拥有优先级。其思想是，调度器会优先选择可运行的高优先级进程，而不是可运行的低优先级进程。这些策略可能变得复杂，因为常常存在相互竞争的目标：例如，操作系统可能还想保证公平性和高吞吐量。

### 8.7 练习

1. 修改 xv6，使得在从一个进程的内核线程切换到另一个进程的内核线程时只使用一次上下文切换，而不是经由调度器线程切换。让出 CPU 的线程需要自己选择下一个线程并调用 `swtch`。挑战在于：防止多个 CPU 意外执行同一个线程；正确处理加锁；以及避免死锁。
