## 多测试单分支（Multiple Tests Single Branch）

本章讨论的最后一种技术旨在通过合并多个测试来最小化动态分支指令数量。其核心思想是避免对大数组的每个元素都执行一次分支。相反，目标是同时执行多个测试，这主要涉及使用 SIMD 指令。多个测试的结果是一个向量掩码（vector mask），可以转换为字节掩码（byte mask），通常可以用单条分支指令处理。这使得我们能够消除大量分支指令，如下所示。你可能会在 JSON/HTML 解析、媒体编解码器（media codecs）等各种算法的 SIMD 实现中见到这种技术。

代码清单 LongestLineNaive 展示了一个逐字符测试在输入字符串中查找最长行的函数。我们遍历输入字符串，搜索行尾（`eol`）字符（`\n`，ASCII 中为 0x0A）。每找到一个 `eol` 字符，就检查当前行是否是最长的，并将当前行长度重置为零。这段代码对每个字符都会执行一条分支指令。[^1]

代码清单：查找最长行（逐字符处理）。

```cpp
uint32_t longestLine(const std::string &str) {
  uint32_t maxLen = 0;
  uint32_t curLen = 0;
  for (auto s : str) {
    if (s == '\n') {
      maxLen = std::max(curLen, maxLen);
      curLen = 0;
    } else {
      curLen++;
    }
  }
  // if no end-of-line in the end
  maxLen = std::max(curLen, maxLen);
  return maxLen;
}
```
代码清单 LongestLineSIMD 展示了一次处理八个字符的替代实现。这种思路通常使用编译器内联函数（compiler intrinsics，参见 [secIntrinsics]）来实现，但为了清晰起见，此处展示标准 C++ 代码。这个具体案例是 Performance Ninja 实验任务之一[^2]，读者可以自行尝试编写 SIMD 代码。请注意，下面展示的代码并不完整，遗漏了一些边角情况，仅用于说明思路。

代码清单：查找最长行（每次处理 8 个字符）。

```cpp
uint32_t longestLine(const std::string &str) {
  uint32_t maxLen = 0;
  const uint64_t eol = 0x0a0a0a0a0a0a0a0a;
  auto *buf = str.data();
  uint32_t lineBeginPos = 0;
  for (uint32_t pos = 0; pos + 7 < str.size(); pos += 8) {
    // Load 8-byte chunk of the input string.
    uint64_t vect = *((const uint64_t*)(buf + pos));
    // Check all characters in this chunk.
    uint8_t mask = compareBytes(vect, eol);
    while (mask) {
      uint16_t eolPos = tzcnt(mask);
      // Compute the length of the current string.
      uint32_t curLen = (pos - lineBeginPos) + eolPos;
      // New line starts with the character after '\n'
      lineBeginPos += curLen + 1;
      // Is this line the longest?
      maxLen = std::max(curLen, maxLen);
      // Shift the mask to check if we have more '\n'
      mask >>= eolPos + 1;
    }
  }
  // process remainder (not shown)
  return maxLen;
}

uint8_t compareBytes(uint64_t a, uint64_t b) {
  // Perform a byte-wise comparison of a and b.
  // Produce a bit mask with the result of comparisons:
  // one if bytes are equal, zero if different.
}

uint8_t tzcnt(uint8_t mask) {
  // Count the number of trailing zero bits in the mask.
}
```
首先准备一个填充了 `eol` 符号的 8 字节掩码。内层循环加载输入字符串的八个字符，并对这些字符与 `eol` 掩码进行逐字节比较。现代处理器中的向量寄存器包含 16/32/64 字节，因此可以同时处理更多字符。八次比较的结果是一个 8 位掩码，对应位置的值为 0 或 1（参见 `compareBytes`）。例如，比较 `0x00FF0A000AFFFF00` 和 `0x0A0A0A0A0A0A0A0A` 时，结果为 `0b00101000`。在 x86 和 ARM ISA 上，`compareBytes` 函数可以用两条向量指令实现。[^4]

若掩码为零，说明当前块中没有 `eol` 字符，可以跳过（见第 11 行）。这是一个关键优化，对于包含长行的输入字符串能带来显著加速。若掩码不为零，说明存在 `eol` 字符，需要找出其位置。为此使用 `tzcnt` 函数，它计算 8 位掩码中尾部零位的个数（即最右边置位的位置）。例如，对于掩码 `0b00101000`，返回值为 3。大多数 ISA 支持用单条指令实现 `tzcnt` 函数。[^3] 第 14 行利用 `tzcnt` 函数的结果计算当前行的长度。将掩码右移后重复，直到掩码中没有置位为止。

对于只有单条超长行的输入字符串（最优情况），SIMD 版本执行的分支指令数量减少为原来的八分之一。然而，在最坏情况下（即输入字符串中全为 `eol` 字符，即零长度行），原始方法更快。笔者使用 AVX2 实现（每块 16 个字符）在多种不同输入上（包括教材和源代码文件）对此技术进行了基准测试。结果表明，在 Intel Core i7-1260P（第 12 代，Alder Lake）上，分支指令数量减少了 5--6 倍，性能提升超过 4 倍。

[^1]: 假设编译器不会为 `std::max` 生成分支指令。
[^2]: Performance Ninja: compiler intrinsics 2 - [https://github.com/dendibakh/perf-ninja/tree/main/labs/core_bound/compiler_intrinsics_2](https://github.com/dendibakh/perf-ninja/tree/main/labs/core_bound/compiler_intrinsics_2).
[^3]: 尽管在 x86 中，`TZCNT` 指令没有支持 8 位输入的版本。
[^4]: 例如，使用 AVX2（256 位向量），可以使用 `VPCMPEQB` 和 `VPMOVMSKB` 指令。
