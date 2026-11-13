## 第四章：陷阱和系统调用

有三种事件会使 CPU 搁置普通指令的执行，强制将控制权转移给处理该事件的特殊内核代码。一种情况是系统调用，即用户程序执行 `ecall` 指令，请求内核为它做某事。另一种情况是*异常*：一条指令（用户指令或内核指令）做了非法的事情，例如从无效的虚拟地址加载。第三种情况是设备*中断*，即设备发出需要注意的信号，例如磁盘硬件完成一个读写请求。

本书使用*陷阱（trap）*作为这些情况的统称。通常，在发生陷阱时正在执行的代码之后需要恢复执行，而不应需要觉察到发生过任何特殊的事情。也就是说，我们常常希望陷阱是透明的；这对设备中断尤其重要，因为被中断的代码通常并未预料到中断。陷阱会强制把控制权转移到内核；内核保存寄存器和其他状态，以便之后恢复执行；内核执行相应的处理程序代码（例如系统调用的实现或设备驱动）；内核恢复保存的状态并从陷阱返回；原来的代码则从它中断的地方恢复执行。

xv6 在内核中处理所有陷阱；陷阱不会被递交给用户代码。在内核中处理陷阱对系统调用来说是自然的。对中断来说也是合理的，因为隔离要求只有内核才被允许使用设备，而且内核能够在多个进程之间共享设备。对异常来说同样合理，因为内核或许能够处理来自用户空间的异常（例子见第 5 章），或者以杀死违规程序作为响应。

xv6 的陷阱处理分为四个阶段：RISC-V CPU 采取的硬件动作，为内核 C 代码做准备的若干汇编指令，决定如何处理陷阱的 C 函数，以及系统调用或设备驱动的服务例程。虽然三种陷阱类型之间的共性表明内核可以用同一条代码路径处理所有陷阱，但事实证明，为两种不同的情况分别设置代码会更方便：来自用户空间的陷阱，以及来自内核空间的陷阱。处理陷阱的内核代码（汇编或 C）通常称为*处理程序*（handler）；处理程序的第一条指令通常用汇编（而不是 C）编写，有时称为*向量*（vector）。

