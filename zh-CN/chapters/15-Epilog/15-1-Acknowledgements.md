# 致谢（Acknowledgments）

撰写这一章节时，我对以下所有提及的人充满深深的感激之情。能够将这些人积累的知识传递给读者，我深感荣幸。

   <br> 

\setlength\intextsep{0pt}

![](../../img/contributors/MarkDawson_circle.png) 
Mark E. Dawson, Jr. 撰写了 [LowLatency]《低延迟调优技术》和 [ContinuousProfiling]《持续性能分析》两章，并对本书第一版贡献良多。Mark 是高频交易（High-Frequency Trading）行业公认的专家，目前就职于 WH Trading。他还拥有个人博客（[https://www.jabperf.com](https://www.jabperf.com)），专注于低延迟及其他性能优化主题。

   <br> 

![](../../img/contributors/UniversityZaragoza_circle.png) 
来自萨拉戈萨大学（University of Zaragoza）的 Agustín Navarro-Torres、Jesús Alastruey-Benedé、Pablo Ibáñez-Marín 和 Víctor Viñals-Yúfera 合著了 [Sensitivity2LLC]《案例研究：对末级缓存大小的敏感性》一章。他们是计算机科学领域的研究人员，发表了多篇与性能工程相关的学术论文。

   <br> 

![](../../img/contributors/JanWassenberg_circle.png) 
Jan Wassenberg 撰写了 [secIntrinsicLibraries]《内联函数包装库》，并对 [Vectorization]《向量化》和 [SIMD]《SIMD 多处理器》两章提出了大量改进建议。Jan 是 Google DeepMind 的软件工程师，主导 Gemma.cpp 的开发[^1]，并发表了多篇研究论文。其个人主页为 [https://wassenberg.dreamhosters.com](https://wassenberg.dreamhosters.com)。

   <br> 

![](../../img/contributors/SwarupSahoo_circle.png) 
Swarup Sahoo 协助我撰写了 [PmuChapter] 中关于 AMD PMU 特性的内容，并撰写了 [IntelVtuneOverview] 中关于 AMD uProf 的部分。Swarup 是 AMD 的高级开发工程师，专注于面向 HPC（OpenMP、MPI）应用的 uProf 性能分析工具。其 LinkedIn 主页：[https://www.linkedin.com/in/swarupsahoo](https://www.linkedin.com/in/swarupsahoo)。

   <br> 

![](../../img/contributors/AloisKraus_circle.png) 
Alois Kraus 撰写了 [ETW]《Windows 事件追踪》以及附录 D，并开发了 `ETWAnalyzer`——一款用于通过简单查询分析 ETW 文件的命令行工具。他就职于西门子医疗（Siemens Healthineers），专注于大型软件系统的性能研究。其个人主页和博客：[https://aloiskraus.wordpress.com](https://aloiskraus.wordpress.com)。

   <br> 

![](../../img/contributors/MarcoCastorina_circle.png) 
Marco Castorina 撰写了 [Tracy]《专用与混合性能分析器》，展示了使用 Tracy 进行性能分析的方法。Marco 目前就职于 AMD 游戏图形性能团队，专注于 DX12。他还是《Mastering Graphics Programming with Vulkan》一书的合著者。其个人主页：[https://marcocastorina.com](https://marcocastorina.com)。

   <br> 

![](../../img/contributors/LallySingh_circle.png) 
Lally Singh 撰写了 [MarkerAPI] 中关于标记 API（Marker APIs）的内容。Lally 目前就职于 Tesla，此前曾在 Datadog 性能团队、Google 搜索性能团队工作，并有低延迟交易系统和嵌入式实时控制系统的从业经历。Lally 拥有弗吉尼亚理工大学（Virginia Tech）计算机科学博士学位，研究方向为分布式 VR 的可扩展性。

   <br> 

![](../../img/contributors/DickSites_circle.png) 
Richard L. Sites 对本书进行了技术审阅。他是半导体行业的资深专家，职业生涯大部分时间都在硬件与软件的交界处工作，尤其专注于 CPU 与软件之间的性能交互。Richard 发明了现代大多数处理器中的性能计数器（Performance Counters）。他曾就职于 DEC、Adobe、Google 和 Tesla。其个人主页：[https://sites.google.com/site/dicksites](https://sites.google.com/site/dicksites)。

   <br> 

![](../../img/contributors/MattGodbolt_circle.png) 
Matt Godbolt 对本书进行了技术审阅。Matt 是 Compiler Explorer 的创建者，这是一款在软件开发者中极为流行的工具。他是一位热衷于高性能代码的 C++ 开发者，拥有超过二十年的专业经验，涉及计算机游戏编程、系统设计、实时嵌入式系统和高频交易等领域。他同时也是演讲者、博主和播客主播。其个人博客：[https://xania.org](https://xania.org)。

  <br> 

此外，我还要感谢以下人士：来自 Arm 的 Jumana Mundichipparakkal，协助我撰写了 [PmuChapter] 中关于 Arm PMU 特性的内容；Zstandard 的作者 Yann Collet，为 [ThreadCountScalingStudy] 提供了有关 Zstd 内部工作原理的信息；Ciaran McHale，在初稿中发现了大量语法错误；Nick Black，对本书最终版本进行了校对和编辑；以及 Peter Veentjer、Amir Aupov 和 Charles-Francois Natali，提供了多项编辑建议。

我同样感谢整个性能社区提供的无数博文和论文。通过阅读 Travis Downs、Daniel Lemire、Andi Kleen、Agner Fog、Bruce Dawson、Brendan Gregg 等人的博客，我收获良多。我站在巨人的肩膀上，本书的成功不应仅归功于我个人。这本书是我向整个社区致谢和回馈的方式。

特别感谢我的家人，他们足够耐心，包容了我错过的无数个周末郊游和傍晚散步。没有他们的支持，我不可能完成这本书。

本书插图使用 excalidraw.com 创作，封面由 Darya Antonova 设计。封面所用字体为 Bebas Neue 和 Raleway，均在 Open Font License 下授权使用。Bebas Neue 由 Ryoichi Tsunekawa 设计，Raleway 由 Matt McInerney、Pablo Impallari 和 Rodrigo Fuenzalida 设计。

还需提及本书第一版的贡献者。以下仅列出姓名和章节标题，更详细的致谢信息请见第一版。

* Mark E. Dawson, Jr. 撰写了 [secDTLB]《DTLB 优化》、[FeTLB]《ITLB 优化》、[CacheWarm]《缓存预热》、[SysTune]《系统调优》等章节。
* Sridhar Lakshmanamurthy 撰写了 [uarch] 中关于 CPU 微架构的大部分内容。
* Nadav Rotem 协助撰写了 [Vectorization] 向量化章节。
* Clément Grégoire 撰写了 [ISPC] 关于 ISPC 编译器的章节。
* 审阅者：Dick Sites、Wojciech Muła、Thomas Dullien、Matt Fleming、Daniel Lemire、Ahmad Yasin、Michele Adduci、Clément Grégoire、Arun S. Kumar、Surya Narayanan、Alex Blewitt、Nadav Rotem、Alexander Yermolovich、Suchakrapani Datt Sharma、Renat Idrisov、Sean Heelan、Jumana Mundichipparakkal、Todd Lipcon、Rajiv Chauhan、Shay Morag 等。

[^1]: Gemma.cpp，在 CPU 上运行 LLM 推理 - [https://github.com/google/gemma.cpp](https://github.com/google/gemma.cpp)
