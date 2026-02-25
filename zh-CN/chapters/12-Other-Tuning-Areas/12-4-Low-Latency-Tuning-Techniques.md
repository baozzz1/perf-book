## 低延迟调优技术

到目前为止，我们讨论了各种旨在提升应用程序整体性能的软件优化方法。在本节中，我们将讨论低延迟系统（low-latency systems）中使用的额外调优技术，例如实时处理（real-time processing）和高频交易（HFT，High-Frequency Trading）。在此类环境中，主要的优化目标是使程序的某一特定部分运行得尽可能快。在 HFT 行业中，每一微秒和纳秒都至关重要，因为它直接影响盈利。通常，低延迟部分实现了实时系统或 HFT 系统的关键循环（critical loop），例如移动机械臂或向交易所发送订单。优化关键路径的延迟有时是以牺牲程序其他部分为代价的，甚至有些技术会以牺牲系统的整体吞吐量（throughput）为代价。

当开发者针对延迟进行优化时，他们会避免在热路径（hot path）上产生任何不必要的开销。这通常涉及系统调用（system calls）、内存分配（memory allocation）、I/O 以及任何具有不确定性延迟的操作。要达到尽可能低的延迟，热路径需要让所有资源立即就绪可用。

一种相对简单的技术是预计算（precompute）您在热路径上需要执行的一些操作。这需要付出使用更多内存的代价，使其无法被系统中的其他进程使用，但可以在关键路径上节省宝贵的周期。但请记住，有时计算结果比从内存中取回结果更快。

由于这是一本关于底层 CPU 性能的书，我们将跳过与我们刚才提到的高层技术类似的讨论。相反，我们将讨论如何在关键路径上避免页错误（page faults）、缓存未命中（cache misses）、TLB 击落（TLB shootdowns）以及核心降频（core throttling）。

### 避免次要页错误

尽管术语中包含"次要（minor）"一词，但次要页错误（minor page faults）对运行时延迟的影响绝非次要。回顾一下，当用户代码分配内存时，操作系统只承诺提供一个页面，但不会立即通过给予已清零的物理页面来履行承诺。相反，它会等到用户代码第一次访问该页面时，才由操作系统履行其职责。首次向新分配的页面写入会触发次要页错误，这是一个由操作系统处理的硬件中断（hardware interrupt）。次要故障的延迟影响可以从不到 1 微秒到数微秒不等，尤其是当您使用具有 5 级页表（5-level page tables）而非 4 级页表的 Linux 内核时。

如何检测应用程序中的运行时次要页错误？一种简单方法是使用 `top` 工具（添加 `-H` 选项可查看线程级视图）。在默认显示列中添加 `vMn` 字段，以查看每个显示刷新间隔内发生的次要页错误数量。代码清单 DumpTopWithMinorFaults 展示了在编译一个大型 C++ 项目时，`top` 命令的输出转储，其中显示了前 10 个进程。额外的 `vMn` 列显示了最近 3 秒内发生的次要页错误数量。

代码清单：在编译大型 C++ 项目时，带有额外 vMn 字段的 Linux top 命令输出转储。

```cpp
   PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND  vMn
341763 dendiba+  20   0  303332 165396  83200 R  99.3   1.0   0:05.09 c++      13k
341705 dendiba+  20   0  285768 153872  87808 R  99.0   1.0   0:07.18 c++       5k
341719 dendiba+  20   0  313476 176236  83328 R  94.7   1.1   0:06.49 c++       8k
341709 dendiba+  20   0  301088 162800  82944 R  93.4   1.0   0:06.46 c++       2k
341779 dendiba+  20   0  286468 152376  87424 R  92.4   1.0   0:03.08 c++      26k
341769 dendiba+  20   0  293260 155068  83072 R  91.7   1.0   0:03.90 c++      22k
341749 dendiba+  20   0  360664 214328  75904 R  88.1   1.3   0:05.14 c++      18k
341765 dendiba+  20   0  351036 205268  76288 R  87.1   1.3   0:04.75 c++      18k
341771 dendiba+  20   0  341148 194668  75776 R  86.4   1.2   0:03.43 c++      20k
341776 dendiba+  20   0  286496 147460  82432 R  76.2   0.9   0:02.64 c++      25k
```
另一种检测运行时次要页错误的方法是使用 `perf stat -e page-faults` 挂载到正在运行的进程上。

