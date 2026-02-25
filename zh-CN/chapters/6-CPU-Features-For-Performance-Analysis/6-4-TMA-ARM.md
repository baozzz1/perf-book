### Arm 平台上的 TMA

Arm CPU 架构师也为其处理器开发了一套 TMA 性能分析方法论，下面我们将加以介绍。Arm 在其文档 [ARMNeoverseV1TopDown] 中将其称为"Topdown"，因此我们沿用这一命名。截至本文撰写时（2023 年末），Topdown 仅在 Arm 设计的核心上受支持，例如 Neoverse N1 和 Neoverse V1[^5]，以及其衍生产品，如 Ampere Altra 和 AWS Graviton3。如需了解 Arm 芯片系列的相关内容，请参阅本书末尾的主要 CPU 微架构列表。Apple 设计的处理器目前尚不支持 Arm Topdown 性能分析方法论。

Arm Neoverse V1 是 Neoverse 系列中第一款支持完整第 1 层 Topdown 指标的 CPU，包括：`错误推测（Bad Speculation）`、`前端受限（Frontend Bound）`、`后端受限（Backend Bound）`和`退休（Retiring）`。在 V1 核心之前，Neoverse N1 仅支持两个 L1 类别：`前端停顿周期（Frontend Stalled Cycles）`和`后端停顿周期（Backend Stalled Cycles）`。[^6]

为了在基于 V1 的处理器上演示 Arm 的 Topdown 分析，我启动了一个由 AWS Graviton3 驱动的 AWS EC2 `m7g.metal` 实例。请注意，由于虚拟化的限制，Topdown 可能无法在其他非 `metal` 实例类型上工作。我使用了由 AWS 管理的 64 位 ARM `Ubuntu 22.04 LTS`，Linux 内核版本为 6.2。所提供的 `m7g.metal` 实例有 64 个 vCPU 和 256 GB RAM。

