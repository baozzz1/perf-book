## 采样（Sampling）

采样是进行性能分析最常用的方法。人们通常将其与在程序中查找热点（hotspots）相关联。更广泛地说，采样有助于找到代码中对某些性能事件数量贡献最多的位置。如果我们要查找热点，问题可以被重新表述为："找到代码中消耗最多 CPU 周期的位置"。人们经常将*性能分析*（profiling）这一术语用于技术上称为*采样*（sampling）的概念。根据[维基百科](https://en.wikipedia.org/wiki/Profiling_(computer_programming))，[^1] 性能分析是一个更广泛的术语，包含收集数据的各种技术，包括采样、代码插桩、跟踪等。

可能令人惊讶的是，能想象到的最简单的采样性能分析器是一个调试器（debugger）。事实上，你可以按如下方式识别热点：a) 在调试器下运行程序；b) 每 10 秒暂停一次程序；c) 记录程序停止的位置。如果你多次重复 b) 和 c)，就可以从收集的样本中构建一个直方图（histogram）。程序停止最多的代码行将是程序中最热的地方。当然，这不是查找热点的有效方法，我们不建议这样做。这只是为了说明概念。尽管如此，这是对真实性能分析工具工作方式的简化描述。现代性能分析工具每秒能够收集数千个样本，这对程序中最热的位置给出了相当准确的估计。

就像调试器示例一样，每次捕获新样本时，被分析程序的执行就会被中断。在中断时，性能分析工具收集程序状态的快照，构成一个样本。每个样本收集的信息可能包括：中断时 CPU 正在执行的指令地址、寄存器状态、调用栈（call stack，参见 [secCollectCallStacks]）等。收集的样本存储在转储文件（dump file）中，可进一步用于显示程序最耗时的部分、调用图（call graph）等。

### 用户模式采样与基于硬件事件的采样（User-Mode and Hardware Event-based Sampling）

采样可以以 2 种不同模式执行，使用用户模式采样（user-mode sampling）或基于硬件事件的采样（hardware event-based sampling，EBS）。用户模式采样是一种纯软件方法，它将代理库（agent library）嵌入到被分析的应用程序中。代理为应用程序中的每个线程设置操作系统定时器。定时器到期时，应用程序接收到由代理处理的 `SIGPROF` 信号。EBS 使用硬件 PMC 触发中断。特别地，使用了 PMU 的计数器溢出特性，我们将很快讨论这一点。

用户模式采样只能用于识别热点，而 EBS 可用于涉及 PMC 的其他分析类型，例如基于缓存缺失的采样、自顶向下微架构分析（参见 [TMA]）等。

用户模式采样比 EBS 产生更高的运行时开销。以 10ms 的间隔采样时，用户模式采样的平均开销约为 5%，而 EBS 的开销低于 1%。由于开销更低，你可以使用更高的采样率来使用 EBS，这将提供更准确的数据。但是，用户模式采样生成的数据较少，处理时间也较短。

### 查找热点（Finding Hotspots）

在本节中，我们将讨论将 PMC 与 EBS 结合使用的机制。图 Sampling 说明了 PMU 的计数器溢出特性，它用于触发性能监控中断（Performance Monitoring Interrupt，PMI），也称为 `SIGPROF`。在基准测试开始时，我们配置要采样的事件。默认情况下，许多性能分析工具会对周期进行采样，因为我们想知道程序将大部分时间花费在哪里。但这不一定是严格的规则；我们可以对任何我们想要的性能事件进行采样。例如，如果我们想知道程序中 L3 缓存缺失次数最多的位置，我们会对相应的事件进行采样，即 `MEM_LOAD_RETIRED.L3_MISS`。

![使用性能计数器进行采样](../../img/perf-analysis/SamplingFlow.png)

初始化寄存器后，我们开始计数并让基准测试运行。由于我们已将 PMC 配置为统计周期，它将在每个周期递增。最终，它会溢出。寄存器溢出时，硬件将触发一个 PMI。性能分析工具被配置为捕获 PMI，并有一个中断服务例程（Interrupt Service Routine，ISR）来处理它们。我们在 ISR 内执行多个步骤：首先，我们禁用计数；之后，我们记录计数器溢出时 CPU 正在执行的指令；然后，我们将计数器重置为 `N` 并恢复基准测试。

现在，让我们回到值 `N`。使用这个值，我们可以控制希望多频繁获得新的中断。假设我们想要更细的粒度，每 100 万个周期获取一个样本。为此，我们可以将计数器设置为 `(unsigned) -1,000,000`，使其在每 100 万个周期后溢出。这个值也称为*采样后*（sample after）值。

