# 附录 D. Windows 事件追踪（Event Tracing for Windows）分析

我们在 [ETW] 中介绍了 ETW（Event Tracing for Windows，Windows 事件追踪）。本节将在此基础上继续探讨记录和分析 ETW 的工具。为了展示这些工具的使用方式，我们通过一个调试程序启动缓慢的案例研究来进行说明。

### Tools to Record ETW traces {.unlisted .unnumbered}

以下是可用于捕获 ETW 追踪数据的工具列表：

- `WPR.exe`：命令行录制工具，属于 Windows 10 和 Windows Performance Toolkit 的一部分。
- `WPRUI.exe`：用于录制 ETW 数据的简单图形界面，属于 Windows Performance Toolkit 的一部分。
- `xperf`：wpr 的命令行前身，属于 Windows Performance Toolkit 的一部分。
- `PerfView`：[^3] 一款图形化录制和分析工具，主要关注 .NET 应用程序。这是由 Microsoft 开发的开源应用程序。
- `Performance HUD`：[^7] 一款不太知名但功能强大的 GUI 工具，可通过实时 ETW 录制所有未平衡资源分配来追踪 UI 延迟以及用户/句柄泄漏，并实时显示泄漏/阻塞调用栈追踪。
- `ETWController`：[^4] 一款录制工具，除 ETW 数据外还能录制键盘输入和截图。这款由 Alois Kraus 开发的开源应用程序还支持在两台机器上同时进行分布式性能分析。
- `UIForETW`：[^6] 由 Bruce Dawson 开发的开源应用程序，是 `xperf` 的封装工具，提供专门针对 Google Chrome 问题录制数据的选项，也可录制键盘和鼠标输入。

### Tools to View and Analyze ETW traces {.unlisted .unnumbered}

- `Windows Performance Analyzer`（WPA）：查看 ETW 数据最强大的 UI 工具。WPA 可以可视化并叠加展示磁盘、CPU、GPU、网络、内存、进程等多种数据源，帮助全面了解系统行为。虽然 UI 功能强大，但对初学者而言可能较为复杂。WPA 支持插件以处理来自其他数据源（不仅限于 ETW 追踪）的数据，可以导入由 Linux perf、LTTNG、Perfetto 等工具生成的 Linux/Android[^8] 性能分析数据，以及多种日志文件格式：dmesg、Cloud-Init、WaLinuxAgent 和 AndroidLogcat。
- `ETWAnalyzer`：[^5] 读取 ETW 数据并生成聚合摘要 JSON 文件，可在命令行进行查询、过滤、排序，或导出为 CSV 文件。
- `PerfView`：主要用于排查 .NET 应用程序问题。针对垃圾回收（Garbage Collection）和 JIT 编译所触发的 ETW 事件会被解析，并以报告或 CSV 数据的形式方便地呈现。

### Case Study - Slow Program Start {.unlisted .unnumbered}

现在我们来看一个使用 ETWController 捕获 ETW 追踪数据并用 WPA 可视化分析的示例。

**问题描述**：在 Windows 资源管理器中双击一个已下载的可执行文件时，启动过程存在明显延迟。似乎有什么原因导致进程启动变慢。这可能是什么原因？磁盘慢？

#### Setup {.unlisted .unnumbered}

- 下载 ETWController 以录制 ETW 数据和截图。
- 下载最新的 Windows 11 Performance Toolkit[^1] 以便用 WPA 查看数据。确保较新的 Win 11 版 `WPR.exe` 在路径中排在前面，方法是在系统环境变量对话框中将 WPT 安装文件夹移至 `C:\\Windows\\system32` 之前。配置完成后应如下所示：

```
C> where wpr 
C:\Program Files (x86)\Windows Kits\10\Windows Performance Toolkit\WPR.exe
C:\Windows\System32\WPR.exe
```

#### Capture traces {.unlisted .unnumbered}

- 启动 ETWController。
- 选择 CSwitch 配置文件以追踪线程等待时间及其他默认录制设置。确保勾选*录制鼠标点击*（Record mouse clicks）和*循环截图*（Take cyclic screenshots）复选框（见图 ETWController_Dialog），这样之后可以借助截图导航至慢速位置。
- 从互联网下载一个应用程序，如需要则解压。使用什么程序无关紧要，目标是观察启动时的延迟。
- 按下*开始录制*（Start Recording）按钮开始性能分析。
- 双击可执行文件启动程序。
- 程序启动后，按下*停止录制*（Stop Recording）按钮停止性能分析。

