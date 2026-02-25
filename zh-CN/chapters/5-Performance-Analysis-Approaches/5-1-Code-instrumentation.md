## 代码插桩（Code Instrumentation）

有史以来最早发明的性能分析方法之一可能就是*代码插桩*。它是一种向程序中插入额外代码以收集特定运行时（runtime）信息的技术。代码清单 CodeInstrumentation 展示了在函数开头插入 `printf` 语句以指示该函数是否被调用的最简单示例。之后，你运行程序并统计输出中出现"foo is called"的次数。也许世界上每个程序员都曾在职业生涯中至少做过一次这样的事。

代码清单：代码插桩

```cpp
int foo(int x) {
+ printf("foo is called\n");
 // function body...
}
```
行首的加号表示该行是新增的，不在原始代码中。通常，插桩代码不应该被推送到代码库中；它只是用于收集所需数据，之后可以删除。

代码清单 CodeInstrumentationHistogram 展示了一个更有趣的代码插桩示例。在这个假设的代码示例中，函数 `findObject` 在地图上搜索具有某些属性 `p` 的对象的坐标。所有对象最终都保证能被找到。函数 `getNewCoords` 在作为参数提供的更大区域内返回新坐标。函数 `findObj` 返回在当前坐标 `c` 处找到正确对象的置信度（confidence level）。如果完全匹配，则停止搜索循环并返回坐标。如果置信度高于 `threshold`，则调用 `zoomIn` 以更精确地定位对象。否则，在 `searchArea` 内获取新坐标，下次继续搜索。

插桩代码由两个类组成：`histogram` 和 `incrementor`。前者跟踪我们感兴趣的变量值及其出现频率，并在程序结束*后*打印直方图（histogram）。后者只是一个将值推入 `histogram` 对象的辅助类。它很简单，可以根据你的具体需求快速调整。[^3]

代码清单：代码插桩

```cpp
+ struct histogram {
+   std::map<uint32_t, std::map<uint32_t, uint64_t>> hist;
+   ~histogram() {
+     for (auto& tripCount : hist)
+       for (auto& zoomCount : tripCount.second)
+         std::cout << "[" << tripCount.first << "]["
+                   << zoomCount.first << "] :  "
+                   << zoomCount.second << "\n";
+   }
+ };
+ histogram h;

+ struct incrementor {
+   uint32_t tripCount = 0;
+   uint32_t zoomCount = 0;
+   ~incrementor() {
+ 	   h.hist[tripCount][zoomCount]++;
+   }
+ };

Coords findObject(const ObjParams& p, Coords searchArea) {
+ incrementor inc;
  Coords c = getNewCoords(searchArea);
  while (true) {
+   inc.tripCount++;
    float match = findObj(p, c);
    if (exactMatch(match))
      return c;
    if (match > threshold) {
      searchArea = zoomIn(searchArea, c);
+     inc.zoomCount++;
    } else {
      c = getNewCoords(searchArea);
    }
  }
  return c;
}
```
在这个假设场景中，我们添加了插桩以了解在找到对象之前 `zoomIn` 的调用频率。变量 `inc.tripCount` 计算循环在退出之前运行的迭代次数，变量 `inc.zoomCount` 计算我们缩小搜索区域（调用 `zoomIn`）的次数。我们始终期望 `inc.zoomCount` 小于或等于 `inc.tripCount`。

函数 `findObject` 被多次调用，输入各不相同。以下是运行插桩程序后可能观察到的输出：

```
// [tripCount][zoomCount]: occurences
[7][6]:  2
[7][5]:  6
[7][4]:  20
[7][3]:  156
[7][2]:  967
[7][1]:  3685
[7][0]:  251004
[6][5]:  2
[6][4]:  7
[6][3]:  39
[6][2]:  300
[6][1]:  1235
[6][0]:  91731
[5][4]:  9
[5][3]:  32
[5][2]:  160
[5][1]:  764
[5][0]:  34142
...
```

方括号中的第一个数字是循环的迭代次数，第二个是在同一循环中调用 `zoomIn` 的次数。冒号后面的数字是该特定数字组合出现的次数。例如，我们观察到两次循环迭代了 7 次并调用了 6 次 `zoomIn`。251004 次循环运行了 7 次迭代且没有 `zoomIn`，以此类推。你可以绘制数据以便更好地可视化，或采用其他统计方法，但我们可以得出的主要结论是：`zoomIn` 并不频繁。
调用 `findObject` 的总次数约为 40 万次；我们可以通过将直方图中所有桶的值相加来计算。如果我们将所有 `zoomCount` 不为零的桶相加，大约得到 1 万次，这就是调用 `zoomIn` 函数的次数。因此，每调用一次 `zoomIn`，就调用 40 次 `findObject` 函数。

