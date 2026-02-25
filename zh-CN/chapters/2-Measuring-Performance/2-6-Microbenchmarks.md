## 微基准测试

微基准测试（microbenchmarks）是人们编写的小型独立程序，用于快速验证某个假设。通常，微基准测试用于选择某个相对小型算法或功能的最佳实现。几乎所有现代语言都有基准测试框架。在 C++ 中，你可以使用 Google [benchmark](https://github.com/google/benchmark)[^3] 库；C# 有 [BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet)[^4] 库；Julia 有 [BenchmarkTools](https://github.com/JuliaCI/BenchmarkTools.jl)[^5] 包；Java 有 [JMH](http://openjdk.java.net/projects/code-tools/jmh/etc)[^6]（Java 微基准测试框架，Java Microbenchmark Harness）；Rust 有 Criterion[^8] 包，等等。

编写微基准测试时，非常重要的一点是确保你想要测试的场景在运行时确实被你的微基准测试执行。优化编译器可能会消除重要的代码，这可能使实验毫无意义，甚至更糟糕——让你得出错误的结论。在下面的示例中，现代编译器很可能消除整个循环：

```cpp
// foo DOES NOT benchmark string creation
void foo() {
  for (int i = 0; i < 1000; i++)
    std::string s("hi");
}
```

论文"始终深入一个层次测量"（"Always Measure One Level Deeper"）[MeasureOneLevelDeeper] 很好地捕捉了此类错误，该论文倡导更科学的方法，从不同角度测量性能。遵循论文的建议，我们应该检查基准测试的性能剖析结果，并确保预期代码作为热点（hotspot）突出显示。有时，异常的计时结果会立即显现出来，因此在分析和比较基准测试运行时请运用常识判断。

防止编译器优化掉重要代码的一种流行方法是使用类似 [`DoNotOptimize`](https://github.com/google/benchmark/blob/c078337494086f9372a46b4ed31a3ae7b3f1a6a2/include/benchmark/benchmark.h#L307)[^7] 的辅助函数，这些函数在底层使用必要的内联汇编魔法：

```cpp
// foo benchmarks string creation
void foo() {
  for (int i = 0; i < 1000; i++) {
    std::string s("hi");
    DoNotOptimize(s);
  }
}
```

如果编写得当，微基准测试可以成为性能数据的良好来源。它们通常用于比较关键函数不同实现的性能。好的基准测试在实际条件下测试性能。相反，如果基准测试使用的合成输入与实践中给出的输入不同，则基准测试很可能会误导你并让你得出错误的结论。除此之外，当基准测试在没有其他高需求进程的系统上运行时，它可以使用所有可用资源，包括 DRAM 和缓存空间。这样的基准测试很可能会选出更快的函数版本，即使它比另一个版本消耗更多内存。然而，如果有邻居进程消耗了大量 DRAM，导致属于基准测试进程的内存区域被交换到磁盘，结果可能正好相反。

出于同样的原因，在从单元测试某个函数中得出结论时要谨慎。现代单元测试框架（如 GoogleTest）提供每个测试的持续时间。然而，这些信息不能替代在实际条件下使用真实输入测试函数的精心编写的基准测试（详见 [fogOptimizeCpp]）。复制实践中完全相同的输入和环境并非总是可能的，但这是开发者在编写好的基准测试时应该考虑的事情。

[^3]: Google benchmark 库 - [https://github.com/google/benchmark](https://github.com/google/benchmark)
[^4]: BenchmarkDotNet - [https://github.com/dotnet/BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet)
[^5]: Julia BenchmarkTools - [https://github.com/JuliaCI/BenchmarkTools.jl](https://github.com/JuliaCI/BenchmarkTools.jl)
[^6]: Java 微基准测试框架 - [http://openjdk.java.net/projects/code-tools/jmh/etc](http://openjdk.java.net/projects/code-tools/jmh/etc)
[^7]: 在 JMH 中，这被称为 `Blackhole.consume()`。
[^8]: Criterion.rs - [https://github.com/bheisler/criterion.rs](https://github.com/bheisler/criterion.rs)
