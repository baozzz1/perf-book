## 动态内存分配

开发者应该意识到动态内存分配（dynamic memory allocation）有很多相关开销。在栈上分配对象是即时完成的，与对象大小无关：只需移动栈指针。然而，动态内存分配是一个更复杂的操作。它涉及调用类似 `malloc` 的标准库函数，该函数可能会将分配请求委托给操作系统。避免不必要的动态内存分配是减少这些开销的第一步。这个过程中容易发现的目标是临时分配，即紧接着就被释放的分配。在 [HeaptrackCaseStudy] 中，我们展示了如何使用 `heaptrack` 来查找动态内存分配的来源。

您可以通过分配一大块内存来摊销（amortize）许多小分配的开销。这是[竞技场分配器（arena allocators）](https://en.wikipedia.org/wiki/Region-based_memory_management)[^16]和内存池（memory pools）背后的核心思想。它为手动内存管理提供了更大的灵活性。您可以获取 OS 分配的内存区域，并在该区域之上设计自己的分配策略。一种简单的策略可以是将该区域分为两部分：一部分用于热数据，一部分用于冷数据，并提供两种分配方法，分别从各自的竞技场中分配。将热数据聚集在一起为更好的缓存利用率创造了机会。这也很可能改善 TLB 利用率，因为热数据会更紧凑，占用更少的内存页面。

动态内存分配的另一个开销出现在应用程序使用多线程时。当两个线程同时尝试分配内存时，OS 必须对它们进行同步。在高并发应用程序中，线程可能花费大量时间等待公共锁来分配内存。内存释放也是如此。同样，自定义分配器可以帮助避免这个问题，例如，通过为每个线程使用独立的竞技场。

有很多标准动态内存分配例程（`malloc` 和 `free`）的即插即用替代品，它们更快、可扩展性更好，并且能更好地解决内存碎片问题。一些最流行的内存分配库是 [jemalloc](http://jemalloc.net/)[^17] 和 [tcmalloc](https://github.com/google/tcmalloc)[^18]。一些项目将 `jemalloc` 和 `tcmalloc` 作为默认内存分配器，并观察到了显著的性能提升。

最后，动态内存分配的一些开销是隐藏的[^20]，无法轻易测量。在所有主流操作系统中，`malloc` 返回的指针只是一个承诺——OS 承诺当页面被访问时会提供所需内存，但实际的物理页面在虚拟地址被访问之前并不会分配。这称为*按需分页（demand paging）*，每次新分配的页面都会产生一次次缺页中断（minor page fault）的开销。我们在 [AvoidPageFaults] 中讨论如何缓解这一开销。此外，出于安全原因，所有现代操作系统在将页面交给下一个进程之前都会清除内容（写入零）。OS 维护一个已清零页面的池，以备分配使用。但当这个池中的可用已清零页面耗尽时，OS 必须按需清零一个页面。这个过程并不是非常昂贵，但也不是免费的，可能会增加内存分配调用的延迟。

[^16]: 基于区域的内存管理 - [https://en.wikipedia.org/wiki/Region-based_memory_management](https://en.wikipedia.org/wiki/Region-based_memory_management)
[^17]: jemalloc - [http://jemalloc.net/](http://jemalloc.net/).
[^18]: tcmalloc - [https://github.com/google/tcmalloc](https://github.com/google/tcmalloc)
[^20]: Bruce Dawson：内存分配的隐藏开销 - [https://randomascii.wordpress.com/2014/12/10/hidden-costs-of-memory-allocation/](https://randomascii.wordpress.com/2014/12/10/hidden-costs-of-memory-allocation/).
