## 核心周期与参考周期（Core vs. Reference Cycles）

大多数 CPU 使用时钟信号（clock signal）来控制其顺序操作的节奏。时钟信号由外部发生器产生，每秒提供固定数量的脉冲。时钟脉冲的频率决定了 CPU 执行指令的速率。因此，时钟越快，CPU 每秒执行的指令就越多。

$$
Frequency = \frac{Clockticks}{Time}
$$

包括 Intel 和 AMD CPU 在内的大多数现代 CPU 没有固定的运行频率。相反，它们实现了动态频率调节（dynamic frequency scaling），在 Intel CPU 中称为*睿频加速*（Turbo Boost），在 AMD 处理器中称为*Turbo Core*。它使 CPU 能够动态地提升和降低其频率。降低频率以牺牲性能为代价来减少功耗，而提高频率则以牺牲节能为代价来改善性能。

核心时钟周期（core clock cycles）计数器按 CPU 核心实际运行的频率来计数时钟周期。参考时钟（reference clock）事件则按处理器运行在基础频率（base frequency）时的假定来计数周期。让我们看一个实验，在一台运行单线程应用程序的 Skylake i7-6000 处理器上进行，该处理器的基础频率为 3.4 GHz：

```bash
$ perf stat -e cycles,ref-cycles -- ./a.exe
  43340884632  cycles		# 3.97 GHz
  37028245322  ref-cycles	# 3.39 GHz
      10,899462364 seconds time elapsed
```

`ref-cycles` 事件计数周期时假设没有频率缩放。该平台上的外部时钟频率为 100 MHz，如果我们乘以*时钟倍数*（clock multiplier），就可以得到处理器的基础频率。Skylake i7-6000 处理器的时钟倍数为 34：这意味着对于每个外部脉冲，当 CPU 以基础频率（即 3.4 GHz）运行时，会执行 34 个内部周期。

`cycles` 事件统计真实的 CPU 周期数，并考虑频率缩放。使用上述公式，我们可以确认平均运行频率为 `43340884632 cycles / 10.899 sec = 3.97 GHz`。当比较一小段代码的两个版本的性能时，以时钟周期而非纳秒来衡量时间更好，因为这样可以避免时钟频率上下波动带来的问题。
