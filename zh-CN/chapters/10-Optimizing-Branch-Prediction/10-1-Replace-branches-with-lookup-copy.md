## 用查找表替换分支（Replace Branches with Lookup）

避免频繁分支预测失误的一种方法是使用查找表（lookup tables）。代码清单 LookupBranches 展示了这种转换可能带来收益的示例。与往常一样，左侧为原始版本，右侧为改进版本。函数 `mapToBucket` 将 `[0-50)` 范围内的值映射到对应的五个桶（bucket）中，对超出范围的值返回 `-1`。对于均匀分布的 `v` 值，每个桶被命中的概率相等。在原始版本生成的汇编代码中，很可能会出现大量分支，这些分支可能具有较高的预测失误率。幸运的是，可以使用单次数组查找来重写 `mapToBucket` 函数，如右侧所示。

代码清单：用查找表替换分支。

```cpp
int8_t mapToBucket(unsigned v) {       int8_t buckets[50] = {
  if      (v < 10) return 0;             0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
  else if (v < 20) return 1;             1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
  else if (v < 30) return 2;      =>     2, 2, 2, 2, 2, 2, 2, 2, 2, 2,
  else if (v < 40) return 3;             3, 3, 3, 3, 3, 3, 3, 3, 3, 3,
  else if (v < 50) return 4;             4, 4, 4, 4, 4, 4, 4, 4, 4, 4 };
  return -1;
}                                      int8_t mapToBucket(unsigned v) {
                                         if (v < (sizeof(buckets) / sizeof(int8_t)))
                                           return buckets[v];
                                         return -1;
                                       }
```
对于右侧 `mapToBucket` 的改进版本，编译器很可能只生成一条分支指令用于防止越界访问 `buckets` 数组。经过该函数的典型热路径（hot path）将执行一条未被命中的分支和一条加载指令。由于预期大多数输入值都落在 `buckets` 数组覆盖的范围内，CPU 分支预测器能够很好地预测该分支。同时，由于 `buckets` 数组较小，很可能驻留在 L1 数据缓存（L1 D-cache）中，因此查找速度也会很快。

如果需要映射更大范围的值，例如 `[0-1M)`，分配一个超大数组并不实际。在这种情况下，可以考虑使用区间映射（interval map）数据结构，它以更少的内存实现相同目标，但查找复杂度为对数级别。读者可以在 [Boost](https://www.boost.org/doc/libs/1_65_0/libs/icl/doc/html/boost/icl/interval_map.html)[^2] 和 [LLVM](https://llvm.org/doxygen/IntervalMap_8h_source.html)[^3] 中找到区间映射容器的现有实现。

[^2]: C++ Boost `interval_map` - [https://www.boost.org/doc/libs/1_65_0/libs/icl/doc/html/boost/icl/interval_map.html](https://www.boost.org/doc/libs/1_65_0/libs/icl/doc/html/boost/icl/interval_map.html)
[^3]: LLVM's `IntervalMap` - [https://llvm.org/doxygen/IntervalMap_8h_source.html](https://llvm.org/doxygen/IntervalMap_8h_source.html)
