## 向量化（Vectorization）

在现代处理器上，使用 SIMD 指令与普通未向量化（标量，scalar）代码相比可以获得巨大的加速。在进行性能分析时，软件工程师的首要任务之一是确保代码的热点部分已被向量化。本节指导工程师发现向量化机会。关于现代 CPU 的 SIMD 能力回顾，读者可查看 [SIMD]。

向量化通常在没有任何用户干预的情况下自动发生；这称为编译器*自动向量化*（autovectorization）。在这种情况下，编译器会自动识别从源代码生成 SIMD 机器代码的机会。

自动向量化非常方便，因为现代编译器可以为各种程序自动生成快速的 SIMD 代码。然而，在某些情况下，如果没有软件工程师的干预，自动向量化不会成功。现代编译器有扩展功能，允许高级用户控制自动向量化过程，并确保代码的某些部分被高效地向量化。我们将提供几个使用编译器自动向量化提示的示例。

在本节中，我们将讨论如何利用编译器自动向量化，特别是内层循环向量化，因为它是最常见的自动向量化类型。其他两种类型（外层循环向量化和超字级并行向量化）不在本书讨论范围内。

### 编译器自动向量化（Compiler Autovectorization）

有多种障碍会阻止自动向量化，其中一些与编程语言的语义固有相关。例如，编译器必须假设循环索引可能溢出，这可能阻止某些循环变换。另一个例子是 C 编程语言的假设：程序中的指针可能指向重叠的内存区域，这会使程序分析非常困难。

另一个主要障碍是处理器本身的设计。在某些情况下，处理器对某些操作没有高效的向量指令。例如，大多数处理器上没有预测（谓词，predicated）（位掩码控制）加载和存储操作。尽管面临所有这些挑战，你可以解决其中许多问题并启用自动向量化。本节后面，我们提供如何与编译器合作并确保热代码由编译器向量化的指导。

向量化器通常分三个阶段组织：合法性检查（legality-check）、盈利性检查（profitability-check）和变换本身：

* **合法性检查**：在这个阶段，编译器检查将循环（或其他类型的代码区域）变换为使用向量是否合法。合法性阶段收集循环向量化合法所需的条件列表。循环向量化器检查循环的迭代是否连续，这意味着循环是线性推进的。向量化器还确保循环中的所有内存和算术操作都可以扩展为连续操作，所有通道（lane）的控制流是一致的，以及内存访问模式是统一的。编译器必须检查或确保生成的代码不会访问不应访问的内存，并且操作顺序将被保留。编译器需要分析指针的可能范围，如果缺少某些信息，它必须假定该变换是非法的。

* **盈利性检查**：接下来，向量化器检查变换是否有利可图。它比较不同的向量化宽度，找出哪个执行最快。向量化器使用代价模型来预测不同操作的成本，例如标量加法或向量加载。它需要考虑将数据混洗到寄存器中的额外指令，预测寄存器压力，并估计确保向量化前提条件的循环保护（loop guards）的开销。检查盈利性的算法很简单：1) 累加代码中所有操作的成本，2) 比较每个版本代码的成本，3) 用成本除以预期执行次数。例如，如果标量代码需要 8 个周期，而向量化代码需要 12 个周期但一次处理 4 次循环迭代，则向量化版本的循环可能更快。

* **变换**：最后，在向量化器确定变换合法且有利可图后，它变换代码。这个过程还包括插入启用向量化的保护。例如，大多数循环使用未知的迭代次数，因此编译器必须生成循环的标量版本（余数处理），以及向量化版本，来处理最后几次迭代。编译器还必须检查指针是否不重叠等。所有这些变换都使用在合法性检查阶段收集的信息完成。

### 发现向量化机会（Discovering Vectorization Opportunities）

发现改善向量化的机会应从分析程序中的热循环开始，并检查编译器执行了哪些优化。检查编译器向量化报告（见 [compilerOptReports]）是最简单的方法。现代编译器可以报告某个循环是否被向量化，并提供其他细节，例如向量化因子（vectorization factor，VF）。如果编译器无法向量化循环，它也能说明失败原因。

