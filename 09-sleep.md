## 第九章：睡眠与唤醒

调度和锁有助于把一个线程的行为对另一个线程隐藏起来，但我们还需要一些帮助线程有意进行交互的抽象。例如，xv6 中管道的读取者可能需要等待某个写入进程产生数据；父进程对 `wait` 的调用可能需要等待子进程退出；读取磁盘的进程需要等待磁盘硬件完成读取。xv6 内核在这些情况（以及许多其他情况）中使用一种称为睡眠与唤醒的机制。睡眠允许一个内核线程等待某个*条件*变为真；另一个线程或中断处理程序可以使该条件变为真（通常是通过修改某些变量），然后调用 `wakeup` 来表示等待该条件的线程应当恢复执行。睡眠与唤醒通常被称为*序列协调（sequence coordination）*或*条件同步（conditional synchronization）*机制。

在继续阅读本章之前，请先阅读 [`kernel/proc.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c) 中的 `sleep_prepare()`、`sleep()` 和 `wakeup()` 函数，以及 [`kernel/pipe.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/pipe.c) 的全部内容。

### 9.1 概述

睡眠/唤醒接口如下所示：

```c
void sleep_prepare(void *chan)
void sleep()
void wakeup(void *chan)
```

进程睡眠是为了等待某个条件变为真，`chan` 参数（即*等待通道（wait channel）*）指明了所等待的条件。`sleep_prepare(chan)` 声明某个进程可能很快会为了等待指定的等待通道而睡眠；它立即返回。`wakeup(chan)` 在 `chan` 所指明的条件已经（或可能已经）变为真时被调用；它唤醒所有先调用了 `sleep_prepare(chan)`、随后又以同一个 `chan` 调用 `sleep()` 的进程。如果在此前（必需的）那次 `sleep_prepare(chan)` 调用之后，`wakeup(chan)` 已经被调用过，那么 `sleep()` 会立即返回；否则，`sleep()` 把调用进程标记为 `SLEEPING`（而不是 `RUNNABLE`），并通过上下文切换到调度器来释放 CPU，使其他进程得以运行。`sleep_prepare()` 和 `sleep()` 由同一个进程调用（通常相隔几行）；`wakeup()` 由另一个进程或中断处理程序调用。

这些函数把 `chan` 当作一个不透明的 64 位值：它们对它做的唯一一件事就是比较是否相等。通常的用法是，调用者把某个方便对象的地址作为 `chan` 参数传入。选择等待通道的约定，以及在应当相互作用的 `sleep_prepare()` 和 `wakeup()` 调用中传入同一个 `chan`，是内核程序员的责任。

内核代码睡眠是为了等待某个条件变为真。例如，从管道读取数据的内核代码在管道缓冲区当前为空时会睡眠；这里的条件是管道缓冲区变为非空（由另一个进程向管道写入造成）。`sleep_prepare()` 和 `sleep` 并不知道如何检查条件：只有调用代码知道。通常的模式是，可能进入睡眠的进程先获取保护该条件的锁，然后检查条件；如果条件不成立，就调用 `sleep_prepare()`、释放锁，再调用 `sleep()`。此后使条件变为真的代码调用 `wakeup`。

下面是 xv6 内核中管道代码如何睡眠的一种概括：

```c
piperead(pipe){
  acquire(&pipe->lock);
  while(there's no data in pipe->buffer){
    sleep_prepare(&pipe);
    release(&pipe->lock);
    sleep();
    acquire(&pipe->lock);
  }
  remove the data from the pipe;
  release(&pipe->lock);
}

pipewrite(pipe){
  acquire(&pipe->lock);
  append data to pipe->buffer;
  wakeup(&pipe);
  release(&pipe->lock);
}
```

这段代码把管道数据结构的地址用作等待通道。

