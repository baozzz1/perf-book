## 本章小结


CPU 前端优化摘要如表 CPU_FE_OPT 所示。

--------------------------------------------------------------------------
Transform  How transformed?  Why helps?    Works best for        Done by
---------  ----------------  ------------  --------------------  ---------
Basic      maintain          not taken     any code, especially  compiler
block      fall through      branches are  with a lot of 
placement  hot code          cheaper;      branches
                             better cache
                             utilization

Basic      shift the hot     better cache  hot loops             compiler
block      code using NOPs   utilization 
alignment

Function   split cold        better cache  functions with        compiler
splitting  blocks of code    utilization   complex CFG when 
           and place them                  there are big blocks 
           in separate                     of cold code between 
           functions                       hot parts

Function   group hot         better cache  many small            linker
reorder    functions         utilization   hot functions
           together
--------------------------------------------------------------------------

表：CPU 前端优化摘要。

* 代码布局改进常常被低估和忽视。I-cache 和 ITLB 缺失等 CPU 前端性能问题占据了大量浪费的周期，尤其对于代码量庞大的应用程序。但即使是中小型应用程序也可以从优化机器码布局中受益。
* 如果能为应用程序提供一组典型使用场景，通常最好的选择是使用 LTO、PGO、BOLT 及类似工具来改善代码布局。对于大型应用程序，这是唯一实用的选择。

