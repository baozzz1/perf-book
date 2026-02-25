## 已退休指令与已执行指令（Retired vs. Executed Instruction）

现代处理器通常会执行比程序流所需更多的指令。这是因为某些指令是推测执行（speculative execution）的，如 [SpeculativeExec] 中所述。对于大多数指令，一旦结果可用，且所有前驱指令已经退休（retired），CPU 就会提交（commit）结果。但对于推测执行的指令，CPU 会保留其结果而不立即提交。当推测被证明是正确的，CPU 会解除对这些指令的阻塞并正常继续执行。但当推测被证明是错误的，CPU 会丢弃推测执行指令所做的所有更改，且不会将其退休。因此，被 CPU 处理过的指令可以是已执行（executed）的，但不一定是已退休（retired）的。考虑到这一点，我们通常可以预期已执行指令的数量会高于已退休指令的数量。

但也有例外。某些指令被识别为惯用语（idiom），无需实际执行即可被解析。例如 NOP、寄存器移动消除（move elimination）和清零操作，如 [uarchBE] 中所述。这类指令不需要执行单元，但仍然会被退休。因此，理论上可能存在已退休指令数量多于已执行指令数量的情况。

大多数现代处理器中都有一个性能监控计数器（PMC，Performance Monitoring Counter），用于收集已退休指令的数量。虽然没有专门的性能事件用于收集已执行指令，但有一种方式可以收集已执行和已退休的 *$\mu$ops*，我们稍后会看到。使用 Linux `perf` 可以很轻松地获取已退休指令的数量：

```bash
$ perf stat -e instructions -- ./a.exe
  2173414  instructions  #    0.80  insn per cycle 
# or just simply run:
$ perf stat -- ./a.exe
```
