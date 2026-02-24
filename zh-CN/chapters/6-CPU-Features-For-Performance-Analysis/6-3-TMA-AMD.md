### AMD 平台上的 TMA

从 Zen4 开始，AMD 处理器支持第 1 层和第 2 层 TMA 分析。根据 AMD 文档，这被称为"流水线利用率"（Pipeline Utilization）分析，但核心思想是相同的。L1 和 L2 分类也与 Intel 的非常相似。自 Linux 内核 6.2 起，Linux 用户可以使用 `perf` 工具收集流水线利用率数据。

接下来，我们将分析 [Crypto++](https://github.com/weidai11/cryptopp)[^1] 中 SHA-256（安全散列算法 256）的实现。SHA-256 是比特币挖矿中的基础密码学算法。Crypto++ 是一个开源 C++ 密码学算法类库，包含许多算法的实现，不仅仅是 SHA-256。不过，在我们的示例中，我通过注释掉 `bench1.cpp` 中 `BenchmarkUnkeyedAlgorithms` 函数里的相应代码行，禁用了其他算法的基准测试。

我在搭载 Ubuntu 22.04、Linux 内核 6.5 的 AMD Ryzen 9 7950X 机器上运行了测试。我使用 GCC 12.3 C++ 编译器编译了 Crypto++ 8.9 版本，采用默认的 `-O3` 优化选项，但这对性能影响不大，因为代码使用了 x86 intrinsics（见 [secIntrinsics]）并利用了 SHA x86 ISA 扩展。

以下是我用于获取 L1 和 L2 流水线利用率指标的命令。输出已经过裁剪，部分统计数据被省略以减少干扰。

```bash
$ perf stat -M PipelineL1,PipelineL2 -- ./cryptest.exe b1 10
 0.0 %  bad_speculation_mispredicts        (20.08%) 
 0.0 %  bad_speculation_pipeline_restarts  (20.08%)
 0.0 %  bad_speculation                    (20.08%)
 6.1 %  frontend_bound                     (20.00%)
 6.1 %  frontend_bound_bandwidth           (20.00%)
 0.1 %  frontend_bound_latency             (20.00%)
65.9 %  backend_bound_cpu                  (20.00%)
 1.7 %  backend_bound_memory               (20.00%)
67.5 %  backend_bound                      (20.00%)
26.3 %  retiring                           (20.08%)
20.2 %  retiring_fastpath                  (19.99%)
 6.1 %  retiring_microcode                 (19.99%)
```

输出中方括号内的数字表示某个指标被监控的运行时间百分比。可以看到，由于复用（multiplexing），每个指标仅被监控了约 20% 的时间。在我们的案例中，由于 SHA256 行为一致，这很可能不是问题，但情况并不总是如此。为了最小化复用的影响，可以在单次运行中收集有限的指标集，例如 `perf stat -M frontend_bound,backend_bound`。

上述流水线利用率指标的描述可在 [AMDUprofManual] 中找到。通过查看指标，可以看到 SHA256 中不存在分支预测错误（`bad_speculation` 为 0%）。可用分发槽位（dispatch slot）中只有 26.3% 被使用（`retiring`），这意味着剩余 73.7% 因前端和后端停顿而被浪费。

高级密码学指令并不简单，因此在内部它们会被分解为更小的片段（$\mu$ops）。一旦处理器遇到这类指令，就会从微码（microcode）中获取其对应的 $\mu$ops。与常规指令解码器相比，从微码序列器（microcode sequencer）获取微操作的带宽更低，这可能成为性能瓶颈的来源。Crypto++ SHA256 实现大量使用了 `SHA256MSG2`、`SHA256RNDS2` 等指令，根据 [uops.info](https://uops.info/table.html)[^2] 网站的数据，这些指令由多个 $\mu$ops 组成。

`retiring_microcode` 指标表明，6.1% 的分发槽位被最终退休的微码操作所使用。与其兄弟指标 `retiring_fastpath` 相比，大约每 4 条指令中就有 1 条是微码操作。再看 `frontend_bound_bandwidth` 指标，会发现 6.1% 的分发槽位因 CPU 前端的带宽瓶颈而未被使用。这表明有 6.1% 的分发槽位被浪费，原因是微码序列器无法提供足够的 $\mu$ops，而后端本可以消费这些操作。在本例中，`retiring_microcode` 和 `frontend_bound_bandwidth` 指标紧密关联，但它们数值相等只是巧合。

大多数周期停顿在 CPU 后端（`backend_bound`），但仅有 1.7% 的周期在等待内存访问时停顿（`backend_bound_memory`）。因此，我们知道该基准测试主要受机器计算能力的限制。正如你将在本书第二部分了解到的，这可能与数据流依赖（data flow dependency）或某些密码学操作的执行吞吐量（execution throughput）有关。与传统的 `ADD`、`SUB`、`CMP` 等指令相比，这些指令的频率较低，因此通常只能在单个执行单元上运行。大量此类操作可能使该特定单元的执行吞吐量饱和。进一步分析应包括仔细查看源代码和生成的汇编代码、检查执行端口利用率、查找数据依赖关系等。

总结来看，Crypto++ 在 AMD Ryzen 9 7950X 上的 SHA-256 实现仅利用了可用分发槽位的 26.3%；6.1% 的分发槽位因微码序列器带宽不足而被浪费，65.9% 因机器计算资源不足而停顿。该算法明显触及了一些硬件限制，因此其性能能否进一步提升尚不明确。

在 Windows 平台上，截至本文撰写时，TMA 方法论仅在 AMD 服务器平台（代号 Genoa）上受支持，而不支持客户端系统（代号 Raphael）。TMA 支持在 AMD uProf 4.1 版本中被加入，但仅限于 AMD uProf 安装包中的命令行工具 `AMDuProfPcm`。你可以参阅 [AMDUprofManual] 了解如何运行分析的更多详细信息。AMD uProf 的图形化版本尚不支持 TMA 分析。

[^1]: Crypto++ - [https://github.com/weidai11/cryptopp](https://github.com/weidai11/cryptopp)
[^2]: uops.info - [https://uops.info/table.html](https://uops.info/table.html)
