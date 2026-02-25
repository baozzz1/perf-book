## 微架构特定的性能问题

在本节中，我们将讨论一些影响大多数现代处理器的常见微架构特定问题（microarchitecture-specific issues）。之所以称其为微架构特定问题，是因为它们由特定微架构功能的实现方式所引起。这些问题非常具体，通常不会作为主要性能瓶颈频繁出现。它们往往被其他更显著的性能问题所掩盖。因此，这些微架构特定的性能问题被视为边角情况（corner cases），不如本书中已讨论的其他问题广为人知。尽管如此，它们可能导致非常令人不快的性能损失。需要注意的是，特定问题的影响在不同平台上可能更为显著或不那么明显。此外，请记住，以下涵盖的微架构特定问题列表并不详尽。

### 内存顺序违规

我在 [uarchLSU] 中介绍了内存排序（memory ordering）的概念。内存重排序（memory reordering）是现代 CPU 的关键方面，它使 CPU 能够并行且乱序地执行内存请求。加载/存储重排序的关键要素是内存消歧义（memory disambiguation），它预测是否可以安全地让加载操作先于早期的存储操作执行。由于内存消歧义是推测性的（speculative），若处理不当可能导致性能问题。

考虑代码清单 MemOrderViolation 左侧的示例。此代码片段计算 8 位灰度图像的直方图，即某种颜色在图像中出现的次数。这段代码在许多地方均有应用，包括大津二值化算法（Otsu's thresholding algorithm）[^1]，该算法用于将灰度图像转换为二值图像。由于输入图像是 8 位灰度图像，因此只有 256 种不同的颜色。

对于图像中的每个像素，您需要读取该像素颜色的当前直方图计数，将其加 1，然后存储回去。这是一个经典的通过内存进行的读-改-写（read-modify-write）依赖。假设图像中有以下连续像素：`pixels = [0xFF,0xFF,0x00,0xFF,...]` 等等。`pixels[1]` 的直方图计数加载值来自前一次迭代（`pixels[0]`）的结果。`pixels[2]` 的直方图计数来自内存，它是独立的，可以被重排序。但 `pixels[3]` 的直方图计数又依赖于处理 `pixels[1]` 的结果，以此类推。迭代 0、1 和 3 是相互依赖的，不能被重排序。

代码清单：内存顺序违规示例。

```cpp
std::array<uint32_t, 256> hist;           std::array<uint32_t, 256> hist1;
hist.fill(0);                             std::array<uint32_t, 256> hist2;
int N = width * height;                   hist1.fill(0);         
for (int i = 0; i < N; ++i)       =>      hist2.fill(0);
  hist[image[i]]++;                       int N = width * height;         
                                          int i = 0;
                                          for (; i + 1 < N; i += 2) {
                                            hist1[image[i+0]]++;
                                            hist2[image[i+1]]++;
                                          }
                                          // remainder
                                          for (; i < N; ++i)
                                            hist1[image[i]]++;
                                          // combine partial histograms
                                          for (int i = 0; i < hist1.size(); ++i)
                                            hist1[i] += hist2[i];
```
回顾 [uarchLSU]，处理器未必能预知两次存储到加载转发（store-to-load forwarding）之间的潜在关联，因此它必须做出预测。如果它正确预测了两次 `0xFF` 颜色更新之间存在内存顺序违规，这些访问将被串行化。性能不会很好，但对于初始代码而言，这已是最优情况。相反，如果处理器预测不存在内存顺序违规，它将推测性地让两次更新并行执行。随后，当它发现错误时，将刷新流水线（flush the pipeline）并重新执行两次更新中较晚的那次。这对性能损害极大。

性能将在很大程度上取决于输入图像的颜色模式。具有相同颜色像素长序列的图像，其性能将比颜色不经常重复的图像差。只要相同颜色两个像素之间的距离足够长，初始版本的性能就会很好。在这里，"足够长"由乱序指令窗口（out-of-order instruction window）的大小决定。相同颜色的读-改-写如果在彼此相隔几次循环迭代内发生，可能触发顺序违规，但如果相隔超过一百次循环迭代则不会。

