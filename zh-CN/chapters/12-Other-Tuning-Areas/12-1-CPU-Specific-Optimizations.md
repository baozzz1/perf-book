## CPU 特定的优化

针对特定 CPU 微架构优化软件，意味着要对代码进行调整，以充分发挥该微架构的优势并规避其弱点。当您知道应用程序的确切目标 CPU 时，这项工作会相对容易。然而，大多数应用程序需要在各种 CPU 上运行。针对具有极高速度要求的跨平台应用程序进行性能优化颇具挑战性，因为来自不同厂商的平台具有不同的设计和实现。尽管如此，在为特定微架构提供精细调优版本的同时，编写出在不同厂商 CPU 上均能合理运行的代码是完全可行的。

x86（通常被视为 CISC）与 RISC ISA（如 ARM 和 RISC-V）之间的主要差异总结如下：

* x86 指令是可变长度的，而 ARM 和 RISC-V 指令是固定长度的。这使得解码 x86 指令更加复杂。
* x86 ISA 拥有众多寻址模式（addressing modes），而 ARM 和 RISC-V 的寻址模式较少。ARM 和 RISC-V 指令的操作数（operands）要么是寄存器，要么是立即数（immediate values），而 x86 指令的输入也可以直接来自内存。这虽然增加了 x86 指令的数量，但也允许使用更强大的单条指令。例如，ARM 需要先将内存位置加载到寄存器，再执行操作；而 x86 可以用一条指令完成这两步。

除此之外，在针对特定微架构进行优化时，还有一些其他差异需要考虑。截至 2024 年，最新的 x86-64 ISA 拥有 16 个通用寄存器（general-purpose registers），而最新的 ARMv8 和 RV64 要求 CPU 提供 32 个通用寄存器。更多的架构寄存器可以减少寄存器溢出（register spilling），从而减少加载/存储的次数。Intel 已宣布推出名为 APX[^1] 的新扩展，将把寄存器数量增加到 32 个。

x86 和 ARM 在内存页面大小（memory page size）上也存在差异。x86 平台的默认页面大小为 4 KB，而大多数 ARM 系统（例如 Apple MacBooks）使用 16 KB 的页面大小，尽管两个平台都支持更大的页面大小（参见 [ArchHugePages] 和 [secDTLB]）。当这些差异成为瓶颈时，都可能影响应用程序的性能。

尽管 ISA 差异*可能*对特定应用程序的性能产生切实影响，但大量研究表明，平均而言，两种最流行的 ISA（即 x86 和 ARM）之间的差异并不会产生可量化的性能影响。在本书中，我刻意避免为任何产品做广告（例如 Intel vs. AMD vs. Apple）以及任何 ISA 的意识形态之争（x86 vs. ARM vs. RISC-V）。[^5] 以下是一些我希望能终结这场争论的参考资料：

* 性能或能耗差异并非由 ISA 差异产生，而是由微架构实现决定的。[RISCvsCISC2013]
* ISA 对所执行指令的数量和类型没有显著影响。[RISCVvsAArch642023] [RISCvsCISC2013]
* CISC 代码的代码密度并不高于 RISC 代码。[CodeDensityCISCvsRISC]
* ISA 的开销可以通过微架构实现有效缓解。例如，$\mu$op 缓存（$\mu$op cache）可以最小化解码开销；指令缓存（instruction cache）可以最小化代码密度的影响。[RISCvsCISC2013] [ChipsAndCheesex86]

尽管如此，这并不否定架构特定优化（architecture-specific optimizations）的价值。在本节中，我们将讨论如何针对特定平台进行优化。我们将介绍 ISA 扩展（ISA extensions）、CPU 分发（CPU dispatch）技术，并探讨如何理解指令延迟（instruction latencies）和吞吐量（throughput）。

### ISA 扩展

ISA 的演进是持续不断的。其重点在于加速特定工作负载，如密码学（cryptography）、AI、多媒体等。利用 ISA 扩展通常能带来显著的性能提升。开发者们不断探索在通用应用程序中利用这些扩展的巧妙方法。因此，即使您不在这些高度专业化的领域，也可能受益于 ISA 扩展。