在 HFT 领域，任何超过 `0` 的值都是问题。但对于其他业务领域的低延迟应用程序，每秒持续发生 100-1000 次故障应促使进一步调查。调查运行时次要页错误的根本原因可以简单到执行 `perf record -e page-faults`，然后执行 `perf report` 来定位有问题的源代码行。

为了避免运行时的页错误损失，您应该在应用程序启动时预先故障（pre-fault）所有内存。一个简单示例如下：

```cpp
char *mem = malloc(size);
int pageSize = sysconf(_SC_PAGESIZE)
for (int i = 0; i < size; i += pageSize)
  mem[i] = 0;
```

首先，此示例代码像往常一样在堆上分配 `size` 大小的内存。然而，紧接着，它逐步向新分配内存的每个页面的第一个字节写入，以确保每个页面都被加载到 RAM 中。这种方法有助于避免未来访问时因次要页错误而导致的运行时延迟。

请参阅代码清单 LockPagesAndNoRelease，它展示了结合 `mlock/mlockall` 系统调用调优 glibc 分配器的更全面方法（取自"Real-time Linux Wiki"[^1]）。

代码清单：调优 glibc 分配器以将页面锁定在 RAM 中并阻止将其释放给操作系统。

```cpp
#include <malloc.h>
#include <sys/mman.h>

mallopt(M_MMAP_MAX, 0);
mallopt(M_TRIM_THRESHOLD, -1);
mallopt(M_ARENA_MAX, 1);

mlockall(MCL_CURRENT | MCL_FUTURE);

char *mem = malloc(size);
for (int i = 0; i < size; i += sysconf(_SC_PAGESIZE))
    mem[i] = 0;
//...
free(mem);
```
代码清单 LockPagesAndNoRelease 中的代码调整了三个 glibc malloc 设置：`M_MMAP_MAX`、`M_TRIM_THRESHOLD` 和 `M_ARENA_MAX`。

- 将 `M_MMAP_MAX` 设置为 `0` 会禁用大型分配使用底层 `mmap` 系统调用——这是必要的，因为当库尝试将 `mmap` 映射的段释放回操作系统时，`munmap` 的使用可能撤销 `mlockall` 的效果，从而使我们的努力付诸东流。
- 将 `M_TRIM_THRESHOLD` 设置为 `-1` 可防止 glibc 在调用 `free` 后将内存归还给操作系统。如前所述，此选项对 `mmap` 映射的段无效。
- 最后，将 `M_ARENA_MAX` 设置为 `1` 可防止 glibc 通过 `mmap` 分配多个内存竞技场（arenas）以适应多核处理器。请注意，后者会妨碍 glibc 分配器的多线程可扩展性功能。

这些设置组合起来，迫使 glibc 使用堆分配，在应用程序结束之前不会将内存释放回操作系统。因此，上述代码中最后一次调用 `free(mem)` 后，堆的大小将保持不变。任何后续的运行时 `malloc` 或 `new` 调用，如果初始化时预分配/预故障的堆区足够大，将简单地复用其中的空间。

更重要的是，在 `for` 循环中预故障的所有堆内存，将由于之前的 `mlockall` 调用而持续驻留在 RAM 中——选项 `MCL_CURRENT` 锁定当前已映射的所有页面，而 `MCL_FUTURE` 锁定将来映射的所有页面。以这种方式使用 `mlockall` 的一个额外好处是，此进程产生的任何线程的栈也将被预故障并锁定。为了更精细地控制页面锁定，开发者应使用 `mlock` 系统调用，该调用允许您选择哪些页面应驻留在 RAM 中。这种技术的缺点是它减少了系统上其他进程可用的内存量。

