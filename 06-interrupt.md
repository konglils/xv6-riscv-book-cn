## 第六章：中断和设备驱动

*驱动*是操作系统中管理特定设备的代码，它有如下功能：配置设备相关的硬件，告诉设备需要执行什么操作，处理设备产生的中断，以及与使用该设备的进程进行交互。驱动程序的代码写起来可能很棘手，因为驱动程序会与它所管理的设备并发执行，而且常常与使用该设备的进程并发执行。此外，驱动必须了解设备的硬件接口，而硬件接口可能很复杂，文档也可能不够完善。

需要操作系统关注的设备通常可以被配置为产生中断，而中断是陷阱的一种类型。内核的陷阱处理代码能够识别设备何时引发了中断，并调用驱动的中断处理程序；在 xv6 中，这个分发过程发生在 [`devintr`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c#L188) 中。

许多设备驱动程序在两种上下文中执行代码：*下半部分（bottom half）*在进程的内核线程中运行，*上半部分（top half）*在中断时执行。下半部分通过 `read` 和 `write` 等系统调用被调用，这些系统调用希望设备执行 I/O。这段代码可能要求硬件开始一个操作（例如要求磁盘读取一个块）；然后代码等待该操作完成。最终设备完成该操作并引发中断。驱动的中断处理程序作为上半部分，判断哪个操作已经完成，在合适时唤醒一个等待的进程，并在有下一个操作时告诉硬件开始处理该操作。

### 6.1 代码：控制台输入

控制台驱动（[`console.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/console.c)）是驱动结构的一个简单示例。控制台驱动通过连接到 RISC-V 的 *UART* 串行端口硬件，接收人工键入的字符。控制台驱动每次累积一行输入，并处理退格键和 control-u 等特殊的输入字符。用户进程（例如 shell）使用 `read` 系统调用从控制台获取输入行。当你在 QEMU 中向 xv6 输入时，你的按键会经由 QEMU 模拟的 UART 硬件传递给 xv6。

与驱动交互的 UART 硬件是由 QEMU 仿真的 16550 芯片（[Michael and Durich 1987 [11]](references.md#ref11)）。在真实的计算机上，16550 会管理一条连接到终端或其他计算机的 RS232 串行链路。在运行 QEMU 时，它连接到你的键盘和显示器。

UART 硬件在软件看来是一组*内存映射（memory-mapped）*的控制寄存器。也就是说，有一些物理地址连接到 UART 设备，因此加载和存储会与设备硬件交互，而不是与 RAM 交互。UART 的内存映射地址从 0x10000000 开始，即 [`UART0`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/memlayout.h#L21)。UART 有几个控制寄存器，每个的宽度都是一个字节。它们相对于 `UART0` 的偏移定义在 [(uart.c:25)](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/uart.c#L25) 中。例如，`LSR` 寄存器中有一些位表示是否有输入字符正在等待驱动读取。这些字符（如果有的话）可以从 `RHR` 寄存器中读取。每当读取一个字符时，UART 硬件就会把它从内部等待字符的 FIFO 中删除，并在 FIFO 为空时清除 `LSR` 中的“就绪”位。要发送数据，驱动向 `THR` 寄存器写入一个字节，这会使 UART 把该字节追加到一个字节 FIFO 中，UART 随后会通过 RS232 串行链路发送这些字节。UART 的发送硬件和接收硬件在很大程度上相互独立。

xv6 的 `main` 调用 [`consoleinit`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/console.c#L193) 来初始化 UART 硬件。这段代码配置 UART：每当 UART 接收到一个输入字节时产生一个接收中断，并且每当 UART 发送完一个输出字节时产生一个*传输完成（transmit complete）*中断 [(uart.c:50)](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/uart.c#L50)。

xv6 的 shell 通过 [`init.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/user/init.c#L19) 打开的文件描述符从控制台读取。`read` 系统调用会经由内核到达 [`consoleread`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/console.c#L88)。`consoleread` 等待输入到达（通过中断）并缓冲在 `cons.buf` 中，然后把输入复制到用户空间，并（在一整行到达之后）返回用户进程。如果用户还没有键入完整的一行，任何正在读取的进程都会在 `sleep` 调用中等待 [(console.c:106)](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/console.c#L106)（第 9 章将解释 `sleep` 的细节）。

当用户键入一个字符时，UART 硬件请求 RISC-V 引发一个中断，从而激活 xv6 的陷阱处理程序。陷阱处理程序调用 [`devintr`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c#L188)，后者查看 RISC-V 的 `scause` 寄存器，发现中断来自一个外部设备。然后它向一个称为 PLIC 的硬件单元（[Waterman et al. 2024 [17]](references.md#ref17)）询问是哪个设备引发了中断 [(trap.c:196)](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c#L196)。如果是 UART，`devintr` 就调用 `uartintr`。

[`uartintr`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/uart.c#L143) 从 UART 硬件读取所有等待中的输入字符，并把它们交给 [`consoleintr`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/console.c#L147)；它不会等待字符，因为未来的输入会引发新的中断。`consoleintr` 的工作是把输入字符累积到 `cons.buf` 中，直到一整行到达。`consoleintr` 会特殊处理退格键和其他几个字符。当换行符到达时，`consoleintr` 会唤醒一个正在等待的 `consoleread`（如果有的话）。

一旦被唤醒，`consoleread` 就会看到 `cons.buf` 中的一整行，把它复制到用户空间，并（经由系统调用机制）返回用户空间。

有一个模式值得注意：设备活动与进程活动通过缓冲和中断实现解耦。控制台驱动即使在没有进程等待读取时也能处理输入；随后的读取将会看到这些输入。这种解耦允许进程与设备 I/O 并发执行，从而提高性能；当设备很慢（如 UART）或需要立即关注（如回显键入的字符）时，这一点尤为重要。这个想法有时被称为 *I/O 并发*。

### 6.2 代码：控制台输出

向连接到控制台的文件描述符发起 `write` 系统调用，会到达 [`consolewrite`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/console.c#L63)，它从用户空间复制一批批字节，并把每一批交给 [`uartwrite`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/uart.c#L81)。在把每个字节写入 UART 的 `THR` 寄存器之前，`uartwrite` 必须等待 UART 准备好接收更多输出。由于 UART 相对较慢，`uartwrite` 使用 `sleep`（它会让出 CPU）来等待，而不是忙循环。当 UART 发送完最近的一个字节时，它会产生中断。中断例程 [`uartintr`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/uart.c#L143) 与 `uartwrite` 协作：在发送每个字节之前，`uartwrite` 读取 UART 的 `LSR` 寄存器，看 UART 能否再接收一个字节，如果不能就调用 `sleep`；当 UART 表示它已准备好接收更多输出时，中断例程调用 `wakeup`，这会使 `uartwrite` 醒来，重新检查 `LSR` 寄存器，并发送下一个字节。

### 6.3 驱动中的并发

你可能已经注意到，`consoleread` 和 `consoleintr` 中调用了 `acquire`。这些调用获取一把锁，保护控制台驱动的数据结构不被并发访问。这里有三种并发风险：不同 CPU 上的两个进程可能同时调用 `consoleread`；硬件可能在该 CPU 已经在 `consoleread` 内部执行时，要求该 CPU 递送一个控制台（实际上是 UART）中断；以及硬件可能在 `consoleread` 执行时在另一个 CPU 上递送一个控制台中断。第 7 章解释了如何使用锁来确保这些风险不会导致错误的结果。

驱动中需要小心处理并发的另一种情况是：一个进程可能正在等待来自设备的输入，但表明输入到达的中断可能是在另一个进程（或者根本没有进程）运行时到来的。因此，中断处理程序不能去考虑它所中断的进程或代码。例如，中断处理程序不能安全地使用当前进程的页表调用 `copyout`。中断处理程序通常只做相对较少的工作（例如，只是把输入数据复制到缓冲区），并唤醒下半部分代码来完成剩下的工作。

### 6.4 定时器中断

xv6 使用定时器中断来维护它对当前时间的概念，并在计算密集型进程之间切换。定时器中断来自连接到每个 RISC-V CPU 的时钟硬件。xv6 对每个 CPU 的时钟硬件进行编程，使其周期性地中断该 CPU。

[`start.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/start.c#L56) 中的代码设置了一些控制位，使监管模式能够访问定时器控制寄存器，然后请求第一个定时器中断。`time` 控制寄存器中包含一个由硬件以稳定速率递增的计数值；它充当当前时间的概念。`stimecmp` 寄存器中包含一个时间，到了该时间 CPU 将引发定时器中断；把 `stimecmp` 设置为 `time` 的当前值加上 *x*，就会安排一个在 *x* 个时间单位之后发生的中断。对于 `qemu` 的 RISC-V 模拟，1000000 个时间单位大约是十分之一秒。

定时器中断像其他设备中断一样，经由 `usertrap` 或 `kerneltrap` 和 `devintr` 到达。定时器中断到达时，`scause` 的低位被置为 5；`trap.c` 中的 `devintr` 检测到这种情况，并调用 [`clockintr`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c#L167)。后一个函数递增 `ticks`，使内核能够跟踪时间的流逝。递增只在一个 CPU 上进行，以免在有多个 CPU 时时间过得更快。`clockintr` 唤醒所有在 `pause` 系统调用中等待的进程，并通过写入 `stimecmp` 来安排下一个定时器中断。

`devintr` 对定时器中断返回 2，以向 `kerneltrap` 或 `usertrap` 表明它们应调用 `yield`，从而使 CPU 能在可运行的进程之间多路复用。

内核代码可能被定时器中断打断，而该中断通过 `yield` 强制进行上下文切换；这正是 `usertrap` 开头的代码在启用中断之前小心保存 `sepc` 等状态的部分原因。这些上下文切换还意味着，编写内核代码时必须意识到，它可能会在毫无征兆的情况下从一个 CPU 迁移到另一个 CPU。

### 6.5 现实世界

与许多操作系统一样，xv6 允许在内核中执行时发生中断，甚至进行上下文切换（通过 `yield`）。这样做的原因是在运行时间很长的复杂系统调用期间保持较快的响应时间。然而，正如上文所述，允许内核中发生中断是一些复杂性的来源；因此，少数操作系统只在执行用户代码时允许中断。

在一台典型的计算机上完整地支持所有设备是一件很辛苦的事情，因为设备很多，设备有很多功能，设备和驱动程序之间的协议可能很复杂，而且文档也不完善。在许多操作系统中，驱动程序所占的代码比核心内核还多。

UART 驱动通过读取 UART 控制寄存器，一次读取一个字节的数据；这种模式称为*编程 I/O（programmed I/O）*，因为数据的移动是由软件驱动的。编程 I/O 很简单，但在高数据速率下太慢，无法使用。需要高速移动大量数据的设备通常使用*直接内存访问（direct memory access，DMA）*。DMA 设备硬件直接把传入的数据写入 RAM，并从 RAM 中读取传出的数据。现代磁盘和网络设备都使用 DMA。DMA 设备的驱动程序会先在 RAM 中准备好数据，然后用对控制寄存器的一次写入来告诉设备处理准备好的数据。

当设备在不可预知的时间需要关注，而且不那么频繁时，中断是很有用的。但中断的 CPU 开销很大。因此，高速设备（如网络和磁盘控制器）使用各种技巧来减少对中断的需求。一种技巧是为整批传入或传出的请求只引发一个中断。另一种技巧是让驱动程序完全禁用中断，并定期检查设备是否需要关注。这种技术称为*轮询（polling）*。如果设备执行操作的速度很高，轮询是有意义的；但如果设备大部分时间处于空闲状态，轮询就会浪费 CPU 时间。一些驱动程序会根据当前的设备负载，在轮询和中断之间动态切换。

UART 驱动先把传入的数据复制到内核中的缓冲区，然后再复制到用户空间。这在低数据速率下是合理的，但对于非常快速地生成或消耗数据的设备，这样的双重复制会显著降低性能。有些操作系统能够直接在用户空间缓冲区和设备硬件之间移动数据，通常使用 DMA。

正如第 1 章所述，控制台在应用程序看来是一个普通文件，应用程序使用 `read` 和 `write` 系统调用读取输入并写出输出。应用程序可能希望控制设备的某些方面，而这些方面无法通过标准的文件系统调用表达（例如，启用或禁用控制台驱动中的行缓冲）。Unix 操作系统为此类情况提供了 `ioctl` 系统调用。

计算机的某些用途要求对外部事件做出“实时”响应：保证在有界时间内做出响应。例如，在安全关键系统中，错过截止时间可能导致灾难。xv6 不适合实时环境。原因之一是，xv6 的调度器在决定接下来运行哪个进程时不会考虑实时截止时间；此外，xv6 有一些在禁用中断的情况下执行的冗长内核代码路径，因此它可能无法迅速响应中断。实时操作系统不仅要解决这些问题，其结构还必须便于分析最坏情况下的响应时间。

### 6.6 练习

1. 修改 `uart.c`，使其完全不使用中断。你可能还需要修改 `console.c`。

2. 添加一个以太网卡驱动。