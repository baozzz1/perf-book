## 基于硬件的采样特性

各主要 CPU 厂商提供了一组附加特性来增强采样能力。由于 CPU 厂商在性能监控上的实现方式各有不同，这些能力不仅名称各异，功能也有所差别。在 Intel 处理器中，它被称为处理器事件采样（Processor Event-Based Sampling，PEBS），最早在 NetBurst 微架构中引入。AMD 处理器上类似的特性称为基于指令的采样（Instruction Based Sampling，IBS），从 AMD Opteron 系列（10h 代）核心开始提供。下面，我们将详细讨论这些特性，包括它们的异同之处。

### Intel 平台上的 PEBS

与最后分支记录（LBR）特性类似，PEBS 在程序剖析期间被用于随每个收集的样本捕获额外数据。当一个性能计数器被配置为 PEBS 时，处理器保存一组附加数据，这些数据具有固定格式，称为 PEBS 记录（PEBS record）。Intel Skylake CPU 的 PEBS 记录格式如图 PEBS_record 所示。它包含通用寄存器（`EAX`、`EBX`、`ESP` 等）的状态、`EventingIP`、`Data Linear Address`（数据线性地址）和`Latency value`（延迟值），以及其他少数字段。PEBS 记录的内容布局因不同的微架构而有所不同，详见 [IntelOptimizationManual]。

![第 6、7、8 代 Intel Core 处理器系列的 PEBS 记录格式。*© 来源：[IntelOptimizationManual]。*](../../img/pmu-features/PEBS_record.png)

自 Skylake 起，PEBS 记录得到增强，可以收集 XMM 寄存器和最后分支记录（LBR）记录。格式也进行了重构，字段按基本组（Basic group）、内存组（Memory group）、通用寄存器组（GPR group）、XMM 组和 LBR 组分类。性能剖析工具可以选择感兴趣的数据组，从而降低记录开销。默认情况下，PEBS 记录仅包含基本组。

使用 PEBS 的一个显著好处是，与常规基于中断的采样相比，采样开销更低。回想一下，当计数器溢出时，CPU 会生成一个中断以收集一个样本。频繁生成中断并让分析工具在中断服务例程内部捕获程序状态，代价非常高昂，因为这涉及操作系统交互。

另一方面，PEBS 保留一个缓冲区，用于临时存储多条 PEBS 记录。假设我们使用 PEBS 对加载指令（load instruction）进行采样。当一个性能计数器被配置为 PEBS 时，计数器的溢出条件不会触发中断，而是激活 PEBS 机制。该机制会捕获下一条加载指令，捕获一条新记录并将其存储在专用 PEBS 缓冲区中。该机制还负责清除计数器溢出状态并将计数器重新加载为初始值。只有当专用缓冲区满时，处理器才会触发一个中断，缓冲区内容才被刷新到内存中。这种机制通过减少中断触发频率来降低采样开销。

Linux 用户可以通过执行 `dmesg` 检查 PEBS 是否已启用：

```bash
$ dmesg | grep PEBS
[    0.113779] Performance Events: XSAVE Architectural LBR, PEBS fmt4+-baseline,  
AnyThread deprecated, Alderlake Hybrid events, 32-deep LBR, full-width counters, Intel PMU driver.
```

