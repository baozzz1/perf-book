## 显式内存预取（Explicit Memory Prefetching）

到目前为止，您应该已经知道，无法从缓存中解决的内存访问通常非常昂贵。现代 CPU 非常努力地通过预测程序将来会访问哪些内存位置并提前预取来降低缓存缺失的惩罚。如果在程序需要某个内存位置时它不在缓存中，我们将不得不承受缓存缺失惩罚，因为我们必须去 DRAM 获取数据。但如果 CPU 设法及时将该内存位置带入缓存，或者请求已被预测且数据正在传输中，那么缓存缺失的惩罚将大大降低。

现代 CPU 有两种解决该问题的自动机制：硬件预取（hardware prefetching）和乱序执行（OOO execution）。硬件预取器通过对重复的内存访问模式发起预取请求来帮助隐藏内存访问延迟。OOO 引擎向前查看 `N` 条指令，提前发出加载指令，以使后续需要该数据的指令能够顺畅执行。

当数据访问模式过于复杂而难以预测时，硬件预取器会失效。软件开发者对此无能为力，因为我们无法控制这个单元的行为。另一方面，OOO 引擎并不像硬件预取那样尝试预测将来需要的内存位置。因此，它成功与否的唯一衡量标准是它通过提前调度加载能够隐藏多少延迟。

考虑代码清单 MemPrefetch1 中的一小段代码，其中 `arr` 是一个包含一百万个整数的数组。索引 `idx` 被赋予一个随机值，立即用于访问 `arr` 中的某个位置，由于是随机的，该位置几乎肯定会缓存缺失。硬件预取器无法预测它，因为每次加载都会到达内存中完全不同的位置。从内存位置的地址已知（由函数 `random_distribution` 返回）到该内存位置的值被需要（调用 `doSomeExtensiveComputation`）之间的时间间隔称为*预取窗口（prefetching window）*。在这个示例中，OOO 引擎没有机会提前发出加载，因为预取窗口非常小。这导致内存访问 `arr[idx]` 的延迟成为执行循环时的关键路径，如图 SWmemprefetch1 所示。程序在没有取得任何进展的情况下等待值返回（图中的阴影矩形）。

您可能会想："但下一次迭代应该会推测性地并行开始执行"。这是真的，确实反映在图 SWmemprefetch1 中。`doSomeExtensiveComputation` 函数需要大量工作，当执行接近第一次迭代结束时，CPU 推测性地开始执行下一次迭代的指令。这在迭代之间创建了正向的执行重叠。事实上，我们呈现了一个乐观的场景，其中处理器能够生成下一个随机数并与前一次迭代并行发出加载。然而，CPU 无法完全隐藏加载的延迟，因为它无法超前当前迭代足够远来足够早地发出加载。也许未来的处理器会有更强大的 OOO 引擎，但目前，有些情况需要程序员的介入。

代码清单：随机数作为后续加载的索引。

```cpp
for (int i = 0; i < N; ++i) {
  size_t idx = random_distribution(generator);
  int x = arr[idx]; // cache miss
  doSomeExtensiveComputation(x);
}
```
![显示加载延迟处于关键路径上的执行时间线。](../../../img/memory-access-opts/SWmemprefetch1.png)

<p align="center"><em>显示加载延迟处于关键路径上的执行时间线。</em></p>


幸运的是，这并非死路一条，有一种方法可以通过将加载与 `doSomeExtensiveComputation` 的执行完全重叠来加速此代码，从而隐藏缓存缺失的延迟。我们可以通过称为*软件流水线（software pipelining）*和*显式内存预取（explicit memory prefetching）*的技术来实现这一点。代码清单 MemPrefetch2 展示了这个想法的实现。我们对随机数的生成进行流水线处理，并开始预取下一次迭代的内存位置，与 `doSomeExtensiveComputation` 并行进行。

代码清单：利用显式软件内存预取提示。

```cpp
size_t idx = random_distribution(generator);
for (int i = 0; i < N; ++i) {
  int x = arr[idx]; 
  idx = random_distribution(generator);
  // prefetch the element for the next iteration
  __builtin_prefetch(&arr[idx]);
  doSomeExtensiveComputation(x);
}
```
此转换的图形说明如图 SWmemprefetch2 所示。我们利用软件流水线为下一次迭代生成随机数。换句话说，在迭代 `M` 上，我们生成将在迭代 `M+1` 上消费的随机数。这使我们能够提前发出内存请求，因为我们已经知道数组中的下一个索引。这种转换使我们的预取窗口大得多，并完全隐藏了缓存缺失的延迟。在迭代 `M+1` 上，实际加载很有可能命中缓存，因为它在迭代 `M` 上已经被预取了。

![通过将缓存缺失延迟与其他执行重叠来隐藏它。](../../../img/memory-access-opts/SWmemprefetch2.png)