Windows 应用程序开发者应了解以下 API：使用 `VirtualLock` 锁定页面，使用带有 `MEM_DECOMMIT` 标志（但不使用 `MEM_RELEASE` 标志）的 `VirtualFree` 来避免立即释放内存。

以上只是防止运行时次要故障的两种示例方法。其中部分或全部技术可能已集成到 jemalloc、tcmalloc 或 mimalloc 等内存分配库中。请查阅您所用库的文档以了解可用功能。

### 缓存预热

在某些应用程序中，对延迟最敏感的代码部分恰恰是执行频率最低的部分。此类应用程序的一个例子可能是 HFT 应用程序，它持续从股票交易所读取市场数据信号，一旦检测到有利的市场信号，便向交易所发送订单。在上述工作负载中，读取市场数据的代码路径执行最频繁，而执行订单的代码路径则很少执行。

由于市场上的其他参与者可能捕捉到相同的市场信号，策略的成功在很大程度上取决于我们能多快做出反应，换句话说，就是我们能多快向交易所发送订单。当我们希望订单以最快速度到达交易所，利用在市场数据中检测到的有利信号时，我们最不希望的就是在决定出发的那一刻遇到障碍。

当某条代码路径有一段时间未被执行时，其指令和相关数据很可能已从指令缓存（I-cache）和数据缓存（D-cache）中被驱逐。然后，就在我们需要那段很少执行的关键代码运行时，我们却承受了 I-cache 和 D-cache 未命中的惩罚，这可能导致我们在竞争中落败。这就是*缓存预热（cache warming）*技术大显身手之处。

