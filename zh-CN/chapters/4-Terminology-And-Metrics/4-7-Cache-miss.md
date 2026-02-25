## 缓存未命中（Cache Miss）

如 [MemHierar] 中所述，在某个特定缓存层级未命中的任何内存请求都必须由更高层级的缓存或 DRAM 来服务。这意味着此类内存访问的延迟会显著增加。内存子系统各组件的典型延迟如表 mem_latency 所示。当内存请求在最后一级缓存（LLC，Last Level Cache）中未命中并一直访问到主内存时，性能会大幅下降。[^3]

| 内存层次结构组件 | 延迟（周期/时间） |
|------------------|-------------------|
| L1 缓存 | 4 个周期（~1 ns） |
| L2 缓存 | 10-25 个周期（5-10 ns） |
| L3 缓存 | ~40 个周期（20 ns） |
| 主内存 | 200+ 个周期（100 ns） |

表：x86 平台内存子系统的典型延迟。

指令和数据的获取都可能在缓存中未命中。根据自顶向下微架构分析（见 [TMA]），指令缓存（I-cache）未命中被归类为前端停顿（Frontend stall），而数据缓存（D-cache）未命中被归类为后端停顿（Backend stall）。指令缓存未命中发生在 CPU 流水线的早期阶段，即指令获取阶段。数据缓存未命中则发生在指令执行阶段，时间要晚得多。

Linux `perf` 用户可以通过以下命令收集 L1 缓存未命中的数量：

```bash
$ perf stat -e mem_load_retired.fb_hit,mem_load_retired.l1_miss,
  mem_load_retired.l1_hit,mem_inst_retired.all_loads -- a.exe
   29580  mem_load_retired.fb_hit
   19036  mem_load_retired.l1_miss
  497204  mem_load_retired.l1_hit
  546230  mem_inst_retired.all_loads
```

以上是 L1 数据缓存和填充缓冲区（fill buffer）的所有加载分类。一次加载可能命中已分配的填充缓冲区（`fb_hit`）、命中 L1 缓存（`l1_hit`），或两者都未命中（`l1_miss`），因此 `all_loads = fb_hit + l1_hit + l1_miss`。[^2] 可以看出，只有 3.5% 的加载在 L1 缓存中未命中，因此 *L1 命中率*为 96.5%。

我们可以进一步分解 L1 数据缓存未命中，并通过运行以下命令分析 L2 缓存的行为：

```bash
$ perf stat -e mem_load_retired.l1_miss,
  mem_load_retired.l2_hit,mem_load_retired.l2_miss -- a.exe
  19521  mem_load_retired.l1_miss
  12360  mem_load_retired.l2_hit
   7188  mem_load_retired.l2_miss
```

从这个示例可以看出，在 L1 D-cache 中未命中的加载中有 37% 也在 L2 缓存中未命中，因此 *L2 命中率*为 63%。L3 缓存的分类可以以类似方式进行。

[^2]: 细心的读者可能会注意到数字存在差异：`fb_hit + l1_hit + l1_miss = 545,820`，与 `all_loads` 不完全匹配。这很可能是由于硬件事件收集的轻微误差，但由于数字非常接近，我们没有对此进行深入调查。
[^3]: 此外，还有一个交互式视图可以将现代系统中不同操作的延迟可视化 - [https://colin-scott.github.io/personal_website/research/interactive_latency.html](https://colin-scott.github.io/personal_website/research/interactive_latency.html)
