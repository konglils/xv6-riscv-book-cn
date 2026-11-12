# xv6：一个简单的、类 Unix 教学操作系统

## 前言与致谢

这是一份为操作系统课程准备的草稿文本。它通过研究一个名为 xv6 的示例内核，来解释操作系统的主要概念。xv6 以 Dennis Ritchie 和 Ken Thompson 的 Unix 第 6 版（v6）（[Ritchie and Thompson 1974 [15]](references.md#ref15)）为蓝本。xv6 大致遵循 v6 的结构和风格，但它是用 ANSI C（[Kernighan 1988 [4]](references.md#ref4)）为多核 RISC-V（[Patterson and Waterman 2017 [13]](references.md#ref13)）实现的。

阅读本文时应同时阅读 xv6 的源代码，这一做法受到 John Lions 的《Commentary on UNIX 6th Edition》（[Lions 2000 [9]](references.md#ref9)）的启发；文中带有指向 [https://github.com/mit-pdos/xv6-riscv](https://github.com/mit-pdos/xv6-riscv) 处源代码的超链接。关于 v6 和 xv6 的在线资源（包括若干使用 xv6 的实验作业）的更多指引，请参见 [https://pdos.csail.mit.edu/6.1810](https://pdos.csail.mit.edu/6.1810)。

我们已在 MIT 的操作系统课程 6.828 和 6.1810 中使用本文。我们感谢这些课程的教师、助教和学生，他们都直接或间接地为 xv6 做出了贡献。我们尤其感谢 Adam Belay、Austin Clements 和 Nickolai Zeldovich。最后，我们感谢以下通过电子邮件向我们报告文中错误或提出改进建议的人：Abutalib Aghayev、Sebastian Boehm、brandb97、Anton Burtsev、Raphael Carvalho、Tej Chajed、Brendan Davidson、Rasit Eskicioglu、Color Fuzzy、Wojciech Gac、Giuseppe、Tao Guo、Haibo Hao、Naoki Hayama、Chris Henderson、Robert Hilderman、Eden Hochbaum、Wolfgang Keller、Paweł Kraszewski、Henry Laih、Jin Li、Austin Liew、lyazj@github.com、Pavan Maddamsetti、Jacek Masiulaniec、Michael McConville、m3hm00d、Mes0903、miguelgvieira、Mark Morrissey、Muhammed Mourad、Harry Pan、Harry Porter、pr3pony、Siyuan Qian、Zhefeng Qiao、Askar Safin、Salman Shah、Huang Sha、Vikram Shenoy、Adeodato Simó、Ruslan Savchenko、Pawel Szczurko、Warren Toomey、tyfkda、tzerbib、unicornx、Vanush Vaswani、Chen Wang、Xi Wang、Zou Chang Wei、Sam Whitlock、Qiongsi Wu、LucyShawYang、ykf1114@gmail.com 以及 Meng Zhou。

如果你发现错误或有改进建议，请发送电子邮件给 Frans Kaashoek 和 Robert Morris（kaashoek,rtm@csail.mit.edu）。