缓存预热（cache warming）涉及周期性地执行对延迟敏感的代码，使其保留在缓存中，同时确保不会执行任何不希望发生的操作。执行延迟敏感代码还能通过将对延迟敏感的数据加载到 D-cache 中来"预热" D-cache。这种技术在 HFT 应用程序中被广泛使用。虽然我不会提供具体的实现示例，但您可以在 [CppCon 2018 闪电演讲](https://www.youtube.com/watch?v=XzRxikGgaHI)[^4] 中一窥究竟。

### 避免 TLB 击落

我们从前面的章节中了解到，TLB（转换后备缓冲区，Translation Lookaside Buffer）是一种快速但容量有限的每核缓存，用于存储虚拟到物理内存地址的转换，从而减少耗时的内核页表遍历需求。与基于 MESI 协议的每核 CPU 缓存（即 L1、L2 和 LLC）不同，硬件本身并不维护核间 TLB 一致性。因此，这项任务必须由操作系统在软件层面完成。

在多线程应用程序中，进程线程共享虚拟地址空间。因此，内核必须在执行任何参与线程的核的 TLB 之间，传达对该共享地址空间的特定类型更新。例如，常用的系统调用（如 `munmap`（可以从 glibc 分配器使用中禁用，参见 [AvoidPageFaults]）、`mprotect` 和 `madvise`）可能会使 TLB 条目失效。这些更新必须在进程的各个组成线程之间传达。内核使用一种特定类型的处理器间中断（IPI，Inter Processor Interrupts）来完成这项工作，称为 *TLB 击落（TLB shootdowns）*，在 x86 平台上通过 `INVLPG` 汇编指令实现。TLB 击落是多线程应用程序中实现低延迟最容易被忽视的陷阱之一。

尽管开发者可能在其代码中避免显式使用上述系统调用，TLB 击落仍可能从外部来源爆发——例如，共享内存分配库或操作系统功能。这种类型的 IPI 不仅会干扰运行时应用程序性能，而且其影响的严重程度会随着涉及的线程数量增加而增大，因为中断是在软件中传递的。

如何在多线程应用程序中检测 TLB 击落？一种简单方法是查看 `/proc/interrupts` 中的 TLB 行。在运行时持续检测 TLB 中断的一个有用方法是使用 `watch` 命令查看此文件。例如，您可以运行 `watch -n5 -d 'grep TLB /proc/interrupts'`，其中 `-n 5` 选项每 5 秒刷新视图，而 `-d` 则高亮显示每次刷新输出之间的差量。

代码清单 ProcInterrupts 展示了 `/proc/interrupts` 的一个转储，其中运行延迟关键线程的 `CPU2` 处理器上有大量 TLB 击落。注意与其他核心之间的数量级差异。在该场景中，此类行为的罪魁祸首是 Linux 内核的一项名为自动 NUMA 均衡（Automatic NUMA Balancing）的功能，可通过 `sysctl -w numa_balancing=0` 轻松禁用。

代码清单：显示 CPU2 上大量 TLB 击落的 /proc/interrupts 转储

```cpp
           CPU0       CPU1       CPU2       CPU3       
...
NMI:          0          0          0          0   Non-maskable interrupts
LOC:     552219    1010298    2272333    3179890   Local timer interrupts
SPU:          0          0          0          0   Spurious interrupts
...
IWI:          0          0          0          0   IRQ work interrupts
RTR:          7          0          0          0   APIC ICR read retries
RES:      18708       9550        771        528   Rescheduling interrupts
CAL:        711        934       1312       1261   Function call interrupts
TLB:       4493       6108      73789       5014   TLB shootdowns
```
但这并不是 TLB 击落的唯一来源。其他来源包括透明大页（Transparent Huge Pages）、内存压缩（memory compaction）、页面迁移（page migration）和页面缓存回写（page cache writeback）。垃圾收集器（Garbage collectors）也可能发起 TLB 击落。这些功能在履行职责的过程中会重新定位页面和/或更改进程页面的权限，这需要更新页表，从而导致 TLB 击落。

防止 TLB 击落需要限制对共享进程地址空间所做的更新次数。在源代码层面，应避免在运行时执行上述系统调用列表，即 `munmap`、`mprotect` 和 `madvise`。在操作系统层面，禁用作为其功能副作用而引发 TLB 击落的内核功能，例如透明大页和自动 NUMA 均衡。有关 TLB 击落的更深入讨论，包括其检测和预防，请阅读 JabPerf 博客上的相关文章[^5]。

### 防止意外的核心降频

C/C++ 编译器是工程学的杰出成果。然而，它们有时会生成令人意外的结果，可能让您走上漫长的排错之路。一个真实案例是编译器优化器生成了您从未打算使用的重型 AVX512 指令。虽然在更现代的芯片上问题较小，但许多较旧世代的 CPU（在本地和云端仍在大量使用）在执行重型 AVX512 指令时会出现严重的核心降频/降速（core throttling/downclocking）。如果您的编译器在您不知情或未经同意的情况下生成这些指令，您可能会在应用程序运行时遇到无法解释的延迟异常。

对于这种特定情况，如果不需要大量使用 AVX512 指令，可以在编译标志中添加 `-mprefer-vector-width=###`，将最高宽度指令集固定为 128 或 256。同样，如果整个服务器集群运行在最新的芯片上，那么这个问题几乎不值一提，因为现在 AVX 指令集的降频影响可以忽略不计。

[^1]: Linux Foundation Wiki：实时应用程序的内存 - [https://wiki.linuxfoundation.org/realtime/documentation/howto/applications/memory](https://wiki.linuxfoundation.org/realtime/documentation/howto/applications/memory)
[^4]: 缓存预热技术 - [https://www.youtube.com/watch?v=XzRxikGgaHI](https://www.youtube.com/watch?v=XzRxikGgaHI)
[^5]: JabPerf 博客：TLB 击落 - [https://www.jabperf.com/how-to-deter-or-disarm-tlb-shootdowns/](https://www.jabperf.com/how-to-deter-or-disarm-tlb-shootdowns/)