对于 LBR，Linux perf 在每个收集的样本中转储整个 LBR 栈的内容，因此可以分析由 Linux perf 收集的原始 LBR 转储。然而，对于 PEBS，Linux `perf` 不像对 LBR 那样导出原始输出，而是处理 PEBS 记录，仅提取特定需求所需的数据子集。因此，无法通过 Linux `perf` 访问原始 PEBS 记录集合。不过，Linux `perf` 提供了从原始样本处理得到的部分 PEBS 数据，可通过 `perf report -D` 访问。要转储原始 PEBS 记录，可以使用 [`pebs-grabber`](https://github.com/andikleen/pmu-tools/tree/master/pebs-grabber)[^1]。

### AMD 平台上的 IBS

基于指令的采样（Instruction-Based Sampling，IBS）是一种 AMD64 处理器特性，可用于收集与指令取指（instruction fetch）和指令执行（instruction execution）相关的特定指标。AMD 处理器的流水线（pipeline）由两个独立阶段组成：取指 AMD64 指令字节的前端（Frontend）阶段和执行操作（ops）的后端（Backend）阶段。由于这两个阶段在逻辑上是分离的，因此存在两种独立的采样机制：IBS Fetch（取指采样）和 IBS Execute（执行采样）。

- IBS Fetch 监控流水线的前端，提供关于 ITLB（命中或缺失）、I-cache（命中或缺失）、取指地址、取指延迟等信息。
- IBS Execute 监控流水线的后端，通过追踪单个操作的执行来提供指令执行行为信息。例如，分支（是否执行、是否预测正确），以及加载/存储（在 D-cache 和 DTLB 中的命中或缺失、线性地址、加载延迟）。

AMD 处理器中 PMC 和 IBS 之间有几个重要区别。PMC 计数器是可编程的，而 IBS 的行为类似于固定计数器（fixed counter）。IBS 计数器只能被启用或禁用以进行监控，不能被编程为任意选择的事件。IBS Fetch 和 Execute 计数器可以独立启用/禁用。使用 PMC 时，用户必须提前决定监控哪些事件。使用 IBS 时，每个被采样指令都会收集丰富的数据集，然后由用户分析其中感兴趣的部分。IBS 选取并标记一条待监控的指令，然后在该指令执行过程中捕获由其引起的微架构事件。Intel PEBS 与 AMD IBS 的更详细比较可参见 [ComparisonPEBSIBS]。

由于 IBS 集成在处理器流水线中并作为固定事件计数器工作，样本收集开销极小。剖析器需要处理 IBS 生成的数据，根据采样间隔、配置的线程数量、是否配置 Fetch/Execute 等因素，数据量可能相当大。在 Linux 内核 6.1 之前，IBS 始终对所有核心收集样本，这一限制会导致巨大的数据收集和处理开销。从内核 6.2 起，Linux perf 支持仅对已配置的核心收集 IBS 样本。

Linux perf 和 AMD uProf 剖析器均支持 IBS。以下是收集 IBS Execute 和 Fetch 样本的示例命令：

```bash
$ perf record -a -e ibs_op/cnt_ctl=1,l3missonly=1/ -- benchmark.exe
$ perf record -a -e ibs_fetch/l3missonly=0/ -- benchmark.exe
$ perf report
```

其中 `cnt_ctl=0` 统计时钟周期，`cnt_ctl=1` 统计一个间隔周期内分发的操作数；`l3missonly=1` 仅保留发生 L3 缺失的样本。这两个参数以及其他一些参数在 [AMDUprofManual] 中有更详细的描述。注意，上述两条命令中都使用了 `-a` 选项，以在 Linux 内核 6.1 或更旧版本上收集所有核心的 IBS 样本，否则 `perf` 会采集失败。从 6.2 版本起，`-a` 选项不再是必需的，除非你想收集所有核心的 IBS 样本。`perf report` 命令将像常规 PMU 事件一样显示归因到函数和源代码行的样本，但会附加我们稍后将讨论的额外功能。AMD uProf 命令行工具可以生成 IBS 原始数据，之后可以转换为 CSV 文件，用 MS Excel 进行后处理，详见 [AMDUprofManual]。

### Arm 平台上的 SPE

Arm 统计剖析扩展（Statistical Profiling Extension，SPE）是一种架构特性，旨在增强 Arm CPU 内部的指令执行剖析能力。SPE 特性扩展在 Armv8-A 架构中被规定，从 Arm v8.2 起提供支持。Arm SPE 扩展在架构上是可选的，这意味着 Arm 处理器厂商无需强制实现。Arm Neoverse 核心从 2019 年推出的 Neoverse N1 核心起就支持 SPE。

与其他解决方案相比，SPE 与 AMD IBS 的相似程度更高，而不是与 Intel PEBS。与 IBS 类似，SPE 独立于通用性能监控计数器（PMC），但与 IBS 有两种类型（取指和执行）不同，SPE 只有一种机制。

SPE 采样过程作为指令执行流水线的内置部分运行。样本收集仍然基于可配置的间隔，但操作是被统计性地选取的。每个被采样的操作生成一条样本记录，其中包含该操作执行的各种数据。SPE 记录保存指令地址、加载/存储访问数据的虚拟地址和物理地址、数据访问来源（缓存或 DRAM），以及用于与系统中其他事件关联的时间戳。此外，它还可以提供各流水线阶段的延迟，例如发射延迟（Issue latency，从分发到执行）、翻译延迟（Translation latency，虚拟地址到物理地址转换的周期数）和执行延迟（Execution latency，加载/存储在功能单元中的延迟）。白皮书 [ARMSPE] 对 Arm SPE 进行了更详细的描述，并展示了使用它进行优化的几个示例。

与 Intel PEBS 和 AMD IBS 类似，Arm SPE 有助于降低采样开销并支持更长时间的收集。此外，它还支持对样本记录进行后过滤（postfiltering），有助于减少存储所需的内存。SPE 剖析在 Linux `perf` 中受支持，可以按如下方式使用：[^6]

```bash
$ perf record -e arm_spe_0/<controls>/ -- test_program
$ perf report --stdio
$ spe-parser perf.data -t csv
```

其中 `<controls>` 允许你可选地为收集指定各种控制选项和过滤器。`perf report` 将根据用户通过 `<controls>` 选项的请求给出常规输出。`spe-parser`[^5] 是 Arm 工程师开发的工具，用于解析捕获的 perf 记录数据并将所有 SPE 记录保存为 CSV 文件。

现在我们已经涵盖了高级采样特性，让我们来讨论它们如何用于改进性能分析。

### 精确事件

采样中的一个主要问题是精确定位导致特定性能事件的确切指令。如 [profiling] 中所讨论的，基于中断的采样是基于统计特定性能事件并等待其溢出的。当溢出发生时，处理器需要一些时间来停止执行并标记导致溢出的指令。这对于现代复杂的乱序执行（out-of-order）CPU 架构尤其困难。

这引入了偏移（skid）的概念，定义为导致事件的指令地址（IP）与事件被标记处的指令地址之间的距离。偏移使得发现导致性能问题的指令变得困难。考虑一个存在大量缓存缺失的应用程序，其热点汇编代码如下：

```x86asm
; load1 
; load2
; load3
```

剖析器可能会将 `load3` 标记为导致大量缓存缺失的指令，而实际上 `load1` 才是罪魁祸首。对于高性能处理器，这种偏移可能达到数百条处理器指令。这通常会给性能工程师带来很大困惑。感兴趣的读者可以在 [Intel Developer Zone 网站](https://software.intel.com/en-us/vtune-help-hardware-event-skid)[^4] 上了解更多关于此类问题的底层原因。

由于处理器本身存储了指令指针（以及其他信息），偏移问题得到了缓解。使用 Intel PEBS，PEBS 记录中的 `EventingIP` 字段指示导致事件的指令。这通常只对受支持事件的一个子集可用，这些事件称为"精确事件"（Precise Events）。特定微架构的精确事件完整列表可在 [IntelOptimizationManual] 中找到。使用 PEBS 精确事件来缓解偏移问题的示例可以在 [easyperf 博客](https://easyperf.net/blog/2018/08/29/Understanding-performance-events-skid)[^2] 上找到。

以下是 Intel Skylake 微架构的精确事件列表：

```
INST_RETIRED.*        OTHER_ASSISTS.*    BR_INST_RETIRED.*     BR_MISP_RETIRED.*
FRONTEND_RETIRED.*    HLE_RETIRED.*      RTM_RETIRED.*         MEM_INST_RETIRED.*
MEM_LOAD_RETIRED.*    MEM_LOAD_L3_HIT_RETIRED.*
```

其中 `.*` 表示组内所有子事件均可被配置为精确事件。

使用 AMD IBS 和 Arm SPE 时，所有收集的样本在设计上都是精确的，因为硬件捕获了准确的指令地址。两者的工作方式非常相似：每当溢出发生时，机制将导致溢出的指令保存到专用缓冲区中，之后由中断处理程序读取。由于地址得以保存，IBS 和 SPE 样本的指令归因是精确的。

Intel 和 AMD 平台上的 Linux `perf` 用户必须在上述某一事件后添加 `pp` 后缀以启用精确标记，如下所示。但在 Arm 平台上，该后缀无效，用户必须使用 `arm_spe_0` 事件。

```bash
$ perf record -e cycles:pp -- ./a.exe
```

精确事件为性能工程师带来了便利，因为它们有助于避免经常令初学者甚至资深开发者感到困惑的误导性数据。TMA 方法论在定位低效执行发生的确切源代码行时，大量依赖精确事件。

### 分析内存访问

内存访问是许多应用程序性能的关键因素。PEBS 和 IBS 都支持收集程序中内存访问的详细信息。例如，可以对加载指令进行采样并收集其目标地址和访问延迟。需要注意的是，这并不追踪所有的存储和加载操作，否则开销会过大。相反，它大约每 10 万次访问才分析一次。你可以自定义每秒想要收集的样本数量。只要收集了足够多的样本，就能提供准确的统计图像。

在 PEBS 中，这一特性称为数据地址剖析（Data Address Profiling，DLA）。为了提供关于被采样加载和存储的附加信息，它使用 PEBS 设施内的 `Data Linear Address`（数据线性地址）和 `Latency Value`（延迟值）字段（见图 PEBS_record）。如果性能事件支持 DLA 设施且 DLA 已启用，处理器将转储被采样内存访问的内存地址和延迟。你还可以过滤延迟高于特定阈值的内存访问，这对于查找长延迟内存访问非常有用，而长延迟内存访问可能是许多应用程序的性能瓶颈。

使用 IBS Execute 和 Arm SPE 采样，还可以对应用程序执行的内存访问进行深入分析。一种方法是转储收集的样本并手动处理。IBS 保存确切的线性地址、其延迟、内存位置从何处取回（缓存或 DRAM），以及是否在 DTLB 中命中或缺失。SPE 可用于估算内存子系统组件的延迟和带宽，估算单个加载/存储的内存延迟等。

这些扩展最重要的使用场景之一是检测真假共享（True and False Sharing），我们将在 [TrueFalseSharing] 中讨论。Linux `perf c2c` 工具大量依赖这三种机制（PEBS、IBS 和 SPE）来查找可能经历真假共享的争用内存访问：它匹配不同线程的加载/存储地址，并检查命中是否发生在被其他线程修改的缓存行（cache line）中。

[^1]: PEBS grabber 工具 - [https://github.com/andikleen/pmu-tools/tree/master/pebs-grabber](https://github.com/andikleen/pmu-tools/tree/master/pebs-grabber). 需要 root 访问权限。
[^2]: 性能偏移 - [https://easyperf.net/blog/2018/08/29/Understanding-performance-events-skid](https://easyperf.net/blog/2018/08/29/Understanding-performance-events-skid)
[^4]: 硬件事件偏移 - [https://software.intel.com/en-us/vtune-help-hardware-event-skid](https://software.intel.com/en-us/vtune-help-hardware-event-skid)
[^5]: Arm SPE 解析器 - [https://gitlab.arm.com/telemetry-solution/telemetry-solution](https://gitlab.arm.com/telemetry-solution/telemetry-solution)
[^6]: 首先需要安装 `arm_spe` 的 Linux perf 驱动程序（参见 [https://developer.arm.com/documentation/ka005362/latest/](https://developer.arm.com/documentation/ka005362/latest/)）。在 Amazon Linux 2 和 2023 上，SPE PMU 在 Graviton metal 实例上默认可用（参见 [https://github.com/aws/aws-graviton-getting-started/blob/main/perfrunbook/debug_hw_perf.md](https://github.com/aws/aws-graviton-getting-started/blob/main/perfrunbook/debug_hw_perf.md)）。
