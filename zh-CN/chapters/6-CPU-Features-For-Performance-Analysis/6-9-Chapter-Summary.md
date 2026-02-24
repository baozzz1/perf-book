## 本章总结 {.unlisted .unnumbered}

\markright{Summary}

* 现代处理器提供了增强性能分析的特性。利用这些特性，可以大大简化发现底层优化机会的过程。
* 自顶向下微架构分析（Top-down Microarchitecture Analysis，TMA）方法论是一种用于识别程序对 CPU 微架构低效使用的强大技术，即使是经验不足的开发者也易于上手。TMA 是一个迭代过程，由多个步骤组成，包括描述工作负载特征和精确定位源代码中发生瓶颈的位置。我们建议将 TMA 作为每次底层调优工作的起点之一。
* 分支记录机制（Branch Record mechanism），如 Intel 的 LBR、AMD 的 LBR 和 ARM 的 BRBE，在程序执行的同时持续记录最近的分支结果，造成极小的减速。这些设施的主要用途之一是收集调用栈。此外，它们还有助于识别热分支、预测错误率，并支持机器码的精确计时。
* 现代处理器通常提供基于硬件的采样（Hardware-Based Sampling）特性，用于高级剖析。这些特性通过在专用缓冲区中存储多个样本（无需软件中断）来降低采样开销。它们还引入了"精确事件"（Precise Events），能够精确定位导致特定性能事件的确切指令。此外，还有其他一些不太常用的使用场景。此类基于硬件的采样特性的典型实现包括 Intel 的 PEBS、AMD 的 IBS 和 ARM 的 SPE。
* Intel 处理器追踪（Intel Processor Traces，PT）是一种 CPU 特性，通过将程序执行编码为高度压缩的二进制格式数据包来记录程序执行过程，可用于重建每条指令附带时间戳的执行流。PT 覆盖范围广，开销相对较小。其主要用途是事后分析（postmortem analysis）和查找性能故障的根因。Intel PT 特性在附录 C 中介绍。基于 ARM 架构的处理器也具有追踪能力，称为 Arm [CoreSight](https://developer.arm.com/ip-products/system-ip/coresight-debug-and-trace)，[^2] 但它主要用于调试而非性能分析。

[^2]: Arm CoreSight - [https://developer.arm.com/ip-products/system-ip/coresight-debug-and-trace](https://developer.arm.com/ip-products/system-ip/coresight-debug-and-trace)

\sectionbreak
