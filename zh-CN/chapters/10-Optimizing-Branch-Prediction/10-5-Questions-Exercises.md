## 问题与练习 {.unlisted .unnumbered}

\markright{Questions and Exercises}

1. 重新审视代码清单 LookupBranches 右侧所示的代码示例。假设我们开始频繁收到 `[0-50)` 范围之外的数值。这将为防止越界访问 `buckets` 数组的保护分支引入大量新的预测失误。你会如何修改代码来消除这些新引入的预测失误？
2. 使用本章讨论的技术完成以下实验任务：
- `perf-ninja::branches_to_cmov_1`
- `perf-ninja::lookup_tables_1`
- `perf-ninja::virtual_call_mispredict`
- `perf-ninja::conditional_store_1`
3. 运行你日常使用的应用程序，收集 TMA 分解数据并检查 `BadSpeculation` 指标。找到分支预测失误次数最多的代码。能否使用本章讨论的技术来避免这些分支？

**编程练习**：编写一个微基准测试（microbenchmark），使其达到 50% 的预测失误率，或尽量接近该目标。目标是编写这样一段代码：其中一半的分支指令都发生了预测失误。这并没有看起来那么简单。一些提示和思路：

* 分支预测失误率的计算公式为 `BR_MISP_RETIRED.ALL_BRANCHES / BR_INST_RETIRED.ALL_BRANCHES`。
* 如果使用 C++ 编程，可以：1）使用类似 perf-ninja 的 Google benchmark 库；2）编写普通控制台程序，使用 Linux `perf` 收集 CPU 计数器；或 3）在微基准测试中集成 libpfm 库（参见 [MarkerAPI]）。
* 不需要发明复杂的算法。一种简单的方法是在 `[0;100)` 范围内生成伪随机数，并检查其是否小于 50。随机数可以提前预生成。
* 请注意，现代 CPU 能够记住较长（但仍有限）的分支结果序列。
