# 附录 A. 减少测量噪声（Reducing Measurement Noise）

以下是一些可能导致性能测量非确定性（non-determinism）增加的特性示例，以及一些减少噪声的技术。我在 [secFairExperiments] 中对该主题进行了介绍。

本节内容主要针对 Linux 操作系统。建议读者自行搜索其他操作系统的配置说明。

## Dynamic Frequency Scaling {.unnumbered .unlisted}

[动态频率缩放](https://en.wikipedia.org/wiki/Dynamic_frequency_scaling)[^11]（Dynamic Frequency Scaling，DFS）是一种通过在系统运行高负载任务时自动提高 CPU 工作频率来提升系统性能的技术。以 DFS 实现为例，Intel CPU 具有 Turbo Boost 功能，AMD CPU 则采用 Turbo Core 功能。

以下是 Turbo Boost 对运行在 Intel® Core™ i5-8259U 上的单线程工作负载（workload）的影响示例：

```bash
# TurboBoost enabled
$ cat /sys/devices/system/cpu/intel_pstate/no_turbo
0
$ perf stat -e task-clock,cycles -- ./a.exe 
    11984.691958  task-clock (msec) #    1.000 CPUs utilized
  32,427,294,227  cycles            #    2.706 GHz
      11.989164338 seconds time elapsed

# TurboBoost disabled
$ echo 1 | sudo tee /sys/devices/system/cpu/intel_pstate/no_turbo
1
$ perf stat -e task-clock,cycles -- ./a.exe 
    13055.200832  task-clock (msec) #    0.993 CPUs utilized
  29,946,969,255  cycles            #    2.294 GHz
      13.142983989 seconds time elapsed
```

开启 Turbo Boost 时平均频率更高（2.7 GHz vs. 2.3 GHz）。

DFS 可以在 BIOS 中永久禁用。若要在 Linux 系统上以编程方式禁用 DFS 功能，需要 root 权限。操作方法如下：

```bash
# Intel
echo 1 > /sys/devices/system/cpu/intel_pstate/no_turbo
# AMD
echo 0 > /sys/devices/system/cpu/cpufreq/boost
```
## Simultaneous Multithreading {.unnumbered .unlisted}

许多现代 CPU 核心支持同步多线程（Simultaneous Multithreading，参见 [SMT]）。SMT 可以在 BIOS 中永久禁用。若要在 Linux 系统上以编程方式禁用 SMT，需要 root 权限。CPU 线程的兄弟对（sibling pairs）可以在以下文件中查找：

```bash
/sys/devices/system/cpu/cpuN/topology/thread_siblings_list
```

以下是如何在拥有 4 个核心和 8 个线程的 Intel® Core™ i5-8259U 上禁用核心 0 的兄弟线程：

```bash
# all 8 hardware threads enabled:
$ lscpu
...
CPU(s):              8
On-line CPU(s) list: 0-7
...
$ cat /sys/devices/system/cpu/cpu0/topology/thread_siblings_list
0,4
$ cat /sys/devices/system/cpu/cpu1/topology/thread_siblings_list
1,5
$ cat /sys/devices/system/cpu/cpu2/topology/thread_siblings_list
2,6
$ cat /sys/devices/system/cpu/cpu3/topology/thread_siblings_list
3,7

# Disabling SMT on core 0
$ echo 0 | sudo tee /sys/devices/system/cpu/cpu4/online
0
$ lscpu
CPU(s):               8
On-line CPU(s) list:  0-3,5-7
Off-line CPU(s) list: 4
...
$ cat /sys/devices/system/cpu/cpu0/topology/thread_siblings_list
0
```

此外，`lscpu --all --extended` 命令对于查看兄弟线程非常有帮助。

## Scaling Governor {.unnumbered .unlisted}

Linux 内核可以出于不同目的控制 CPU 频率，例如节省电能。在此情况下，调频调节器（scaling governor）可能会决定降低 CPU 频率。为了进行性能测量，建议将调频调节器策略设置为 `performance`，以避免低于标称频率运行。以下是如何为所有核心进行设置：

```bash
echo performance | sudo tee /sys/devices/system/cpu/cpufreq/policy*/scaling_governor
```

## CPU Affinity {.unnumbered .unlisted}

[处理器亲和性](https://en.wikipedia.org/wiki/Processor_affinity)[^8]（Processor affinity）可以将进程绑定到特定的 CPU 核心。在 Linux 中，可以使用 [`taskset`](https://linux.die.net/man/1/taskset)[^9] 工具实现。

```bash
# no affinity
$ perf stat -e context-switches,cpu-migrations -r 10 -- a.exe
               151      context-switches
                10      cpu-migrations

# process is bound to the CPU0
$ perf stat -e context-switches,cpu-migrations -r 10 -- taskset -c 0 a.exe 
               102      context-switches
                 0      cpu-migrations
```

注意 `cpu-migrations` 的次数降到了 `0`，即进程从未离开 `core0`。

另外，你也可以使用 [cset](https://github.com/lpechacek/cpuset)[^10] 工具为基准测试程序预留 CPU 核心。如果使用 `Linux perf`，至少保留两个核心，让 `perf` 运行在一个核心上，程序运行在另一个核心上。下面的命令将把所有线程移出 N1 和 N2（`-k on` 表示连内核线程也一并移出）：

```bash
$ cset shield -c N1,N2 -k on
```

下面的命令将在隔离的 CPU 上运行 `--` 之后的命令：
```bash
$ cset shield --exec -- perf stat -r 10 <cmd>
```

在 Windows 上，可以使用以下命令将程序固定到特定核心：

```bash
$ start /wait /b /affinity 0xC0 myapp.exe
```

其中 `/wait` 选项等待应用程序终止，`/b` 启动应用程序时不打开新命令窗口，`/affinity` 指定 CPU 亲和性掩码（affinity mask）。此处掩码 `0xC0` 表示应用程序将在第 6 和第 7 个核心上运行。

在 macOS 上，由于操作系统不提供相关 API，无法将线程固定到特定核心。

## Process Priority {.unnumbered .unlisted}

在 Linux 中，可以使用 `nice` 工具提高进程优先级。通过提高优先级，进程获得更多 CPU 时间，调度器（scheduler）相比普通优先级进程更倾向于调度它。优先级范围（niceness）从 `-20`（最高优先级）到 `19`（最低优先级），默认值为 `0`。

注意在前面的示例中，被测进程的执行被操作系统中断了超过 100 次。如果通过 `sudo nice -n -<N>` 提高进程优先级来运行基准测试：

```bash
$ perf stat -r 10 -- sudo nice -n -5 taskset -c 1 a.exe
    0   context-switches
    0   cpu-migrations
```

可以看到上下文切换（context switches）次数降为 `0`，进程获得了不中断的全部计算时间。

[^4]: SMT - [https://en.wikipedia.org/wiki/Simultaneous_multithreading](https://en.wikipedia.org/wiki/Simultaneous_multithreading).
[^7]: Documentation for Linux CPU frequency governors: [https://www.kernel.org/doc/Documentation/cpu-freq/governors.txt](https://www.kernel.org/doc/Documentation/cpu-freq/governors.txt).
[^8]: Processor affinity - [https://en.wikipedia.org/wiki/Processor_affinity](https://en.wikipedia.org/wiki/Processor_affinity).
[^9]: `taskset` manual - [https://linux.die.net/man/1/taskset](https://linux.die.net/man/1/taskset).
[^10]: `cpuset` manual - [https://github.com/lpechacek/cpuset](https://github.com/lpechacek/cpuset).
[^11]: Dynamic frequency scaling - [https://en.wikipedia.org/wiki/Dynamic_frequency_scaling](https://en.wikipedia.org/wiki/Dynamic_frequency_scaling).
