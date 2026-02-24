## 减少 DTLB 缺失

如 [TLBs] 中所述，TLB（Translation Lookaside Buffer，转换后备缓冲区）是每个核心的快速但有限的缓存，用于缓存内存地址的虚拟到物理地址转换。没有它，应用程序的每次内存访问都需要耗时的内核页表遍历（page walk）来计算每个被引用虚拟地址的正确物理地址。在具有 5 级页表（5-level page table）的系统中，需要访问至少 5 个不同的内存位置才能获得地址转换。在 [FeTLB] 节中，我们将讨论如何使用巨页（huge pages）用于代码。在这里，我们将了解如何将它们用于数据。

任何对大块内存区域进行随机访问的算法都可能遭受 DTLB 缺失（DTLB misses）。此类应用程序的示例包括：在大数组中进行二分搜索、访问大型哈希表以及遍历图。使用巨页有可能加速此类应用程序。

在 x86 平台上，默认页面大小为 4KB。考虑一个频繁引用 20MB 内存空间的应用程序。使用 4KB 页面，OS 需要分配许多小页面。此外，进程将访问许多 4KB 大小的页面，每个页面都在争用有限数量的 TLB 条目。相比之下，使用 2MB 巨页，20MB 内存只需 10 个页面即可映射，而使用 4KB 页面则需要 5120 个页面。这意味着使用巨页时需要的 TLB 条目更少，从而减少了 TLB 缺失次数。这不会按 512 的比例成比例减少，因为 2MB 条目的数量要少得多。例如，在 Intel 的 Skylake 核心系列中，L1 DTLB 对 4KB 页面有 64 个条目，而对 2MB 页面只有 32 个条目。除了 2MB 巨页外，AMD 和 Intel 的 x86 芯片还支持 1GB 超大页（gigantic pages）用于数据，但不支持用于指令。使用 1GB 页面代替 2MB 页面可以进一步减少 TLB 压力。

使用巨页通常会减少页表遍历次数，而且在 TLB 缺失时遍历内核页表的惩罚也会减少，因为页表本身更加紧凑。使用巨页带来的性能提升有时可以高达 30%，具体取决于应用程序所承受的 TLB 压力。期望 2 倍的加速幅度要求过高，因为 TLB 缺失极少是主要瓶颈。论文 [Luo2015] 介绍了在 SPEC2006 基准测试套件上使用巨页的评估结果。结果可以总结如下：在套件的 29 个基准测试中，15 个的加速幅度在 1% 以内，可以作为噪声忽略；6 个基准测试的加速幅度在 1%-4% 之间；4 个基准测试的加速幅度在 4% 到 8% 之间；2 个基准测试的加速幅度为 10%；收益最大的 2 个基准测试分别享有 22% 和 27% 的加速。

许多真实世界的应用程序已经充分利用了巨页，例如 KVM、MySQL、PostgreSQL、Java 的 JVM 等。通常，这些软件包提供了启用该功能的选项。使用类似应用程序时，请查阅其文档以了解是否可以启用巨页。

Windows 和 Linux 都允许应用程序建立巨页内存区域。有关在 Windows 和 Linux 上启用巨页的说明，请参阅附录 B。在 Linux 上，应用程序中使用巨页有两种方式：显式巨页（Explicit Huge Pages）和透明巨页（Transparent Huge Pages）。Windows 的支持不如 Linux 丰富，将在后面讨论。

### 显式巨页（Explicit Huge Pages）

显式巨页（EHP）作为系统内存的一部分提供，通过巨页文件系统 `hugetlbfs` 暴露。EHP 应该在系统启动时或应用程序启动前预留。有关操作方法，请参阅附录 B。在启动时预留 EHP 可以增加成功分配的可能性，因为此时内存尚未发生严重碎片化。显式预分配的页面驻留在保留的物理内存块中，不能在内存压力下被换出。此外，这部分内存空间不能用于其他目的，因此用户应谨慎，只预留所需数量的页面。

在 Linux 应用程序中使用 EHP 的最简单方法是使用 `MAP_HUGETLB` 标志调用 `mmap`，如代码清单 ExplicitHugepages1 所示。在此代码中，指针 `ptr` 将指向为 EHP 显式预留的 2MB 内存区域。注意，如果事先没有预留 EHP，分配可能会失败。在附录 B 中提供了在用户代码中使用 EHP 的其他不太常用的方式。此外，开发者可以编写自己的基于竞技场的分配器，利用 EHP。

代码清单：从显式分配的巨页映射内存区域。

