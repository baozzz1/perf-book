## 用条件选择替换分支（Replace Branches with Selection）

某些分支可以通过执行分支的两个部分，然后选择正确的结果来有效消除。代码清单 ReplaceBranchesWithSelection 展示了这种转换可能带来收益的示例。如果 TMA 显示 `if (cond)` 分支具有非常高的预测失误次数，可以尝试通过右侧所示的转换来消除该分支。

代码清单：用条件选择替换分支。

```cpp
int a;                                             int x = computeX();
if (cond) { /* frequently mispredicted */   =>     int y = computeY();
  a = computeX();                                  int a = cond ? x : y;
} else {                                           foo(a);
  a = computeY();
}
foo(a);
```
对于右侧的代码，编译器可以替换来自三元运算符的分支，转而生成 `CMOV`（条件移动）x86 指令。`CMOVcc` 指令检查 `EFLAGS` 寄存器中一个或多个状态标志位（`CF, OF, PF, SF` 和 `ZF`）的状态，若标志位满足指定状态或条件，则执行移动操作。类似的转换可用于浮点数，通过 `FCMOVcc` 和 `VMAXSS/VMINSS` 指令实现。在 ARM ISA 中，有 `CSEL`（条件选择）指令，此外还有 `CSINC`（选择并加一）、`CSNEG`（选择并取反）以及其他若干条件指令。

代码清单：用条件选择替换分支——x86 汇编代码。

```bash
# original version              # branchless version
400504: test ebx,ebx            400537: mov eax,0x0
400506: je 400514               40053c: call <computeX> # compute x; a = x
400508: mov eax,0x0             400541: mov ebp,eax     # ebp = x
40050d: call <computeX>    =>   400543: mov eax,0x0
400512: jmp 40051e              400548: call <computeY> # compute y; a = y
400514: mov eax,0x0             40054d: test ebx,ebx    # test cond
400519: call <computeY>         40054f: cmovne eax,ebp  # override a with x if needed
40051e: mov edi,eax             400552: mov edi,eax
400521: call <foo>              400554: call <foo>
```
代码清单 ReplaceBranchesWithSelectionAsm 展示了原始版本和无分支版本的汇编代码。与原始版本相比，无分支版本不包含跳转指令。然而，无分支版本会独立计算 `x` 和 `y` 的值，然后选择其中一个并丢弃另一个。虽然这种转换消除了分支预测失误的惩罚，但它做了比原始代码更多的工作。

我们已知原始版本（左侧）中的分支难以预测，这正是尝试无分支版本的动机。在此示例中，这种改变带来的性能提升取决于 `computeX` 和 `computeY` 函数的特性。如果函数很小[^1]且编译器能内联它们，条件选择可能带来明显的性能收益。如果函数很大[^2]，承受分支预测失误的代价可能比同时执行 `computeX` 和 `computeY` 更划算。最终，性能测量结果决定哪个版本更优。

再看一遍代码清单 ReplaceBranchesWithSelectionAsm。在左侧，处理器可以预测例如 `je 400514` 分支将被执行，投机性地调用 `computeY`，并开始运行函数 `foo` 中的代码。请记住，分支预测在我们知道分支实际结果的许多周期之前就已发生。当我们开始解析该分支时，尽管仍是投机状态，可能已经执行了 `foo` 函数的一半。若预测正确，节省了大量周期；若预测错误，必须接受惩罚并从正确路径重新开始。在后一种情况下，已完成的那部分 `foo` 执行毫无益处，全部必须丢弃。若预测失误频繁发生，恢复惩罚会超过投机执行带来的收益。

而使用条件选择则不同。没有分支，处理器无需进行投机执行，可以并行执行 `computeX` 和 `computeY`。但是，在计算出 `CMOVNE` 指令的结果之前，无法开始运行 `foo` 中的代码，因为 `foo` 使用该结果作为参数（数据依赖）。使用条件选择指令时，将控制流依赖（control flow dependency）转换为数据流依赖（data flow dependency）。

总结而言，对于执行简单操作的小型 `if-else` 语句，条件选择比分支更高效，但前提是该分支难以预测。因此，不要强制编译器为每个条件语句生成条件选择。对于总能被正确预测的条件语句，使用分支指令可能是最优选择，因为它允许处理器（正确地）进行投机执行，提前运行。同时不要忘记测量改动的实际影响。

在没有性能分析数据的情况下，编译器无法了解预测失误率。因此，默认情况下它们通常倾向于生成分支指令。编译器在使用条件选择时比较保守，即便在简单情况下也可能抗拒生成 `CMOV` 指令。权衡本就复杂，在没有运行时数据的情况下很难做出正确决策。[^4] 从 Clang-17 开始，编译器针对 x86 目标支持 `__builtin_unpredictable` 提示，用于告知编译器某个分支条件不可预测。这有助于影响编译器的决策，但不保证一定会生成 `CMOV` 指令。以下是 `__builtin_unpredictable` 的使用示例：

```cpp
int a;
if (__builtin_unpredictable(cond)) { 
  a = computeX();
} else {
  a = computeY();
}
```

[^1]: 仅少量指令，可在数个周期内完成。
[^2]: 超过二十条指令，需要超过二十个周期。
[^4]: 基于硬件的 PGO（参见 [secPGO]）将在这方面带来重大进步。