使用编译器优化报告的另一种方式是检查汇编输出。最好分析来自分析工具的输出，该输出显示了给定循环的源代码与生成的汇编指令之间的对应关系。这样你只需关注重要的代码，即热代码。然而，理解汇编语言比理解 C++ 等高级语言要困难得多。弄清楚编译器生成的指令的语义可能需要一些时间。但这种技能非常有价值，往往能提供宝贵的见解。

有经验的开发者只需查看指令助记符和这些指令使用的寄存器名称，就能迅速判断代码是否被向量化。例如，在 x86 ISA 中，向量指令操作打包数据（因此名称中有 `P`），并使用 `XMM`、`YMM` 或 `ZMM` 寄存器，例如 `VMULPS XMM1, XMM2, XMM3` 将 `XMM2` 和 `XMM3` 中的四个单精度浮点数相乘并将结果保存在 `XMM1` 中。但要注意，人们经常因为看到使用了 `XMM` 寄存器就得出结论认为是向量代码——这并不一定正确。例如，`VMULSS XMM1, XMM2, XMM3` 指令只会乘以一个单精度浮点值，而不是四个。

潜在向量化机会的另一个指标是较高的 `Retiring` 指标（超过 80%）。在 [TMA] 中，我们说过 `Retiring` 指标是代码性能良好的良好指标。其背后的道理是执行没有停顿，CPU 以较高的速率退休指令。然而，有时它可能隐藏真正的性能问题，即低效的计算。也许工作负载执行了大量可以被向量指令替换的简单指令。在这种情况下，高 `Retiring` 指标并不转化为高性能。

开发者在尝试加速可向量化代码时经常遇到几种常见情况。下面我们介绍四种典型场景，并对每种情况给出一般性指导。

#### 向量化不合法（Vectorization Is Illegal）

在某些情况下，遍历数组元素的代码根本无法向量化。优化报告在解释出了什么问题以及编译器为何无法向量化代码方面非常有效。清单 VectDep 展示了一个循环内阻止向量化的依赖示例。[^31]

清单：向量化：读后写依赖。

```cpp
void vectorDependence(int *A, int n) {
  for (int i = 1; i < n; i++)
    A[i] = A[i-1] * 2;
}
```
虽然某些循环由于硬性限制（如读后写依赖）而无法向量化，但其他循环在放宽某些约束后可以向量化。例如，清单 VectIllegal 中的代码无法被编译器自动向量化，因为这会改变浮点操作的顺序，可能导致不同的舍入结果和略有不同的结果。浮点加法满足交换律，这意味着可以交换左右两侧而不改变结果：`(a + b == b + a)`。然而，它不满足结合律，因为舍入发生在不同时刻：`((a + b) + c) != (a + (b + c))`。

清单：向量化：浮点算术。

```cpp
// a.cpp
float calcSum(float* a, unsigned N) {
  float sum = 0.0f;
  for (unsigned i = 0; i < N; i++) {
    sum += a[i];
  }
  return sum;
}
```
如果你告诉编译器可以容忍最终结果的一些变化，它将自动向量化代码。Clang 和 GCC 编译器有一个标志 `-ffast-math`，[^29] 允许这种变换，即使结果程序可能给出略有不同的结果：

```bash
$ clang++ -c a.cpp -O3 -march=core-avx2 -Rpass-analysis=.*
...
a.cpp:5:9: remark: loop not vectorized: cannot prove it is safe to reorder floating-point operations; allow reordering by specifying '#pragma clang loop vectorize(enable)' before the loop or by providing the compiler option '-ffast-math'. [-Rpass-analysis=loop-vectorize]
...
$ clang++ -c a.cpp -O3 -march=core-avx2 -ffast-math -Rpass=.*
...
a.cpp:4:3: remark: vectorized loop (vectorization width: 4, interleaved count: 2) [-Rpass=loop-vectorize]
...
```

不幸的是，这个标志涉及微妙且潜在危险的行为变化，包括对非数字（Not-a-Number）、有符号零、无穷大和次正规数（subnormals）的处理。由于第三方代码可能没有准备好应对这些影响，在没有仔细验证结果（包括边缘情况）的情况下，不应在大量代码中启用此标志。自 Clang 18 起，你可以使用专用 pragma 来限制变换的范围，例如 `#pragma clang fp reassociate(on)`。[^4]

