# PCL Rust 科普准备

## Log

### 2020-11-13

新的风暴又已经来了吗？哭哭哭

大概需要从下面几个方面来进行说明吧，而且基本上在嵌入式的角度，和 C 进行比较。

* 性能方面：包括执行性能和编译性能（指可执行文件的大小）
* 编程方面提供的方便：Rust 自身的语法、库生态
* 相关可用的 IDE/工具

大概总时长需要 1 小时吧。我说这也有点恐怖了。

好。下面开始疯狂收集素材。

### 2020-11-15

陈老师知道了情况之后，和杨沙洲老师讨论了一下，最后决定下周二下午五点给一个 20 分钟的简单宣讲即可。

素材就是陈老师在华为做的长达两个小时报告的 PPT，还有张汉东老师做的《关于 Rust 你需要了解的》讲座，这个我之前看过并留下了详细笔记。

尽管如此，我觉得 cs110L 和 Rust Embedded Book 作为我自身的学习材料仍然是不可错过的，所以今天还是看这些材料，能看多少就看多少。

## 素材-性能比较

[一篇比较 Rust 和 C 性能的 Blog](https://kornel.ski/rust-c-speed)

[另一篇支持 Rust 某种程度上没有那么好的 Blog](https://www.viva64.com/en/b/0733/)

[C++ 和 Rust 的运行性能比较](https://benchmarksgame-team.pages.debian.net/benchmarksgame/fastest/gpp-rust.html)

## 素材-嵌入式 Rust 生态

[awesome-embedded-rust 主页](https://github.com/rust-embedded/awesome-embedded-rust)

> 里面包含很多嵌入式 Rust 的库。

[rtic 官方文档](https://rtic.rs/0.5/book/en/)

[rust embedded book](https://rust-embedded.github.io/book/)

## 素材-不局限于 no_std 的生态

## 素材-Rust 和 C 进行比较

[一篇比较详细的比较 C/Rust 的 Blog](https://medium.com/better-programming/in-depth-comparison-of-rust-and-cpp-579b1f93a5e9)

## 素材-rCore

## 素材-zCore

## 素材-可用的工具

## 素材-陈老师华为 PPT

C 语言的局限性：指针灵活但是非常不安全、缺乏有效的并发支持

当前 OS 的困难 1：内核膨胀，bug 增加，难以发现

比如：内存安全 bg（如使用空指针、use-after-free）、并发相关 bug（如死锁、数据竞争）

当前 OS 的困难 2：不能像开发应用一样开发内核（调试工具缺乏、移到用户态会降低性能）

用高级语言写 OS：Biscuit OS in Golang(2018)，Tock OS in Rust(2017)

[ixy](https://github.com/ixy-languages/ixy-languages) 是一个用多种不同语言开发的网卡的用户态驱动，并进行了性能的比较，这是相关的[论文](https://www.net.in.tum.de/fileadmin/bibtex/publications/papers/the-case-for-writing-network-drivers-in-high-level-languages.pdf)。从中可以看出 Rust 和 C 几乎相同（特别是并发程度提高之后）并领先于其他编程语言。

Tsinghua 的尝试：rCore/zCore

## 素材-张汉东老师讲座@上海科技大学

## 素材-张汉东老师知乎 Live

## 提纲（暂定）

### Rust 的安全性

比如：内存安全、并发安全、类型安全。

### Rust 的高性能

可能来源于：没有 GC；零成本抽象，运行时比较小。

### Rust 的易用性

### Rust 的嵌入式生态

### Rust IDE/相关工具简介