我们多次重复该过程以构建足够的样本集合。如果我们之后对这些样本进行聚合，就可以构建一个程序中最热位置的直方图，如下面 Linux `perf record/report` 输出中所示。这给出了程序函数开销按降序排列的分解（热点）。以下是对 [Phoronix 测试套件](https://www.phoronix-test-suite.com/),[^8] 中 [x264](https://openbenchmarking.org/test/pts/x264)[^7] 基准测试进行采样的示例：

```bash
$ time -p perf record -F 1000 -- ./x264 -o /dev/null --slow --threads 1 ../Bosphorus_1920x1080_120fps_420_8bit_YUV.y4m
[ perf record: Captured and wrote 1.625 MB perf.data (35035 samples) ]
real 36.20 sec
$ perf report -n --stdio
# Samples: 35K of event 'cpu_core/cycles/'
# Event count (approx.): 156756064947

# Overhead  Samples  Shared Object  Symbol                                                     
# ........  .......  .............  ........................................
  7.50%     2620     x264           [.] x264_8_me_search_ref
  7.38%     2577     x264           [.] refine_subpel.lto_priv.0
  6.51%     2281     x264           [.] x264_8_pixel_satd_8x8_internal_avx2
  6.29%     2212     x264           [.] get_ref_avx2.lto_priv.0
  5.07%     1787     x264           [.] x264_8_pixel_avg2_w16_sse2
  3.26%     1145     x264           [.] x264_8_mc_chroma_avx2
  2.88%     1013     x264           [.] x264_8_pixel_satd_16x8_internal_avx2
  2.87%     1006     x264           [.] x264_8_pixel_avg2_w8_mmx2
  2.58%      904     x264           [.] x264_8_pixel_satd_8x8_avx2
  2.51%      882     x264           [.] x264_8_pixel_sad_16x16_sse2
  ...
```

Linux `perf` 共收集了 `35,035` 个样本，这意味着发生了相同数量的进程中断。我们还使用了 `-F 1000`，将采样率设置为每秒 1000 个样本。这与整体运行时间 36.2 秒大致吻合。请注意，Linux `perf` 提供了经过的总周期的近似数量。如果我们用样本数来除它，我们得到 `156756064947 cycles / 35035 samples = 4.5 million cycles`（每个样本 450 万周期）。这意味着 Linux `perf` 将 `N` 设置为大约 `4500000` 以每秒收集 1000 个样本。Linux `perf` 可以根据实际 CPU 频率动态调整数字 `N`。

当然，对我们最有价值的是按归属于每个函数的样本数排序的热点列表。在知道最热的函数是什么之后，我们可能希望深入一个层次：每个函数内部的热代码部分是什么？要查看也被内联（inlined）的函数的性能分析数据以及为特定源代码区域生成的汇编代码，我们需要使用调试信息（`-g` 编译器标志）构建应用程序。

Linux `perf` 没有丰富的图形支持，因此查看源代码的热点部分不是很方便，但仍然可行。Linux `perf` 将源代码与生成的汇编代码混合显示，如下所示：

```bash
# snippet of annotating source code of 'x264_8_me_search_ref' function
$ perf annotate x264_8_me_search_ref --stdio
Percent | Source code & Disassembly of x264 for cycles:ppp 
----------------------------------------------------------
  ...
        :                 bmx += square1[bcost&15][0];   <== source code
  1.43  : 4eb10d:  movsx  ecx,BYTE PTR [r8+rdx*2]        <== corresponding machine code
        :                 bmy += square1[bcost&15][1];
  0.36  : 4eb112:  movsx  r12d,BYTE PTR [r8+rdx*2+0x1]
        :                 bmx += square1[bcost&15][0];
  0.63  : 4eb118:  add    DWORD PTR [rsp+0x38],ecx
        :                 bmy += square1[bcost&15][1];
  ...
```

大多数具有图形用户界面（GUI）的性能分析工具，如 Intel VTune Profiler，可以并排显示源代码和相关汇编代码。此外，还有一些工具可以通过类似于 Intel VTune 和其他工具的丰富图形界面来可视化 Linux `perf` 原始数据的输出。你将在 [secOverviewPerfTools] 中更详细地了解这些内容。

采样能对程序执行提供良好的统计表示，但这种技术的一个缺点是它有盲点，不适合检测异常行为。每个样本代表程序执行某一部分的聚合视图。聚合并不能给我们提供那段时间间隔内究竟发生了什么的足够细节。我们无法放大以进一步了解执行细节。当我们将时间间隔压缩为样本时，我们丢失了有价值的信息，这对分析持续时间非常短的事件变得毫无用处。例如，对响应网络数据包的程序（如股票交易软件）进行性能分析可能并不十分有用，因为它会将大多数样本归属于忙等待循环（busy wait loop）。增加采样间隔，例如每秒超过 1000 个样本，可能会给你更好的图像，但仍然可能不够。作为解决方案，你应该使用跟踪，因为它不跳过感兴趣的事件。

### 收集调用栈（Collecting Call Stacks）

在采样时，我们经常可能遇到这样的情况：程序中最热的函数从多个函数调用。图 CallStacks 中展示了这种场景的一个示例。性能分析工具的输出可能会揭示 `foo` 是程序中最热的函数之一，但如果它有多个调用者，我们想知道哪个调用者调用 `foo` 的次数最多。对于在热点中出现 `memcpy` 或 `sqrt` 等库函数的应用程序来说，这是一种典型情况。要理解为什么某个特定函数出现为热点，我们需要知道程序的控制流图（Control Flow Graph，CFG）中哪条路径对此负责。

![控制流图：热函数"foo"有多个调用者。](../../img/perf-analysis/CallStacksCFG.png)

分析 `foo` 的所有调用者的源代码可能非常耗时。我们只想关注那些导致 `foo` 成为热点的调用者。换句话说，我们想找出程序 CFG 中最热的路径。性能分析工具通过在收集性能样本时捕获进程的调用栈以及其他信息来实现这一点。然后，所有收集的栈被分组，使我们能够看到导致特定函数的最热路径。

在 Linux `perf` 中收集调用栈有三种方法：

1. 帧指针（frame pointers）（`perf record --call-graph fp`）。它要求二进制文件使用 `--fno-omit-frame-pointer` 构建。从历史上看，帧指针（`RBP` 寄存器）被用于调试，因为它使我们能够在不弹出堆栈上所有参数的情况下获取调用栈（也称为*栈展开*（stack unwinding））。帧指针可以立即告诉返回地址。它能实现非常廉价的栈展开，从而降低性能分析开销，但仅为此目的消耗了一个额外的寄存器。在架构寄存器数量较少的时候，使用帧指针在运行时性能方面代价高昂。如今，Linux 社区正在重新使用帧指针，因为它提供了更好质量的调用栈和较低的性能分析开销。
2. DWARF 调试信息（`perf record --call-graph dwarf`）。它要求二进制文件使用 DWARF 调试信息（`-g`）构建。它也通过栈展开过程获取调用栈，但此方法比使用帧指针开销更大。
3. Intel 最后分支记录（Last Branch Record，LBR）。此方法利用硬件特性，通过以下命令访问：`perf record --call-graph lbr`。它通过解析 LBR 栈（一组硬件寄存器）来获取调用栈。生成的调用图不如前两种方法产生的那么深。更多关于 LBR 调用栈模式的信息，请参见 [lbr]。

下面是使用 LBR 在程序中收集调用栈的示例。通过查看输出，我们知道 55% 的时间 `foo` 是从 `func1` 调用的，33% 的时间从 `func2` 调用，11% 从 `fun3` 调用。我们可以清楚地看到 `foo` 调用者之间开销的分布，现在可以将注意力集中在程序 CFG 中最热的边上，即 `func1` &rarr; `foo`，但我们也应该关注 `func2` &rarr; `foo` 这条边。

```bash
$ perf record --call-graph lbr -- ./a.out
$ perf report -n --stdio --no-children
# Samples: 65K of event 'cycles:ppp'
# Event count (approx.): 61363317007
# Overhead       Samples  Command  Shared Object     Symbol
# ........  ............  .......  ................  ......................
    99.96%         65217  a.out    a.out             [.] foo
            |
             --99.96%--foo
                        |
                        |--55.52%--func1
                        |          main
                        |          __libc_start_main
                        |          _start
                        |
                        |--33.32%--func2
                        |          main
                        |          __libc_start_main
                        |          _start
                        |
                         --11.12%--func3
                                   main
                                   __libc_start_main
                                   _start
```

使用 Intel VTune Profiler 时，你可以通过在配置分析时勾选相应的"Collect stacks"复选框来收集调用栈数据。使用命令行界面时，指定 `-knob enable-stack-collection=true` 选项。

[^1]: Profiling（维基百科）- [https://en.wikipedia.org/wiki/Profiling_(computer_programming)](https://en.wikipedia.org/wiki/Profiling_(computer_programming))。
[^7]: x264 benchmark - [https://openbenchmarking.org/test/pts/x264](https://openbenchmarking.org/test/pts/x264)。
[^8]: Phoronix test suite - [https://www.phoronix-test-suite.com/](https://www.phoronix-test-suite.com/)。
