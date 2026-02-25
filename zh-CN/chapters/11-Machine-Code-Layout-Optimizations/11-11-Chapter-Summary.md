## 本章小结


CPU 前端优化摘要如表 CPU_FE_OPT 所示。

| Transform | How transformed? | Why helps? | Works best for | Done by |
|-----------|-----------------|------------|----------------|---------|
| Basic block placement | maintain fall through hot code | not taken branches are cheaper; better cache utilization | any code, especially with a lot of branches | compiler |
| Basic block alignment | shift the hot code using NOPs | better cache utilization | hot loops | compiler |
| Function splitting | split cold blocks of code and place them in separate functions | better cache utilization | functions with complex CFG when there are big blocks of cold code between hot parts | compiler |
| Function reorder | group hot functions together | better cache utilization | many small hot functions | linker |

表：CPU 前端优化摘要。

* 代码布局改进常常被低估和忽视。I-cache 和 ITLB 缺失等 CPU 前端性能问题占据了大量浪费的周期，尤其对于代码量庞大的应用程序。但即使是中小型应用程序也可以从优化机器码布局中受益。
* 如果能为应用程序提供一组典型使用场景，通常最好的选择是使用 LTO、PGO、BOLT 及类似工具来改善代码布局。对于大型应用程序，这是唯一实用的选择。

