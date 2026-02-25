## 多线程程序中的性能扩展

在处理单线程应用程序时，优化程序的某一部分通常会带来正向的性能提升。然而，对于多线程应用程序来说，情况并非如此。可能存在这样的应用程序：线程 `A` 执行一个长时间运行的操作，而线程 `B` 提前完成任务后只是等待线程 `A` 完成。无论我们如何改进线程 `B`，应用程序的延迟都不会降低，因为它受限于运行时间更长的线程 `A`。

这种效应广为人知，即[阿姆达尔定律](https://en.wikipedia.org/wiki/Amdahl's_law)（Amdahl's law），[^6]它指出并行程序的加速比受其串行部分的限制。图 MT_AmdahlsLaw 展示了程序执行延迟的理论加速比与执行处理器数量的函数关系。对于一个 75% 部分可并行的程序，加速比收敛于 4。

<div id="fig:AmdahlUSLLaws">
![根据阿姆达尔定律，理论加速比上限与处理器数量的函数关系。](../../../img/mt-perf/AmdahlsLaw.png)

<p align="center"><em>根据阿姆达尔定律，理论加速比上限与处理器数量的函数关系。</em></p>

![线性加速比、阿姆达尔定律和通用可扩展性定律。](../../../img/mt-perf/USL.png)

<p align="center"><em>线性加速比、阿姆达尔定律和通用可扩展性定律。</em></p>


阿姆达尔定律和通用可扩展性定律。
</div>

实际上，进一步向系统添加计算节点可能会导致加速比倒退。我们将在下一节中看到相关示例。Neil Gunther 以[通用可扩展性定律](http://www.perfdynamics.com/Manifesto/USLscalability.html#tth_sEc1)（Universal Scalability Law，USL）[^8]解释了这种效应，它是阿姆达尔定律的扩展。USL 将计算节点（线程）之间的通信描述为制约性能的另一个因素。随着系统规模的扩大，开销开始抵消收益。超过某个临界点后，系统的处理能力开始下降（见图 MT_USL）。USL 被广泛用于对系统的容量和可扩展性进行建模。

USL 所描述的性能下降由几个因素驱动。首先，随着计算节点数量的增加，它们开始竞争资源（竞争，contention），这导致需要花费额外时间来同步这些访问。另一个问题出现在多个工作节点共享资源时。我们需要在多个工作节点之间维护共享资源的一致状态（一致性，coherence）。例如，当多个工作节点频繁修改一个全局可见对象时，这些修改需要广播到所有使用该对象的节点。突然之间，由于需要额外维护一致性，普通操作开始需要更多时间才能完成。优化多线程应用程序不仅涉及本书迄今为止描述的所有技术，还涉及检测和缓解上述竞争和一致性效应。

[^6]: Amdahl's law - [https://en.wikipedia.org/wiki/Amdahl's_law](https://en.wikipedia.org/wiki/Amdahl's_law).
[^8]: USL law - [http://www.perfdynamics.com/Manifesto/USLscalability.html#tth_sEc1](http://www.perfdynamics.com/Manifesto/USLscalability.html#tth_sEc1).
