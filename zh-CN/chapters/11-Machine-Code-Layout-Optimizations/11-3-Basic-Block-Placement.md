## 基本块放置（Basic Block Placement）

假设程序中有一段热路径（hot path），其中穿插着一些错误处理代码（`coldFunc`）：

```cpp
// hot path
if (cond)
  coldFunc();
// hot path again
```
图 BBLayout 展示了这段代码的两种可能的物理布局。图 BB_default 是大多数编译器在没有提示的情况下默认生成的布局。如果我们反转条件 `cond` 并将热代码作为直通（fall through）路径，就可以得到图 BB_better 所示的布局。

<div id="fig:BBLayout">
![default layout](../../img/cpu_fe_opts/BBLayout_Default.png)
![improved layout](../../img/cpu_fe_opts/BBLayout_Better.png)

上述代码片段的两种机器码布局版本。
</div>

哪种布局更好？这取决于 `cond` 通常是真还是假。如果 `cond` 通常为真，那么默认布局更好，否则我们将执行两次跳转而不是一次。此外，如果 `coldFunc` 是一个相对较小的函数，我们希望将其内联。然而，在这个特定示例中，我们知道 `coldFunc` 是一个错误处理函数，很少被执行。选择图 BB_better 的布局，可以在热代码片段之间保持直通，并将已采用分支（taken branch）转换为未采用分支（not taken branch）。

图 BB_better 的布局性能更好有几个原因。首先，图 BB_better 的布局更好地利用了指令缓存和 $\mu$op 缓存（DSB，参见 [uarchFE]）。所有热代码连续存放，不存在缓存行碎片化：L1 I-cache 中的所有缓存行都被热代码使用。$\mu$op 缓存也是如此，因为它同样基于底层代码布局进行缓存。其次，已采用分支对于取指单元（fetch unit）来说也更加昂贵。CPU 的前端以连续对齐的字节块（通常是 16、32 或 64 字节，取决于架构）来获取指令。对于每一个已采用分支，跳转指令之后、分支目标之前的取指块中的字节都是未使用的，这降低了最大有效取指吞吐量。最后，在某些架构上，未采用分支从根本上比已采用分支开销更小。例如，Intel Skylake CPU 每个周期可以执行两个未采用分支，但每两个周期只能执行一个已采用分支。[^2]

要建议编译器生成改进版的机器码布局，可以使用 `[[likely]]` 和 `[[unlikely]]` 属性提供提示，这些属性自 C++20 起可用。使用该提示的代码如下：

```cpp
// hot path
if (cond) [[unlikely]] 
  coldFunc();
// hot path again
```

上述代码中，`[[unlikely]]` 提示将告知编译器 `cond` 不太可能为真，因此编译器应相应地调整代码布局。在 C++20 之前，开发者可以使用 [`__builtin_expect`](https://llvm.org/docs/BranchWeightMetadata.html#builtin-expect)[^3] 构造。他们通常会创建 `LIKELY` 包装提示以使代码更易读。例如：

```cpp
#define LIKELY(EXPR)   __builtin_expect((bool)(EXPR), true)
#define UNLIKELY(EXPR) __builtin_expect((bool)(EXPR), false)
// hot path
if (UNLIKELY(cond)) // NOT 
  coldFunc();
// hot path again
```

优化编译器不仅会在遇到"likely/unlikely"提示时改进代码布局，还会在其他地方利用这些信息。例如，当应用了 `[[unlikely]]` 属性时，编译器会阻止内联 `coldFunc`，因为它现在知道该函数不太可能被频繁执行，更有利的做法是将其优化为更小的尺寸，即只保留一个 `CALL` 调用。

如代码清单 BuiltinSwitch 所示，在 switch 语句中插入 `[[likely]]` 属性也是可行的。使用这个提示，编译器可以稍微不同地重排代码，并优化热 switch 以更快地处理 `ADD` 指令。

代码清单：在 switch 语句中使用 likely 属性

```cpp
for (;;) {
  switch (instruction) {
               case NOP: handleNOP(); break;
    [[likely]] case ADD: handleADD(); break;
               case RET: handleRET(); break;
    // handle other instructions
  }
}
```
[^2]: 不过，有一种针对小循环的特殊优化，允许非常小的循环每个周期执行一个已采用分支。
[^3]: 关于 builtin-expect 的更多信息：[https://llvm.org/docs/BranchWeightMetadata.html#builtin-expect](https://llvm.org/docs/BranchWeightMetadata.html#builtin-expect)。
[^10]: C++ 标准 `[[likely]]` 属性：[https://en.cppreference.com/w/cpp/language/attributes/likely](https://en.cppreference.com/w/cpp/language/attributes/likely)。