让我们看另一种典型情况，编译器可能需要开发者的支持来执行向量化。当编译器无法证明循环操作的数组具有非重叠内存区域时，它们通常会选择保守方式。给定清单 OverlappingMemRefions 中的代码，编译器应该考虑数组 `a`、`b` 和 `c` 的内存区域重叠的情况。

清单：a.c

```cpp
void foo(float* a, float* b, float* c, unsigned N) {
  for (unsigned i = 1; i < N; i++) {
    c[i] = b[i];
    a[i] = c[i-1];
  }
}
```
以下是 GCC 10.2 提供的优化报告（使用 `-fopt-info` 启用）：

```bash
$ gcc -O3 -march=core-avx2 -fopt-info
a.cpp:2:26: optimized: loop vectorized using 32-byte vectors
a.cpp:2:26: optimized:  loop versioned for vectorization because of possible aliasing
```

GCC 识别到内存区域之间的潜在重叠，并创建了循环的多个版本。编译器插入了运行时检查（runtime checks）[^36] 来检测内存区域是否重叠。根据这些检查，它在向量化版本和标量版本之间进行调度。在这种情况下，向量化的代价是插入可能开销较大的运行时检查。如果开发者知道数组 `a`、`b` 和 `c` 的内存区域不重叠，可以在循环前插入 `#pragma GCC ivdep`[^37]，或使用 `__restrict__` 关键字，如 [compilerOptReports] 所示。这类编译器提示将消除 GCC 编译器插入上述运行时检查的需要。

一些动态工具，如 Intel Advisor，可以检测循环中是否存在跨迭代依赖或访问具有重叠内存区域的数组等问题。但要注意，这类工具只提供建议。不加谨慎地插入编译器提示可能会导致真正的问题。

#### 向量化无利可图（Vectorization Is Not Beneficial）

在某些情况下，编译器可以向量化循环，但决定这样做不划算。在清单 VectNotProfit 中展示的代码中，编译器可以向量化对数组 `A` 的内存访问，但需要将对数组 `B` 的访问分割成多个标量加载。散集（scatter/gather）模式相对昂贵，能够模拟操作成本的编译器通常决定避免向量化具有此类模式的代码。

清单：向量化：无利可图。

```cpp
// a.cpp
void stridedLoads(int *A, int *B, int n) {
  for (int i = 0; i < n; i++)
    A[i] += B[i * 3];
}
```
以下是清单 VectNotProfit 中代码的编译器优化报告：

```bash
$ clang -c -O3 -march=core-avx2 a.cpp -Rpass-missed=loop-vectorize
a.cpp:3:3: remark: the cost-model indicates that vectorization is not beneficial [-Rpass-missed=loop-vectorize]
  for (int i = 0; i < n; i++)
  ^
```

用户可以使用 `#pragma` 提示强制 Clang 编译器向量化循环，如清单 VectNotProfitOverriden 所示。但请记住，向量化是否有利可图在很大程度上取决于运行时数据，例如循环的迭代次数。编译器没有这些信息，[^1] 因此往往倾向于保守。不过你可以将这类提示用于性能实验。

清单：向量化：无利可图。

```cpp
// a.cpp
void stridedLoads(int *A, int *B, int n) {
#pragma clang loop vectorize(enable)
  for (int i = 0; i < n; i++)
    A[i] += B[i * 3];
}
```
开发者应该意识到使用向量化代码的隐性成本。使用 AVX 尤其是 AVX-512 向量指令可能导致频率降频（frequency downclocking）或启动开销，在某些 CPU 上这也会影响后续代码长达几微秒。向量化部分的代码应该足够热，才能证明使用 AVX-512 是合理的。[^38] 例如，研究发现排序 80 KiB 足以摊销这种开销并使向量化物有所值。[^39]

#### 循环已向量化但使用了标量版本（Loop Vectorized but Scalar Version Used）

在某些情况下，编译器成功向量化了代码，但它在分析器中没有显示为正在执行。检查循环的对应汇编时，通常很容易找到循环体的向量化版本，因为它使用了向量寄存器，而向量寄存器在程序的其他部分并不常用。

