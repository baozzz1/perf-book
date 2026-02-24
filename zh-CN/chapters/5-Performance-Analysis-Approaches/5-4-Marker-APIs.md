### 使用标记 API（Using Marker APIs）

在某些情况下，我们可能对分析特定代码区域的性能感兴趣，而不是整个应用程序。这可能是你正在开发一段新代码并希望专注于该代码的情况。自然地，你希望跟踪优化进度并捕获额外的性能数据来帮助你。大多数性能分析工具提供了特定的*标记 API*（marker APIs），让你可以做到这一点。以下是两个示例：

* Intel VTune 有 `__itt_task_begin / __itt_task_end` 函数。
* AMD uProf 有 `amdProfileResume / amdProfilePause` 函数。

这种混合方法结合了插桩和性能事件计数的优点。标记 API 不是测量整个程序，而是允许我们将性能统计数据归属到代码区域（循环、函数）或功能片段（远程过程调用（RPCs）、输入事件等）。你获取的数据质量可以轻松证明所付出的努力是值得的。例如，在调查仅在特定类型 RPC 下发生的性能错误时，你可以仅为该类型的 RPC 启用监控。

下面我们提供一个使用 [libpfm4](https://sourceforge.net/p/perfmon2/libpfm4/ci/master/tree/),[^1] 的非常基本的示例，libpfm4 是用于收集性能监控事件的流行 Linux 库之一。它构建在 Linux `perf_events` 子系统之上，允许你直接访问性能事件计数器。`perf_events` 子系统相当底层，因此 `libpfm4` 包在这里很有用，因为它为识别 CPU 上可用事件添加了发现工具，并为原始 `perf_event_open` 系统调用提供了包装库。以下代码清单展示了如何使用 `libpfm4` 来对 [C-Ray](https://openbenchmarking.org/test/pts/c-ray)[^2] 基准测试的 `render` 函数进行插桩。

```cpp
+#include <perfmon/pfmlib.h>
+#include <perfmon/pfmlib_perf_event.h>
...
/* render a frame of xsz/ysz dimensions into the provided framebuffer */
void render(int xsz, int ysz, uint32_t *fb, int samples) {
   ...
+  pfm_initialize();
+  struct perf_event_attr perf_attr;
+  memset(&perf_attr, 0, sizeof(perf_attr));
+  perf_attr.size = sizeof(struct perf_event_attr);
+  perf_attr.read_format = PERF_FORMAT_TOTAL_TIME_ENABLED | 
+                          PERF_FORMAT_TOTAL_TIME_RUNNING | PERF_FORMAT_GROUP;
+   
+  pfm_perf_encode_arg_t arg;
+  memset(&arg, 0, sizeof(pfm_perf_encode_arg_t));
+  arg.size = sizeof(pfm_perf_encode_arg_t);
+  arg.attr = &perf_attr;
+   
+  pfm_get_os_event_encoding("instructions", PFM_PLM3, PFM_OS_PERF_EVENT_EXT, &arg);
+  int leader_fd = perf_event_open(&perf_attr, 0, -1, -1, 0);
+  pfm_get_os_event_encoding("cycles", PFM_PLM3, PFM_OS_PERF_EVENT_EXT, &arg);
+  int event_fd = perf_event_open(&perf_attr, 0, -1, leader_fd, 0);
+  pfm_get_os_event_encoding("branches", PFM_PLM3, PFM_OS_PERF_EVENT_EXT, &arg);
+  event_fd = perf_event_open(&perf_attr, 0, -1, leader_fd, 0);
+  pfm_get_os_event_encoding("branch-misses", PFM_PLM3, PFM_OS_PERF_EVENT_EXT, &arg);
+  event_fd = perf_event_open(&perf_attr, 0, -1, leader_fd, 0);
+
+  struct read_format { uint64_t nr, time_enabled, time_running, values[4]; };
+  struct read_format before, after;

  for(j=0; j<ysz; j++) {
    for(i=0; i<xsz; i++) {
      double r = 0.0, g = 0.0, b = 0.0;
+     // capture counters before ray tracing
+     read(event_fd, &before, sizeof(struct read_format));

      for(s=0; s<samples; s++) {
        struct vec3 col = trace(get_primary_ray(i, j, s), 0);
        r += col.x;
        g += col.y;
        b += col.z;
      }
+     // capture counters after ray tracing
+     read(event_fd, &after, sizeof(struct read_format));

+     // save deltas in separate arrays
+     nanosecs[j * xsz + i] = after.time_running - before.time_running;
+     instrs  [j * xsz + i] = after.values[0] - before.values[0];
+     cycles  [j * xsz + i] = after.values[1] - before.values[1];
+     branches[j * xsz + i] = after.values[2] - before.values[2];
+     br_misps[j * xsz + i] = after.values[3] - before.values[3];

      *fb++ = ((uint32_t)(MIN(r * rcp_samples, 1.0) * 255.0) & 0xff) << RSHIFT |
              ((uint32_t)(MIN(g * rcp_samples, 1.0) * 255.0) & 0xff) << GSHIFT |
              ((uint32_t)(MIN(b * rcp_samples, 1.0) * 255.0) & 0xff) << BSHIFT;
  } }
+ // aggregate statistics and print it
  ...
}
```

在这个代码示例中，我们首先初始化 `libpfm` 库，然后配置性能事件以及我们将用来读取它们的格式。在 C-Ray 基准测试中，`render` 函数只被调用一次。在你自己的代码中，要注意不要多次进行 `libpfm` 初始化。

然后，我们选择要分析的代码区域。在我们的情况下，它是一个内部有 `trace` 函数调用的循环。我们用两个 `read` 系统调用包围这个代码区域，它们将在循环前后捕获性能计数器的值。接下来，我们保存差值以供后续处理。在本例中，我们通过计算平均值（average）、第 90 百分位数（90th percentile）和最大值（maximum）进行了聚合（代码未显示）。在 Intel Alder Lake 机器上运行，我们得到如下输出。不需要 root 权限，但 `/proc/sys/kernel/perf_event_paranoid` 应设置为小于 1。当读取某个线程的计数器时，值仅针对该线程。它可以选择性地包括归属于该线程的内核代码。

```bash
$ ./c-ray-f -s 1024x768 -r 2 -i sphfract -o output.ppm
Per-pixel ray tracing stats:
                      avg         p90         max
-------------------------------------------------
nanoseconds   |      4571 |      6139 |     25567
instructions  |     71927 |     96172 |    165608
cycles        |     20474 |     27837 |    118921
branches      |      5283 |      7061 |     12149
branch-misses |        18 |        35 |       146
```

请记住，我们的插桩测量的是每像素光线追踪（ray tracing）统计数据。将平均数乘以像素数（`1024x768`）应该大致得到程序的总体统计数据。在这种情况下，一个好的健全性检查是运行 `perf stat` 并比较我们收集的性能事件的整体 C-Ray 统计数据。

C-ray 基准测试主要强调 CPU 核心的浮点运算性能，通常不应导致测量结果的高方差，换句话说，我们期望所有测量值都非常接近。然而，我们看到情况并非如此，`p90` 值是平均值的 1.33 倍，`max` 比平均情况慢 5 倍。最可能的解释是，对于某些像素，算法碰到了边界情况，执行了更多指令，因此运行时间更长。但通过研究源代码或扩展插桩以捕获"慢"像素的更多数据来确认假设总是好的。

我们示例中的额外插桩代码导致了 17% 的开销，这对于本地实验来说是可以接受的，但对于在生产环境中运行来说相当高。大多数大型分布式系统的目标是低于 1% 的开销，某些情况下最多 5% 是可以接受的，但用户不太可能会对 17% 的降速感到满意。管理插桩的开销至关重要，尤其是当你选择在生产环境中启用它时。

开销的计算通常以单位时间或工作量（RPC、数据库查询、循环迭代等）的发生率来表示。如果我们系统上的 `read` 系统调用大约需要 1.6 微秒的 CPU 时间，并且我们对每个像素（外层循环的一次迭代）调用它两次，则每像素的开销为 3.2 微秒的 CPU 时间。

有许多策略可以降低开销。作为一般规则，你的插桩应该始终有固定成本，例如确定性的系统调用（syscall），而不是列表遍历或动态内存分配。否则它会干扰测量结果。插桩代码有三个逻辑部分：收集信息、存储信息和报告信息。为了降低第一部分（收集）的开销，我们可以降低采样率，例如每 10 个 RPC 采样一次，跳过其余的。对于长时间运行的应用程序，性能可以通过相对廉价的随机采样来监控，即随机选择每个样本中要监控的 RPC。这些方法牺牲了收集精度，但仍然提供了对整体性能特征的良好估计，同时产生非常低的开销。

对于第二和第三部分（存储和聚合），建议仅收集、处理和保留你了解系统性能所需的数据。你可以通过使用计算均值（mean）、方差（variance）、最小值、最大值和其他指标的"在线"算法（online algorithms）来避免将每个样本存储在内存中。这将大大减少插桩的内存占用。例如，方差和标准差可以使用 Knuth 的在线方差算法计算。一个好的实现[^3]使用不到 50 字节的内存。

对于长例程，你可以在开始和结束时以及中间的某些部分收集计数器。在连续运行中，你可以通过二分搜索（binary search）找到例程中性能最差的部分并进行优化。重复此过程，直到消除所有性能较差的地方。如果尾延迟（tail latency）是主要关注点，那么在特别慢的运行中发出日志消息可以提供有用的洞察。

在我们的示例中，我们同时收集了 4 个事件，尽管 CPU 有 6 个可编程计数器。你可以打开更多组，启用不同的事件集。内核将选择不同的组依次运行。`time_enabled` 和 `time_running` 字段表示多路复用情况。它们都以纳秒为单位表示持续时间。`time_enabled` 字段表示事件组被启用了多少纳秒。`time_running` 字段表示在该启用时间内实际收集事件的时间。如果你同时启用了两个无法在硬件计数器上同时放置的事件组，你可能会看到两个组的运行时间都收敛到 `time_running = 0.5 * time_enabled`。

同时捕获多个事件使得计算我们在第 4 章中讨论的各种指标成为可能。例如，捕获 `INSTRUCTIONS_RETIRED` 和 `UNHALTED_CLOCK_CYCLES` 使我们能够测量 IPC。我们可以通过比较 CPU 周期（`UNHALTED_CORE_CYCLES`）与固定频率参考时钟（`UNHALTED_REFERENCE_CYCLES`）来观察频率缩放（frequency scaling）的影响。通过请求消耗的 CPU 周期（`UNHALTED_CORE_CYCLES`，仅在线程运行时计数）并将其与挂钟时间（wall-clock time）进行比较，可以检测线程何时未运行。此外，我们可以将数字归一化以获得每秒/每时钟/每指令的事件率。例如，通过测量 `MEM_LOAD_RETIRED.L3_MISS` 和 `INSTRUCTIONS_RETIRED`，我们可以获得 `L3MPKI` 指标。如你所见，这给了很大的灵活性。

事件分组的重要属性是计数器在同一个 `read` 系统调用下以原子方式可用。这些原子束（atomic bundles）非常有用。首先，它允许我们关联每个组内的事件。例如，假设我们测量某段代码的 IPC，发现它非常低。在这种情况下，我们可以将两个事件（指令和周期）与第三个事件（比如 L3 缓存缺失）配对，以检查该事件是否导致了我们正在处理的低 IPC。如果没有，我们可以使用其他事件继续因素分析（factor analysis）。其次，事件分组有助于在工作负载具有不同阶段时减少偏差。由于组内的所有事件都在同一时间测量，它们总是捕获相同的阶段。

在某些情况下，插桩可能成为功能或特性的一部分。例如，开发者可以实现一个插桩逻辑，用于检测 IPC 下降（例如，当有繁忙的兄弟硬件线程运行时）或 CPU 频率下降（例如，由于重负载导致系统节流）。当此类事件发生时，应用程序自动推迟低优先级工作以补偿临时增加的负载。

[^1]: libpfm4 - [https://sourceforge.net/p/perfmon2/libpfm4/ci/master/tree/](https://sourceforge.net/p/perfmon2/libpfm4/ci/master/tree/)

[^2]: C-Ray benchmark - [https://openbenchmarking.org/test/pts/c-ray](https://openbenchmarking.org/test/pts/c-ray)

[^3]: 精确计算运行方差 - [https://www.johndcook.com/blog/standard_deviation/](https://www.johndcook.com/blog/standard_deviation/)
