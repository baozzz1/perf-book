## 为什么要关注性能？

除了硬件单线程性能增长放缓之外，还有几个商业层面的理由值得关注性能问题。在 PC 时代，[^12] 慢软件的代价由用户承担，因为低效软件运行在用户的计算机上，软件供应商没有直接的动力去优化其应用程序的代码。随着 SaaS（软件即服务，software as a service）和云计算的兴起，慢软件的代价回到了软件提供商身上，而非用户。如果你是像 Meta 或 Netflix[^4] 这样的 SaaS 公司，无论是在自有机房运行服务，还是使用公有云，你都要为服务器消耗的电力付费。低效的软件直接侵蚀你的利润空间和市场估值。根据 Synergy Research Group[^5] 的数据，2020 年全球云服务支出突破 1000 亿美元，而根据 Gartner[^6] 的预测，这一数字将在 2024 年超过 6750 亿美元。

多年来，性能工程（performance engineering）一直是一个小众的极客领域，但如今它正逐渐成为主流。许多公司已经意识到性能工程的重要性，并愿意为此类工作支付可观的薪酬。

要达到表 PlentyOfRoom 中第 4 行的性能水平其实并不难，实际上你根本不需要这本书就能做到。用一种原生编程语言编写程序，将工作分发给多个线程，选择一个优秀的优化编译器，你就能实现这一目标。遗憾的是，你的程序性能大约会比最优目标慢 200 倍。

本书中的方法论专注于从你的应用程序中榨取最后一丝性能。此类转换对应表 PlentyOfRoom 中的第 6 行和第 7 行。将要讨论的改进类型通常幅度不大，往往不超过 10%。但请不要低估 10% 的性能提升的重要性。SQLite 今天的普及，并不是因为其开发者某天将其速度提升了 50%，而是因为他们多年来精心积累了数百次 0.1% 的改进。这些微小改进的累积效应，才是造就差异的关键所在。

微小改进的影响对于在云端运行的大型分布式应用程序尤为显著。根据 [HennessyGoogleIO]，2018 年 Google 在实际运行云服务的计算服务器上的支出，与其在电力和冷却基础设施上的支出大致相当。能源效率是一个非常重要的问题，可以通过优化软件来改善。

> "在 [Google] 这样的规模下，了解性能特征变得至关重要——即使是微小的性能或利用率改进，也可以转化为巨额的成本节约。" [GoogleProfiling]

除了云成本，还有另一个因素在起作用：人们对慢速软件的感知。Google 报告称，搜索延迟 500 毫秒导致流量下降了 20%。[^3] 雅虎（Yahoo!）的页面加载速度提高 400 毫秒，带来了 5-9% 的额外流量。[^8] 在大数量博弈中，微小的改进可以产生显著的影响。这些例子证明，服务运行越慢，使用它的人就越少。

在云服务之外，还有许多其他对性能要求极高的行业，在这些行业中性能工程的价值无需论证，如人工智能（AI）、高性能计算（HPC, High-Performance Computing）、高频交易（HFT, High-Frequency Trading）、游戏开发等。此外，性能不仅在高度专业化的领域才被需要，它对通用应用程序和服务同样重要。我们每天使用的许多工具，如果无法满足其性能要求，根本就不会存在。例如，集成到 Microsoft Visual Studio IDE 中的 Visual C++ IntelliSense[^2] 功能有着非常严格的性能约束。为了让 IntelliSense 自动补全功能正常工作，它必须在毫秒级内解析整个源代码库。[^9] 如果一个源代码编辑器需要数秒才能给出自动补全建议，没人会使用它。此类功能必须响应迅速，并在用户输入新代码时提供有效的补全选项。

> "并非所有快速软件都是世界级的，但所有世界级软件都是快速的。性能是*最*杀手级的功能。" ——Tobi Lutke，Shopify 首席执行官。

我希望不言而喻的是，人们讨厌使用慢速软件，尤其是当慢速降低了他们的工作效率时。表 1.2 显示，大多数人认为 2 秒或更长的延迟就是"漫长的等待"，在等待 10 秒后（我认为会更早）会切换到其他事情。如果你想留住用户的注意力，你的应用程序必须响应迅速。


-----------------------------------------------------------------------------
交互类别      人类感知                                          响应时间
------------- -----------------------------------------------  --------------
快速          几乎察觉不到延迟                                  100ms--200ms

交互式        快速，但还不足以被描述为"快"                      300ms--500ms
                
暂停          不够快，但仍感觉有响应                            500ms--1 sec
               
