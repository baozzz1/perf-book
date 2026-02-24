## 问题与练习 {.unlisted .unnumbered}

\markright{Questions and Exercises}

1. 完成 `perf-ninja::pgo` 和 `perf-ninja::lto` 实验作业。
2. 尝试为代码段使用大页。选取一个较大的应用程序（能访问源代码是加分项，但不是必须的），其二进制文件大小超过 100MB。尝试使用 [FeTLB] 中描述的方法之一将其代码段重新映射到大页上。观察性能变化、`/proc/meminfo` 中的大页分配情况，以及测量 ITLB 加载和缺失的 CPU 性能计数器。
3. 运行你日常工作中使用的应用程序。应用 PGO、llvm-bolt 或 Propeller 并检查结果。比较"之前"和"之后"的性能分析数据，以理解性能提升的来源。