本书后面的章节包含许多关于如何将此类信息用于优化的示例。在我们的案例中，我们得出结论：`findObj` 经常无法找到对象。这意味着循环的下一次迭代将尝试使用新坐标在同一搜索区域内找到对象。了解到这一点，我们可以尝试多种优化：1) 并行运行多个搜索，并在任何一个搜索成功时同步；2) 对当前搜索区域预计算某些内容，从而消除 `findObj` 内部的重复工作；3) 编写一个软件流水线（software pipeline），调用 `getNewCoords` 生成下一组所需坐标，并从内存中预取（prefetch）相应的地图位置。本书第 2 部分将更深入地探讨其中一些技术。

代码插桩在需要了解程序执行的特定信息时，能提供非常详细的信息。它允许我们跟踪程序中任何变量的任何信息。使用这种方法通常在优化大型代码片段时能提供最佳洞察，因为你可以使用自顶向下的方法（首先插桩主函数，然后深入到其被调用者）来更好地理解应用程序的行为。代码插桩使开发者能够观察应用程序的架构和流程。这种技术对于处理不熟悉代码库的开发者尤其有帮助。

代码插桩技术在实时场景的性能分析中被广泛使用，例如视频游戏和嵌入式开发。一些性能分析工具（profilers）将插桩与跟踪或采样等其他技术结合使用。我们将在 [Tracy] 中介绍一种名为 Tracy 的混合性能分析工具。

虽然代码插桩在许多情况下功能强大，但它无法提供任何关于代码如何从操作系统或 CPU 角度执行的信息。例如，它无法告诉你进程被调入和调出执行的频率（由操作系统知晓），或发生了多少次分支预测错误（由 CPU 知晓）。插桩代码是应用程序的一部分，与应用程序本身具有相同的权限。它在用户空间（userspace）运行，无法访问内核（kernel）。

这种技术更重要的一个缺点是，每次需要插桩新的内容（比如另一个变量），都需要重新编译。这可能成为一种负担，并增加分析时间。不幸的是，还有其他缺点。由于你通常关注应用程序中的热路径（hot paths），你正在对位于代码性能关键部分的内容进行插桩。在热路径中注入插桩代码很容易导致整体基准测试降速 2 倍。记住不要对插桩程序进行基准测试。通过对代码进行插桩，你改变了程序的行为，因此可能看不到之前看到的相同效果。

以上所有因素都增加了实验之间的时间并消耗了更多开发时间，这就是为什么工程师现在不经常手动插桩其代码的原因。然而，自动化代码插桩仍然被编译器广泛使用。编译器能够自动插桩整个程序（第三方库除外），以收集有关执行的有趣统计数据。自动化插桩最广为人知的用例是代码覆盖率分析（code coverage analysis）和配置文件引导优化（Profile-Guided Optimization，参见 [secPGO]）。

在讨论插桩时，重要的是要提及*二进制插桩*（binary instrumentation）技术。二进制插桩背后的理念与代码插桩类似，但它是在已构建的可执行文件上进行的，而不是在源代码上进行。二进制插桩有两种类型：静态的（提前完成）和动态的（在程序执行时按需插入插桩代码）。动态二进制插桩的主要优点是不需要重新编译和重新链接程序。此外，通过动态插桩，可以将插桩量限制在感兴趣的代码区域，而不是对整个程序进行插桩。

二进制插桩在性能分析和调试中非常有用。最流行的二进制插桩工具之一是 Intel Pin[^1] 工具。Pin 在程序中有趣事件发生时拦截程序的执行，并从程序中的该点开始生成新的插桩代码。这使得收集各种运行时信息成为可能。构建在 Pin 之上最流行的工具之一是 Intel SDE（Software Development Emulator，软件开发模拟器）。[^2] 另一个知名的二进制插桩工具叫做 DynamoRIO。[^4] 以下是可以使用二进制插桩工具收集的一些内容：

* 指令计数和函数调用计数，
* 指令混合分析（instruction mix analysis），
* 拦截函数调用和应用程序中任意指令的执行，
* 内存强度（memory intensity）和内存占用（footprint）（参见 [MemoryIntensityFootprint]）。

与代码插桩一样，二进制插桩只对用户级代码进行插桩，速度可能非常慢。

[^1]: Pin - [https://software.intel.com/en-us/articles/pin-a-dynamic-binary-instrumentation-tool](https://software.intel.com/en-us/articles/pin-a-dynamic-binary-instrumentation-tool)
[^2]: Intel SDE - [https://www.intel.com/content/www/us/en/developer/articles/tool/software-development-emulator.html](https://www.intel.com/content/www/us/en/developer/articles/tool/software-development-emulator.html)
[^3]: 我有一个稍微高级一些的此代码版本，通常会将其复制粘贴到我正在处理的任何项目中，之后再删除。
[^4]: DynamoRIO - [https://github.com/DynamoRIO/dynamorio](https://github.com/DynamoRIO/dynamorio)。它支持 Linux 和 Windows 操作系统，可在 x86 和 ARM 硬件上运行。
