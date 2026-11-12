# xv6-riscv-book-cn

本项目是 MIT 操作系统课程教材《xv6: a simple, Unix-like teaching operating system》（简称 xv6 book）的中文翻译。

本版依据官方最新版在线书稿（[https://mit-pdos.github.io/xv6-riscv-book/](https://mit-pdos.github.io/xv6-riscv-book/)）重新整理翻译：

- 基于 [zhenyu-zang/xv6-riscv-book-Chinese](https://github.com/zhenyu-zang/xv6-riscv-book-Chinese), [pleasewhy/xv6-book-2020-Chinese](pleasewhy/xv6-book-2020-Chinese) 校对修改；
- 新版新增或改写的章节（第 5 章 缺页异常、第 9 章 睡眠与唤醒、第 11 章 日志等）使用 LLM 按原译文风格新译并人工核验；
- 各章节的图片来源于官方在线书稿。

## 目录

- [前言与致谢（Foreword and acknowledgments）](acks.md)
- [第一章：操作系统接口（Operating system interfaces）](01-unix.md)
- [第二章：操作系统组织（Operating system organization）](02-first.md)
- [第三章：页表（Page tables）](03-mem.md)
- [第四章：陷阱和系统调用（Traps and system calls）](04-trap.md)
- [第五章：缺页异常（Page faults）](05-pgfault.md)
- [第六章：中断和设备驱动（Interrupts and device drivers）](06-interrupt.md)
- [第七章：锁（Locking）](07-lock.md)
- [第八章：调度（Scheduling）](08-sched.md)
- [第九章：睡眠与唤醒（Sleep and Wakeup）](09-sleep.md)
- [第十章：文件系统（File system）](10-fs.md)
- [第十一章：日志（Logging）](11-log.md)
- [第十二章：再论并发（Concurrency revisited）](12-lock2.md)
- [第十三章：总结（Summary）](13-sum.md)
- [参考文献（References）](references.md)

## 参考

- [xv6-riscv-book 官方在线版](https://mit-pdos.github.io/xv6-riscv-book/)
- [xv6-riscv GitHub 仓库](https://github.com/mit-pdos/xv6-riscv)
- [MIT 6.1810 课程](https://pdos.csail.mit.edu/6.1810)
