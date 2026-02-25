## 流水线槽（Pipeline Slot）

某些性能工具使用的另一个重要指标是*流水线槽*（pipeline slot）的概念。流水线槽代表处理一个 $\mu$op 所需的硬件资源。图 PipelineSlot 展示了一个每周期有 4 个分配槽（allocation slot）的 CPU 执行流水线。这意味着核心每周期可以为 4 个新的 $\mu$ops 分配执行资源（重命名后的源和目标寄存器、执行端口、ROB 条目等）。这样的处理器通常被称为 *4-wide 机器*。在图中连续六个周期中，只有一半的可用槽被利用（以黄色高亮显示）。从微架构的角度来看，执行此类代码的效率仅为 50%。

![4-wide CPU 的流水线示意图。](../../../img/terms-and-metrics/PipelineSlot.jpg)

<p align="center"><em>4-wide CPU 的流水线示意图。</em></p>


Intel 的 Skylake 和 AMD Zen3 核心具有 4-wide 分配。Intel 的 Sunny Cove 微架构是 5-wide 设计。截至 2023 年底，最新的 Golden Cove 和 Zen4 架构均为 6-wide 分配。Apple M1 和 M2 设计为 8-wide，Apple M3 的执行带宽为 9-$\mu$op，详见 [AppleOptimizationGuide]。机器的宽度对 IPC 设定了上限。这意味着处理器的最大可达 IPC 等于其宽度。[^2] 例如，当你的计算结果显示 Golden Cove 核心上的 IPC 超过 6 时，应该持怀疑态度。

极少数应用程序能达到机器的最大 IPC。例如，Intel Golden Cove 核心理论上每个时钟周期可以执行四次整数加法/减法，加上一次加载，加上一次存储（共计六条指令），但应用程序极不可能在相邻位置拥有适当组合的独立指令来充分利用所有这些潜在的并行性。

流水线槽利用率是自顶向下微架构分析（Top-down Microarchitecture Analysis，见 [TMA]）的核心指标之一。例如，前端受限（Frontend Bound）和后端受限（Backend Bound）指标被表示为由于各种瓶颈导致的未利用流水线槽的百分比。

[^2]: 尽管存在一些例外。例如，宏融合的比较并跳转指令只需要一个流水线槽，但被计为两条指令。在某些极端情况下，这可能导致 IPC 大于机器宽度。