在继续之前，请阅读 [`kernel/trampoline.S`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trampoline.S)，以及 [`kernel/trap.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c) 中的 `usertrap()` 和 `prepare_return()`。

### 4.1 RISC-V 陷阱机制

每个 RISC-V CPU 都有一组硬件控制寄存器，内核通过写入这些寄存器来告诉 CPU 如何处理陷阱，也可以通过读取它们来了解已经发生的陷阱。RISC-V 文档包含完整的说明（[Waterman et al. 2024 [17]](references.md#ref17)）。[`riscv.h`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/riscv.h) 包含 xv6 使用的定义。下面是最重要的寄存器的概述：

- `stvec`：内核在这里写入其陷阱处理程序代码的地址；RISC-V 跳转到 `stvec` 中的地址来处理陷阱。
- `sepc`：发生陷阱时，RISC-V 把程序计数器保存在这里（因为随后 `pc` 会被 `stvec` 中的值覆盖）。`sret`（从陷阱返回）指令把 `sepc` 复制到 `pc`。内核可以写入 `sepc`，以控制 `sret` 返回到哪里。
- `scause`：RISC-V 在这里放入一个描述陷阱原因的数字。
- `sscratch`：内核陷阱处理程序代码使用 `sscratch`，以帮助自己在保存用户寄存器之前不覆盖它们。
- `sstatus`：`sstatus` 中的 `SIE` 位控制设备中断是否启用。如果内核清除 `SIE`，RISC-V 将推迟设备中断，直到内核设置 `SIE`。`SPP` 位表示陷阱来自用户模式还是监管模式，并控制 `sret` 返回到哪个模式。

上述寄存器只能在监管模式下（即由内核）访问；CPU 会阻止用户代码读取或写入它们。

多核芯片上的每个 CPU 都有自己的一组这些寄存器，而且在任一时刻都可能有多个 CPU 在处理陷阱。

当 RISC-V 硬件强制进入陷阱时，它执行以下操作：

1. 如果该陷阱是设备中断，并且 `sstatus` 的 `SIE` 位为 0，则不执行以下任何操作。
2. 通过清除 `sstatus` 中的 `SIE` 位来禁用中断。
3. 把 `pc` 复制到 `sepc`。
4. 把当前模式（用户或监管模式）保存在 `sstatus` 的 `SPP` 位。
5. 把 `scause` 设置为表示陷阱原因的数字。
6. 把模式设置为监管模式。
7. 把 `stvec` 复制到 `pc`。
8. 从新的 `pc` 开始执行。

CPU 不会切换到内核页表，不会切换到内核中的栈，也不会保存 `pc` 以外的任何寄存器。这些任务必须由内核软件完成。CPU 在陷阱期间只做很少的工作，一个原因是为了给软件提供灵活性；例如，一些操作系统在某些情况下省略页表切换，以提高陷阱处理的性能。

值得想一想，为了追求更快的陷阱处理，上面列出的步骤是否有些可以被省略。虽然在有些情况下更简单的序列也能工作，但一般来说，省略其中许多步骤是危险的。例如，假设 CPU 不切换程序计数器。那么来自用户空间的陷阱就可能切换到监管模式，却仍在执行用户指令。这些用户指令可能破坏用户/内核隔离，例如通过修改 `satp` 寄存器，让它指向一个允许访问全部物理内存的页表。因此，CPU 必须切换到内核指定的指令地址，即 `stvec`。

### 4.2 来自用户空间的陷阱

![Figure 4.1](fig/trap.svg)

*图 4.1：来自用户代码的陷阱处理流程概述。*

xv6 对陷阱的处理方式取决于陷阱是在内核中执行时发生，还是在用户代码中执行时发生。这里讲述来自用户代码的陷阱；第 4.5 节讲述来自内核代码的陷阱。

在用户空间执行时，如果用户程序进行了系统调用（`ecall` 指令），或者做了非法的事情，或者发生了设备中断，都可能发生陷阱。如图 4.1 所示，来自用户空间的陷阱的高层路径是 [`uservec`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trampoline.S#L22)，然后是 [`usertrap`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c#L38)；当内核准备返回时，`usertrap` 返回到 [`userret`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trampoline.S#L101)，由它执行 `sret` 返回用户空间。

xv6 陷阱处理设计上的一个主要约束是：RISC-V 硬件在强制进入陷阱时不会切换页表。这意味着 `stvec` 中的陷阱处理程序地址必须在用户页表中有有效的映射，因为陷阱处理代码开始执行时生效的正是用户页表。此外，xv6 的陷阱处理代码需要切换到内核页表；为了在切换之后还能继续执行，内核页表也必须为 `stvec` 指向的处理程序提供映射。

xv6 使用一个 *trampoline 页*来满足这些要求。这个页包含 `uservec`，即 `stvec` 所指向的 xv6 陷阱处理代码。trampoline 页在每个进程的页表中都映射到虚拟地址 `0x3ffffff000`（称为 `TRAMPOLINE`），这是虚拟地址空间的最后一页，因此位于程序自用内存之上。trampoline 页在内核页表中也映射到同一个虚拟地址。参见图 2.3 和图 3.3。因为 trampoline 页在用户页表中被映射，所以陷阱可以在监管模式下从这里开始执行。因为 trampoline 页在内核地址空间中映射到相同地址，所以陷阱处理程序在切换到内核页表之后仍能继续执行。

[`trampoline.S`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trampoline.S#L22) 中是 `uservec` 陷阱处理程序的代码。当 `uservec` 启动时，全部 32 个寄存器都包含被中断的用户代码所拥有的值。这 32 个值需要保存到内存中的某个地方，以便内核之后在返回用户空间之前能恢复它们。向内存存储需要使用一个寄存器来保存存储的目标地址，但此时没有任何通用寄存器可用！幸运的是，RISC-V 通过 `sscratch` 寄存器伸出了援手。`uservec` 开头的 `csrw` 指令把 `a0` 保存到 `sscratch`。现在 `uservec` 有一个寄存器（`a0`）可以使用了。

`uservec` 的下一个任务是保存 32 个用户寄存器。内核为每个进程分配一页内存，用于一个 `trapframe` 结构，该结构（除了别的内容以外）有空间保存这 32 个用户寄存器 [(proc.h:40)](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.h#L40)。因为 `satp` 仍然指向用户页表，所以 `uservec` 需要 trapframe 被映射在用户地址空间中。xv6 在每个进程的用户页表中把该进程的 trapframe 映射到虚拟地址 `TRAPFRAME`（`0x3fffffe000`），即 `TRAMPOLINE` 下面的一页。每个进程的 `p->trapframe` 包含该进程 trapframe 的内核虚拟地址。

`uservec` 把寄存器 `a0` 设置为地址 `TRAPFRAME`，并把所有用户寄存器保存到那里。然后它从 `sscratch` 中取回用户的 `a0`，并把它保存到 trapframe 中。

内核此前已经初始化了 trapframe，使其包含一些对 `uservec` 有用的值：当前进程的内核栈地址、当前 CPU 的 hartid、`usertrap` 函数的地址，以及内核页表的地址。`uservec` 取出这些值，把 `satp` 切换到内核页表，并跳转到 C 函数 `usertrap`。

`usertrap` 的任务是确定陷阱的原因、处理它，然后返回 [(trap.c:38)](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c#L38)。它首先修改 `stvec`，这样在内核中发生的陷阱将由 `kernelvec` 而不是 `uservec` 处理。它保存 `sepc` 寄存器（保存的用户程序计数器），以供之后返回用户空间时使用。如果陷阱是系统调用，`usertrap` 调用 `syscall` 来处理；如果是设备中断，调用 `devintr`；如果是缺页异常，调用 `vmfault`；否则就是异常（例如使用了非法地址），内核会杀死出错的进程。系统调用路径会在保存的用户程序计数器上加四，因为对于系统调用，RISC-V 留下的程序指针指向 `ecall` 指令，而用户代码需要在紧随其后的指令处恢复执行。`usertrap` 检查进程是否已被杀死，或者是否应该让出 CPU（如果这个陷阱是定时器中断）。

返回用户空间的第一步是调用 [`prepare_return`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c#L101)。这个函数设置 RISC-V 控制寄存器，为将来来自用户空间的陷阱做准备：把 `stvec` 设置为 `uservec`，并准备好 `uservec` 所依赖的 trapframe 字段。`prepare_return` 把 `sepc` 设置为先前保存的用户程序计数器。最后，`usertrap` 返回到 trampoline 页中的 [`userret`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trampoline.S#L101)，并通过 `a0` 传回指向用户页表的指针。

`userret` 把 `satp` 切换到该进程的用户页表。回想一下，用户页表既映射了 trampoline 页和 `TRAPFRAME`，但不映射内核的其他任何内容。由于 trampoline 页在用户页表和内核页表中映射到相同的虚拟地址，`userret` 在修改 `satp` 之后仍能继续执行。从这时起，`userret` 能使用的数据只有寄存器内容和 trapframe 的内容。`userret` 把 `TRAPFRAME` 地址加载到 `a0`，通过 `a0` 从 trapframe 恢复保存的用户寄存器，恢复保存的用户 `a0`，并执行 `sret` 返回用户空间。

`uservec` 和 `userret` 用汇编语言编写，因为很难用 C 代码来保存或恢复所有寄存器，或者安全地经历页表切换。

### 4.3 代码：调用系统调用

用户程序通过调用库函数来进行系统调用。例如，shell 用下面的函数调用显示提示符（在 `user/sh.c` 中）：

```c
write(2, "$ ", 2);
```

下面是 `user/usys.S` 中的库函数：

```asm
write:
 li a7, SYS_write
 ecall
 ret
```

C 编译器为这个函数调用生成的代码把三个参数加载到寄存器 `a0`、`a1` 和 `a2`。然后 `write()` 函数把系统调用号 `SYS_write`（16）加载到 `a7`。内核会查看这些寄存器，以弄清意图是哪个系统调用，以及参数是什么。`ecall` 指令从用户空间陷入内核，使 `uservec`、`usertrap` 以及随后的 `syscall` 执行。

此时，请阅读 [`kernel/syscall.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/syscall.c)、[`kernel/sysfile.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sysfile.c) 中的 `sys_write()`，以及 [`kernel/vm.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/vm.c) 中的 `copyout()`、`copyin()` 和 `copyinstr()`。

