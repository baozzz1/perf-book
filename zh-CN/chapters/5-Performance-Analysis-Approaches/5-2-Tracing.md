## 跟踪（Tracing）

跟踪在概念上与插桩非常相似，但又略有不同。代码插桩假设用户可以完全访问其应用程序的源代码。另一方面，跟踪依赖于已有的插桩。例如，`strace` 工具使我们能够跟踪系统调用（system calls），可以被视为对 Linux 内核的插桩。Intel 处理器跟踪（Intel Processor Traces，Intel PT，参见附录 C）使你能够记录处理器执行的指令，可以被视为对 CPU 的插桩。跟踪可以从预先进行了适当插桩的组件中获取，且这些组件无需更改。跟踪通常用作黑盒（black-box）方法，用户无法修改应用程序的代码，但仍想深入了解程序正在执行什么。

代码清单 strace 提供了一个使用 Linux `strace` 工具跟踪系统调用的示例，显示了运行 `git status` 命令时输出的前几行。通过使用 `strace` 跟踪系统调用，可以知道每个系统调用的时间戳（最左列）、退出状态（`=` 号后面）以及每个系统调用的持续时间（尖括号中）。

代码清单：使用 strace 跟踪系统调用。

```bash
$ strace -tt -T -- git status
17:46:16.798861 execve("/usr/bin/git", ["git", "status"], 0x7ffe705dcd78
                  /* 75 vars */) = 0 <0.000300>
17:46:16.799493 brk(NULL)               = 0x55f81d929000 <0.000062>
17:46:16.799692 access("/etc/ld.so.nohwcap", F_OK) = -1 ENOENT
                  (No such file or directory) <0.000063>
17:46:16.799863 access("/etc/ld.so.preload", R_OK) = -1 ENOENT
                  (No such file or directory) <0.000074>
17:46:16.800032 openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
                  <0.000072>
17:46:16.800255 fstat(3, {st_mode=S_IFREG|0644, st_size=144852, ...}) = 0
                  <0.000058>
17:46:16.800408 mmap(NULL, 144852, PROT_READ, MAP_PRIVATE, 3, 0)
                  = 0x7f6ea7e48000 <0.000066>
17:46:16.800619 close(3)                = 0 <0.000123>
...
```
跟踪的开销取决于我们具体要跟踪什么。例如，如果我们跟踪一个很少进行系统调用的程序，在 `strace` 下运行它的开销接近于零。另一方面，如果我们跟踪一个严重依赖系统调用的程序，开销可能会非常大，例如 100 倍。[^1] 此外，由于跟踪不会跳过任何样本，可能会生成大量数据。为了应对这一问题，跟踪工具提供了过滤器，使你能够将数据收集限制在特定的时间片或特定的代码段。

与插桩类似，跟踪可以用于探索系统中的异常（anomalies）。例如，你可能想确定应用程序在 10 秒无响应期间发生了什么。正如你将在后面看到的，采样方法并非为此设计，但通过跟踪，你可以了解是什么导致程序无响应。例如，使用 Intel PT，你可以重建程序的控制流（control flow），并精确知道执行了哪些指令。

跟踪也非常适合调试。其底层特性支持基于记录的跟踪进行"记录与回放"（record and replay）的用例。其中一个这样的工具是 Mozilla [rr](https://rr-project.org/)[^2] 调试器，它可以对进程进行记录和回放，支持向后单步执行，以及更多功能。大多数跟踪工具能够为事件添加时间戳，这使我们能够发现与那段时间内发生的外部事件之间的相关性。也就是说，当我们观察到程序中的故障（glitch）时，我们可以查看应用程序的跟踪记录，并将此故障与那段时间内整个系统正在发生的情况关联起来。

[^1]: B. Gregg 关于 `strace` 的文章 - [http://www.brendangregg.com/blog/2014-05-11/strace-wow-much-syscall.html](http://www.brendangregg.com/blog/2014-05-11/strace-wow-much-syscall.html)

[^2]: Mozilla rr 调试器 - [https://rr-project.org/](https://rr-project.org/)。