![Starting ETW collection with ETWController UI.](../../img/perf-tools/ETWController_Dialog.png)

第一次停止性能分析时会稍慢一些，因为需要为所有托管代码生成程序调试数据库文件（PDB），这是一次性操作。分析到达已停止（Stopped）状态后，可以按下*在 WPA 中打开*（Open in WPA）按钮，使用 ETWController 提供的配置文件将 ETL 文件加载到 Windows Performance Analyzer 中。CSwitch 配置文件会生成大量数据，存储在 4 GB 环形缓冲区中，允许录制 1-2 分钟，之后最旧的事件会被覆盖。有时在正确的时间点停止录制需要一点技巧。如果问题偶发，可以保持录制持续数小时，当某个事件（如文件中出现特定日志条目）触发时再停止，这可以通过轮询脚本来检测。

Windows 支持事件日志（Event Log）和性能计数器（Performance Counter）触发器，当性能计数器达到阈值或特定事件写入事件日志时可以启动脚本。如果需要更复杂的停止触发条件，可以使用 PerfView；它允许定义性能计数器阈值，该阈值必须达到并持续 `N` 秒才会停止性能分析，从而避免随机峰值触发误报。

#### Analysis in WPA {.unlisted .unnumbered}

  <br>

图 WPA_MainView 展示了在 Windows Performance Analyzer（WPA）中打开的已录制 ETW 数据。WPA 视图纵向分为三个部分：*CPU 使用率（采样）*（CPU Usage (Sampled)）、*通用事件*（Generic Events）和*CPU 使用率（精确）*（CPU Usage (Precise)）。为了理解它们之间的区别，我们来深入分析。上部图表*CPU 使用率（采样）*用于识别 CPU 时间花在何处，数据通过以固定时间间隔对所有运行线程进行采样来收集。该*CPU 使用率（采样）*图表与其他性能分析工具中的*热点*（Hotspots）视图非常类似。

![Windows Performance Analyzer: root causing a slow start of an application.](../../img/perf-tools/WPA_MainView.png)

接下来是*通用事件*（Generic Events）视图，显示鼠标点击和已捕获截图等事件。请记住，我们在 ETWController 窗口中启用了对这些事件的拦截。由于事件被放置在时间线上，可以方便地将 UI 操作与系统响应关联起来。

底部图表*CPU 使用率（精确）*（CPU Usage (Precise)）使用的数据源与*采样*视图不同。采样数据只捕获正在运行的线程，而 CSwitch 收集还考虑了进程未运行的时间间隔。*CPU 使用率（精确）*视图的数据来自 Windows 线程调度器（Thread Scheduler）。该图表追踪线程在哪个 CPU 上运行了多长时间（CPU 使用率）、在内核调用中阻塞了多长时间（等待时间 Waits）、以何种优先级运行，以及线程等待 CPU 空闲了多长时间（就绪时间 Ready Time）等。因此，*CPU 使用率（精确）*视图不显示 CPU 消耗最多的前几名，但对于理解某个进程被阻塞了多长时间以及*原因*非常有帮助。

熟悉了 WPA 界面之后，我们来观察图表。首先，可以在时间线上找到 `MouseButton` 事件 63 和 64。ETWController 将收集过程中拍摄的所有截图保存在新创建的文件夹中。性能分析数据本身保存在名为 `SlowProcessStart.etl` 的文件中，并有一个名为 `SlowProcessStart.etl.Screenshots` 的新文件夹。该文件夹包含截图和可在 Web 浏览器中查看的 `Report.html` 文件。每条已录制的键盘/鼠标交互都保存在文件名中包含事件编号的文件中，例如 `Screenshot_63.jpg`。图 ETWController_ClickScreenshot（裁剪版）显示了鼠标双击（事件 63 和 64）。鼠标指针位置标记为绿色方块，如果发生了点击事件则为红色。这样可以轻松识别鼠标点击的时间和位置。