代码清单 MemOrderViolation 右侧展示了内存顺序违规问题的解决方案。可以看到，我复制了直方图，现在像素的处理在两个部分直方图之间交替进行。最终，我们合并两个部分直方图以获得最终结果。这个带有两个部分直方图的新版本仍然容易受到某些潜在问题模式的影响，例如 `0xFF 0x00 0xFF 0x00 0xFF ...`。然而，通过此更改，原来的最坏情况（例如 `0xFF 0xFF 0xFF ...`）将比以前快两倍。根据输入图像的颜色模式，创建四个或八个部分直方图可能更有益。该代码已收录于 Performance Ninja 课程的 [mem_order_violation_1](https://github.com/dendibakh/perf-ninja/tree/main/labs/memory_bound/mem_order_violation_1)[^2] 实验任务中，欢迎自行实验。

在一小组输入图像上，我在各种平台上观察到 10% 到 50% 的性能提升。值得一提的是，右侧版本额外消耗了 1 KB 内存，在本例中可能不算太大，但仍需关注。

### 内存未对齐访问

如果变量存储在可被其大小整除的内存地址处，则访问效率最高。例如，`int` 需要 4 字节对齐，即其地址应为 4 的倍数。在 C++ 中，这称为*自然对齐（natural alignment）*，它默认适用于基本数据类型，如整数、浮点数或双精度浮点数。当您声明这些类型的变量时，编译器确保它们存储在地址为其大小倍数的内存中。相比之下，数组、结构体和类可能需要特殊对齐，您将在本节中了解到这一点。

数据对齐（data alignment）重要性的一个典型场景是 SIMD 代码，其中加载和存储用单次操作访问大块数据。在大多数处理器中，L1 缓存被设计为能够以任意对齐方式读写数据。通常，即使加载/存储未对齐，但不跨越缓存行（cache line）边界，也不会有任何性能损失。

然而，当加载或存储跨越缓存行边界时，此类访问需要读取两个缓存行（*拆分加载/存储，split load/store*）。这需要使用*拆分寄存器（split register）*来保存两个部分，一旦两个部分都取回，它们将被合并到单个寄存器中。拆分寄存器的数量是有限的。当偶尔执行时，拆分访问（split accesses）不会对整体执行产生可观察的性能影响。然而，如果这种情况频繁发生，未对齐的内存访问将遭受延迟。

如果一个内存地址是特定大小的倍数，则称该地址是*对齐的（aligned）*。例如，当一个 16 字节对象在 64 字节边界上对齐时，其地址的低 6 位为零。否则，当一个 16 字节对象跨越 64 字节边界时，称其为*未对齐的（misaligned）*。在文献中，您还可能遇到*拆分加载/存储（split load/store）*一词来描述这种情况。如果连续许多拆分加载/存储耗尽了所有可用的拆分寄存器，则可能产生性能损失。Intel 的 TMA 方法论通过 `Memory_Bound` &rarr; `L1_Bound` &rarr; `Split Loads` 指标来跟踪这一问题。

例如，AVX2 内存操作可以访问最多 32 个字节。如果一个数组从偏移量 `0x30`（48 字节）开始，第一次 AVX2 加载将从 `0x30` 获取数据到 `0x4F`，第二次加载将从 0x50 获取数据到 0x6F，依此类推。第一次加载跨越了缓存行边界（`0x40`）。事实上，每隔一次加载都会跨越缓存行边界，这可能会减慢执行速度。图 SplitLoads 对此进行了说明。将数据向前推移 16 字节可以使数组与缓存行边界对齐，从而消除拆分加载。代码清单 AligningData 展示了如何使用 C++11 的 `alignas` 关键字修复此示例。

![AVX2 在未对齐数组中的加载。每隔一次加载就会跨越缓存行边界。](../../../img/memory-access-opts/SplitLoads.png)

<p align="center"><em>AVX2 在未对齐数组中的加载。每隔一次加载就会跨越缓存行边界。</em></p>


代码清单：使用 "alignas" 关键字对齐数据。

```cpp
// Array of 16-bit integers aligned at a 64-byte boundary
#define CACHELINE_ALIGN alignas(64) 
CACHELINE_ALIGN int16_t a[N];
```
对于动态分配，C++17 使其变得更加简单。运算符 `new` 现在接受一个额外的参数，您可以用它来控制动态分配内存的对齐方式。使用标准容器（如 `std::vector`）时，您可以定义自定义分配器（custom allocator）。代码清单 AlignedStdVector 展示了一个自定义分配器的最简示例，该分配器在缓存行边界处对齐内存缓冲区。

代码清单：定义在缓存行边界对齐的 std::vector。

```cpp
// Returns aligned pointers when allocations are requested. 
template <typename T>
class CacheLineAlignedAllocator {
public:
  using value_type = T;
  static std::align_val_t constexpr ALIGNMENT{64};
  [[nodiscard]] T* allocate(std::size_t N) {
    return reinterpret_cast<T*>(::operator new[](N * sizeof(T), ALIGNMENT));
  }
  void deallocate(T* allocPtr, [[maybe_unused]] std::size_t N) {
    ::operator delete[](allocPtr, ALIGNMENT);
  }
};
template<typename T> 
using AlignedVector = std::vector<T, CacheLineAlignedAllocator<T> >;
```
为了演示未对齐内存访问的影响，我在 Performance Ninja 在线课程中创建了 [mem_alignment_1](https://github.com/dendibakh/perf-ninja/tree/main/labs/memory_bound/mem_alignment_1)[^5] 实验任务。它展示了一个非常简单的矩阵乘法示例，初始版本完全不考虑矩阵的对齐。该任务要求将矩阵对齐到缓存行边界并测量性能差异。欢迎自行实验并在您的平台上测量效果。

解决该任务中拆分加载/存储的第一步是对齐矩阵的起始偏移量。操作系统可能以已对齐到缓存行边界的方式为矩阵分配内存。然而，您不应依赖此行为，因为这并不是有保证的。简单的修复方法是使用代码清单 AlignedStdVector 中的 `AlignedVector` 来为矩阵分配内存。

然而，仅对齐矩阵的起始偏移量是不够的。图 MemAlignment 展示了一个 `9x9` 的 `float` 值矩阵示例。如果缓存行为 64 字节，它可以存储 16 个 `float` 值。使用 AVX2 指令时，程序每次加载/存储 8 个元素（256 位）。在每一行中，前 8 个元素将以 SIMD 方式处理，而最后一个元素将由循环尾部（loop remainder）以标量方式处理。第二次向量加载/存储（元素 10-17）跨越了缓存行边界，许多后续的向量加载/存储也是如此。图 MemAlignment 中突出显示的问题影响任何列数不是 8 的倍数的矩阵（针对 AVX2 向量化）。SSE 和 ARM Neon 向量化需要 16 字节对齐；AVX-512 需要 64 字节对齐。

![使用 AVX2 向量化时 9x9 矩阵中的拆分加载/存储。拆分内存访问用黄色高亮显示。](../../../img/memory-access-opts/MemAlignment.png)

<p align="center"><em>使用 AVX2 向量化时 9x9 矩阵中的拆分加载/存储。拆分内存访问用黄色高亮显示。</em></p>


因此，除了对齐起始偏移量之外，矩阵的每一行也应该对齐。例如在图 MemAlignment 中，可以通过在矩阵中插入七列哑元（dummy columns）来实现，从而有效地将其变成一个 `9x16` 的矩阵。这将使第二行（元素 10-18）在偏移量 `0x40` 处对齐，所有其他行同样会对齐。哑元列不会被算法处理，但它们将确保实际数据在缓存行边界对齐。在我的测试中，此更改的性能影响高达 30%，具体取决于矩阵大小和平台配置。

对齐和填充（padding）会产生含有未使用字节的空洞，这可能降低内存带宽利用率。对于小矩阵（如我们的 9x9 矩阵），填充将导致每行近一半的空间未被使用。然而，对于大矩阵（如 1025x1025），填充的影响并不那么大。尽管如此，对于某些算法（例如 AI 领域），内存带宽可能是更大的关注点。请谨慎使用这些技术，并始终进行测量，以确认对齐带来的性能收益是否值得为未使用字节付出代价。

跨越 4 KB 边界的访问会引入更多复杂性，因为虚拟到物理地址转换（virtual to physical address translations）通常以 4 KB 页面为单位处理。处理此类访问需要访问两个 TLB 条目。除非 TLB 支持每个周期多次查找，否则此类加载可能导致显著的性能下降。

### 缓存别名

某些特定的数据访问模式可能导致令人不快的性能问题。这些边角情况与缓存组织方式（cache organization）紧密相关，例如缓存中的组数（sets）和路数（ways）。我们在 [CacheHierarchy] 中讨论了缓存组织，如需复习可参阅该章节。内存位置在缓存中的存放位置由其地址决定。缓存控制器根据地址位进行组选择（set selection），即确定包含所取内存位置的缓存行将放入哪个组。

如果两个内存位置映射到同一个组，它们将竞争该组中有限数量的可用槽位（路，ways）。当程序反复访问映射到同一组的内存位置时，它们将不断相互驱逐（evicting）。这可能导致缓存中某个组饱和而其他组未被充分利用。这种现象称为*缓存别名（cache aliasing）*，您也可能会遇到*缓存竞争（cache contention）*、*缓存冲突（cache conflicts）*或*缓存抖动（cache trashing）*等术语来描述这种效果。

缓存别名的一个简单例子可以在矩阵转置（matrix transposition）中观察到，在 [fogOptimizeCpp] 中有详细解释。我鼓励读者阅读该手册以进一步了解其原因。我在几款现代处理器上重复了这个实验，并确认这仍然是一个相关问题。图 CacheAliasing 展示了在 Intel 第 12 代 core i7-1260P 处理器上转置 32 位浮点值矩阵的性能。

![在 Intel 第 12 代处理器上运行的矩阵转置中观察到的缓存别名效果。大小为 2 的幂次方或 128 的倍数的矩阵导致超过 10 倍的性能下降。](../../../img/memory-access-opts/CacheAliasing.png)

<p align="center"><em>在 Intel 第 12 代处理器上运行的矩阵转置中观察到的缓存别名效果。大小为 2 的幂次方或 128 的倍数的矩阵导致超过 10 倍的性能下降。</em></p>


图表中有几个峰值，对应于导致缓存别名的矩阵大小。当矩阵大小为 2 的幂次方（如 256、512）或 128 的倍数（如 384、640、768、896）时，性能显著下降。[^6] 这是因为属于同一列的内存位置映射到 L1D 和 L2 缓存的同一组。这些内存位置竞争该组中有限数量的路，导致同一缓存行在处理该行上的每个元素之前被多次重新加载。

在 Intel 处理器上，可以借助 `L1D.REPLACEMENT` 性能事件来诊断此问题，该事件计算 L1 缓存行替换次数。例如，矩阵大小为 `256x256` 时的缓存行替换次数是 `255x255` 时的 17 倍。我测试了从 `64x64` 到 `10,000x10,000` 的所有大小，发现该模式非常一致地重复出现。我还在基于 Intel Skylake 的处理器和 Apple M1 芯片上运行了相同的实验，确认这些芯片都容易受到缓存别名效应的影响。

为了缓解缓存别名，您可以使用我们在 [LoopOptsHighLevel] 中讨论的缓存分块（cache blocking）技术。其思路是以能够放入缓存的较小块来处理矩阵。这样可以避免缓存行驱逐，因为缓存中有足够的空间。另一种解决方法是用额外的列填充（pad）矩阵，例如，不使用 `256x256` 矩阵，而是分配 `256x264` 矩阵；类似于我们在前一节中的做法。但请注意不要陷入未对齐内存访问问题。

### 缓慢的浮点运算

某些进行大量浮点（FP）值计算的应用程序容易遇到一个非常微妙的问题，可能导致性能下降。当应用程序遇到_次正规（subnormal）_ FP 值时，就会出现这个问题，我们将在本节中讨论。您也可能找到*非规格化（denormal）* FP 值这一术语，它指的是同一事物。根据 IEEE 标准 754，[^4] 次正规数是指数（exponent）小于最小正规数（smallest normal number）的非零数。[^3] 代码清单 Subnormals 展示了一个非常简单的次正规值实例化示例。

在实际应用中，次正规值通常代表小到与零无法区分的信号。在音频中，它可能意味着人耳听觉范围之外的极小信号。在图像处理中，它可能意味着像素的任何 RGB 颜色分量非常接近零，等等。有趣的是，次正规值存在于许多生产软件包中，包括天气预报、光线追踪（ray tracing）、物理模拟等。

代码清单：实例化一个正规和次正规的 FP 值

```cpp
unsigned usub = 0x80200000; // -2.93873587706e-39 (subnormal)
unsigned unorm = 0x411a428e; // 9.641248703 (normal)
float sub = *((float*)&usub);
float norm = *((float*)&unorm);
assert(std::fpclassify(sub) == FP_SUBNORMAL);
assert(std::fpclassify(norm) != FP_SUBNORMAL);
```
如果没有次正规值，两个 FP 值 `a - b` 的减法可能下溢（underflow）并产生零，即使这两个值不相等。次正规值允许计算逐渐损失精度，而不会将结果舍入为零。不过，这可能带来一定的代价，我们稍后将看到。当一个值在带有减法或除法的循环中不断减小时，次正规值也可能出现在生产软件中。

从硬件角度来看，处理次正规数比处理正规 FP 值更困难，因为它需要特殊处理，通常被视为异常情况。应用程序不会崩溃，但会受到性能惩罚。产生或使用次正规数的计算比处理正规数的类似计算慢，可能慢 10 倍或更多。例如，Intel 处理器目前通过微码辅助（microcode assist）处理次正规数的操作。当处理器识别到次正规 FP 值时，微码序列器（Microcode Sequencer，MSROM）将提供计算结果所需的微操作（$\mu$ops）。

在许多情况下，次正规值由算法自然生成，因此不可避免。大多数处理器提供了将次正规值刷新为零（flush to zero）的选项，从一开始就不产生次正规数。注重性能的应用程序开发者可能更愿意接受略微不精确的结果，而不是让代码变慢。

假设您的应用程序不需要次正规值，您如何检测和缓解相关代价？虽然您可以使用代码清单 Subnormals 中所示的运行时检查，但在整个代码库中插入这些检查并不实际。有一种更好的方法可以使用 PMU（性能监控单元，Performance Monitoring Unit）检测应用程序是否产生次正规值。在 Intel CPU 上，您可以收集 `FP_ASSIST.ANY` 性能事件，每当使用或产生次正规值时，该事件就会递增。TMA 方法论将此类瓶颈归类于 `Retiring` 类别，是的，这是另一种高 `Retiring` 并不意味着性能好的情况。

一旦确认存在次正规值，您可以启用 FTZ 和 DAZ 模式：

* __DAZ__（非规格化数视为零，Denormals Are Zero）。任何非规格化的输入在使用前被替换为零。
* __FTZ__（刷新为零，Flush To Zero）。任何结果为非规格化的输出被替换为零。

当它们启用后，CPU 浮点运算中就不再需要对次正规值进行昂贵的处理。在基于 x86 的平台上，`MXCSR`（全局控制和状态寄存器）中有两个独立的位域。在 ARM Aarch64 中，两种模式由 `FPCR` 控制寄存器的 `FZ` 和 `AH` 位控制。如果使用 `-ffast-math` 编译应用程序，则无需担心，编译器将在程序启动时自动插入所需代码以启用这两个标志。`-ffast-math` 编译器选项功能较多，因此 GCC 开发人员创建了一个单独的 `-mdaz-ftz` 选项，仅控制次正规值的行为。如果您更希望从源代码中控制，代码清单 EnableFTZDAZ 展示了一个可供使用的示例。如果选择此选项，请避免频繁更改 `MXCSR` 寄存器，因为该操作相对昂贵。读取 MXCSR 寄存器的延迟相当长，而写入该寄存器是一个串行化指令（serializing instruction）。

代码清单：手动启用 FTZ 和 DAZ 模式

```cpp
unsigned FTZ = 0x8000;
unsigned DAZ = 0x0040;
unsigned MXCSR = _mm_getcsr();
_mm_setcsr(MXCSR | FTZ | DAZ);
```
请记住，`FTZ` 和 `DAZ` 模式都与 IEEE 标准 754 不兼容。它们在硬件中实现，是为了提高下溢（underflow）常见且不需要产生非规格化结果的应用程序的性能。我在一些使用次正规值的生产浮点应用程序中观察到了 3%-5% 的性能损失。

[^1]: 大津二值化方法 - [https://en.wikipedia.org/wiki/Otsu%27s_method](https://en.wikipedia.org/wiki/Otsu%27s_method)
[^2]: Performance Ninja 实验任务：内存顺序违规 - [https://github.com/dendibakh/perf-ninja/tree/main/labs/memory_bound/mem_order_violation_1](https://github.com/dendibakh/perf-ninja/tree/main/labs/memory_bound/mem_order_violation_1)
[^3]: 次正规数 - [https://en.wikipedia.org/wiki/Subnormal_number](https://en.wikipedia.org/wiki/Subnormal_number)
[^4]: IEEE 标准 754 - [https://ieeexplore.ieee.org/document/8766229](https://ieeexplore.ieee.org/document/8766229)
[^5]: Performance Ninja 实验任务：内存对齐 - [https://github.com/dendibakh/perf-ninja/tree/main/labs/memory_bound/mem_alignment_1](https://github.com/dendibakh/perf-ninja/tree/main/labs/memory_bound/mem_alignment_1)
[^6]: 此外，在大小为 341、683 和 819 时也有几个峰值。据推测，这些大小也受到相同缓存别名效应的影响，但我没有对其进行深入研究。
