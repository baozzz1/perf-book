## 案例研究：测量代码占用（Code Footprint）

正如我在本章中多次提到的，代码布局优化对代码量庞大的应用程序影响最为显著。明确程序中热代码大小的最佳方式是测量其*代码占用*，即程序在执行过程中所触及的带有机器指令的字节数/缓存行数/页面数。

较大的代码占用本身并不一定会对性能产生负面影响。代码占用不是决定性指标，它不能立即告诉你是否存在问题。尽管如此，它已被证明是性能分析中有用的补充数据点。结合 TMA 的 `Frontend_Bound`、L1 指令缓存缺失率和其他指标，它可能有助于加强投入时间优化应用程序机器码布局的论据。

目前，能够可靠测量代码占用的工具很少。在本案例研究中，我将演示 [perf-tools](https://github.com/aayasin/perf-tools)，[^1] 这是一个基于 Linux `perf` 构建的开源性能分析工具集合。为了估算[^2]代码占用，`perf-tools` 利用了 Intel 的 LBR（参见 [lbr]），因此目前不能在基于 AMD 或 ARM 的系统上工作。以下是收集代码占用数据的示例命令：

```
$ perf-tools/do.py profile --profile-mask 100 -a <your benchmark>
```

`--profile-mask 100` 启动 LBR 采样，`-a` 使你能够指定要运行的程序。此命令将收集代码占用以及各种其他数据。我不展示工具的输出，好奇的读者可以研究文档并进行实验。

我选取了四个基准测试：Clang C++ 编译、Blender 光线追踪、Cloverleaf 流体动力学和 Stockfish 国际象棋引擎；这些工作负载在 [PerfMetricsCaseStudy] 中应该已经很熟悉，我们在那里分析了它们的性能特征。我在基于 Intel Alder Lake 的处理器上运行了它们。[^5]

在开始查看结果之前，让我们先花时间了解一下术语。程序代码的不同部分可能以不同的频率被执行，因此某些部分会比其他部分更"热"。`perf-tools` 包不区分这一点，使用术语"非冷代码"（non-cold code）来指代至少执行过一次的代码。这被称为*双向拆分*（two-way splitting），因为它将代码分为冷和非冷两部分。其他工具（如 Meta 的 HHVM）使用*三向拆分*（three-way splitting），区分热、温和冷代码，温热之间有可调阈值。本节中，我们使用"热代码"来指代非冷代码。

四个基准测试的结果如表 code_footprint 所示。二进制文件和 `.text` 大小通过标准 Linux `readelf` 工具获取，而其他指标通过 `perf-tools` 收集。`non-cold code footprint [KB]`（非冷代码占用 [KB]）指标是程序至少触及一次的带有机器指令的千字节数。`non-cold code [4KB-pages]`（非冷代码 [4KB 页面]）指标告诉我们程序至少触及一次的含有机器指令的非冷 4KB 页面数量。它们共同帮助我们了解这些非冷内存位置的密集程度或稀疏程度。一旦我们深入分析数字，这一点就会变得清晰。最后，我们还展示了前端受限（Frontend Bound）百分比，这是你在 [TMA] 关于 TMA 的章节中应该已经熟悉的指标。

| Metric | Clang17 compilation | Blender | CloverLeaf | Stockfish |
|--------|---------------------|---------|------------|-----------|
| Binary size [KB] | 113844 | 223914 | 672 | 39583 |
| `.text` size [KB] | 67309 | 133009 | 598 | 238 |
| non-cold code footprint [KB] | 5042 | 313 | 104 | 99 |
| non-cold code [4KB-pages] | 6614 | 546 | 104 | 61 |
| Frontend Bound [%] | 52.3 | 29.4 | 5.3 | 25.8 |

表：案例研究中基准测试的代码占用情况。

我们先看一下二进制文件和 `.text` 的大小。与 Clang17 和 Blender 相比，CloverLeaf 是一个很小的应用程序；Stockfish 嵌入了神经网络文件，这占据了二进制文件的最大部分，但其代码段相对较小；Clang17 和 Blender 拥有庞大的代码库。`.text size` 指标是我们应用程序的上界，即我们假设[^3]代码占用不应超过 `.text` 大小。

通过分析代码占用数据，可以得出几个有趣的观察结论。首先，尽管 Blender 的 `.text` 段非常大，但 Blender 非冷代码不足 1%：313 KB（占 133 MB 的不到 1%）。因此，仅仅因为二进制文件很大，并不意味着应用程序会遭受 CPU 前端瓶颈。重要的是热代码的数量。其他基准测试的这一比例更高：Clang17 为 7.5%，CloverLeaf 为 17.4%，Stockfish 为 41.6%。从绝对数量来看，Clang17 编译触及的带有机器指令的字节数比其他三个应用程序加起来多一个数量级。

其次，让我们检查表中的 `non-cold code [4KB-pages]` 行。对于 Clang17，5042 KB 的非冷代码分布在 6614 个 4KB 页面上，页面利用率为 `5042 / (6614 * 4) = 19%`。此指标告诉我们热代码部分的密集/稀疏程度。每条热缓存行与另一条热缓存行的距离越近，存储热代码所需的页面就越少。页面利用率越高越好。本章前面讨论的基本块放置和函数重排是提高页面利用率的变换的完美示例。其他基准测试的百分比为：Blender 14%，CloverLeaf 25%，Stockfish 41%。

现在我们量化了四个应用程序的代码占用，很想思考 L1 指令缓存和 L2 缓存的大小，以及热代码是否适合其中。在我的基于 Alder Lake 的机器上，L1 I-cache 只有 32 KB，不足以完全覆盖我们分析的任何基准测试。但请记住，本节开头我们说过，大的代码占用并不立即指向问题所在。是的，庞大的代码库对 CPU 前端施加了更大压力，但指令访问模式对性能同样至关重要。与数据访问相同的局部性原则同样适用。这就是为什么我们将其与来自自顶向下分析的 Frontend Bound 指标配合使用。

对于 Clang17，5 MB 的非冷代码造成了巨大的 52.3% 前端受限性能瓶颈：超过一半的周期浪费在等待指令上。在所有展示的基准测试中，它从 PGO 类型的优化中受益最多。CloverLeaf 没有遭受低效指令获取的困扰；其 75% 的分支是向后跳转，这表明这些可能是反复执行的相对较小的循环。Stockfish 虽然非冷代码占用与 CloverLeaf 大致相同，但对 CPU 前端构成了更大的挑战（25.8%）。它有更多的间接跳转和函数调用。最后，Blender 的间接跳转和调用甚至比 Stockfish 更多。

我在此结束分析，因为进一步的调查超出了本案例研究的范围。对于有兴趣继续分析的读者，我建议根据 TMA 方法深入研究前端受限（Frontend Bound）类别，并查看 `ICache_Misses`、`ITLB_Misses`、`DSB coverage` 等指标。

另一个研究代码占用的有用工具是 [llvm-bolt-heatmap](https://github.com/llvm/llvm-project/blob/main/bolt/docs/Heatmaps.md)，[^4] 它是 LLVM 的 BOLT 项目的一部分。该工具可以生成代码热图（code heatmaps），提供对应用程序代码布局的精细理解。它主要用于评估热代码的原始布局并确认优化后的布局更加紧凑。

[^1]: perf-tools - [https://github.com/aayasin/perf-tools](https://github.com/aayasin/perf-tools)
[^2]: `perf-tools` 收集的代码占用数据并不精确，因为它基于采样 LBR 记录。其他工具如 Intel 的 `sde -footprint` 遗憾地不提供代码占用。但是，自己编写一个基于 PIN 的工具来测量精确代码占用并不难。
[^3]: 这并不总是正确的：应用程序本身可能很小，但会调用多个其他动态链接库，或者可能大量使用内核代码。
[^4]: llvm-bolt-heatmap - [https://github.com/llvm/llvm-project/blob/main/bolt/docs/Heatmaps.md](https://github.com/llvm/llvm-project/blob/main/bolt/docs/Heatmaps.md)
[^5]: 收集代码占用使用哪台机器并不重要，因为它取决于程序和输入数据，而不是特定机器的特性。作为健全性检查，我在基于 Skylake 的机器上运行并得到了非常相似的结果。