[`syscall`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/syscall.c#L137) 从 trapframe 中保存的 `a7` 取得系统调用号，并用它作为索引来查 [`syscalls`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/syscall.c#L109)。对于我们的例子，`a7` 中包含 [`SYS_write`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/syscall.h#L17)，因此会调用系统调用的实现函数 `sys_write`。

当 `sys_write` 返回时，`syscall` 把它的返回值记录在 `p->trapframe->a0` 中。这会使原先用户空间对 `write()` 的调用返回该值，因为 RISC-V 上的 C 调用约定把返回值放在 `a0` 中。系统调用按惯例返回负数表示错误，返回零或正数表示成功。

### 4.4 代码：系统调用参数

系统调用参数最初存放在用户寄存器中，然后由内核陷阱代码移动到 trap frame。内核函数 `argint`、`argaddr` 和 `argfd` 分别以整数、指针或文件描述符的形式，从 trap frame 中取出第 *n* 个系统调用参数。

有些系统调用把指针作为参数传递，内核必须使用这些指针来读取或写入用户内存。例如，`write` 系统调用把一个指向待写入数据的用户空间指针传递给内核。这样的指针带来两个挑战。首先，用户程序可能是有缺陷的或恶意的，可能把一个无效的指针传递给内核，或者传递一个旨在诱骗内核访问内核内存而不是用户内存的指针。其次，xv6 内核页表映射与用户页表映射并不相同，所以内核不能使用普通指令从用户提供的地址加载或存储。

内核实现了在用户提供的地址与内核之间安全传输数据的函数。`fetchstr` 就是一个例子 [(syscall.c:26)](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/syscall.c#L26)。像 `exec` 这样的文件系统调用使用 `fetchstr` 从用户空间取出字符串形式的文件名参数。`fetchstr` 调用 `copyinstr` 来完成困难的工作。

[`copyinstr`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/vm.c#L412) 从用户页表 `pagetable` 中的虚拟地址 `srcva` 复制最多 `max` 字节到 `dst`。由于 `pagetable` *不是*当前页表，`copyinstr` 使用 `walkaddr`（它调用 `walk`）在 `pagetable` 中查找 `srcva`，得到物理地址 `pa0`。内核页表把所有物理 RAM 映射到与 RAM 物理地址相等的虚拟地址。这使得 `copyinstr` 可以直接把字符串字节从 `pa0` 复制到 `dst`。[`walkaddr`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/vm.c#L122) 检查用户提供的虚拟地址是否属于该进程的用户地址空间，所以程序不能诱骗内核读取其他内存。一个类似的函数 `copyout` 把数据从内核复制到用户提供的地址。

### 4.5 来自内核空间的陷阱

请阅读 [`kernel/kernelvec.S`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/kernelvec.S)，以及 [`kernel/trap.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c) 中的 `kerneltrap()`。

xv6 处理来自内核代码的陷阱的方式与处理来自用户代码的陷阱不同。进入内核时，`usertrap` 把 `stvec` 指向 [`kernelvec`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/kernelvec.S#L12) 处的汇编代码。由于 `kernelvec` 只在 xv6 已经处于内核中时才执行，它可以依赖 `satp` 已被设置为内核页表，也可以依赖栈指针指向一个有效的内核栈。`kernelvec` 把所有调用者保存寄存器压入当前栈，之后从那里恢复它们，以便被中断的内核代码能不受干扰地恢复执行。

`kernelvec` 把寄存器保存在被中断的内核线程的栈上，这是合理的，因为寄存器值属于那个线程。如果陷阱导致切换到另一个线程，这一点尤其重要——在这种情况下，陷阱实际上会从新线程的栈返回，而把被中断线程的已保存寄存器安全地留在它自己的栈上。

`kernelvec` 在保存寄存器后跳转到 [`kerneltrap`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c#L137)。`kerneltrap` 只为一种类型的陷阱做准备：设备中断。它调用 [`devintr`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c#L188) 来处理这些中断。如果陷阱不是设备中断，那它一定是异常，例如内核代码试图使用无效指针。这只可能是内核代码中的缺陷造成的。内核在这种情况下没有办法恢复，所以它调用 `panic()`，后者打印一条错误信息然后停机。

如果 `kerneltrap` 是因为定时器中断而被调用，并且正在运行的是某个进程的内核线程（而不是调度器线程），`kerneltrap` 会调用 `yield`，给其他线程运行的机会。在某个时刻，其中一个线程会让出，让我们这个线程及其 `kerneltrap` 再次恢复执行。第 8 章解释了 `yield` 中发生的事情。

当 `kerneltrap` 的工作完成时，它需要返回到被陷阱中断的代码。因为 `yield` 可能已经破坏了 `sepc` 以及 `sstatus` 中保存的先前模式，`kerneltrap` 在开始时保存它们。现在它恢复这些控制寄存器，并返回到 [`kernelvec`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/kernelvec.S#L38)。`kernelvec` 从栈中弹出保存的寄存器并执行 `sret`，`sret` 把 `sepc` 复制到 `pc`，恢复被中断的内核代码。

当 CPU 从用户空间进入内核时，xv6 把该 CPU 的 `stvec` 设置为 `kernelvec`；你可以在 [`usertrap`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c#L47) 中看到这一点。但存在一个时间窗口：内核已经开始执行，而 `stvec` 仍被设置为 `uservec`，在这个窗口期内绝不能发生设备中断，这一点至关重要。幸运的是，RISC-V 在开始处理陷阱时总是禁用中断，而 `usertrap` 直到设置好 `stvec` 之后才会重新启用中断。

### 4.6 现实世界

如果内核内存被映射到每个进程的用户页表中（同时清除 `PTE_U` 位），就可以消除对 trampoline 页的需求。这也会消除从用户空间陷入内核时切换页表的需求。这反过来又会让内核中的系统调用实现能够利用当前进程的用户内存已被映射这一优势，使内核代码可以直接解引用用户指针。许多操作系统利用这些想法来提高效率。xv6 回避了这些做法，以减少因无意使用用户指针而导致内核出现安全漏洞的可能性，并减少为确保用户和内核虚拟地址不重叠所需的一些复杂性。

### 4.7 练习

1. `trampoline.S` 和 `kernelvec.S` 中的部分或全部代码能否用 C 而不是汇编来编写？
2. 有没有办法消除每个用户地址空间中特殊的 `TRAPFRAME` 页映射？例如，能否修改 `uservec`，让它简单地把 32 个用户寄存器压入内核栈，或者把它们保存到 `proc` 结构中？
3. 能否修改 xv6，以消除特殊的 `TRAMPOLINE` 页映射？
