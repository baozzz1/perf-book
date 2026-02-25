## 编译器内建函数（Compiler Intrinsics）

有些应用程序的热点值得大力调优。然而，编译器在这些热点处生成的代码并不总是符合我们的期望。有时，人类专家能想出比编译器生成的代码性能更好的代码。这通常涉及一些巧妙或专用的算法，编译器很难甚至不可能自己找到。在这种情况下，可能无法通过使用 C 和 C++ 语言的标准构造来让编译器生成所需的汇编代码。

当绝对必要生成特定的汇编指令时，不应依赖编译器自动向量化。在这种情况下，代码可以使用*编译器内建函数*（compiler intrinsics）来编写。在大多数情况下，编译器内建函数提供与汇编指令的一对一映射。清单 Intrinsics 中的示例展示了如何使用编译器内建函数对数组元素进行水平求和（见清单 VectIllegal）的 C++ 版本。

清单：使用编译器内建函数对数组元素求和。
		
```cpp
#include <immintrin.h>

float calcSum(float* a, unsigned N) {  
  __m128 sum = _mm_setzero_ps();      // init sum with zeros
  unsigned i = 0;
  for (; i + 3 < N; i += 4) {
    __m128 vec = _mm_loadu_ps(a + i); // load 4 floats from array
    sum = _mm_add_ps(sum, vec);       // accumulate vec into sum
  }

  // Horizontal sum of the 128-bit vector
  __m128 shuf = _mm_movehdup_ps(sum); // broadcast elements 3,1 to 2,0
  sum = _mm_add_ps(sum, shuf);        // partial sums [0+1] and [2+3]
  shuf = _mm_movehl_ps(shuf, sum);    // high half -> low half
  sum = _mm_add_ss(sum, shuf);        // result in the lower element
  float result = _mm_cvtss_f32(sum);  // nop (compiler eliminates it)

  // Process any remaining elements
  for (; i < N; i++)
      result += a[i];
  return result;
}
```
当为 SSE 目标编译清单 VectIllegal 时，编译器会生成与清单 Intrinsics 几乎相同的汇编代码。我展示这个示例只是为了说明目的。显然，如果编译器能生成相同的机器代码，就没必要使用内建函数。只有当编译器无法生成所需代码时，才应使用内建函数。

当你利用编译器自动向量化时，它会插入所有必要的运行时检查。例如，它会确保有足够的元素来供给向量执行单元（见清单 Intrinsics 第 6 行）。此外，编译器还会生成循环的标量版本来处理余数（第 19 行）。当你使用内建函数时，必须自己处理这些安全方面。

内建函数比内联汇编（inline assembly）更好使用，因为编译器执行类型检查，负责寄存器分配，并做进一步优化，如窥孔变换（peephole transformations）和指令调度（instruction scheduling）。然而，它们仍然经常冗长且难以阅读。