如果向量代码没有被执行，一个可能的原因是生成的代码假设的循环行程计数高于程序实际使用的值。例如，编译器可能决定以每次迭代处理 64 个元素的方式对循环进行向量化和展开。输入数组可能没有足够的元素，即使是循环的单次迭代也不够。在这种情况下，将使用循环的标量版本（余数）。检测这些情况很容易，因为标量循环会在分析器中高亮显示，而向量化代码保持冷状态。

解决这个问题的方法是强制向量化器使用较低的向量化因子或展开次数，以减少循环每次处理的元素数量。可以通过 `#pragma` 提示来实现。对于 Clang 编译器，可以使用 `#pragma clang loop vectorize_width(N)`，如 Easyperf 博客的文章所述。[^30]

#### 循环以次优方式向量化（Loop Vectorized in a Suboptimal Way）

当你看到循环被自动向量化并在运行时执行时，程序的这一部分很可能已经表现良好。然而也有例外。有时循环的未向量化标量版本的性能比向量化版本更好。这可能是由于昂贵的向量操作，如 `gather/scatter` 加载、掩码（masking）、`inserting/extracting` 元素、数据混洗（data shuffling）等，当编译器需要使用这些操作来实现向量化时。性能工程师也可以尝试以不同方式禁用向量化。对于 Clang 编译器，可以通过编译器选项 `-fno-vectorize` 和 `-fno-slp-vectorize` 来实现，或者使用特定于特定循环的提示，例如 `#pragma clang loop vectorize(disable)`。

重要的是要注意，有一系列问题中 SIMD 非常重要，而自动向量化根本不起作用，短期内也不太可能起作用。一个例子可以在 [Mula_Lemire_2019] 中找到。另一个例子是外层循环自动向量化，编译器目前没有尝试这一点。向量化浮点代码是有问题的，因为重新排序浮点算术操作会导致不同的舍入和略有不同的值。

自动向量化还有一个更微妙的问题。随着编译器的发展，它们进行的优化也在变化。在上一个编译器版本中成功进行的代码自动向量化可能在下一个版本中停止工作，反之亦然。此外，在代码维护或重构过程中，代码结构可能会发生变化，导致自动向量化突然失败。这可能在原始软件编写很久之后才发生，届时修复或重新实现的代价会更高。

#### 具有显式向量化的语言（Languages with Explicit Vectorization）

向量化也可以通过将程序的部分内容用专用于并行计算的编程语言重写来实现。这些语言使用特殊的构造和程序数据知识，将代码高效地编译成并行程序。最初，这类语言主要用于将工作卸载到特定处理单元，如图形处理单元（GPU）、数字信号处理器（DSP）或现场可编程门阵列（FPGA）。然而，其中一些编程模型也可以针对你的 CPU（如 OpenCL 和 OpenMP）。

