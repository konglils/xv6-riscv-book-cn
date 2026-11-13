# 术语翻译表

| 术语 | 中文译名 | 说明 |
|---|---|---|
| absorption | 吸收 | 日志优化：同一事务内同一块多次写入合并为一个日志槽 |
| address space | 地址空间 |  |
| atomic instruction | 原子指令 | 如 `amoswap`、`lr/sc` 等 |
| bitmap | 位图 | 记录磁盘块分配情况 |
| block | 块 | 磁盘块 |
| buffer cache | 缓冲区缓存 | 缓存磁盘块并保证同一块只有一份副本 |
| clock tick | 时钟滴答 | 时钟中断的周期 |
| condition lock | 条件锁 | 保护睡眠条件的锁，`sleep` 前须释放 |
| condition variable | 条件变量 |  |
| context switch | 上下文切换 |  |
| copy-on-write (COW) | 写时复制 |  |
| crash recovery | 崩溃恢复 |  |
| critical section | 临界区 |  |
| demand paging | 按需分页 |  |
| evict | 踢出 | 把物理页换出到磁盘交换区 |
| exception | 异常 |  |
| fence | 栅栏 | 与“内存屏障（memory barrier）”同义 |
| file descriptor | 文件描述符 |  |
| free list | 空闲链表 |  |
| group commit | 组提交 | 把多个事务合并提交 |
| guard page | 保护页 | 权限为无效的页，用于捕捉越界访问 |
| heap | 堆 |  |
| inode | 索引节点 | 保留原文 |
| interrupt | 中断 |  |
| interrupt handler | 中断处理程序 |  |
| kernel | 内核 |  |
| kernel space | 内核空间 |  |
| kernel thread | 内核线程 |  |
| lazy allocation | 惰性分配 |  |
| lock | 锁 |  |
| log | 日志 |  |
| lost wake-up | 丢失唤醒 |  |
| machine mode | 机器模式 |  |
| memory barrier | 内存屏障 |  |
| memory-mapped | 内存映射 |  |
| microkernel | 微内核 |  |
| monolithic kernel | 宏内核 |  |
| page | 页 |  |
| page fault | 缺页异常 |  |
| page in / page out | 换入 / 换出 |  |
| page table | 页表 |  |
| page table entry (PTE) | 页表项 | 正文通常直接用 `PTE` |
| paging area | 交换区 | 磁盘上存放换出页的区域 |
| path name | 路径名 |  |
| permission bit | 权限位 |  |
| physical address | 物理地址 |  |
| PID (process identifier) | PID（进程标识符） | 通常直接用 `PID` |
| pipe | 管道 |  |
| privilege | 特权 |  |
| privileged instruction | 特权指令 |  |
| process | 进程 |  |
| race condition | 竞争条件 |  |
| reference count | 引用计数 |  |
| scheduler | 调度器 |  |
| server process | 服务器进程 |  |
| sleep lock | 睡眠锁 |  |
| spinlock | 自旋锁 |  |
| stack | 栈 |  |
| super page | 大页 | RISC-V 的大尺寸页（如 2 MiB、1 GiB），与 4 KiB 的普通页相对 |
| superblock | 超级块 |  |
| supervisor mode | 监管模式 |  |
| system call | 系统调用 |  |
| thread | 线程 |  |
| trampoline (page) | trampoline 页（跳板页） | 保留原文 |
| transaction | 事务 |  |
| trap | 陷阱 |  |
| trapframe | 陷阱帧 | 保留原文 |
| user mode | 用户模式 |  |
| user space | 用户空间 |  |
| virtual address | 虚拟地址 |  |
| wait channel | 等待通道 |  |
