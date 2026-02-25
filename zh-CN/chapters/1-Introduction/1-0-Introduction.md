# 引言

性能为王：十年前如此，现在更是如此。根据 [Domo2017] 的数据，2017 年全球每天产生 2.5 百亿亿[^1]字节的数据。[Statista2024] 预测，这一数字到 2024 年将达到每天 400 百亿亿字节。在我们这个日益以数据为中心的世界里，信息交换的增长需要更快的软件和更快的硬件。

数十年来，软件程序员一直过着"轻松日子"，这要归功于摩尔定律（Moore's law）。即使软件供应商没有在代码改进上投入人力资源，他们也可以依赖新一代硬件来加速其软件产品。然而这种策略已经不再奏效。从图 50YearsProcessorTrend 中可以看出，单线程[^2]性能的增长正在放缓。从 1990 年到 2000 年，SPECint 基准测试上的单线程性能增长了约 25 到 30 倍，这主要得益于更高的 CPU 频率和改进的微架构（microarchitecture）。

![微处理器 50 年趋势数据。*© K. Rupp 通过 karlrupp.net 提供的图片*。2010 年之前的原始数据由 M. Horowitz、F. Labonte、O. Shacham、K. Olukotun、L. Hammond 和 C. Batten 收集并绘制。2010-2021 年的新图表和数据由 K. Rupp 收集。](../../../img/intro/50-years-processor-trend.png)

<p align="center"><em>微处理器 50 年趋势数据。© K. Rupp 通过 karlrupp.net 提供的图片。2010 年之前的原始数据由 M. Horowitz、F. Labonte、O. Shacham、K. Olukotun、L. Hammond 和 C. Batten 收集并绘制。2010-2021 年的新图表和数据由 K. Rupp 收集。</em></p>


从 2000 年到 2010 年，单线程 CPU 性能增长更为温和（增长了 4 至 5 倍）。彼时，由于功耗、散热挑战、电压缩放限制（Dennard 缩放定律[^3]）以及其他基本问题，时钟频率上限约为 4GHz。尽管时钟速度停滞不前，但架构上的进步仍在持续：更好的分支预测（branch prediction）、更深的流水线（pipelines）、更大的缓存（caches）和更高效的执行单元（execution units）。

从 2010 年到 2020 年，单线程性能仅增长了 2 至 3 倍。在此期间，CPU 制造商开始更多地关注多核处理器（multi-core processors）和并行性（parallelism），而不是单纯地提升单线程性能。

现代处理器的晶体管（transistor）数量持续增加。例如，Apple 芯片中的晶体管数量在大约四年时间里从 M1 的 160 亿增长到 M2 的 200 亿，再到 M3 的 250 亿，再到 M4 的 280 亿。晶体管数量的增长使制造商能够在处理器中添加更多的核心（cores）。截至 2024 年，你可以购买到单个 CPU 插槽上拥有超过 100 个逻辑核心的高端服务器处理器，这非常令人印象深刻。遗憾的是，这并不总是能转化为更好的性能。很多时候，应用程序的性能无法随着额外的 CPU 核心线性提升。

既然每一代硬件都提供显著性能提升的时代已经过去，我们就必须开始更加关注代码的运行速度。在寻求性能提升的途径时，开发者不应再依赖硬件，而应开始优化其应用程序的代码。

> "今天的软件效率极其低下；软件程序员是时候在优化方面大展身手了。" —— Marc Andreessen，美国企业家和投资人（a16z Podcast）

[^1]: 百亿亿（quintillion）是 10 的 18 次方（10^18^）。
[^2]: 单线程性能（single-threaded performance）是指在隔离环境下测量的 CPU 核心内单个硬件线程的性能。
[^3]: Dennard 缩放定律 - [https://en.wikipedia.org/wiki/Dennard_scaling](https://en.wikipedia.org/wiki/Dennard_scaling)