`sleep_prepare(chan)` 必须在检查条件之前调用，或者在释放保护该条件的锁之前调用。该调用记录下这个进程可能需要在指定的 `chan` 上睡眠这一事实。如果另一个进程（或中断）在 `sleep_prepare(chan)` 和 `sleep()` 的两次调用之间调用了 `wakeup(chan)`，那么 `wakeup()` 会在即将睡眠的进程里设置一个标志，表示它不应当真的睡眠。如果 `sleep()` 看到这个标志，它就返回，而不是睡眠。

为什么需要 `sleep_prepare(chan)`？在管道读取者看到缓冲区为空之后、在它调用 `sleep` 之前，条件有可能已经变为真，并且 `wakeup()` 已经被调用。如果不做点巧妙处理，这个 `wakeup()` 就找不到任何正在睡眠的进程，也就不会产生任何效果。随后该进程还是会调用 `sleep()`，尽管管道缓冲区中已经有数据可读；而如果（情况可能正是如此）管道写入者此后再也没有写入任何数据，读取者就可能错误地永远睡眠下去。这种不希望出现的情形称为*丢失唤醒（lost wake-up）*。上一段描述的行为避免了这个问题。

在管道这个例子中，和大多数睡眠代码一样，有一把锁保护条件，并阻止 `wakeup()` 在即将睡眠的进程检查条件之后、调用 `sleep_prepare()` 之前被调用。这把锁通常称为*条件锁（condition lock）*。进程必须在调用 `sleep()` 之前释放该锁（以便其他进程或中断处理程序可以修改条件并调用 `wakeup()`），并在 `sleep()` 返回之后重新获取该锁。

### 9.2 代码：睡眠与唤醒

xv6 的 [`sleep_prepare`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L547)、[`sleep`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L562) 和 [`wakeup`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L576) 实现了上面例子中使用的接口。其基本思想是让 `sleep` 把当前进程标记为 `SLEEPING`，然后调用 `sched` 让出 CPU；`wakeup` 则寻找在给定等待通道上睡眠的进程，并把它们标记为 `RUNNABLE`。

`sleep_prepare(chan)` 把等待通道记录在 `p->chan` 中，然后返回。如果调用 `sleep()` 时 `p->chan` 仍然非零，`sleep()` 就把调用进程的状态改为 `SLEEPING`，并调用 `sched()` 让其他进程运行。

另一个进程可能正在并发地进入在 `chan` 上睡眠的过程，`wakeup(chan)` 要处理两种情形。如果那个进程已经走到把自己标记为 `SLEEPING` 这一步，那么 `wakeup` 会看到它处于该状态，并把它改为 `RUNNABLE`；于是它对 `sleep()` 的调用就会返回。如果那个进程已经走到调用 `sleep_prepare(chan)` 这一步（但它的 `sleep()` 还没走到获取 `p->lock` 那一步），`wakeup(chan)` 会看到 `p->chan` 等于 `chan`，于是把 `p->chan` 置零。在这种情况下，等那个进程真的调用 `sleep` 时，它会看到这个零值并立即返回。这样做的目的，是避免在另一个进程正要调用 `sleep()` 时调用 `wakeup()` 而造成丢失唤醒。

`sleep_prepare()` 的使用规则使我们不必去操心另一种情形：即将睡眠的进程还没有调用 `sleep_prepare()`。要么该进程用一把锁把它的条件检查和 `sleep_prepare()` 包起来，而这把锁阻止任何针对相关通道的 `wakeup()` 调用；要么该进程必须在检查条件之前调用 `sleep_prepare()`。

有时会有多个进程在同一个通道上睡眠；例如，有不止一个进程从管道中读取数据。一次 `wakeup` 调用会把它们全部唤醒。其中一个会先运行，（就管道而言）获取管道锁并读走正在等待的数据。其他进程会发现，尽管被唤醒了，却没有数据可读。从它们的角度看，这次唤醒是“虚假的”，它们必须再次睡眠。出于这个原因，睡眠总是发生在重新检查条件的循环内部，就像上面的 `piperead` 那样。

