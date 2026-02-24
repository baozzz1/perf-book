## CPU 利用率（CPU Utilization）

CPU 利用率是指在某段时间内核心处于忙碌状态的时间百分比。从技术角度来说，当 CPU 没有运行内核的 `idle`（空闲）线程时，就被认为是已利用状态。

$$
CPU~Utilization = \frac{CPU\_CLK\_UNHALTED.REF\_TSC}{TSC},
$$

其中 `CPU_CLK_UNHALTED.REF_TSC` 统计核心未处于停机（halt）状态时的参考周期数。`TSC` 代表时间戳计数器（timestamp counter，详见 [timers]），它始终在计数。

如果 CPU 利用率较低，通常意味着应用程序性能较差，因为 CPU 有一部分时间被浪费了。然而，高 CPU 利用率并不总是良好性能的标志。它仅仅表明系统正在做一些工作，但并不说明它在做什么：即使 CPU 因等待内存访问而停顿（stall），其利用率也可能很高。在多线程场景中，线程也可能在等待资源时进行自旋（spin）。稍后，在 [secMT_metrics] 中，我们将讨论并行效率指标，特别是"有效 CPU 利用率"（Effective CPU utilization），它会过滤掉自旋时间。

Linux `perf` 会自动计算系统中所有 CPU 的 CPU 利用率：

```bash
$ perf stat -- a.exe
  0.634874  task-clock (msec) #    0.773 CPUs utilized   
```
