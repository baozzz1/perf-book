## 分支预测错误（Mispredicted Branch）

现代 CPU 会尝试预测条件分支指令（taken 或 not taken）的结果。例如，当处理器看到如下代码时：

```bash
dec eax
jz .zero
# eax is not 0
...
zero:
# eax is 0
```

在上述示例中，`jz` 指令是一个条件分支。为了提高性能，现代处理器每次遇到分支指令时都会尝试猜测其结果。这被称为*推测执行*（Speculative Execution），我们在 [SpeculativeExec] 中已经讨论过。处理器会推测（例如）该分支不会被跳转，并执行对应于 `eax is not 0` 情况的代码。但是，如果猜测错误，就会发生*分支预测错误*（branch misprediction），CPU 需要撤销它最近所做的所有推测工作。

分支预测错误通常会导致 10 到 25 个时钟周期的惩罚（penalty）。首先，所有根据错误预测获取和执行的指令需要从流水线中清除（flush）。之后，一些缓冲区可能需要清理，以恢复到错误推测开始之前的状态。

Linux `perf` 用户可以通过以下命令检查分支预测错误的数量：

```bash
$ perf stat -e branches,branch-misses -- a.exe
   358209  branches
    14026  branch-misses #    3,92% of all branches        
# or simply do:
$ perf stat -- a.exe
```
