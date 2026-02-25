## 微操作（Micro-operations）

具有 x86 架构的微处理器会将复杂的 CISC 指令转换为简单的 RISC 微操作（micro-operations），缩写为 μops。一条简单的寄存器到寄存器的加法指令（如 `ADD rax, rbx`）只会生成一个 μop，而像 `ADD rax, [mem]` 这样更复杂的指令则可能生成两个：一个用于从 `mem` 内存位置加载数据到临时（未命名）寄存器，另一个用于将其与 `rax` 寄存器相加。指令 `ADD [mem], rax` 则会生成三个 μops：一个用于从内存加载，一个用于相加，一个用于将结果存储回内存。

将指令拆分为微操作的主要优势在于，μops 可以：

* **乱序（Out of order）执行**：以 `PUSH rbx` 指令为例，该指令将栈指针减小 8 个字节，然后将源操作数存储到栈顶。假设 `PUSH rbx` 在解码后被"拆解"为两个相互依赖的微操作：
  ```
  SUB rsp, 8
  STORE [rsp], rbx
  ```
  通常，函数序言（function prologue）会通过多条 `PUSH` 指令保存多个寄存器。在这种情况下，下一条 `PUSH` 指令可以在前一条 `PUSH` 指令的 `SUB` μop 完成后立即开始执行，而不必等待 `STORE` μop，后者现在可以异步执行。

* **并行（In parallel）执行**：以 `HADDPD xmm1, xmm2` 指令为例，该指令将 `xmm1` 和 `xmm2` 中的两个双精度浮点值进行规约求和，并将两个结果存储到 `xmm1` 中，如下所示：
  ```
  xmm1[63:0] = xmm2[127:64] + xmm2[63:0]
  xmm1[127:64] = xmm1[127:64] + xmm1[63:0]
  ```
  对该指令进行微编码的一种方式是：1) 规约 `xmm2` 并将结果存储在 `xmm_tmp1[63:0]`，2) 规约 `xmm1` 并将结果存储在 `xmm_tmp2[63:0]`，3) 将 `xmm_tmp1` 和 `xmm_tmp2` 合并到 `xmm1`。共计三个 μops。注意步骤 1) 和 2) 是独立的，因此可以并行执行。

尽管我们刚刚讨论了指令如何被拆分为更小的部分，但有时 μops 也可以被融合在一起。现代 x86 CPU 中有两种融合类型：

* **微融合（Microfusion）**：融合同一机器指令中的 μops。微融合只能应用于两种组合：内存写操作和读-改-写操作。例如：

  ```bash
  add    eax, [mem]
  ```
  该指令包含两个 μops：1) 读取内存位置 `mem`，2) 将其加到 `eax`。通过微融合，两个 μops 在解码阶段被合并为一个。
  
* **宏融合（Macrofusion）**：融合不同机器指令中的 μops。在特定情况下，解码器可以将算术或逻辑指令与后续的条件跳转指令融合为一个"计算-分支"μop。例如：

  ```bash
  .loop:
    dec rdi
    jnz .loop
  ```
  通过宏融合，`DEC` 和 `JNZ` 指令的两个 μops 被融合为一个。Zen4 微架构还增加了对 DIV/IDIV 和 NOP 宏融合的支持 [amd_zen4]。

微融合和宏融合都能节省流水线（pipeline）各个阶段（从解码到退休）的带宽。融合后的操作共享重排序缓冲区（ROB，reorder buffer）中的单个条目。当融合的 μop 只占用一个条目时，ROB 的容量得到了更好的利用。这样的融合 ROB 条目随后会被分发（dispatch）到两个不同的执行端口（execution port），但仍作为单个单元退休。读者可以在 [fogMicroarchitecture] 中了解更多关于 μop 融合的内容。

要收集应用程序中已发射（issued）、已执行（executed）和已退休（retired）μops 的数量，可以使用 Linux `perf`：

```bash
$ perf stat -e uops_issued.any,uops_executed.thread,uops_retired.slots -- ./a.exe
  2856278  uops_issued.any             
  2720241  uops_executed.thread
  2557884  uops_retired.slots
```

指令被拆分为微操作的方式可能因 CPU 世代而异。通常，一条指令使用的 μops 数量越少，意味着硬件对其支持越好，延迟（latency）可能更低，吞吐量（throughput）可能更高。对于最新的 Intel 和 AMD CPU，绝大多数指令只生成一个 μop。近期微架构上 x86 指令的延迟、吞吐量、端口使用情况以及 μops 数量可以在 [uops.info](https://uops.info/table.html)[^1] 网站上找到。

[^1]: x86 指令延迟和吞吐量 - [https://uops.info/table.html](https://uops.info/table.html)
