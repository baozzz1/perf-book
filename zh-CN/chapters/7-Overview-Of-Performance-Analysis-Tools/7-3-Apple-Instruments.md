## Apple Xcode Instruments

在 macOS 上进行类似性能分析最便捷的方式是使用 Xcode Instruments。这是一款随 Xcode 免费提供的应用程序性能分析器和可视化工具。Instruments 分析器构建在 DTrace 追踪框架之上，该框架从 Solaris 移植到了 macOS。它拥有许多用于检查应用程序性能的工具，使我们能够完成其他分析器（如 Intel VTune）所能做的大部分基本操作。获取该分析器的最简单方式是从 Apple App Store 安装 Xcode。该工具无需配置，安装后即可使用。

在 Instruments 中，你可以使用称为仪器（instruments）的专用工具，随时间追踪应用程序、进程和设备的不同方面。Instruments 具有强大的可视化机制。它在分析时收集数据，并以实时方式向你呈现结果。你可以收集不同类型的数据并并排查看，这使你能够发现执行中的模式、关联系统事件，并找出非常微妙的性能问题。

在本章中，我们仅展示"CPU Counters"（CPU 计数器）仪器，这与本书最为相关。Instruments 还可以可视化 GPU、网络和磁盘活动，追踪内存分配和释放，捕获用户事件（如鼠标点击），提供功耗效率方面的洞察等等。你可以在 Instruments 的[文档](https://help.apple.com/instruments/mac/current)中了解更多相关用例。[^1]

### 可以用它做什么 {.unlisted .unnumbered}

- 访问 Apple 处理器上的硬件性能计数器。
- 查找程序中的热点及其调用栈。
- 将生成的 ARM 汇编代码与源代码并排检查。
- 针对时间线上选定的时间区间过滤数据。

### 不能用它做什么 {.unlisted .unnumbered}

与其他基于采样的分析器类似，Xcode Instruments 具有与 VTune 和 uProf 相同的盲点。

### 示例：分析 Clang 编译过程 {.unlisted .unnumbered}

在本示例中，我将展示如何在配备 M1 处理器、macOS 13.5.1 Ventura 和 16 GB RAM 的 Apple Mac mini 上收集硬件性能计数器。我选取了 LLVM 代码库中最大的文件之一，并使用 Clang C++ 编译器 15.0 版本对其编译过程进行了分析。

![Xcode Instruments：时间线与统计面板。](../../img/perf-tools/XcodeInstrumentsView.jpg)

以下是我使用的命令行：

```bash
$ clang++ -O3 -DNDEBUG -arch arm64 <other options ...> -c llvm/lib/Transforms/Vectorize/LoopVectorize.cpp
```

图 InstrumentsView 展示了 Xcode Instruments 的主时间线视图。此截图是在编译完成后截取的。稍后我们会回到它，但首先让我们展示如何启动分析会话。

首先，打开 *Instruments* 并选择 *CPU Counters* 分析类型。第一步需要配置采集。点击并按住红色目标图标（见图 InstrumentsView 中的①），然后从菜单中选择 *Recording Options...*。这将显示图 InstrumentsDialog 中所示的对话框窗口。在这里可以添加要收集的硬件性能监控事件（hardware performance monitoring events）。Apple 在其手册 [AppleOptimizationGuide] 中记录了其硬件性能监控事件。

![Xcode Instruments：CPU Counters 选项。](../../img/perf-tools/XcodeInstrumentsDialog.png)

第二步是设置分析目标。为此，点击并按住应用程序名称（图 InstrumentsView 中标记为②），然后选择你感兴趣的应用程序。如有需要，设置参数和环境变量。现在可以开始采集了；按下红色目标图标①。

Instruments 会显示时间线并持续更新有关运行中应用程序的统计信息。程序结束后，Instruments 将显示如图 InstrumentsView 所示的结果。编译耗时 7.3 秒，我们可以看到事件数量随时间的变化情况。例如，在运行末期，执行的分支指令数量和预测错误数量有所增加。可以在时间线上放大该区间以检查涉及的函数。

底部面板显示数值统计信息。要检查类似 Intel VTune 自底向上视图的热点，在菜单③中选择 *Profile*，然后点击 *Call Tree* 菜单④并勾选 *Invert Call Tree* 选项。这正是我们在图 InstrumentsView 中所做的操作。

Instruments 同时显示原始计数和总计百分比，这对于计算 IPC（每周期指令数）、MPKI（每千指令缺失次数）等二次指标非常有用。右侧显示了函数 `llvm::FoldingSetBase::FindNodeOrInsertPos` 的最热调用栈。双击某个函数，可以检查为源代码生成的 ARM 汇编指令。

据我所知，在 macOS 平台上没有其他同等质量的分析工具可用。高级用户可以通过编写简短（或较长）的命令行脚本来直接使用 `dtrace` 框架，但如何操作超出了本书的范围。

[^1]: Instruments documentation - [https://help.apple.com/instruments/mac/current](https://help.apple.com/instruments/mac/current)