![A mouse click screenshot captured with ETWController.](../../img/perf-tools/ETWController_ClickScreenshot.png)

双击标志着 1.2 秒延迟的开始，此期间我们的应用程序在等待某些操作。在时间戳 `35.1` 处，`explorer.exe` 处于活动状态，尝试启动新应用程序。但之后它几乎没有做什么工作，应用程序也没有启动。取而代之的是，`MsMpEng.exe` 接管了执行直到时间 `35.7`。目前来看，这像是在允许已下载的可执行文件启动之前进行的防病毒扫描。但我们还不能 100% 确定 `MsMpEng.exe` 是否阻塞了新应用程序的启动。

由于我们关注的是延迟，我们对等待时间感兴趣。这些信息在*CPU 使用率（精确）*面板的下拉菜单中选择*等待时间*（Waits）后可以看到。在那里，我们可以找到 `explorer.exe` 等待的进程列表，以柱状图形式显示并与上方面板的时间线对齐。不难发现与*防病毒 - Windows Defender* 对应的长柱，等待时间为 1.068 秒。因此，可以得出结论：应用程序启动延迟是由 Defender 扫描活动造成的。如果深入查看调用栈（此处未展示），可以看到 `CreateProcess` 系统调用在内核中被 `WDFilter.sys`（Windows Defender 过滤驱动程序）延迟。它在扫描完潜在恶意文件内容之前阻止进程启动。防病毒软件可以拦截一切，导致难以预测的性能问题，如果没有 ETW 提供的全面内核视图，这类问题很难诊断。谜题解开了？还没完全解开。

知道 Defender 是问题所在只是第一步。如果再次查看上方面板，可以看到延迟并非完全由忙碌的防病毒扫描引起。`MsMpEng.exe` 进程从时间 `35.1` 到 `35.7` 保持活动，但应用程序在此之后并没有立即启动。从时间 `35.7` 到 `36.2` 还有额外的 0.5 秒延迟，期间 CPU 大部分时间处于空闲状态，没有做任何事情。要找到这一问题的根本原因，需要跨进程追踪线程唤醒历史，这里不展开介绍。最终会发现一个阻塞性的 Web 服务调用 `MpClient.dll!MpClient::CMpSpyNetContext::UpdateSpynetMetrics`，它在等待某个 Microsoft Defender Web 服务响应。如果启用 TCP/IP 或 socket ETW 追踪，还可以找出 Microsoft Defender 正在与哪个远程端点通信。因此，延迟的第二部分是由 `MsMpEng.exe` 进程等待网络响应造成的，这也阻塞了我们的应用程序运行。

这个案例研究只展示了可以用 WPA 有效分析的一类问题，实际上还有许多其他类型的问题。WPA 界面功能丰富，高度可定制。它支持自定义配置文件，以您喜欢的方式配置图表和表格来可视化事件数据。最初，WPA 是为设备驱动程序开发人员开发的，其内置配置文件并不专注于应用程序开发。ETWController 带有自己的配置文件（*Overview.wpaprofile*），可以在*配置文件（Profiles）&rarr; 保存启动配置文件（Save Startup Profile）*下将其设为默认配置文件，以便始终使用性能概览配置文件。

[^1]: Windows SDK Downloads - [https://developer.microsoft.com/en-us/windows/downloads/sdk-archive/](https://developer.microsoft.com/en-us/windows/downloads/sdk-archive/)
[^3]: PerfView - [https://github.com/microsoft/perfview](https://github.com/microsoft/perfview)
[^4]: ETWController - [https://github.com/alois-xx/etwcontroller](https://github.com/alois-xx/etwcontroller)
[^5]: ETWAnalyzer - [https://github.com/Siemens-Healthineers/ETWAnalyzer](https://github.com/Siemens-Healthineers/ETWAnalyzer)
[^6]: UIforETW - [https://github.com/google/UIforETW](https://github.com/google/UIforETW)
[^7]: Performance HUD - [https://www.microsoft.com/en-us/download/100813](https://www.microsoft.com/en-us/download/100813)
[^8]: Microsoft Performance Tools Linux / Android - [https://github.com/microsoft/Microsoft-Performance-Tools-Linux-Android](https://github.com/microsoft/Microsoft-Performance-Tools-Linux-Android)
