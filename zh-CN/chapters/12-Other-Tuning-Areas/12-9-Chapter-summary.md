## 章节总结


* 不同厂商的处理器并不相同。它们在所支持的指令集架构（ISA，Instruction Set Architecture）和微架构（microarchitecture）实现上存在差异。达到峰值性能通常需要利用最新的 ISA 扩展，并针对特定的 CPU 微架构对应用程序进行调优。
* CPU 分发（CPU dispatching）是一种能够引入平台特定优化的技术。借助该技术，您可以为特定微架构提供快速路径，同时为其他平台保留通用实现。
* 我们探讨了几种由应用程序与 CPU 微架构交互所引起的性能边角情况（performance corner cases），包括内存顺序违规（memory ordering violations）、内存未对齐访问（misaligned memory accesses）、缓存别名（cache aliasing）以及非规格化浮点数（denormal floating-point numbers）。
* 我们还讨论了几种低延迟调优技术（low-latency tuning techniques），这些技术对于需要快速响应时间的应用程序至关重要。我们展示了如何在关键路径上避免页错误、缓存未命中（cache misses）、TLB 击落（TLB shootdowns）以及核心降频（core throttling）。
* 系统调优（System tuning）是最后一块拼图。某些旋钮和设置可能会影响应用程序的性能。确保系统固件（firmware）、操作系统或内核不会破坏应用程序调优工作的所有成果，这一点至关重要。

