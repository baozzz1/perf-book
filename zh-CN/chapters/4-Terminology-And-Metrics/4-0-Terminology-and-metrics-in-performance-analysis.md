# 性能分析中的术语与指标

与许多工程学科一样，性能分析（performance analysis）领域使用大量专业术语和指标。对于初学者来说，阅读 Linux `perf` 或 Intel VTune Profiler 等分析工具生成的性能报告可能会非常困难。这些工具涉及许多复杂的术语和指标，然而，如果你决心从事严肃的性能工程（performance engineering）工作，这些指标是必须掌握的。

既然提到了 Linux `perf`，我先简要介绍一下这个工具，因为在本章及后续章节中我会大量使用它。Linux `perf` 是一款性能分析器（performance profiler），可用于查找程序中的热点（hotspot）、收集各种底层 CPU 性能事件、分析调用栈，以及完成许多其他任务。我将在全书中广泛使用 Linux `perf`，因为它是最流行的性能分析工具之一。我偏好展示 Linux `perf` 的另一个原因是它是开源软件，这使得富有探索精神的读者可以深入了解现代分析工具的内部机制。这对于学习本书所介绍的概念尤为有用，因为基于 GUI 的工具（如 Intel® VTune™ Profiler）往往会隐藏所有的复杂性。我们将在 [secOverviewPerfTools] 中对 Linux `perf` 进行更详细的概述。

本章是对性能分析中基本术语和指标的入门介绍。我们将首先定义基本概念，如已退休/已执行指令（retired/executed instructions）、IPC/CPI、μops、核心/参考时钟（core/reference clocks）、缓存未命中（cache misses）和分支预测错误（branch mispredictions）。然后我们将了解如何测量系统的内存延迟（memory latency）和带宽（bandwidth），并介绍一些更高级的指标。最后，我们将对四个行业工作负载进行基准测试，并查看收集到的指标。