```cpp
void ptr = mmap(nullptr, size, PROT_READ | PROT_WRITE,
                MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB, -1, 0);
if (ptr == MAP_FAILED)
  throw std::bad_alloc{};                
...
munmap(ptr, size);
```
过去，有一个 [libhugetlbfs](https://github.com/libhugetlbfs/libhugetlbfs)[^1] 库可以覆盖现有动态链接可执行文件中使用的 `malloc` 调用，以在 EHP 中分配内存。遗憾的是，该项目已不再维护。它不需要用户修改代码或重新链接二进制文件，只需在命令行前加上 `LD_PRELOAD=libhugetlbfs.so HUGETLB_MORECORE=yes <你的应用程序命令行>` 即可使用。但幸运的是，其他库可以使 `malloc` 使用巨页（不是 EHP），我们将在下面讨论。

### 透明巨页（Transparent Huge Pages）

Linux 还提供透明巨页支持（Transparent Huge Page Support，THP），有两种操作模式：系统级和进程级。当系统级启用 THP 时，内核自动管理巨页，对应用程序透明。OS 内核在需要大块内存且可能分配时，会尝试为任何进程分配巨页，因此不需要手动预留巨页。如果 THP 按进程启用，内核只为通过 `madvise` 系统调用标记的各个进程内存区域分配巨页。您可以通过以下命令检查系统中是否启用了 THP：

```bash
$ cat /sys/kernel/mm/transparent_hugepage/enabled
always [madvise] never
```

括号中显示的值是当前设置。如果该值为 `always`（系统级）或 `madvise`（进程级），则 THP 对您的应用程序可用。有关每个选项的详细规范，可以在 Linux 内核关于 THP 的[文档](https://www.kernel.org/doc/Documentation/vm/transhuge.txt)[^2]中找到。

当系统级启用 THP 时，巨页会自动用于普通内存分配，无需应用程序显式请求。要观察巨页对应用程序的效果，用户只需通过 `echo "always" | sudo tee /sys/kernel/mm/transparent_hugepage/enabled` 启用系统级 THP。这将自动启动一个名为 `khugepaged` 的守护进程，开始扫描应用程序的内存空间，将常规页面提升为巨页。有时内核可能无法将多个常规页面合并为一个巨页，原因是找不到连续的 2MB 内存块。

系统级 THP 模式适合快速实验，检查巨页是否能提升性能。它自动工作，即使对于不了解 THP 的应用程序也是如此，因此开发者不必修改代码就能看到巨页对其应用程序的好处。当系统级启用巨页时，应用程序最终可能会分配比所需更多的内存资源。这就是为什么系统级模式默认是禁用的。完成实验后不要忘记禁用系统级 THP，因为它可能影响整体系统性能。

使用 `madvise`（进程级）选项时，THP 仅在通过 `madvise` 系统调用带 `MADV_HUGEPAGE` 标志标记的内存区域内启用。如代码清单 TransparentHugepages1 所示，指针 `ptr` 将指向内核动态分配的 2MB 匿名（透明）内存区域。如果内核找不到连续的 2MB 内存块，`mmap` 调用将失败。

代码清单：将内存区域映射到透明巨页。

```cpp
void ptr = mmap(nullptr, size, PROT_READ | PROT_WRITE | PROT_EXEC,
                MAP_PRIVATE | MAP_ANONYMOUS, -1 , 0);
if (ptr == MAP_FAILED)
  throw std::bad_alloc{};
madvise(ptr, size, MADV_HUGEPAGE);
// use the memory region `ptr`
munmap(ptr, size);
```
开发者可以基于代码清单 TransparentHugepages1 中的代码构建自定义 THP 分配器。但也可以在应用程序调用的 `malloc` 中使用 THP。许多内存分配库通过覆盖 `libc` 中 `malloc` 的实现来提供该功能。以下是使用 `jemalloc`（最受欢迎的选项之一）的示例。如果您有权访问应用程序的源代码，可以使用额外的 `-ljemalloc` 选项重新链接二进制文件。这将动态链接您的应用程序与 `jemalloc` 库，该库将处理所有 `malloc` 调用。然后使用以下选项为堆分配启用 THP：

```bash
$ MALLOC_CONF="thp:always" <你的应用程序命令行>
```

如果您没有源代码访问权限，仍然可以通过预加载动态库来使用 `jemalloc`：

```bash
$ LD_PRELOAD=/usr/local/libjemalloc.so.2 MALLOC_CONF="thp:always" <你的应用程序命令行>
```

Windows 仅提供通过 `VirtualAlloc` 系统调用以类似 Linux THP 进程级模式的方式使用巨页。详情请参阅附录 B。

### 显式巨页与透明巨页的比较

Linux 用户可以以三种不同模式使用巨页：

* 显式巨页（Explicit Huge Pages）
* 系统级透明巨页（System-wide Transparent Huge Pages）
* 进程级透明巨页（Per-process Transparent Huge Pages）

让我们比较这些选项。首先，EHP 在虚拟内存中预先预留，THP 则不是。这使得使用 EHP 的软件包更难发布，因为它们依赖于机器管理员进行的特定配置设置。此外，EHP 静态地驻留在内存中，占用宝贵的 DRAM，即使不使用时也是如此。

系统级透明巨页非常适合快速实验。无需更改用户代码即可测试在应用程序中使用巨页的好处。然而，将软件包发布给客户并要求他们启用系统级 THP 是不明智的，因为它可能对该系统上运行的其他程序产生负面影响。通常，开发者会识别代码中可能从巨页中受益的分配，并在这些地方使用 `madvise` 提示（进程级模式）。

进程级 THP 没有上述任何一种缺点，但有另一个缺点。我们之前讨论过，内核的 THP 分配对用户是透明的。分配过程可能涉及多个内核进程，负责在虚拟内存中腾出空间，可能包括将内存交换到磁盘、碎片整理或页面提升。透明巨页的后台维护会因内核管理不可避免的碎片化和交换问题而产生不确定的延迟开销。EHP 不受内存碎片化影响，也不能被交换到磁盘，因此产生的延迟开销要小得多。

总体而言，THP 更易于使用，但会产生更大的分配延迟开销。这就是 THP 在高频交易（High-Frequency Trading）和其他超低延迟行业中不受欢迎的原因；它们更倾向于使用 EHP。另一方面，虚拟机提供商和数据库倾向于使用进程级 THP，因为要求额外的系统配置可能会给用户带来负担。

[^1]: libhugetlbfs - [https://github.com/libhugetlbfs/libhugetlbfs](https://github.com/libhugetlbfs/libhugetlbfs).
[^2]: Linux 内核 THP 文档 - [https://www.kernel.org/doc/Documentation/vm/transhuge.txt](https://www.kernel.org/doc/Documentation/vm/transhuge.txt)