当使用非可移植的平台特定内建函数编写代码时，还应为其他架构提供后备选项。Intel 平台所有可用内建函数的列表可以在此[参考文档](https://software.intel.com/sites/landingpage/IntrinsicsGuide/)[^11]中找到。对于 ARM，可以在 Arm 的网站上找到此类列表。[^14]

### 内建函数的包装库（Wrapper Libraries for Intrinsics）

在低工作量但不可预测的自动向量化，与冗长/难以阅读但可预测的内建函数之间，可以使用内建函数的包装库作为中间路径。这些库往往更具可读性，提供可移植性，同时仍给开发者提供对生成代码的控制。存在许多这样的库，在覆盖最新或"特殊"操作的范围，以及它们支持的平台数量上有所不同。

ISPC 的"编写一次，面向多目标"模型很有吸引力。然而，你可能希望与 C++ 程序更紧密地集成。例如，与模板的互操作性，或避免单独的构建步骤和使用相同的编译器。相反，内建函数提供更多控制，但开发成本更高。

包装库结合了两者的优点，通过使用所谓的嵌入式领域特定语言（embedded domain-specific language）来避免各自的缺点，其中向量操作被表达为普通的 C++ 函数。你可以将这些函数视为"可移植内建函数"。甚至可以使用预处理器在唯一的命名空间中"重复"你的代码，但使用不同的编译器设置，从而在普通的 C++ 库中完成多次编译代码（每个指令集一次）。此类库的一个例子是 Highway，[^12] 它只需要 C++11 标准。

清单 HWY_code 展示了对数组元素求和的 Highway 版本。`ScalableTag<float> d` 是一个类型描述符，代表"可扩展"类型，意味着它可以根据目标硬件上可用的向量宽度进行调整（例如，AVX2 或 NEON）。`Zero(d)` 将 `sum` 初始化为填充了零的向量。这个变量将在函数遍历 `array` 时存储累积的总和。for 循环每次处理 `Lanes(d)` 个元素，其中 `Lanes(d)` 表示可以加载到单个 SIMD 向量中的浮点数数量。`LoadU` 操作从 `array` 加载 `Lanes(d)` 个连续元素。`Add` 操作对加载的值和当前的 `sum` 执行元素级加法，在 `sum` 中累积结果。

清单：对数组元素求和的 Highway 版本。

```cpp
#include <hwy/highway.h>

float calcSum(const float* HWY_RESTRICT array, size_t count) {
  const ScalableTag<float> d;  // type descriptor; no actual data
  auto sum = Zero(d);
  size_t i = 0;
  for (; i + Lanes(d) <= count; i += Lanes(d)) {
    sum = Add(sum, LoadU(d, array + i));
  }
  sum = Add(sum, MaskedLoad(FirstN(d, count - i), d, array + i));
  return ReduceSum(d, sum);
}
```
注意循环处理向量大小 `Lanes(d)` 的倍数后余数的显式处理。虽然这更冗长，但它使实际发生的事情可见，并允许优化，例如使用最后一个向量的重叠而不是依赖 `MaskedLoad`，或者当 `count` 已知为向量大小的倍数时完全跳过余数处理。最后，`ReduceSum` 操作通过将所有元素相加，将向量 `sum` 中的所有元素归约为单个标量值。

与 ISPC 类似，Highway 也支持检测最佳可用指令集，在 x86 上按"集群"分组，对应 Intel Core（S-SSE3）、Nehalem（SSE4.2）、Haswell（AVX2）、Skylake（AVX-512）或 Icelake/Zen4（带扩展的 AVX-512）。然后从对应的命名空间调用你的代码。与内建函数不同，代码保持可读性（每个函数没有前缀/后缀）和可移植性。

当你使用内建函数或包装库时，仍然建议使用 C++ 编写初始实现。这允许快速原型化和通过比较原始代码与新向量化实现的结果来验证正确性。

Highway 支持超过 200 个操作，可分为以下类别：


\tightlist
- 初始化（Initialization）
- 获取/设置通道（Getting/setting lanes）
- 获取/设置块（Getting/setting blocks）
- 打印（Printing）
- 元组（Tuples）
- 算术（Arithmetic）
- 逻辑（Logical）
- 掩码（Masks）
- 比较（Comparisons）
- 内存（Memory）
- 缓存控制（Cache control）
- 类型转换（Type conversion）
- 组合（Combine）
- 混洗/排列（Swizzle/permute）
- 128 位块内混洗（Swizzling within 128-bit blocks）
- 归约（Reductions）
- 加密（Crypto）


完整的操作列表，请参见其文档。[^13] Highway 不是此类唯一的库。其他库包括 nsimd、SIMDe、VCL 和 xsimd。注意，从 Vc 库开始的 C++ 标准化工作产生了 std::experimental::simd，然而，它提供的操作集非常有限，截至本文撰写时，并非所有主要编译器都支持。

[^11]: Intel intrinsics 指南 - [https://software.intel.com/sites/landingpage/IntrinsicsGuide/](https://software.intel.com/sites/landingpage/IntrinsicsGuide/)。
[^12]: Highway 库：[https://github.com/google/highway](https://github.com/google/highway)
[^13]: Highway 快速参考 - [https://github.com/google/highway/blob/master/g3doc/quick_reference.md](https://github.com/google/highway/blob/master/g3doc/quick_reference.md)
[^14]: ARM intrinsics 指南 - [https://developer.arm.com/architectures/instruction-sets/intrinsics/](https://developer.arm.com/architectures/instruction-sets/intrinsics/)
