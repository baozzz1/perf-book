## 并行效率指标

首先介绍几个对分析多线程应用程序性能非常重要的指标。在处理多线程应用程序时，工程师在分析基本指标时需要格外谨慎，例如 CPU 利用率。某个线程可能显示出较高的 CPU 利用率，但结果可能只是该线程在等待锁时处于忙等待（busy-wait）循环中。因此，在评估应用程序的并行效率时，建议使用*有效 CPU 利用率*（Effective CPU Utilization），该指标仅基于*有效时间*（Effective time）。

### 有效 CPU 利用率

该指标表示应用程序对可用 CPU 的有效利用程度。它显示系统上所有逻辑 CPU 的平均 CPU 利用率百分比。该指标仅基于*有效时间*，不包括并行运行时系统[^11]引入的开销和自旋时间（Spin time）。*有效 CPU 利用率*为 100% 表示应用程序在整个运行期间始终保持所有逻辑 CPU 核心忙碌。

对于指定的时间间隔 `T`，*有效 CPU 利用率*可计算如下：

$$
\textrm{Effective CPU Utilization} = \frac{\sum_{i=1}^{\textrm{ThreadCount}}\textrm{Effective CPU Time(T,i)}}{\textrm{T}~\times~\textrm{ThreadCount}}
$$

$$
\textrm{Effective CPU Time} = \textrm{CPU Time}~-~(\textrm{Overhead Time}~+~\textrm{Spin Time})
$$

测量开销时间和自旋时间可能具有挑战性，建议使用 Intel VTune Profiler 等性能分析工具，它可以提供这些指标。

### 线程数

大多数并行应用程序都有可配置的线程数，使其能够在具有不同核心数量的平台上高效运行。使用少于系统可用线程数的线程运行应用程序会造成资源利用不足。另一方面，运行过多线程可能导致*过度订阅*（oversubscription）；某些线程将等待轮到自己运行。

除实际工作线程外，多线程应用程序通常还有其他管理线程：主线程、输入/输出线程等。如果这些线程消耗大量时间，它们将占用工作线程的执行时间，因为它们也需要 CPU 核心来运行。这就是为什么了解总线程数并适当配置工作线程数量非常重要。

为了避免线程创建和销毁的开销，工程师通常分配一个[线程池](https://en.wikipedia.org/wiki/Thread_pool)[^14]，其中多个线程等待由监督程序分配任务以并发执行。这对于执行短生命周期任务尤为有益。

### 等待时间

*等待时间*（Wait Time）发生在软件线程因阻塞或导致上下文切换的 API 调用而等待时。等待时间是按线程计算的；因此，总*等待时间*可能超过应用程序的经过时间。

OS 调度器可能因同步或抢占而将线程切换出执行。因此，*等待时间*可进一步划分为*同步等待时间*（Sync Wait Time）和*抢占等待时间*（Preemption Wait Time）。大量的*同步等待时间*可能表明应用程序具有竞争激烈的同步对象。我们将在后续章节中探讨如何找到它们。显著的*抢占等待时间*可能表明线程过度订阅问题，原因可能是应用程序线程数过多，或与系统上的 OS 线程或其他应用程序冲突。在这种情况下，开发人员应考虑减少总线程数或增加每个工作线程的任务粒度。

### 自旋时间

*自旋时间*（Spin time）是 CPU 处于忙碌状态的*等待时间*。这通常发生在同步 API 导致 CPU 在软件线程等待时轮询的情况下。实际上，内核同步原语的实现会在锁上自旋一段时间，而不是立即让出给另一个线程。然而，过多的自旋时间可能会浪费本可用于生产性工作的机会。

其他并行效率指标列表可在 Intel 的 VTune [页面](https://software.intel.com/en-us/vtune-help-cpu-metrics-reference)中找到。[^15]

[^11]: 线程库（如 `pthread`、`OpenMP` 和 `Intel TBB`）会产生用于创建和管理线程的额外开销。
[^14]: Thread pool - [https://en.wikipedia.org/wiki/Thread_pool](https://en.wikipedia.org/wiki/Thread_pool)
[^15]: CPU metrics reference - [https://software.intel.com/en-us/vtune-help-cpu-metrics-reference](https://software.intel.com/en-us/vtune-help-cpu-metrics-reference)