我将 Topdown 方法论应用于 [AI Benchmark Alpha](https://ai-benchmark.com/alpha.html)[^1]，这是一个用于评估各种硬件平台（包括 CPU、GPU 和 TPU）AI 性能的开源 Python 库。该基准测试依赖 TensorFlow 机器学习库来测量关键深度学习模型的推理和训练速度。AI Benchmark Alpha 共包含 42 项测试，涵盖分类、图像分割、文本翻译等任务。

Arm 工程师开发了 [topdown-tool](https://learn.arm.com/install-guides/topdown-tool/)[^2]，下面我们将使用该工具。该工具在 Linux 和 Windows on ARM 上均可使用。在 Linux 上，它利用标准 perf 工具；在 Windows 上，它使用 [WindowsPerf](https://gitlab.com/Linaro/WindowsPerf/windowsperf)[^3]，这是一款 Windows on ARM 性能剖析工具。与 Intel 的 TMA 类似，Arm 方法论采用"逐层深入"的概念，即先确定高层性能瓶颈，再深入分析以找出更细致的根因。以下是我们使用的命令：

```bash
$ topdown-tool --all-cpus -m Topdown_L1 -- python -c "from ai_benchmark import AIBenchmark; results = AIBenchmark(use_CPU=True).run()"
Stage 1 (Topdown metrics)
=========================
[Topdown Level 1]
Frontend Bound... 16.48% slots
Backend Bound.... 54.92% slots
Retiring......... 27.99% slots
Bad Speculation..  0.59% slots
```

其中 `--all-cpus` 选项启用对所有 CPU 的全系统收集，`-m Topdown_L1` 收集 Topdown 第 1 层指标。`--` 后面的内容是运行 AI Benchmark Alpha 套件的命令行。

从上述输出可以得出结论：该基准测试不受分支预测错误的影响。此外，如果不深入了解相关工作负载，很难判断 16.5% 的`前端受限`是否值得关注，因此我们将注意力转向`后端受限`指标，这是停顿周期的主要来源。基于 Neoverse V1 的芯片没有第二层分解，方法论建议通过收集一组对应指标来进一步探索有问题的类别。以下是我们如何深入进行更详细的`后端受限`分析：

```bash
$ topdown-tool --all-cpus -n BackendBound -- python -c "from ai_benchmark import AIBenchmark; results = AIBenchmark(use_CPU=True).run()"
Stage 1 (Topdown metrics)
=========================
[Topdown Level 1]
Backend Bound......................... 54.70% slots

Stage 2 (uarch metrics)
=======================
  [Data TLB Effectiveness]
  DTLB MPKI........................... 0.413 misses per 1,000 instructions
  L1 Data TLB MPKI.................... 3.779 misses per 1,000 instructions
  L2 Unified TLB MPKI................. 0.407 misses per 1,000 instructions
  DTLB Walk Ratio..................... 0.001 per TLB access
  L1 Data TLB Miss Ratio.............. 0.013 per TLB access
  L2 Unified TLB Miss Ratio........... 0.112 per TLB access

  [L1 Data Cache Effectiveness]
  L1D Cache MPKI...................... 13.114 misses per 1,000 instructions
  L1D Cache Miss Ratio................  0.046 per cache access

  [L2 Unified Cache Effectiveness]
  L2 Cache MPKI....................... 1.458 misses per 1,000 instructions
  L2 Cache Miss Ratio................. 0.027 per cache access

  [Last Level Cache Effectiveness]
  LL Cache Read MPKI.................. 2.505 misses per 1,000 instructions
  LL Cache Read Miss Ratio............ 0.219 per cache access
  LL Cache Read Hit Ratio............. 0.783 per cache access

  [Speculative Operation Mix]
  Load Operations Percentage.......... 25.36% operations
  Store Operations Percentage.........  2.54% operations
  Integer Operations Percentage....... 29.60% operations
  Advanced SIMD Operations Percentage. 10.93% operations
  Floating Point Operations Percentage  6.85% operations
  Branch Operations Percentage........ 10.04% operations
  Crypto Operations Percentage........  0.00% operations
```

上述命令中，`-n BackendBound` 选项收集与`后端受限`类别及其子类别相关的所有指标。输出中每个指标的描述可在 [ARMNeoverseV1TopDown] 中找到。注意，它们与我们在 [PerfMetrics] 中讨论的内容非常相似，你可能需要回顾一下。

我们的目标不是优化基准测试，而是描述性能瓶颈的特征。但如果给出这样的任务，以下是我们的分析思路。`L1 Data TLB`（数据 TLB）缺失次数相当多（3.8 MPKI），但其中 90% 的缺失会命中 L2 TLB（见 `L2 Unified TLB Miss Ratio`）。总体来看，只有 0.1% 的 TLB 访问会导致页表遍历（page table walk，见 `DTLB Walk Ratio`），这表明它不是我们主要关注的问题，但利用大页（huge page）进行快速实验仍然值得一试。

查看 `L1/L2/LL Cache Effectiveness`（缓存有效性）指标，可以发现数据缓存缺失存在潜在问题。大约每 22 次 L1D 缓存访问中有 1 次缺失（见 `L1D Cache Miss Ratio`），这是可以接受的，但代价仍然不小。对于 L2，这一数字是每 37 次有 1 次缺失（见 `L2 Cache Miss Ratio`），情况好得多。然而对于 LLC（末级缓存），`LL Cache Read Miss Ratio` 令人不满意：每 5 次访问就有 1 次缺失。由于这是一个 AI 基准测试，大部分时间很可能花在矩阵乘法上，循环分块（loop blocking）等代码变换可能会有所帮助（见 [LoopOpts]）。AI 算法以"内存密集型"著称，然而 Neoverse V1 Topdown 并未显示是否存在因内存带宽饱和而导致的停顿。

最后一个类别提供了操作组合（operation mix），在某些场景下非常有用。这些百分比给出了各类指令（包括推测执行的指令）执行数量的估计。各项数字加起来不等于 100%，因为其余部分归属于隐式的"其他"类别（`topdown-tool` 不打印），在我们的案例中约为 15%。我们应该对 SIMD 操作比例偏低感到担忧，尤其是考虑到使用了高度优化的 `Tensorflow` 和 `numpy` 库。相比之下，整数操作和分支指令的比例似乎偏高。我检查后发现，大多数执行的分支指令是跳转到下一次循环迭代的循环回跳边（loop backward jump）。整数操作比例高可能是由于缺乏向量化，或者由于线程同步。[ARMNeoverseV1TopDown] 中给出了一个利用`推测操作组合（Speculative Operation Mix）`类别中的数据发现向量化机会的示例。

在我们的案例研究中，我们运行了基准测试两次，但在实践中，通常一次运行就足够了。不带选项运行 `topdown-tool` 将在单次运行中收集所有可用指标。此外，`-s combined` 选项会按 L1 类别对指标进行分组，并以类似 `Intel VTune`、`toplev` 及其他工具的格式输出数据。进行多次运行的唯一实际理由，是当工作负载具有非常短的突发阶段且这些阶段具有不同性能特征时。在这种情况下，为了避免事件复用（见 [secMultiplex]）并提高收集精度，最好多次运行工作负载。

AI Benchmark Alpha 包含多种测试，可能表现出不同的性能特征。上面呈现的输出汇聚了全部 42 项测试，给出了总体分解。如果各个测试确实具有不同的性能瓶颈，这种汇聚通常并不理想。你需要对每项测试分别进行 Topdown 分析。`topdown-tool` 可以使用 `-i` 选项来帮助实现这一点，该选项会按可配置的时间间隔输出数据，你可以比较不同时间段的数据并决定下一步行动。

[^1]: AI Benchmark Alpha - [https://ai-benchmark.com/alpha.html](https://ai-benchmark.com/alpha.html)
[^2]: Arm `topdown-tool` - [https://learn.arm.com/install-guides/topdown-tool/](https://learn.arm.com/install-guides/topdown-tool/)
[^3]: WindowsPerf - [https://gitlab.com/Linaro/WindowsPerf/windowsperf](https://gitlab.com/Linaro/WindowsPerf/windowsperf)
[^5]: 2024 年 9 月，Arm 发布了针对 Neoverse V2 和 V3 核心的性能分析指南 - [https://developer.arm.com/telemetry](https://developer.arm.com/telemetry)
[^6]: Neoverse V2 和 V3 核心的性能分析指南在本节撰写完成后才发布。参见以下页面 - [https://developer.arm.com/telemetry](https://developer.arm.com/telemetry)。