其中一种并行语言是 Intel® 隐式 SPMD 程序编译器 [(ISPC)](https://ispc.github.io/)，[^33] 我将在本节简要介绍。ISPC 语言基于 C 编程语言，使用 LLVM 编译器基础设施为许多不同架构生成优化代码。ISPC 的关键特性是"接近金属"（close to the metal）的编程模型和跨 SIMD 架构的性能可移植性。它需要从传统的编写程序思维方式转变，但给了程序员更多对 CPU 资源利用的控制权。

ISPC 的另一个优势是其互操作性和易用性。ISPC 编译器生成标准目标文件，可以与传统 C/C++ 编译器生成的代码链接。由于用 ISPC 编写的函数可以像 C 代码一样被调用，ISPC 代码可以轻松地插入到任何原生项目中。

清单 ISPC_code 展示了我之前在清单 VectIllegal 中展示的函数的 ISPC 版本。ISPC 考虑程序将在并行实例中运行，基于目标指令集。例如，当使用带有 `float` 的 SSE 时，它可以并行计算 4 个操作。每个程序实例将操作 `i` 的向量值为 `(0,1,2,3)`，然后是 `(4,5,6,7)`，依此类推，有效地一次计算 4 个总和。如你所见，使用了一些在 C 和 C++ 中不典型的关键字：

* `export` 关键字表示该函数可以从 C 兼容语言调用。

* `uniform` 关键字表示变量在程序实例之间共享。

* `varying` 关键字表示每个程序实例都有自己的变量本地副本。

* `foreach` 与经典的 `for` 循环相同，只是它将工作分配到不同的程序实例。

清单：对数组元素求和的 ISPC 版本。

```cpp
export uniform float calcSum(const uniform float array[], 
                             uniform ptrdiff_t count)
{
    varying float sum = 0;
    foreach (i = 0 ... count)
        sum += array[i];
    return reduce_add(sum);
}
```
由于函数 `calcSum` 必须返回单个值（一个 `uniform` 变量），而我们的 `sum` 变量是 `varying`，我们需要使用 `reduce_add` 函数*聚合*（gather）每个程序实例的值。ISPC 还负责根据需要生成预处理和余数循环，以考虑未正确对齐或不是向量宽度倍数的数据。

**"接近金属"的编程模型**：传统 C 和 C++ 语言的一个问题是编译器并不总是向量化代码的关键部分。ISPC 通过默认将每个操作视为 SIMD 来帮助解决这个问题。例如，ISPC 语句 `sum += array[i]` 被隐式视为并行进行多个加法的 SIMD 操作。ISPC 不是自动向量化编译器，它不会自动发现向量化机会。由于 ISPC 语言与 C 和 C++ 非常相似，它比 intrinsics（见 [secIntrinsics]）更具可读性，因为它允许你专注于算法而非低级指令。据报道，它在性能方面与手写 intrinsics 代码相匹配 [ISPC_Paper] 甚至超越。[^34]

**性能可移植性**：ISPC 可以自动检测 CPU 的特性，充分利用所有可用资源。程序员可以编写一次 ISPC 代码并编译到多种向量指令集，如 SSE4、AVX2 和 ARM NEON。

[^1]: 除了配置文件引导优化（Profile Guided Optimizations，见 [secPGO]）。
[^2]: 例如，编译器优化报告，见 [compilerOptReports]。
[^29]: 编译器标志 `-Ofast` 同时启用 `-ffast-math` 和 `-O3` 编译模式。
[^30]: 使用 Clang 的优化 pragma - [https://easyperf.net/blog/2017/11/09/Multiversioning_by_trip_counts](https://easyperf.net/blog/2017/11/09/Multiversioning_by_trip_counts)
[^31]: 一旦你展开循环的几次迭代，就很容易发现读后写依赖。参见 [compilerOptReports] 中的示例。
[^33]: ISPC 编译器：[https://ispc.github.io/](https://ispc.github.io/)。
[^34]: 使用 SIMD intrinsics 的 Unreal Engine 部分被使用 ISPC 重写，获得了加速：[https://software.intel.com/content/www/us/en/develop/articles/unreal-engines-new-chaos-physics-system-screams-with-in-depth-intel-cpu-optimizations.html](https://software.intel.com/content/www/us/en/develop/articles/unreal-engines-new-chaos-physics-system-screams-with-in-depth-intel-cpu-optimizations.html)。
[^36]: 参见 Easyperf 博客上的示例：[https://easyperf.net/blog/2017/11/03/Multiversioning_by_DD](https://easyperf.net/blog/2017/11/03/Multiversioning_by_DD)。
[^37]: 这是 GCC 特定的 pragma。对于其他编译器，请查看相应的手册。
[^38]: 更多详情请阅读此博文：[https://travisdowns.github.io/blog/2020/01/17/avxfreq1.html](https://travisdowns.github.io/blog/2020/01/17/avxfreq1.html)。
[^39]: AVX-512 降频研究：在 [VQSort readme](https://github.com/google/highway/blob/master/hwy/contrib/sort/README.md#study-of-avx-512-downclocking) 中
[^4]: LLVM 指定浮点标志的扩展 - [https://clang.llvm.org/docs/LanguageExtensions.html#extensions-to-specify-floating-point-flags](https://clang.llvm.org/docs/LanguageExtensions.html#extensions-to-specify-floating-point-flags)
