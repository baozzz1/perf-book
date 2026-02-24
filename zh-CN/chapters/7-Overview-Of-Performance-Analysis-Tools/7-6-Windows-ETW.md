## Windows 事件追踪

Microsoft 开发了一种系统级追踪机制，名为 Windows 事件追踪（Event Tracing for Windows，ETW）。它最初旨在帮助设备驱动程序开发人员，但后来也被用于分析通用应用程序。ETW 适用于所有受支持的 Windows 平台（x86 和 ARM），并提供相应的平台相关安装包。ETW 在用户代码和内核代码中以完整调用栈追踪支持记录结构化事件，使你能够观察运行中系统的软件动态（software dynamics），并解决许多具有挑战性的性能问题。

### 如何配置 {.unlisted .unnumbered}

自 Windows 10 起，无需任何额外下载即可使用 `WPR.exe` 记录 ETW 数据。但要启用系统范围的性能分析，你必须是管理员并启用 `SeSystemProfilePrivilege`。\underline{W}indows \underline{P}erformance \underline{R}ecorder（Windows 性能记录器）工具支持一组适用于常见性能问题的内置记录配置文件。你可以通过编写带有 `.wprp` 扩展名的自定义性能记录器配置文件 xml 来定制记录需求。

如果不仅要记录还要查看已记录的 ETW 数据，则需要安装 Windows Performance Toolkit（WPT）。可以从 Windows SDK[^1] 或 ADK[^2] 下载页面下载。Windows SDK 体积庞大；不一定需要其所有部分。在我们的情况下，只需勾选 Windows Performance Toolkit 的复选框即可。允许将 WPT 作为自己应用程序的一部分进行重新分发。

### 可以用它做什么 {.unlisted .unnumbered}

- 以可配置的 CPU 采样率（从 125 微秒到 10 秒）识别热点。默认值为 1 毫秒，运行时开销约为 5--10%。
- 确定某个线程被阻塞的原因及持续时间（例如，延迟的事件信号、不必要的线程睡眠等）。
- 检查磁盘处理读写请求的速度，并发现是什么发起了这些操作。
- 检查文件访问性能和模式（包括导致没有磁盘 IO 的缓存读写）。
- 追踪 TCP/IP 协议栈，以查看数据包如何在网络接口和计算机之间流动。

以上所有项目均以可配置的调用栈追踪方式（内核和用户模式调用栈合并）针对所有进程进行系统范围记录。还可以添加自己的 ETW 提供者，以将系统范围的追踪与应用程序行为关联起来。可以通过插桩代码来扩展收集的数据量。例如，可以在源代码的函数中注入进入/退出 ETW 追踪钩子，以测量某个函数的执行频率。

### 不能用它做什么 {.unlisted .unnumbered}

ETW 追踪不适用于检查 CPU 微架构瓶颈。为此，请使用特定于供应商的工具，如 Intel VTune、AMD uProf、Apple Instruments 等。

ETW 追踪可以在系统级别捕获所有进程的动态，但可能会产生大量数据。例如，捕获线程上下文切换数据以观察各种等待和延迟，每分钟轻易就能产生 1--2 GB 数据。因此，在不覆盖之前存储的追踪数据的情况下，长时间记录高容量事件是不现实的。

如果想进一步了解 ETW，附录 D 中有更详细的讨论。我们探讨了用于记录和分析 ETW 的工具，并展示了一个调试程序启动缓慢的案例研究。

[^1]: Windows SDK Downloads - [https://developer.microsoft.com/en-us/windows/downloads/sdk-archive/](https://developer.microsoft.com/en-us/windows/downloads/sdk-archive/)
[^2]: Windows ADK Downloads - [https://learn.microsoft.com/en-us/windows-hardware/get-started/adk-install#other-adk-downloads](https://learn.microsoft.com/en-us/windows-hardware/get-started/adk-install#other-adk-downloads)
