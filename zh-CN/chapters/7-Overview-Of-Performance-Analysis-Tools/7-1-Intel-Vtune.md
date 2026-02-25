## Intel VTune Profiler

VTune Profiler（前身为 VTune Amplifier）是一款面向 x86 架构机器的性能分析工具，具有丰富的图形界面。它可以在 Linux 或 Windows 操作系统上运行。由于 VTune 不支持 Apple 芯片（如 M1 和 M2），而基于 Intel 的 MacBook 也正在迅速淡出市场，因此我们跳过对 VTune 在 macOS 上的支持讨论。

VTune 可以在 Intel 和 AMD 系统上使用。但是，高级的基于硬件的采样（hardware-based sampling）需要 Intel 生产的 CPU。例如，在 AMD 系统上无法使用 Intel VTune 收集硬件性能计数器（hardware performance counters）。

截至 2023 年初，VTune 可作为独立工具或作为 Intel oneAPI Base Toolkit 的一部分免费获取。[^1]

### 如何配置

在 Linux 上，VTune 可以使用两种数据采集器：Linux perf 和 VTune 自己的驱动程序 SEP。前者用于用户模式采样，但如果想进行高级分析，则需要构建并安装 SEP 驱动程序，这并不太难。

```bash
# 进入 vtune 安装目录中的 sepdk 文件夹
$ cd ~/intel/oneapi/vtune/latest/sepdk/src
# 构建驱动程序
$ ./build-driver
# 添加 vtune 用户组并将你的用户加入该组
# 创建新 shell，或重启系统
$ sudo groupadd vtune
$ sudo usermod -a -G vtune `whoami`
# 安装 sep 驱动程序
$ sudo ./insmod-sep -r -g vtune
```

完成以上步骤后，你应该能够使用高级分析类型，如微架构探索（Microarchitectural Exploration）和内存访问（Memory Access）。

在 Windows 上，安装 VTune 后无需任何额外配置。收集硬件性能事件需要管理员权限。

### 可以用它做什么

- 查找热点（hotspots）：函数、循环、语句。
- 监控各种 CPU 特定的性能事件，例如分支预测错误（branch mispredictions）和 L3 缓存缺失（L3 cache misses）。
- 定位这些事件发生的代码行。
- 使用 TMA（Top-down Microarchitecture Analysis，自顶向下微架构分析）方法表征 CPU 性能瓶颈。
- 针对特定函数、进程、时间段或逻辑核心过滤数据。
- 观察工作负载随时间的行为变化（包括 CPU 频率、内存带宽利用率等）。

VTune 可以提供有关运行中进程的非常丰富的信息。如果你希望提升应用程序的整体性能，它是合适的工具。VTune 始终提供一段时间内的聚合数据，因此可用于找出"平均情况"的优化机会。

### 不能用它做什么

- 分析持续时间极短的执行异常。
- 观察系统范围内复杂的软件动态。

由于该工具的采样特性，它最终会错过持续时间非常短的事件（例如，亚微秒级别）。

### 示例

以下是 VTune 最有趣功能的一系列截图。在本示例中，我使用了 POV-Ray，这是一款用于创建 3D 图形的光线追踪器。图 VtuneHotspots 展示了内置 POV-Ray 3.7 基准测试的热点分析，该测试使用 clang14 编译器以 `-O3 -ffast-math -march=native -g` 选项编译，在搭载 Intel Alder Lake 处理器（Core i7-1260P，4 个性能核 + 8 个能效核）的系统上以 4 个工作线程运行。

![VTune 对 povray 内置基准测试的热点视图。](../../../img/perf-tools/VtunePovray.png)

<p align="center"><em>VTune 对 povray 内置基准测试的热点视图。</em></p>


![VTune 对 povray 内置基准测试的源代码视图。](../../../img/perf-tools/VtunePovray_SourceView.png)

<p align="center"><em>VTune 对 povray 内置基准测试的源代码视图。</em></p>


在图像左侧，可以看到工作负载中热函数的列表，以及对应的 CPU 时间百分比和已退休指令数（retired instructions）。在右侧面板中，可以看到导致调用 `pov::Noise` 函数的最频繁调用栈。根据该截图，`pov::Noise` 函数有 `44.4%` 的时间是从 `pov::Evaluate_TPat` 调用的，而后者又是从 `pov::Compute_Pigment` 调用的。[^20]

