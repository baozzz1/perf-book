## 问题与练习 {.unlisted .unnumbered}

\markright{Questions and Exercises}

1. 完成 `perf-ninja::data_packing` 实验作业，在该作业中您需要使数据结构更加紧凑。
2. 使用我们在 [secDTLB] 中讨论的方法完成 `perf-ninja::huge_pages_1` 实验作业。观察性能变化、`/proc/meminfo` 中的巨页分配情况，以及测量 DTLB 加载和缺失的 CPU 性能计数器。
3. 通过为将来的循环迭代实现显式内存预取，完成 `perf-ninja::swmem_prefetch_1` 实验作业。
4. 用通俗的语言描述使一段代码成为缓存友好的所需条件。
5. 运行您日常工作中使用的应用程序。使用我们在 [MemoryProfiling] 中讨论的内存分析器测量其内存利用率并分析堆分配情况。使用 Linux perf、Intel VTune 或其他分析器识别热内存访问。是否有改善这些访问的方法？