了解所有具体指令是不现实的，但我建议您熟悉目标平台上可用的主要 ISA 扩展。例如，如果您正在开发使用 `fp16`（16 位半精度浮点数，16-bit half-precision floating-point）数据类型的 AI 应用程序，并以某款现代 ARM 处理器为目标，请确保您程序的机器码中包含相应的 `fp16` ISA 扩展。如果您正在开发加密/解密软件，请检查它是否利用了目标 ISA 的加密扩展。以此类推。

以下是一些值得关注的 x86 ISA 扩展：

* SSE/AVX/AVX2：为浮点数和整数运算提供 SIMD 指令。
* AVX512：以 512 位寄存器及许多新指令扩展了 AVX2。
* AVX512_FP16/AVX512_BF16：增加了对 16 位半精度和 `Bfloat16` 浮点值的支持。
* AES/SHA：提供用于 AES 加密、解密和 SHA 哈希的指令。
* BMI/BMI2：提供用于位操作（bit manipulation）的指令。
* AVX_VNNI/AVX512_VNNI：用于加速深度学习工作负载的向量神经网络指令（Vector Neural Network Instructions）。
* AMX：用于加速矩阵乘法的高级矩阵扩展（Advanced Matrix Extensions）。

以下是一些值得关注的 ARM ISA 扩展：

* Advanced SIMD：也称为 NEON，提供算术 SIMD 指令。
* Cryptographic Instructions：提供用于加密、哈希和校验和的指令。
* FP16/BF16：提供 16 位半精度和 `Bfloat16` 浮点指令。
* UDOT/SDOT：支持用于加速机器学习工作负载的点积（dot product）指令。
* SVE：支持可扩展向量长度（scalable vector length）指令。
* SME：用于加速矩阵乘法的可扩展矩阵扩展（Scalable Matrix Extension）。

编译应用程序时，请确保启用必要的编译器标志（compiler flags）以激活所需的 ISA 扩展。在 GCC 和 Clang 编译器上，使用 `-march` 选项。例如，`-march=native` 将激活宿主系统（即运行编译的机器）的 ISA 特性；或者您可以指定具体的 ISA 版本，例如 `-march=armv8.6-a`。在 MSVC 编译器上，使用 `/arch` 选项，例如 `/arch:AVX2`。

我不建议在生产构建中使用 `-march=native`，因为代码生成将取决于您构建代码的机器。许多 CI/CD 系统使用旧机器。在这些机器上使用 `-march=native` 进行构建，当应用程序在更新的机器上运行时，可能导致性能欠佳。建议使用 `-march` 并指定您想要针对的具体微架构。

### CPU 分发

当您希望为特定微架构提供快速路径的同时保留其他平台的通用实现时，可以使用 *CPU 分发（CPU dispatching）*。这是一种允许程序检测处理器所具备特性，并据此决定执行哪个版本代码的技术。它使您能够在单一代码库中引入平台特定的优化。根据经验，最好先从通用实现开始，然后逐步引入微架构特定的优化，确保为不具备所需特性的架构提供回退方案（fallback）。例如：

```cpp
if (__builtin_cpu_supports ("avx512f")) {
  avx512_impl();
} else {
  generic_impl();
}
```

上述代码演示了 GCC 和 Clang 编译器中内置函数的使用方法。除了检测所支持的 ISA 扩展外，还有 `__builtin_cpu_is` 函数可用于检测确切的处理器型号。编写 CPU 分发的编译器无关方式包括使用 `CPUID` 指令（仅限 x86）、`getauxval(AT_HWCAP)` Linux 系统调用，或 macOS 上的 `sysctlbyname`。

CPU 分发构造通常用于仅优化代码的特定部分，例如热函数（hot function）或循环。这些平台特定的实现通常使用编译器内联函数（compiler intrinsics，参见 [secIntrinsics]）来生成所需指令。