等待          由于场景工作量大，不够快                          1 sec--3 sec
               
漫长等待      不再感觉有响应                                    2 sec--5 sec

受迫等待      保留给不可避免的长时/复杂场景                     5 sec--10 sec
               
长时运行      用户可能会在操作期间切换到其他事情               10 sec--30 sec

------------------------------------------------------------------------------

表：人机交互类别。*来源：Microsoft Windows Blogs*。[^11]

应用程序性能可能将你的客户推向竞争对手的产品。通过强调性能，你可以为你的产品提供竞争优势。

有时，快速的工具会找到其最初设计之外的应用场景。例如，虚幻引擎（Unreal）和 Unity 等游戏引擎被用于建筑、3D 可视化、电影制作等领域。正因为游戏引擎性能出色，它们自然成为需要 2D 和 3D 渲染、物理模拟、碰撞检测、声音、动画等功能的应用程序的首选。

> "快速的工具不仅能让用户更快地完成任务；它们还让用户能够以全新的方式完成全新类型的任务。" ——Nelson Elhage 在其博客中写道。[^1]

在开始性能相关工作之前，确保你有充分的理由这样做。纯粹为了优化而优化是无意义的，如果它不能为你的产品带来价值的话。[^10] 有意识的性能工程始于明确定义的性能目标。清楚地了解你想要达到什么目标，并为这项工作提供理由。建立你用来衡量成功的指标。

既然我们已经谈到了性能工程的价值，接下来让我们揭示它的组成。当你试图改善一个程序的性能时，你需要找到问题（性能分析，performance analysis），然后加以改进（调优，tuning），这与常规的调试活动非常相似。接下来我们将讨论这些内容。

[^12]: 20 世纪 90 年代末至 21 世纪初，个人计算机主导计算设备市场的时期。
[^4]: 2024 年，Meta 主要使用自有机房，而 Netflix 使用 AWS 公有云。
[^5]: 2020 年全球云服务支出 - [https://www.srgresearch.com/articles/2020-the-year-that-cloud-service-revenues-finally-dwarfed-enterprise-spending-on-data-centers](https://www.srgresearch.com/articles/2020-the-year-that-cloud-service-revenues-finally-dwarfed-enterprise-spending-on-data-centers)
[^6]: 2024 年全球云服务支出 - [https://www.gartner.com/en/newsroom/press-releases/2024-05-20-gartner-forecasts-worldwide-public-cloud-end-user-spending-to-surpass-675-billion-in-2024](https://www.gartner.com/en/newsroom/press-releases/2024-05-20-gartner-forecasts-worldwide-public-cloud-end-user-spending-to-surpass-675-billion-in-2024)

[^1]: N. Elhage 关于软件性能的思考 - [https://blog.nelhage.com/post/reflections-on-performance/](https://blog.nelhage.com/post/reflections-on-performance/)
[^2]: Visual C++ IntelliSense - [https://docs.microsoft.com/en-us/visualstudio/ide/visual-cpp-intellisense](https://docs.microsoft.com/en-us/visualstudio/ide/visual-cpp-intellisense)
[^3]: Google I/O '08 主题演讲（Marissa Mayer） - [https://www.youtube.com/watch?v=6x0cAzQ7PVs](https://www.youtube.com/watch?v=6x0cAzQ7PVs)
[^8]: Stoyan Stefanov 的幻灯片 - [https://www.slideshare.net/stoyan/dont-make-me-wait-or-building-highperformance-web-applications](https://www.slideshare.net/stoyan/dont-make-me-wait-or-building-highperformance-web-applications)
[^9]: 事实上，在毫秒级内解析整个代码库是不可能的。IntelliSense 实际上只重建已更改部分的 AST（抽象语法树）。关于 Microsoft 团队如何实现这一点的更多细节，请观看视频：[https://channel9.msdn.com/Blogs/Seth-Juarez/Anders-Hejlsberg-on-Modern-Compiler-Construction](https://channel9.msdn.com/Blogs/Seth-Juarez/Anders-Hejlsberg-on-Modern-Compiler-Construction)
[^10]: 除非你只是想练习性能优化，这当然也是可以的。
[^11]: Microsoft Windows Blogs - [https://blogs.windows.com/windowsdeveloper/2023/05/26/delivering-delightful-performance-for-more-than-one-billion-users-worldwide/](https://blogs.windows.com/windowsdeveloper/2023/05/26/delivering-delightful-performance-for-more-than-one-billion-users-worldwide/)
