# Performance Characterization of Modern Storage Stacks: POSIX I/O, libaio,SPDK,and io\_uring

## 摘要

Linux 存储栈提供了多种存储 io栈和 API，像 posix io，libaio， 高性能异步 io\_uring 或者 SPDK，最后一种完全的 bypass 了内核。除去他们的可用性，没有人系统的研究过他们的性能和资源消耗。为了增加我们的理解，我们系统的描述了性能，扩展性和在流行 Linux io api在高性能硬件上的微架构属性。我们的研究显示了：

1. &#x20;在低延时负载下，所有的 API相差不大，polling 可以提升1.7倍性能，但是消耗了2.3倍的 CPU指令数
2. 在高负载下，io\_uring 比 SPDK 慢一个数量级
3. 在高负载下，benchmark 工具 fio 本身会成为瓶颈
4. 现在 linux block io 调度器提升显著

## 研究背景和动机

现代存储设备（如 Intel Optane SSDs）能够提供百万IOPS和单微妙级别的 I/O 访问延迟。然而由于摩尔定律驱动的 CPU 性能提升趋于停滞，这种高性能存储硬件暴露了存储堆栈实现中许多以前隐藏的软件开销。为了理解这些开销并优化存储堆栈，论文系统地研究了不同I/O API 在高性能存储硬件上的性能和效率。

## 主要研究问题

1. 不同 I/O API 及其配置之间的性能差距是什么
2. 为什么会有性能差距，这些 API 如何使用 CPU 时间、周期和指令
3. 随着 CPU核心数量和存储设备数量的增加，性能差距如何变化
4. I/O调度程序对这些 I/O API性能的影响是什么？

## 研究方法

研究在一台配备7个 Intel Optane SSDs 和双路 Intel Xeon Silver 4210R CPU 的服务器上进行，测试包括 posix I/O，libai，io\_uring （包括默认配置，完成轮询，和提交轮询），SPDK。这些测试涵盖了低负载和高负载条件下的性能评估，并使用 fio 和 SPDK自带的 perf 工具进行基准测试。

## 主要发现

1. 低 I/O 负载下的性能：在单个 I/O请求和单个 CPU 核心的条件下，非轮询 API(posix，libaio，io\_uring)表现相近，但轮询能提高1.7倍性能，代价是消耗2.3倍的 CPU资源。
2. 高负载下的性能：在高负载（128 qd, 7 devices）下，io\_uring 的效率比 spdk 第一个数量级，SPDK 可以使用5个 core（fio）或者一个core（使用 spdk 的轻量级 perf benchmark）来打满硬件，而 io\_uring 需要13个 core
3. benchmark 工具的瓶颈：在高负载下，fio 本身成为了瓶颈，spdk 使用 fio 时需要5个 core 才能达到4.2m iops，而使用 spdk 自己的 perf benchmark只需要一个 core，fio 带来了5倍的开销
4. I/O调度程序的开销：当前的 Linux I/O调度程序（BFQ，mq-deadline，Kyber）引入了显著的（最高50%）开销，并且他们使用了全局锁阻碍了可扩展性

## 贡献

本文的主要贡献在于对 Linux 存储 API在现代高性能存储设备上的性能和可扩展性进行了详细的表征。所有的实验代码和数据均已开放，以便其他研究人员复现和进一步研究。

## 结论

不同 I/O API 在性能和效率上存在显著差异，特别是在高负载条件下。io\_uring 虽然引入了一些高性能特性，但在高负载下效率不如 SPDK。同时，现有的 I/O 调度程序在高性能存储硬件上表现出交大的开销和可扩展问题。

&#x20;
