## 问题与练习


1. 使用本章讨论的技术解决以下实验任务：
- `perf-ninja::function_inlining_1` 
- `perf-ninja::vectorization` 1 和 2
- `perf-ninja::dep_chains` 1 和 2
- `perf-ninja::compiler_intrinsics` 1 和 2
- `perf-ninja::loop_interchange` 1 和 2
- `perf-ninja::loop_tiling_1`
2. 描述你将采取哪些步骤来找出应用程序是否利用了所有使用 SIMD 代码的机会。
3. 在真实代码上手动练习循环优化。确保所有测试仍然通过。
4. 假设你正在处理一个 IpCall（每次调用指令数）指标非常低的应用程序。你会尝试应用/强制哪些优化？
5. 运行你日常使用的应用程序。找到最热的循环。它是否被向量化？是否可以强制编译器进行自动向量化？该循环是否因依赖链或执行吞吐量而成为瓶颈？
