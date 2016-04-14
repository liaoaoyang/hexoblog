title: LeetCode-6
date: 2016-04-14 20:25:06
categories: Try
tags: LeetCode
---

# 概述

作为一道简单题，[ZigZag Conversion][1] 即将字符串按照之字形显示后，再逐行打印字符串。

# 分析

这一题题意自然是很明显的，但是，这个之字形的形状需要注意，这一个之字形的形状应该是：

```
A     G     M     S     Y
B   F H   L N   R T   X Z
C E   I K   O Q   U W
D     J     P     V
```

而不是类似字母 `N` 的模样。这里是需要注意的一点，题目中的示例为三行，并不是很明显。

简单来说，可以直接模拟重新排布字字母的过程，字母可以看做填写在一个 `m * n` 的矩阵中（`m * n` >= `s.length()`）。

如果想要在 *O(n)* 的时间复杂度内解决问题，那么最理想的方法应该是根据行列的位置，确定当前是填充字母还是留空。

观察这一个矩阵，若行数为 `n`，那么可以看做有多个 `n * (n - 1)` 的子矩阵，形状都是重复的，只需要得到各个位置上的字符与源字符串中的下标的对应关系即可得知当前应该出现的字母。

对于一个行数为 `n` 的子矩阵，包含的字符个数应为 <= 2n - 2。从左往右，第 `i` 个（`i >= 1`）子矩阵的字符串包含的字母下标范围为 [`(i - 1) * (2n - 2)`, `i * (2n - 2)`)

从行的角度来看，每行在每个子矩阵中，子矩阵的第一列一定是有字母的，而这个字母在源字符串中正是连续的，所以，每行直接输出子矩阵对应的字母的前 n 个字符即可。

而剩下的 n - 2 个字符，则是类似斜率为1的一条直线的排布，我们要做的，即当运行到对应点时，结合横纵位置，判断当前是否有字符，如果没出现下标越界，说明此处应显示对应的字母。

# 解法

```java
public class Solution {
    public String convert(String s, int numRows) {
        if (numRows == 1) {
            return s;
        }

        int sLen = s.length();
        int colNum = (int) Math.ceil(1.0 * sLen / numRows);
        int sectionNum = 2 * numRows - 2;

        String result = "";

        for (int r = 0; r < numRows; ++r) {
            for (int c = 0; c < colNum; ++c) {
                int idx = c * sectionNum + r;

                if (idx < sLen) {
                    result += s.substring(idx, idx + 1);

                    if (r >= 1 && r <= numRows - 2) {
                        int idxAppend = (c + 1) * sectionNum - r;

                        if (idxAppend >= 0 && idxAppend < sLen) {
                            result += s.substring(idxAppend, idxAppend + 1);
                        }
                    }
                }
            }
        }

        return result;
    }
}
```



[1]: https://leetcode.com/problems/zigzag-conversion/