<p align="center"><em>通过将缓存缺失延迟与其他执行重叠来隐藏它。</em></p>


注意 [`__builtin_prefetch`](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html)[^4] 的使用，这是开发者可以用来显式请求 CPU 预取特定内存位置的特殊提示。另一个选项是使用编译器内部函数（compiler intrinsics）。在 x86 平台上有 `_mm_prefetch` 内部函数，在 ARM 平台上有 `__pld` 内部函数。编译器将为 x86 生成 `PREFETCH` 指令，为 ARM 生成 `pld` 指令。

有些情况下无法进行软件内存预取。例如，在遍历链表时，预取窗口很小，不可能隐藏指针追踪的延迟。

在代码清单 MemPrefetch2 中，我们看到了为下一次迭代预取的示例，但您也可能经常遇到需要为 2、4、8 甚至更多次迭代预取的情况。代码清单 MemPrefetch3 中的代码就是这样一种可能受益的情况。它展示了填充图的边的典型代码。如果图非常稀疏且有很多顶点，访问 `this->out_neighbors` 和 `this->in_neighbors` 向量很可能频繁缓存缺失。这是因为每条边很可能连接到当前不在缓存中的新顶点。

这段代码与前面的示例不同，因为每次迭代没有大量计算，因此缓存缺失的惩罚可能主导每次迭代的延迟。但我们可以利用这样一个事实：我们知道将来会访问所有元素。`edges` 向量的元素是顺序访问的，因此可能会被硬件预取器及时带入 L1 缓存。我们的目标是将缓存缺失的延迟与执行足够多的迭代重叠，以完全隐藏它。

作为一般规则，预取提示必须足够提前插入，以便在加载值被用于其他计算时，它已经在缓存中。但也不应该插入得太早，因为它可能用长时间不使用的数据污染缓存。注意，在代码清单 MemPrefetch3 中，`lookAhead` 是一个模板参数，使程序员能够尝试不同的值并查看哪个给出最佳性能。更高级的用户可以尝试使用 [timed_lbr] 中描述的方法估算预取窗口；在 Easyperf 博客上可以找到使用此方法的示例。[^5]

代码清单：为接下来 8 次迭代进行软件预取的示例。

```cpp
template <int lookAhead = 8>
void Graph::update(const std::vector<Edge>& edges) {
  for(int i = 0; i + lookAhead < edges.size(); i++) {
    VertexID v = edges[i].from;
    VertexID u = edges[i].to;
    this->out_neighbors[u].push_back(v);
    this->in_neighbors[v].push_back(u);

    // prefetch elements for future iterations
    VertexID v_next = edges[i + lookAhead].from;
    VertexID u_next = edges[i + lookAhead].to;
    __builtin_prefetch(this->out_neighbors.data() + v_next);
    __builtin_prefetch(this->in_neighbors.data()  + u_next);
  }
  // process the remainder of the vector `edges` ...
}
```
显式内存预取最常用于循环中，但您也可以将这些提示插入父函数中；同样，这一切都取决于可用的预取窗口。

这种技术是一种强大的武器，但应极其谨慎地使用，因为不容易做对。首先，显式内存预取不是可移植的，这意味着如果它在一个平台上带来性能提升，并不保证在另一个平台上有类似的加速效果。它非常依赖于具体实现，平台不需要遵从这些提示。在这种情况下，它很可能会降低性能。我的建议是使用所有可用工具验证影响是正面的。不仅要检查性能数字，还要确保缓存缺失次数（特别是 L3）有所下降。一旦变更提交到代码库，请在运行应用程序的所有平台上监控性能，因为它可能对周围代码的变化非常敏感。如果收益不超过潜在的维护负担，请考虑放弃这个想法。

对于一些复杂的场景，确保代码预取了正确的内存位置。当循环的当前迭代依赖于前一次迭代时，例如存在 `continue` 语句或下一个要处理的元素受 `if` 条件保护时，这可能会变得棘手。在这种情况下，我的建议是为代码添加工具测试来验证预取提示的准确性。因为当使用不当时，它可能通过驱逐其他有用数据来降低缓存性能。

最后，显式预取会增加代码大小，并给 CPU 前端（CPU Frontend）增加压力。预取提示只是一个进入内存子系统但没有目标寄存器的假加载。就像任何其他指令一样，它消耗 CPU 资源。极其谨慎地应用它，因为当使用错误时，它可能使程序性能变差。

[^4]: GCC 内建函数 - [https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html).
[^5]: "使用 Linux perf 精确计时机器代码" - [https://easyperf.net/blog/2019/04/03/Precise-timing-of-machine-code-with-Linux-perf#application-estimating-prefetch-window](https://easyperf.net/blog/2019/04/03/Precise-timing-of-machine-code-with-Linux-perf#application-estimating-prefetch-window).
