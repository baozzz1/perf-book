## 系统调优

在完成了利用 CPU 微架构各种复杂设施对应用程序进行调优的艰苦工作之后，我们最不希望看到的是系统固件（firmware）、操作系统或内核摧毁我们的所有努力。即使是调优最精细的应用程序，如果被周期性的系统中断打断并使整个系统停顿，其意义也将大打折扣。这种中断每次可能持续数十毫秒甚至数百毫秒。

开发者通常对应用程序的执行环境几乎没有控制权。当我们发布产品时，调优每个客户可能拥有的配置是不现实的。通常，足够大的组织会有专门的运维团队（Operations Teams，Ops）来处理此类问题。我们的利益在于为他们提供如何配置系统以获得应用程序最佳性能的建议。

现代系统中有许多可调优的内容，避免基于系统的干扰并非易事。x86 服务器部署性能调优手册的一个例子是 Red Hat [指南](https://access.redhat.com/sites/default/files/attachments/201501-perf-brief-low-latency-tuning-rhel7-v2.1.pdf)[^5]。在那里，您将找到消除或显著减少来自系统 BIOS、Linux 内核和设备驱动程序等应用干扰源所导致的缓存中断性中断的提示。这些指南应作为所有新服务器构建的基准配置，在部署任何应用程序到生产环境之前使用。

大多数开箱即用的平台都配置为在尽可能节省功耗的同时实现最佳吞吐量（throughput）。但某些行业有实时性要求，更关注低延迟而非其他一切。此类行业的一个例子是汽车装配线上运行的机器人。此类机器人执行的动作由外部事件触发，通常有预定的时间预算来完成，因为下一个中断很快就会到来（通常称为"控制循环"，control loop）。为此类平台满足实时目标可能需要牺牲机器的整体吞吐量或允许其消耗更多能量。该领域的一种流行技术是禁用处理器休眠状态（processor sleeping states）[^7]，使其保持就绪状态以立即响应。另一种有趣的技术称为缓存锁定（Cache Locking）[^8]，它将 CPU 缓存的某些部分保留给特定的数据集，有助于精简应用程序内的内存延迟。

提升性能的更极端手段是超频（overclocking）CPU。超频是以高于 CPU 设计频率运行的过程。这是一项有风险的操作，可能使保修失效并损坏 CPU。超频不适合生产环境，通常由愿意为了性能而承担风险的发烧友进行。要对 CPU 超频，您需要合适的硬件，主要是支持超频的主板、具有解锁时钟频率的 CPU，以及能够处理增加热量输出的散热系统。2024 年初，超频专家在广泛可用的 CPU 上突破了 9 GHz 大关。[^9]

了解应用程序的性能瓶颈对于调优正确的设置很有帮助。可扩展性研究（Scalability studies）可以帮助您确定应用程序对各种系统设置的敏感程度。例如，您可能会发现应用程序随核心数量的增加无法线性扩展（参见 [ThreadCountScalingStudy]），或者受到内存延迟的限制。利用这些信息，您可以就调优系统设置或为计算系统购买新硬件组件做出有根据的决策。在下一个案例研究中，我们将展示如何确定应用程序对末级缓存（LLC，Last-Level Cache）大小的敏感程度。

[^5]: Red Hat 低延迟调优指南 - [https://access.redhat.com/sites/default/files/attachments/201501-perf-brief-low-latency-tuning-rhel7-v2.1.pdf](https://access.redhat.com/sites/default/files/attachments/201501-perf-brief-low-latency-tuning-rhel7-v2.1.pdf)
[^7]: 电源管理状态：P 态、C 态 - [https://software.intel.com/content/www/us/en/develop/articles/power-management-states-p-states-c-states-and-package-c-states.html](https://software.intel.com/content/www/us/en/develop/articles/power-management-states-p-states-c-states-and-package-c-states.html)
[^8]: 缓存锁定。缓存锁定技术综述 [CacheLocking]。将缓存的一部分伪锁定（pseudo-locking），然后在 Linux 文件系统中以字符设备形式公开供 `mmap` 使用的示例：[https://events19.linuxfoundation.org/wp-content/uploads/2017/11/Introducing-Cache-Pseudo-Locking-to-Reduce-Memory-Access-Latency-Reinette-Chatre-Intel.pdf](https://events19.linuxfoundation.org/wp-content/uploads/2017/11/Introducing-Cache-Pseudo-Locking-to-Reduce-Memory-Access-Latency-Reinette-Chatre-Intel.pdf)
[^9]: CPU 超频记录 - [https://press.asus.com/news/press-releases/rog-maximus-z790-apex-encore-sets-3-overclocking-world-records/](https://press.asus.com/news/press-releases/rog-maximus-z790-apex-encore-sets-3-overclocking-world-records/)
