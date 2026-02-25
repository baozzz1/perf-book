## 性能监控单元（Performance Monitoring Unit）

每款现代 CPU 都提供监控性能的设施，这些设施统称为性能监控单元（PMU，Performance Monitoring Unit）。该单元包含帮助开发人员分析其应用程序性能的功能。图 PMU 展示了现代 Intel CPU 中 PMU 的示例。大多数现代 PMU 都有一组性能监控计数器（PMC，Performance Monitoring Counters），可用于收集程序执行期间发生的各种性能事件。在后面的 [counting] 中，我们将讨论如何将 PMC 用于性能分析。此外，PMU 还有其他增强性能分析的特性，如 LBR、PEBS 和 PT，这些主题将在 [PmuChapter] 中专门讨论。

![现代 Intel CPU 的性能监控单元。](../../img/uarch/PMU.png)

随着每一代新的 CPU 设计的演进，其 PMU 也随之演进。在 Linux 上，可以使用 `cpuid` 命令确定 CPU 中 PMU 的版本，如清单 QueryPMU 所示。类似的信息可以通过检查 `dmesg` 命令的输出从内核消息缓冲区提取。有关每个 Intel PMU 版本的特性以及与前一版本的变化，可以在 [IntelOptimizationManual] 中找到。

清单：查询你的 PMU

```bash
$ cpuid
...
Architecture Performance Monitoring Features (0xa/eax):
      version ID                               = 0x4 (4)
      number of counters per logical processor = 0x4 (4)
      bit width of counter                     = 0x30 (48)
...
Architecture Performance Monitoring Features (0xa/edx):
      number of fixed counters    = 0x3 (3)
      bit width of fixed counters = 0x30 (48)
...
```
### 性能监控计数器（Performance Monitoring Counters）

如果我们设想一个处理器的简化视图，它可能看起来类似于图 PMC 所示。正如我们在本章前面所讨论的，现代 CPU 有缓存、分支预测器、执行流水线和其他单元。当连接到多个单元时，PMC 可以从它们收集有趣的统计数据。例如，它可以计算已过了多少个时钟周期、执行了多少条指令、在此期间发生了多少次缓存未命中或分支预测错误以及其他性能事件。

![带有性能监控计数器的 CPU 简化视图。](../../img/uarch/PMC.png)

通常，PMC 是 48 位宽的，这使得分析工具可以长时间运行而不中断程序执行。[^2] 性能计数器是作为特定型号寄存器（MSR，Model-Specific Register）实现的硬件寄存器。这意味着计数器的数量和宽度可能因型号而异，你不能依赖 CPU 中相同数量的计数器。你应该始终首先查询，例如使用 `cpuid` 等工具。PMC 可通过 `RDMSR` 和 `WRMSR` 指令访问，这些指令只能从内核空间执行。幸运的是，如果你是性能分析工具的开发人员（如 Linux `perf` 或 Intel VTune 性能分析器），你才需要关心这一点。这些工具处理 PMC 编程的所有复杂性。

当工程师分析其应用程序时，收集已执行指令数量和已过去周期数是非常常见的。这就是为什么一些 PMU 有专用 PMC 来收集此类事件。固定计数器（Fixed counters）始终测量 CPU 核心内的同一事物。使用可编程计数器（programmable counters），由用户选择他们想要测量的内容。

例如，在 Intel Skylake 架构（PMU 版本 4，参见清单 QueryPMU）中，每个物理核心有三个固定计数器和八个可编程计数器。三个固定计数器设置为计数核心时钟、参考时钟和退休指令（有关这些指标的更多详细信息，参见 [secMetrics]）。AMD Zen4 和 Arm Neoverse V1 核心每个处理器核心支持 6 个可编程性能监控计数器，没有固定计数器。

PMU 提供超过一百个可供监控的事件并不罕见。图 PMU 只显示了现代 Intel CPU 上可供监控的性能监控事件的一小部分。不难注意到，可用 PMC 的数量远小于性能事件的数量。无法同时计数所有事件，但分析工具通过在程序执行期间在性能事件组之间进行多路复用（multiplexing）来解决这个问题（参见 [secMultiplex]）。

- 对于 Intel CPU，完整的性能事件列表可以在 [IntelOptimizationManual] 或 [perfmon-events.intel.com](https://perfmon-events.intel.com/) 中找到。
- AMD 没有为每款 AMD 处理器发布性能监控事件列表。好奇的读者可能会在 Linux `perf` 源[代码](https://github.com/torvalds/linux/blob/master/arch/x86/events/amd/core.c)[^3]中找到一些信息。此外，你可以使用 AMD uProf 命令行工具列出可供监控的性能事件。有关 AMD 性能计数器的一般信息，可以在 [AMDProgrammingManual] 中找到。
- 对于 ARM 芯片，性能事件没有得到很好的定义。供应商按照 ARM 架构实现核心，但性能事件有所不同，包括它们的含义和支持的事件。对于 Arm 自己设计的 Arm Neoverse V1 核心，性能事件列表可以在 [ARMNeoverseV1] 中找到。对于 Arm Neoverse V2 和 V3 微架构，性能事件列表可以在 Arm 的网站上找到。[^4]

[^2]: 当 PMC 的值溢出时，程序的执行必须被中断。性能分析工具应该保存溢出的事实。我们将在 [sec_PerfApproaches] 中更详细地讨论这一点。
[^3]: AMD 核心的 Linux 源代码 - [https://github.com/torvalds/linux/blob/master/arch/x86/events/amd/core.c](https://github.com/torvalds/linux/blob/master/arch/x86/events/amd/core.c)
[^4]: Arm 遥测 - [https://developer.arm.com/telemetry](https://developer.arm.com/telemetry)