尽管 CPU 分发是运行时检查，但其开销并不高。您可以在启动时一次性识别硬件能力并将其保存在某个变量中，这样在运行时它只是一个单一的、预测良好的分支。关于 CPU 分发，一个更值得关注的问题可能是维护成本。每个新的专用分支都需要精细调优和验证。

### 指令延迟与吞吐量

除 ISA 扩展外，了解处理器中执行单元（execution units）的数量和类型也很有价值（例如，处理器每个周期能发出多少次加载、存储、除法和乘法）。对于大多数处理器，CPU 厂商会在相应的技术手册中公布这些信息。然而，特定指令的延迟和吞吐量数据通常不会披露。尽管如此，人们已经对单条指令进行了基准测试，相关数据可在线访问。对于最新的 Intel 和 AMD CPU，指令的延迟、吞吐量、端口使用情况以及 $\mu$op 数量可以在 [uops.info](https://uops.info/table.html)[^2] 网站上查阅。对于 Apple 处理器，类似数据可在 [AppleOptimizationGuide] 中获取。[^6] 除了指令延迟和吞吐量之外，开发者还对微架构的其他方面进行了逆向工程，例如分支预测历史缓冲区大小、乱序执行缓冲区（reorder buffer）容量、加载/存储缓冲区大小等。

在根据指令延迟和吞吐量数值得出结论时要非常谨慎。在许多情况下，指令延迟被乱序执行引擎（out-of-order execution engine）所隐藏，一条指令的延迟是 4 个周期还是 8 个周期可能并不重要。如果它不阻碍前向执行（forward progress），该指令将在"后台"处理，不会影响性能。然而，当一条指令处于关键依赖链（critical dependency chain）上时，其延迟就变得重要了，因为它会延迟依赖操作的执行。

相反，如果您的循环执行大量*独立*操作，则应关注指令吞吐量而非延迟。当操作相互独立时，它们可以并行处理。在这种情况下，关键因素是每个周期内可以执行多少次某类操作，即*执行吞吐量（execution throughput）*。也存在一些"中间"情形，其中指令延迟和吞吐量都可能影响性能。

当您分析某个热循环的机器码时，可能会发现多条指令被分配到同一个执行端口（execution port）。这种情况称为*执行端口竞争（execution port contention）*。因此，挑战在于找到将部分指令替换为不分配到关键端口的替代指令的方法。例如，在 Intel 处理器上，如果您严重瓶颈于 `port5`，则可能发现 `port0` 上的两条指令优于 `port5` 上的一条指令。这通常不是一项容易的任务，需要深厚的 ISA 和微架构知识。如有疑问，请在专业论坛上寻求帮助。此外，请记住，这些情况在未来的 CPU 世代中可能会发生变化，因此考虑使用 CPU 分发来隔离代码更改的效果。

### 案例研究：FMA 指令何时会损害性能

在 [FMAThroughput] 中，我们看过一个 FMA 指令吞吐量变得关键的例子。现在让我们来看另一个涉及 FMA 延迟的例子。在代码清单 FMAlatency 左侧，我们有 `sqSum` 函数，它计算每个元素的平方和。右侧展示了当使用 `-O3 -march=core-avx2` 编译时，Clang-18 生成的相应机器码。注意，我们没有使用 `-ffast-math`，也许是因为我们希望在多个平台上保持精确的位级（bit-exact）结果。这就是编译器没有自动向量化（autovectorize）该代码的原因。

代码清单：FMA 延迟

```cpp
float sqSum(float *a, int N) {         │ .loop:
  float sum = 0;                       │  vmovss xmm1, dword ptr [rcx + 4*rdx]
  for (int i = 0; i < N; i++ )         │  vfmadd231ss xmm0, xmm1, xmm1
    sum += a[i] * a[i];                │  inc rdx
  return sum;                          │  cmp rax, rdx
}                                      │  jne .loop
```
在循环的每次迭代中，我们有两个操作：计算 `a[i]` 的平方值，并将乘积累加到 `sum` 变量中。仔细观察可以发现，各次乘法之间相互独立，因此可以并行执行。右侧生成的机器码使用融合乘加（FMA，Fused Multiply-Add）指令，用单条指令完成这两项操作。问题在于，通过使用 FMA，编译器将乘法纳入了循环的关键依赖链中。

`vfmadd231ss` 指令计算 `a[i]`（存于 `xmm1` 中）的平方值，然后将结果累加到 `xmm0` 中。在 `xmm0` 上存在数据依赖：处理器无法在上一条 `vfmadd231ss` 指令完成之前发出新的 `vfmadd231ss` 指令，因为 `xmm0` 既是 `vfmadd231ss` 的输入又是输出。即使 FMA 的乘法部分相互独立，这些指令也需要等待所有输入就绪。此循环的性能受限于 FMA 延迟，在 Intel 的 Alder Lake 中为 4 个周期。

在这种情况下，将乘法和加法融合在一起反而损害了性能。使用两条独立指令会更好。下面的 `nanobench` 实验证明了这一点：

```
# ran on Intel Core i7-1260P (Alder Lake)
$ sudo ./kernel-nanoBench.sh -f -basic │ $ sudo ./kernel-nanoBench.sh -f -basic
 -loop 100 -unroll 1000                │  -loop 100 -unroll 1000 
 -warm_up_count 10 -asm "              │  -warm_up_count 10 -asm "
vmovss xmm1, dword ptr [R14];          │ vmovss xmm1, dword ptr [R14];
vfmadd231ss xmm0, xmm1, xmm1;"         │ vmulss xmm1, xmm1, xmm1;
-asm_init "<not shown>"                │ vaddss xmm0, xmm0, xmm1;"
                                       │ -asm_init "<not shown>"
Instructions retired: 2.00             │ 
Core cycles: 4.00                      │ Instructions retired: 3.00
                                       │ Core cycles: 2.00
```

左侧版本每次迭代运行 4 个周期，对应 FMA 延迟。然而，右侧的 `vmulss` 指令相互独立，可以并行运行。虽然 `vaddss` 指令（FADD）在 `xmm0` 上仍存在循环携带依赖（loop carry dependency），但 FADD 的延迟仅为 2 个周期，这就是右侧版本每次迭代仅需 2 个周期的原因。其他处理器的延迟和吞吐量特性可能有所不同。[^7]

通过这个实验，我们了解到，如果编译器没有决定将乘法和加法融合成一条指令，对于这个循环将获得两倍的性能提升。这一点只有在检查循环依赖并比较 FMA 和 FADD 指令的延迟之后才能明确。从 Clang 18 开始，您可以使用 `#pragma clang fp contract(off)` 在某个作用域内阻止生成 FMA 指令。[^4]

[^1]: Intel APX - [https://www.intel.com/content/www/us/en/developer/articles/technical/advanced-performance-extensions-apx.html](https://www.intel.com/content/www/us/en/developer/articles/technical/advanced-performance-extensions-apx.html)
[^2]: x86 指令延迟和吞吐量 - [https://uops.info/table.html](https://uops.info/table.html)
[^4]: LLVM 用于指定浮点标志的扩展 - [https://clang.llvm.org/docs/LanguageExtensions.html#extensions-to-specify-floating-point-flags](https://clang.llvm.org/docs/LanguageExtensions.html#extensions-to-specify-floating-point-flags)
[^5]: 这场争论其实也并不有趣，因为经过 $\mu$op 转换后，x86 本质上也成为了 RISC 风格的微架构——复杂指令被分解为更简单的指令。
[^6]: 此外，还有通过逆向工程实验收集的指令吞吐量和延迟数据，例如 [https://dougallj.github.io/applecpu/firestorm-simd.html](https://dougallj.github.io/applecpu/firestorm-simd.html)。由于这是非官方数据来源，应持审慎态度。
[^7]: 由于浮点值的舍入方式不同，两个版本将产生略有差异的结果。
