## 软件和硬件计时器

为了对执行时间进行基准测试，工程师通常使用两种不同的计时器，所有现代平台都提供这两种计时器：

 - **系统级高分辨率计时器（System-wide high-resolution timer）**：这是一种系统计时器，通常实现为自一个任意起始日期（称为[纪元](https://en.wikipedia.org/wiki/Epoch_(computing))[^1]）以来已流逝的时钟滴答（ticks）数的简单计数。该时钟是单调的（monotonic），即始终向前走。系统时间可以通过系统调用从操作系统获取。在 Linux 系统上，可以通过 `clock_gettime` 系统调用访问系统计时器。系统计时器具有纳秒（nanosecond）分辨率，在所有 CPU 之间保持一致，且独立于 CPU 频率。尽管系统计时器可以返回纳秒精度的时间戳，但它不适合测量短时间运行的事件，因为通过 `clock_gettime` 系统调用获取时间戳需要相对较长的时间。但它适合测量持续时间超过一微秒的事件。在 C++ 中访问系统计时器的标准方式是使用 `std::chrono`，如清单 Chrono 所示。

   清单：使用 C++ std::chrono 访问系统计时器
   
   ```cpp
   #include <cstdint>
   #include <chrono>

   // returns elapsed time in nanoseconds
   uint64_t timeWithChrono() {
     using namespace std::chrono;
     auto start = steady_clock::now();
     // run something
     auto end = steady_clock::now();
     uint64_t delta = duration_cast<nanoseconds>(end - start).count();
     return delta;
   }
   ```
   
 - **时间戳计数器（TSC, Time Stamp Counter）**：这是一种实现为硬件寄存器的硬件计时器。TSC 是单调的，且具有恒定速率，即不考虑频率变化。每个 CPU 都有自己的 TSC，它只是已流逝的参考周期数（参见 [secRefCycles]）。它适合测量持续时间从纳秒到一分钟的短时事件。在 x86 平台上，可以使用编译器的内置函数 `__rdtsc` 来获取 TSC 的值，如清单 TSC 所示，该函数在底层使用 `RDTSC` 汇编指令。有关使用 `RDTSC` 汇编指令对代码进行基准测试的更多底层细节，可以在白皮书 [IntelRDTSC] 中找到。在 ARM 平台上，可以读取 `CNTVCT_EL0`（计数器-计时器虚拟计数寄存器）。

   清单：使用 __rdtsc 编译器内置函数访问 TSC

   ```cpp
   #include <x86intrin.h>
   #include <cstdint>

   // returns the number of elapsed reference clocks
   uint64_t timeWithTSC() {
       uint64_t start = __rdtsc();
       // run something
       return __rdtsc() - start;
   }
   ```

选择使用哪种计时器非常简单，取决于你想测量的事物持续多长时间。如果测量时间非常短暂，TSC 会给你更好的精度。相反，使用 TSC 来测量运行数小时的程序是没有意义的。除非需要周期级精度，系统计时器对大多数情况来说已经足够。重要的是要记住，访问系统计时器通常比访问 TSC 的延迟更高。进行 `clock_gettime` 系统调用可能比执行 `RDTSC` 指令慢得多。后者约需 5 纳秒（20 个 CPU 周期），而前者约需 500 纳秒。这在最小化测量开销时可能变得重要，尤其是在生产环境中。CppPerformanceBenchmarks 仓库的 wiki 页面上有不同平台上各种计时器访问 API 的性能比较。[^3]

[^1]: Unix 纪元从 1970 年 1 月 1 日 00:00:00 UTC 开始：[https://en.wikipedia.org/wiki/Unix_epoch](https://en.wikipedia.org/wiki/Unix_epoch)。
[^3]: CppPerformanceBenchmarks wiki - [https://gitlab.com/chriscox/CppPerformanceBenchmarks/-/wikis/ClockTimeAnalysis](https://gitlab.com/chriscox/CppPerformanceBenchmarks/-/wikis/ClockTimeAnalysis)
