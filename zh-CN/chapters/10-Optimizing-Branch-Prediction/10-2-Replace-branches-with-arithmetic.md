## 用算术运算替换分支（Replace Branches with Arithmetic）

在某些场景下，分支可以被算术运算所替代。代码清单 LookupBranches 中的代码也可以用简单的算术公式改写，如代码清单 ArithmeticBranches 所示。请注意，对于这段代码，Clang-17 编译器将代价高昂的除法替换为代价更低的乘法和右移操作。

代码清单：用算术运算替换分支。

```cpp
int8_t mapToBucket(unsigned v) {             │    mov al, -1
  constexpr unsigned BucketRangeMax = 50;    │    cmp edi, 49
  if (v < BucketRangeMax)                    │    ja .exit
    return v / 10;                           │    movzx eax, dil
  return -1;                                 │    imul eax, eax, 205
}                                            │    shr eax, 11
                                             │  .exit:
                                             │    ret
```
截至 2024 年，编译器通常无法自行找到这些捷径，因此需要程序员手动完成。如果能找到用算术运算替换频繁预测失误分支的方法，很可能会看到明显的性能提升。