如果两处睡眠/唤醒的用法不小心选到了同一个通道，也不会有害：它们会看到虚假的唤醒，但上面描述的这种循环可以容忍这个问题。睡眠/唤醒的魅力很大一部分在于，它既是轻量级的（不需要创建特殊的数据结构来充当等待通道），又提供了一层间接性（调用者不需要知道自己在与哪个具体的进程交互）。

### 9.3 代码：管道

xv6 的管道是一个用睡眠来同步生产者和消费者的代码例子。我们在第1章看到了管道的接口：写入管道一端的字节被复制到内核缓冲区，然后可以从管道的另一端读取。后续章节将考察管道周围的文件描述符支持，但我们现在先来看看 `pipewrite` 和 `piperead` 的实现。

每个管道由一个 `struct pipe` 表示，其中包含一个 `lock` 和一个 `data` 缓冲区。`nread` 和 `nwrite` 两个字段分别统计从缓冲区读取和向缓冲区写入的字节总数。缓冲区是环形的：在 `buf[PIPESIZE-1]` 之后写入的下一个字节是 `buf[0]`。这两个计数不回绕。这个约定使实现能够区分满缓冲区（`nwrite == nread+PIPESIZE`）和空缓冲区（`nwrite == nread`），但这也意味着对缓冲区做索引时必须用 `buf[nread % PIPESIZE]`，而不能只用 `buf[nread]`（`nwrite` 同理）。

