## 本章小结


* 我们快速概述了三个主要平台上最流行的工具：Linux、Windows 和 macOS。根据 CPU 厂商的不同，性能分析工具的选择也会有所不同。对于搭载 Intel 处理器的系统，我们推荐使用 VTune；对于搭载 AMD 处理器的系统，使用 uProf；在 Apple 平台上使用 Xcode Instruments。
* Linux perf 可能是 Linux 上使用最频繁的性能分析工具。它支持所有主要 CPU 厂商的处理器。它没有图形界面。然而，有工具可以可视化 `perf` 的性能分析数据。
* 我们还讨论了 Windows 事件追踪（ETW），它旨在观察运行中系统的软件动态。Linux 有一个类似的工具叫做 [KUtrace](https://github.com/dicksites/KUtrace)，[^1] 在书籍 [DickSitesBook] 中有所介绍。
* 有混合分析器（hybrid profilers）结合了代码插桩、采样和追踪等技术。这集合了这些方法的优点，允许用户获取关于特定代码段的非常详细的信息。在本章中，我们研究了 Tracy，它在游戏开发者中相当流行。
* 内存分析器提供关于内存使用情况、堆分配、内存占用和其他指标的信息。内存性能分析帮助你了解应用程序随时间如何使用内存。
* 持续性能分析工具已经成为监控生产环境性能不可或缺的一部分。它们以调用栈为单位收集系统级性能指标，持续数天、数周乃至数月。这类工具更容易发现性能变化开始的时间点，并确定问题的根本原因。

[^1]: KUtrace - [https://github.com/dicksites/KUtrace](https://github.com/dicksites/KUtrace)

