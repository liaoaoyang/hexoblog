title: LeetCode-4
date: 2016-04-12 00:43:13
categories: Try
tags: LeetCode
---

# 概述

作为复杂题的第一题([Median of Two Sorted Arrays][1])，在时间复杂度为*O(log (m+n))*（
`m`，`n`均为数组长度）的要求之下，求两个已排序数组的中位数。

# 分析

首先中位数的定义可以在 [WikiPedia][2] 上了解到。在已排序的前提下，即奇数元素个数的数组，为中间值；偶数元素个数的数组则为中间二值的算数平均数。

那么两个序列求中位数，可以考虑先将二者合并排序之后，进行中位数的求值。

考虑到两数组已经排序，那么我们所需要做的就是进行一次归并排序。

归并排序作为一个相当稳定的排序方式，在时间复杂度上能够满足需求。

考虑到虽然数组是排序数组，但是并没有说明是升序或是降序，无妨，如果是降序，那么将下标从尾部开始即可，但是个人测试，应该没有出现这类 case 。

# 解法

```java
public class Solution {
    private static final int ASC = 0;
    private static final int DESC = 1;

    public double findMedianSortedArrays(int[] nums1, int[] nums2) {
        double median = 0;

        if (nums1.length + nums2.length == 0) {
            return 0;
        }

        if (nums1.length == 1 && nums2.length == 1) {
            return (nums1[0] + nums2[0]) / 2.0;
        }

        int[] mergedNums = new int[nums1.length + nums2.length];

        int i = 0;
        int j = 0;
        int nums1order = (nums1.length >= 2 && nums1[0] > nums1[1]) ? DESC : ASC;
        int nums2order = (nums2.length >= 2 && nums2[0] > nums2[1]) ? DESC : ASC;
        int mergedIdx = 0;

        for (;i < nums1.length || j < nums2.length;) {
            int realI = i;
            int realJ = j;

            if (i >= nums1.length) {
                mergedNums[mergedIdx] = nums2[realJ];
                ++j;
                ++mergedIdx;
                continue;
            }

            if (j >= nums2.length) {
                mergedNums[mergedIdx] = nums1[realI];
                ++mergedIdx;
                ++i;
                continue;
            }

            int iVal = nums1[realI];
            int jVal = nums2[realJ];

            if (iVal == jVal) {
                mergedNums[mergedIdx] = iVal;
                ++mergedIdx;
                mergedNums[mergedIdx] = jVal;
                ++mergedIdx;
                ++i;
                ++j;
                continue;
            }

            if (iVal > jVal) {
                mergedNums[mergedIdx] = jVal;
                ++mergedIdx;
                ++j;
                continue;
            }

            if (iVal < jVal) {
                mergedNums[mergedIdx] = iVal;
                ++mergedIdx;
                ++i;
            }
        }

        int mergedSize = mergedNums.length;
        int medianIdx = (mergedSize + 1) / 2 - 1;

        if (mergedSize % 2 == 1) {
            median = mergedNums[medianIdx];
        } else {
            medianIdx = mergedIdx / 2;
            median = (mergedNums[medianIdx] + mergedNums[medianIdx - 1]) / 2.0;
        }

        return median;
    }
}
```

[1]: https://leetcode.com/problems/median-of-two-sorted-arrays/
[2]: https://en.wikipedia.org/wiki/Median