假设对 `piperead` 和 `pipewrite` 的调用同时发生在两个不同的 CPU 上。[`pipewrite`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/pipe.c#L77) 首先获取管道的锁，这个锁保护计数、数据以及它们相关的不变量。[`piperead`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/pipe.c#L112) 随后也试图获取这个锁，但获取不到。它在 [`acquire`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/spinlock.c#L22) 中自旋，等待锁。在 `piperead` 等待期间，`pipewrite` 循环处理要写入的字节（`addr[0..n-1]`），依次把每个字节添加到管道中 [(pipe.c:101)](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/pipe.c#L101)。在这个循环过程中，可能会发生缓冲区被填满的情况 [(pipe.c:88)](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/pipe.c#L88)。在这种情况下，`pipewrite` 调用 `wakeup`，提醒任何正在睡眠的读取者缓冲区中有数据在等待，然后释放锁，在 `&pi->nwrite` 通道上睡眠，等待某个读取者从缓冲区中取走一些字节。

`piperead` 现在获取了管道的锁并进入它的临界区：它发现 `pi->nread != pi->nwrite` [(pipe.c:119)](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/pipe.c#L119)（`pipewrite` 之所以睡眠，是因为 `pi->nwrite == pi->nread + PIPESIZE` [(pipe.c:88)](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/pipe.c#L88)），于是进入 `for` 循环，把数据从管道中复制出来 [(pipe.c:129)](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/pipe.c#L129)，并按复制的字节数增加 `nread`。缓冲区中这么多的空间现在可以用来写入了，所以 `piperead` 在返回之前调用 [`wakeup`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/pipe.c#L140) 来唤醒任何正在睡眠的写入者。`wakeup` 找到一个在 `&pi->nwrite` 上睡眠的进程（也就是原先运行 `pipewrite`、但在缓冲区填满时停下来的那个进程）。`wakeup` 把该进程标记为 `RUNNABLE`。

管道代码为读取者和写入者使用不同的等待通道（`&pi->nread` 和 `&pi->nwrite`）；在有大量读取者和写入者等待同一个管道的情况下，这可能会使系统更有效率。管道代码在检查睡眠条件的循环内部睡眠；如果有多个读取者或写入者，除第一个被唤醒的进程外，其他进程都会发现条件为假，于是再次睡眠。

### 9.4 代码：wait、exit 和 kill

请阅读 [`kernel/proc.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c) 中 `kwait()`、`kexit()` 和 `kkill()` 函数的代码；它们是对应系统调用的内部实现。

第1章中介绍的一个关于睡眠的有趣例子，是子进程的 `exit` 与父进程的 `wait` 之间的交互。在子进程死亡的时候，父进程可能已经在 `wait` 中睡眠，也可能在做别的事情；在后一种情况下，后续的 `wait` 调用必须观察到子进程的死亡，而这也许发生在子进程调用 `exit` 之后很久。xv6 在 `wait` 观察到子进程死亡之前记录这一事实的方式，是让 `exit` 把调用进程置为 `ZOMBIE` 状态，它会在那里停留，直到父进程的 `wait` 注意到它、把子进程的状态改为 `UNUSED`、复制子进程的退出状态，并把子进程的进程 ID 返回给父进程。如果父进程比子进程先退出，父进程就把子进程交给 `init` 进程，而 `init` 进程会永远循环地调用 `wait`；这样每个子进程都有一个父进程来为它收尾。一个挑战是避免并发的父进程 `wait` 与子进程 `exit`、以及并发的 `exit` 与 `exit` 之间的竞争和死锁。

`kwait` 是 `wait` 的内核实现，它首先获取 [`wait_lock`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L370)，这个锁充当条件锁，帮助确保 `kwait` 不会错过正在退出的子进程发出的唤醒。然后 `kwait` 扫描进程表。如果它发现一个处于 `ZOMBIE` 状态的子进程，它就释放该子进程的资源及其 `proc` 结构，把子进程的退出状态复制到提供给 `wait` 的地址（如果该地址不为 0），并返回子进程的进程 ID。如果 `kwait` 找到了子进程但没有一个已经退出，它就睡眠，等待其中任何一个退出 [(proc.c:414)](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L414)，然后再次扫描。`kwait` 经常持有两把锁：`wait_lock` 和某个进程的 `pp->lock`；避免死锁的顺序是先 `wait_lock`，再 `pp->lock`。

[`kexit`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L326) 记录退出状态，释放一些资源，调用 `reparent` 把所有子进程交给 `init` 进程，在父进程正处于 `wait` 中时唤醒它，把调用进程标记为僵尸，并永久让出 CPU。`kexit` 在这个序列中同时持有 `wait_lock` 和 `p->lock`。它持有 `wait_lock`，因为这是 `wakeup(p->parent)` 的条件锁，可以防止处于 `wait` 中的父进程丢失唤醒。`kexit` 在这个序列中还必须持有 `p->lock`，以防止处于 `wait` 中的父进程在子进程最终调用 `swtch` 之前，就看到子进程处于 `ZOMBIE` 状态。`kexit` 以与 `kwait` 相同的顺序获取这些锁，以避免死锁。

`kexit` 在把自己的状态设置为 `ZOMBIE` 之前就唤醒父进程，看起来可能不正确，但这是安全的：尽管 `wakeup` 可能使父进程运行起来，父进程 `kwait` 中的循环却无法检查子进程，直到子进程的 `p->lock` 被 `scheduler` 释放为止；因此 `kwait` 无法查看正在退出的进程，直到 `kexit` 已经把它的状态设置为 [`ZOMBIE`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L358)。

`exit` 允许一个进程自行终止，而 `kill` 系统调用 [(proc.c:601)](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L601) 则允许一个进程请求另一个进程终止。让 `kill` 直接摧毁受害进程会过于复杂，因为受害进程可能正在另一个 CPU 上执行，也许正处于更新内核数据结构的某个敏感序列中间。因此 `kkill` 做的事情很少：它只设置受害进程的 `p->killed`，如果它正在睡眠，就唤醒它。最终受害进程会进入或离开内核，此时如果 `p->killed` 已被设置，`usertrap` 中的代码会调用 `kexit`（它通过调用 [`killed`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c#L633) 来检查）。如果受害进程正在用户空间运行，它会在下一次通过系统调用进入内核、或者因为定时器（或其他设备）中断而进入内核时，发现自己已经被杀死。

如果受害进程正在睡眠，`kkill` 会把它的状态改为 `RUNNABLE`，使受害进程从 `sleep` 返回。这是有潜在危险的，因为所等待的条件可能并不为真。然而，xv6 对 `sleep` 的调用总是被包裹在一个 `while` 循环中，在 `sleep` 返回之后重新检查条件。有些 `sleep` 调用还会在循环中检查 `p->killed`，如果它被设置就放弃当前活动。只有在这样放弃是正确的时候才会这样做。例如，管道读写代码 [(pipe.c:84)](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/pipe.c#L84) 在 killed 标志被设置时会返回；最终代码会返回到陷阱处理代码，陷阱处理代码会再次检查 `p->killed` 并退出。

有些 xv6 的 `sleep` 循环不检查 `p->killed`，因为代码正处在某个应当原子的多步骤系统调用的中间（也就是说，如果中途被放弃就会出错）。virtio 驱动 [(virtio_disk.c:287)](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/virtio_disk.c#L287) 就是一个例子：它不检查 `p->killed`，因为一次磁盘操作可能是一组写操作中的一次，而为了让文件系统处于正确状态，这些写操作全都需要完成。一个在等待磁盘 I/O 时被杀死的进程，要等到它完成当前系统调用、并且 `usertrap` 看到 killed 标志之后才会退出。

### 9.5 进程锁

与每个进程关联的锁（`p->lock`）是 xv6 中最复杂的锁。理解 `p->lock` 的一个简单方法是：在读取或写入以下任何一个 `struct proc` 字段时，都必须持有它：`p->state`、`p->chan`、`p->killed`、`p->xstate` 和 `p->pid`。这些字段可能被其他进程使用，也可能被其他 CPU 上的调度器线程使用，所以它们必须由一把锁来保护，这是很自然的。

然而，`p->lock` 的大多数用途是保护 xv6 进程数据结构和算法中更高层次的不变量。`p->lock` 所做的事情完整列表如下：

- 连同 `p->state` 一起，防止为新进程分配 `proc[]` 槽位时出现竞争。
- 在进程被创建或销毁的过程中把它隐藏起来，使它不被看到。
- 防止父进程的 `wait` 收走一个已经把自己的状态设置为 `ZOMBIE`、但还没有让出 CPU 的进程。
- 防止另一个 CPU 的调度器在一个让出 CPU 的进程已经把状态设置为 `RUNNABLE`、但还没有完成 `swtch` 的时候决定运行它。
- 确保只有一个 CPU 的调度器决定运行某个 `RUNNABLE` 进程。
- 防止定时器中断在进程处于 `swtch` 中时使它让出 CPU。
- 防止 `sleep` 和 `wakeup` 之间的竞争。
- 防止 `kill` 的受害进程在 `kkill` 检查 `p->pid` 之后、设置 `p->killed` 之前退出，并可能被重新分配。
- 使 `kkill` 对 `p->state` 的检查和写入成为原子操作。

`p->parent` 字段由全局锁 `wait_lock` 保护，而不是由 `p->lock` 保护。只有进程的父进程会修改 `p->parent`，不过该进程本身以及寻找自己子进程的其他进程都会读取这个字段。`wait_lock` 的作用是在 `wait` 睡眠等待任何子进程退出时充当条件锁。一个正在退出的子进程会一直持有 `wait_lock` 或 `p->lock`，直到它把状态设置为 `ZOMBIE`、唤醒父进程并让出 CPU 之后。`wait_lock` 还把父进程和子进程并发的 `exit` 串行化，从而保证（继承该子进程的）`init` 进程能从它的 `wait` 中被唤醒。`wait_lock` 是一个全局锁，而不是每个父进程各自持有的一把锁，因为一个进程在获取它之前，无法知道自己的父进程是谁。

### 9.6 现实世界

`sleep` 和 `wakeup` 是一种简单而有效的同步方法，但还有很多其他方法；信号量（semaphore）（[Dijkstra 1965 [2]](references.md#ref2)）就是一个例子。所有这些方法中的第一个挑战，都是避免我们在本章开头看到的“丢失唤醒”问题。最初的 Unix 内核的 `sleep` 只是禁用中断，这就足够了，因为 Unix 运行在单 CPU 系统上。Plan 9 的 `sleep` 使用一个回调函数，它就在进入睡眠之前、在持有调度锁的情况下运行；这个函数充当对睡眠条件的最后时刻检查，以避免丢失唤醒。Linux 内核的 `sleep` 使用一个显式的进程队列，称为等待队列，而不是等待通道；该队列有自己的内部锁。

在 `wakeup` 中扫描整个进程集合效率很低。更好的解决办法是把 `sleep` 和 `wakeup` 中的 `chan` 换成一个保存着在其上睡眠的进程列表的数据结构，比如 Linux 的等待队列。Plan 9 的 `sleep` 和 `wakeup` 把这种结构称为 rendezvous point。许多线程库把同一个结构称为条件变量；在这种语境下，`sleep` 和 `wakeup` 这两个操作被称为 `wait` 和 `signal`。所有这些机制都有相同的特点：睡眠条件由某种锁保护，而这把锁在睡眠过程中被原子地释放。

xv6 的 `wakeup` 会唤醒在某个特定等待通道上等待的所有进程。如果这样的进程不止一个，它们都会尝试获取条件锁并重新检查条件；在很多情况下，只有一个进程能做成有用的事（例如把管道中等待的所有数据读走）。其余进程会发现条件已不再为真，于是回去继续睡眠；唤醒它们白白浪费了 CPU 时间。因此，大多数条件变量的设计都提供两个原语：`signal`，它唤醒等待该条件变量的进程中的一个；以及 `broadcast`，它唤醒它们全部。

强制杀死进程会带来一些问题。例如，一个被杀的进程可能在内核深处睡眠，而解开它的栈需要小心处理，因为调用栈上的每个函数可能都需要做一些清理工作。有些语言通过提供异常机制来帮忙，但 C 语言没有提供。此外，还有其他一些事件可以导致一个睡眠进程被唤醒，即使它正在等待的事件还没有发生。例如，当一个 Unix 进程处于睡眠状态时，另一个进程可能会向它发送一个 `signal`。在这种情况下，该进程将从中断的系统调用中返回，返回值为 -1，错误代码被设置为 EINTR。应用程序可以检查这些值并决定做什么。xv6 不支持信号，因此不会出现这种复杂性。

xv6 对 `kill` 的支持并不完全令人满意：有些睡眠循环可能应该检查 `p->killed`。一个相关的问题是，即使是检查 `p->killed` 的 `sleep` 循环，`sleep` 和 `kill` 之间也存在竞争；后者可能在受害进程的循环检查完 `p->killed` 之后、但在它调用 `sleep` 之前，就设置 `p->killed` 并试图唤醒受害进程。如果出现这个问题，受害进程要等到它所等待的条件发生之后才会注意到 `p->killed`。这可能晚很多，甚至永远不会发生（例如，受害进程正在等待来自控制台的输入，而用户没有输入任何内容）。

### 9.7 练习

1. 在 xv6 中实现计数信号量。挑选 xv6 中几处使用睡眠和唤醒的地方，用信号量替换它们。判断结果。

2. 你能实现这样一种睡眠/唤醒方案吗：它只需要调用 `sleep()`（也许带有额外的参数），而不需要像调用 `sleep_prepare()` 这样的东西？

3. 修正上面提到的 `kill` 和 `sleep` 之间的竞争，使得发生在受害进程的睡眠循环检查 `p->killed` 之后、但在它调用 `sleep` 之前的 `kill`，会让受害进程放弃当前系统调用。

4. 设计一个方案，让每一个睡眠循环都检查 `p->killed`，这样，例如处于 virtio 驱动中的进程如果被另一个进程杀死，就能从 `while` 循环中快速返回。