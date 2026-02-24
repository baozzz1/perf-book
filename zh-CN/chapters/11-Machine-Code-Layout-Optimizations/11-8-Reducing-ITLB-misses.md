## 减少 ITLB 缺失（Reducing ITLB Misses）

调优前端效率的另一个重要领域是内存地址的虚拟地址到物理地址的转换。这些转换主要由 TLB（Translation Lookaside Buffer，转换后备缓冲区，参见 [TLBs]）提供服务，TLB 在专用条目中缓存最近使用的内存页转换。当 TLB 无法满足转换请求时，就需要进行耗时的内核页表遍历（page walk），以计算每个引用的虚拟地址的正确物理地址。每当你在 TMA 摘要中看到较高的 ITLB 开销百分比时，本节中的建议可能会有所帮助。

一般来说，相对较小的应用程序不容易受到 ITLB 缺失的影响。例如，Golden Cove 微架构的 ITLB 可以覆盖最多 1MB 的内存空间。如果你的应用程序的机器码适合在 1MB 以内，则不应受到 ITLB 缺失的影响。当应用程序中频繁执行的部分分散在内存中时，问题就开始出现了。当许多函数开始频繁地相互调用时，它们开始竞争 ITLB 中的条目。一个例子是 Clang 编译器，在撰写本文时，其代码段约为 60MB。在配备主流 Intel Coffee Lake 处理器的笔记本电脑上运行时，ITLB 开销约为 7%，这意味着 7% 的周期花费在处理 ITLB 缺失上：进行按需页表遍历和填充 TLB 条目。

另一类频繁受益于使用大页（huge pages）的大内存应用程序包括关系数据库（如 MySQL、PostgreSQL、Oracle）、托管运行时（如 JavaScript V8、Java JVM）、云服务（如网络搜索）、Web 工具（如 node.js）。

减少 ITLB 压力的总体思路是将应用程序性能关键代码的部分映射到 2MB（大页）页面上。通常为了简单起见，会重映射应用程序的整个代码段。此变换发生的关键要求是将代码段对齐到 2MB 边界。在 Linux 上，这可以通过两种不同方式实现：使用附加链接器选项重新链接二进制文件，或在运行时重新映射代码段。两种选项都在 Easyperf[^1] 博客上有展示。据我所知，这在 Windows 上不可行，因此我只介绍如何在 Linux 上实现。

第一种方式可以通过使用以下选项链接二进制文件来实现：`-Wl,-zcommon-page-size=2097152` `-Wl,-zmax-page-size=2097152`。这些选项指示链接器将代码段放置在 2MB 边界处，以便在启动时由加载器将其放置在 2MB 大页上。这种放置方式的缺点是链接器将被迫插入最多 2MB 的填充（浪费的）字节，使二进制文件进一步膨胀。以 Clang 编译器为例，它将二进制文件的大小从 111 MB 增加到了 114 MB。重新链接二进制文件后，我们在 ELF 二进制文件头中设置一个特殊位，该位决定文本段在默认情况下是否应由大页支持。最简单的方法是使用 [libhugetlbfs](https://github.com/libhugetlbfs/libhugetlbfs/blob/master/HOWTO)[^12] 包中的 `hugeedit` 或 `hugectl` 工具。例如：

```bash
# Permanently set a special bit in the ELF binary header.
$ hugeedit --text /path/to/clang++
# Code section will be loaded using huge pages by default.
$ /path/to/clang++ a.cpp

# Overwrite default behavior at runtime.
$ hugectl --text /path/to/clang++ a.cpp
```

第二种方式是在运行时重新映射代码段。此方式不要求代码段对齐到 2MB 边界，因此无需重新编译应用程序即可工作。当你无法访问源代码时，这特别有用。该方法的思路是在程序启动时分配大页，并将代码段传输到那里。该方法的参考实现在 [iodlr](https://github.com/intel/iodlr)[^2] 库中。一种选择是从你的 `main` 函数中调用该功能。另一种更简单的选择是构建动态库并在命令行中预加载：

```bash
$ LD_PRELOAD=/usr/lib64/liblppreload.so clang++ a.cpp
```

虽然第一种方式仅适用于显式大页，但使用 `iodlr` 的第二种方式同时适用于显式大页和透明大页（transparent huge pages）。有关如何在 Windows 和 Linux 上启用大页的说明可在附录 B 中找到。

将代码段映射到大页上可以将 ITLB 缺失次数减少高达 50% [IntelBlueprint]，对某些应用程序可带来高达 10% 的性能提升。但是，与许多其他功能一样，大页并不适合每个应用程序。可执行文件只有几 KB 的小程序最好使用常规的 4KB 页面而不是 2MB 大页；这样内存使用更加高效。

除了使用大页之外，优化 I-cache 性能的标准技术也可以用于改善 ITLB 性能。即：重新排列函数使热函数更好地集中在一起、通过链接时优化（Link-Time Optimizations，LTO/IPO）减少热区域大小、使用配置文件引导优化（PGO）和 BOLT、以及减少激进的内联。

BOLT 提供了 `-hugify` 选项，可根据性能分析数据自动为热代码使用大页。使用此选项时，`llvm-bolt` 将注入代码，在运行时将热代码放置在 2MB 大页上。该实现利用了 Linux 透明大页（Transparent Huge Pages，THP）。这种方法的好处是只有一小部分代码被映射到大页，所需大页的数量被最小化，从而减少页面碎片化。

[^1]: "使用大页提升代码性能的好处" - [https://easyperf.net/blog/2022/09/01/Utilizing-Huge-Pages-For-Code](https://easyperf.net/blog/2022/09/01/Utilizing-Huge-Pages-For-Code)。
[^2]: iodlr 库，仅适用于 Linux - [https://github.com/intel/iodlr](https://github.com/intel/iodlr)。
[^12]: libhugetlbfs - [https://github.com/libhugetlbfs/libhugetlbfs/blob/master/HOWTO](https://github.com/libhugetlbfs/libhugetlbfs/blob/master/HOWTO)。
