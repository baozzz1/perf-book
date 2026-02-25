## Linux Perf

Linux Perf 可能是世界上使用最广泛的性能分析器，因为它在大多数 Linux 发行版上都可用，使得广大用户能够轻松获取。Perf 在许多流行的 Linux 发行版中原生支持，包括 Ubuntu、Red Hat 和 Debian。它被包含在内核中，因此可以在任何运行 Linux 的系统上获取操作系统级别的统计信息（缺页错误、CPU 迁移等）。截至 2024 年中期，该分析器支持 x86、ARM、PowerPC64、UltraSPARC 以及其他几种 CPU 类型。[^2] 在这些平台上，`perf` 提供对硬件性能监控特性的访问，例如性能计数器。更多关于 Linux `perf` 的信息可在其[维基页面](https://perf.wiki.kernel.org/index.php/Main_Page)获取。[^1]

### 如何配置

安装 Linux perf 非常简单，只需一条命令：

```bash
$ sudo apt-get install linux-tools-common linux-tools-generic linux-tools-`uname -r`
```

另外，在安全性不是主要考虑因素的情况下，建议修改以下默认设置：

```bash
# 允许非特权用户进行内核分析和访问 CPU 事件
$ echo 0 | sudo tee /proc/sys/kernel/perf_event_paranoid
$ echo kernel.perf_event_paranoid=0 | sudo tee -a /etc/sysctl.d/local.conf
# 为非特权用户启用内核模块符号解析
$ echo 0 | sudo tee /proc/sys/kernel/kptr_restrict
$ echo kernel.kptr_restrict=0 | sudo tee -a /etc/sysctl.d/local.conf
```

### 可以用它做什么

通常，Linux `perf` 可以完成其他分析器能做的大多数相同的事情。硬件厂商优先在 Linux `perf` 中启用其特性，使得当新 CPU 上市时，`perf` 已经提供了相应支持。有两个主要命令是大多数人常用的。第一个 `perf stat` 报告指定性能事件的计数。第二个 `perf record` 以采样模式对应用程序或系统进行分析，通常紧随其后的是 `perf report`，用于从采样数据生成报告。

`perf record` 命令的输出是原始样本转储文件（raw dump）。许多构建在 Linux `perf` 之上的工具可以解析这些原始转储文件并提供新的分析类型。以下是最值得注意的几个：

- 火焰图（Flame graphs），在 [secFlameGraphs] 中讨论。
- [KDAB Hotspot](https://github.com/KDAB/hotspot)，[^3] 一款以与 Intel VTune 非常相似的界面可视化 Linux `perf` 数据的工具。如果你有使用 Intel VTune 的经验，KDAB Hotspot 会让你感到非常熟悉。
- Netflix [Flamescope](https://github.com/Netflix/flamescope)。[^4] 该工具显示应用程序运行期间采样事件的热力图（heat map）。可以观察工作负载行为的不同阶段和模式。Netflix 工程师使用此工具发现了一些非常微妙的性能缺陷。此外，还可以在热力图上选择时间范围，并为该时间范围生成火焰图。

### 不能用它做什么

Linux perf 是一款命令行工具，缺乏图形用户界面（GUI），这使得过滤数据、观察工作负载行为随时间的变化、缩放到运行时的某一部分等操作较为困难。通过 `perf report` 命令提供了有限的控制台输出，对于快速分析而言尚可，但不如其他 GUI 分析器方便。幸运的是，正如我们刚才提到的，有 GUI 工具可以对 Linux `perf` 的原始输出进行后处理和可视化。

[^1]: Linux perf wiki - [https://perf.wiki.kernel.org/index.php/Main_Page](https://perf.wiki.kernel.org/index.php/Main_Page).
[^2]: RISCV 尚未作为官方内核的一部分获得支持，尽管来自厂商的定制工具已经存在。
[^3]: KDAB Hotspot - [https://github.com/KDAB/hotspot](https://github.com/KDAB/hotspot).
[^4]: Netflix Flamescope - [https://github.com/Netflix/flamescope](https://github.com/Netflix/flamescope).