如果双击 `pov::Noise` 函数，将看到图 VtuneSourceView 中显示的图像。为了节省空间，只显示了最重要的列。左侧面板显示源代码以及对应每行代码的 CPU 时间。右侧显示汇编指令以及归因于它们的 CPU 时间。高亮的机器指令对应左侧面板中的第 476 行。每个面板中所有 CPU 时间百分比之和（不仅仅是可见的部分）等于归因于 `pov::Noise` 函数的总 CPU 时间，即 `26.8%`。

![VTune 对 povray 内置基准测试的性能事件时间线视图。](../../../img/perf-tools/VtunePovray_EventTimeline.jpg)

<p align="center"><em>VTune 对 povray 内置基准测试的性能事件时间线视图。</em></p>


当你使用 VTune 分析运行在 Intel CPU 上的应用程序时，它可以收集许多不同的性能事件。为了说明这一点，我运行了一种不同的分析类型——微架构探索（Microarchitecture Exploration）。要访问原始事件计数，可以切换视图至如图 VtuneTimelineView 所示的 Hardware Events 视图。要启用视图切换，需要在 *Options* &rarr; *General* &rarr; *Show all applicable viewpoints* 中勾选选项。在图 VtuneTimelineView 的顶部附近，可以看到 *Platform* 标签页已被选中。另外两个页面也很有用：*Summary* 页面给出从 CPU 计数器收集的原始性能事件的绝对数量；*Event Count* 页面提供相同数据，并按函数进行分解。

图 VtuneTimelineView 内容较多，需要一些解释。顶部面板（标记为①）是时间线视图，显示了我们的四个工作线程随时间变化的 L1 缓存缺失情况，以及主线程（TID: `3102135`）的一些微小活动，主线程负责生成所有工作线程。黑色柱越高，表示在任意给定时刻发生的事件（本例中为 L1 缓存缺失）越多。注意所有四个工作线程中 L1 缺失的偶发性峰值。我们可以使用此视图来观察工作负载的不同或重复阶段。然后，为了找出该时间段内执行了哪些函数，可以选择一个区间并点击"filter in"以专注于该部分运行时间。标记为②的区域就是此类过滤的示例。要查看更新后的函数列表，可以切换至 *Event Count* 视图。这种过滤和缩放功能在所有 VTune 时间线视图中均可使用。

标记为③的区域显示了收集的性能事件及其随时间的分布。这里不是每线程视图，而是显示所有线程的聚合数据。除了观察执行阶段外，还可以从中直观地提取一些有趣的信息。例如，可以看到执行的分支数量很高（`BR_INST_RETIRED.ALL_BRANCHES`），但预测错误率相当低（`BR_MISP_RETIRED.ALL_BRANCHES`）。这可以让你得出结论：分支预测错误对 POV-Ray 来说不是瓶颈。向下滚动，会发现 L3 缺失数为零，L2 缓存缺失也极为罕见。这告诉我们，99% 的内存访问请求由 L1 缓存提供服务，其余由 L2 提供服务。结合这两个观察，可以得出结论：该应用程序可能受计算（compute）限制，即 CPU 忙于计算，而非等待内存或从预测错误中恢复。

最后，底部面板④显示了四个硬件线程的 CPU 频率图。将鼠标悬停在不同时间切片上，可以看到这些核心的频率在 3.2--3.4GHz 范围内波动。

[^1]: Intel oneAPI Base Toolkit - [https://www.intel.com/content/www/us/en/developer/tools/oneapi/base-toolkit.html](https://www.intel.com/content/www/us/en/developer/tools/oneapi/base-toolkit.html)
[^3]: VTune microarchitecture analysis - [https://software.intel.com/en-us/vtune-help-general-exploration-analysis](https://software.intel.com/en-us/vtune-help-general-exploration-analysis). In pre-2019 versions of Intel® VTune Profiler, it was called as "General Exploration" analysis.
[^4]: 7zip benchmark - [https://github.com/llvm-mirror/test-suite/tree/master/MultiSource/Benchmarks/7zip](https://github.com/llvm-mirror/test-suite/tree/master/MultiSource/Benchmarks/7zip).
[^19]: Per-function view of TMA metrics is a feature unique to Intel® VTune profiler.
[^20]: 注意，调用栈并未一直追溯到 `main` 函数。这是因为在基于硬件的采集中，VTune 使用 LBR（Last Branch Record）来采样调用栈，其深度有限。这里很可能涉及递归函数，若要进一步调查，用户需要深入研究代码。
