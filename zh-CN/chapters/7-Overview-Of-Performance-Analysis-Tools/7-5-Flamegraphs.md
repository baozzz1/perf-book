## 火焰图

火焰图（Flame Graph）是一种流行的性能分析数据可视化方式，用于展示程序中最频繁的代码路径。它使我们能够看出哪些函数调用占用了最大比例的执行时间。图 FlameGraph 展示了 [x264](https://openbenchmarking.org/test/pts/x264) 视频编码基准测试的火焰图示例，该图使用 Brendan Gregg 开发的开源[脚本](https://github.com/brendangregg/FlameGraph)[^1]生成。如今，只要在分析会话期间收集了调用栈，几乎所有分析器都能自动生成火焰图。

![x264 基准测试的火焰图。](../../img/perf-tools/Flamegraph.jpg)

在火焰图中，每个矩形（水平条）代表一次函数调用，矩形的宽度表示该函数本身及其被调用函数所占用的相对执行时间。函数调用从下往上进行，因此可以看到程序中最热的路径是 `x264` &rarr; `threadpool_thread_internal` &rarr; `...` &rarr; `x264_8_macroblock_analyse`。函数 `threadpool_thread_internal` 及其被调用函数占程序总时间的 74%。但其自身时间（self-time），即函数本身花费的时间，相当小。类似地，可以对 `x264_8_macroblock_analyse` 进行相同分析，它占运行时间的 66%。这种可视化方式能让你对时间主要花费在哪里形成非常直观的认识。

火焰图是交互式的。可以点击图像上的任意条形，它会放大到该特定代码路径。可以持续放大，直到找到不符合预期的地方，或者到达叶子/尾部函数——此时你就获得了可以在分析中使用的可行信息。另一种策略是找出程序中最热的函数（从这张火焰图上并不能直接看出），然后从火焰图自底向上地追踪，尝试理解这个最热函数从何处被调用。

某些工具更倾向于使用*冰柱图*（icicle graph），即火焰图的倒置版本（参见 [ContinuousProfiling] 中的示例）。

[^1]: Flame graphs by Brendan Gregg - [https://github.com/brendangregg/FlameGraph](https://github.com/brendangregg/FlameGraph)
[^2]: x264 video encoding benchmark - [https://openbenchmarking.org/test/pts/x264](https://openbenchmarking.org/test/pts/x264)
