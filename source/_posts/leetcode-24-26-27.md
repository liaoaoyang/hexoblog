title: LeetCode-24-26-27
date: 2016-05-02 13:53:25
categories: Try
tags: LeetCode
---

[Swap Nodes in Pairs](https://leetcode.com/problems/swap-nodes-in-pairs/)，[Remove Duplicates from Sorted Array](https://leetcode.com/problems/remove-duplicates-from-sorted-array/)，[Remove Element](https://leetcode.com/problems/remove-element/) 均为简单题，合并到一起写解题思路。

# 24 Swap Nodes in Pairs

## 概述

[Swap Nodes in Pairs](https://leetcode.com/problems/swap-nodes-in-pairs/) 即是将相邻的两个节点交换形成新的链表，要求在于不能修改值，只能做链表节点操作。

对于空间复杂度上也要要求，只能在 `O(1)` 这个复杂度上完成。

## 分析

对于链表操作，无需多说，这里需要注意的地方在于：

+ 第一组元素（前2个）的第二个节点需要变成新链表的首节点
+ 在循环中，前一组的第一个节点要连接的是下一组的第二个节点

明确上述两个关键步骤之后，需要做的就是申请变量暂存目前处理节点的位置以及记录上一组节点的第一个节点。

## 解法

```java
public class Solution {
    public ListNode swapPairs(ListNode head) {
        ListNode firstOfPair = null;
        ListNode nowNode = head;
        ListNode privNode = null;
        int i = 1;

        while (nowNode != null) {
            if (i % 2 == 1) {
                firstOfPair = nowNode;
                nowNode = nowNode.next;
            } else {
                if (firstOfPair == null) {
                    return head;
                }

                if (privNode != null) {
                    privNode.next = nowNode;
                }

                privNode = firstOfPair;
                firstOfPair.next = nowNode.next;
                nowNode.next = firstOfPair;

                if (i == 2) {
                    head = nowNode;
                }

                nowNode = firstOfPair.next;
            }

            ++i;
        }

        return head;
    }
}
```

# 26 Remove Duplicates from Sorted Array

## 概述

[Remove Duplicates from Sorted Array](https://leetcode.com/problems/remove-duplicates-from-sorted-array/) 即通过数组元素移动操作，去除数组中重复的数字，并且返回不重复数字的个数。

## 分析

本题的需要注意，单纯返回非重复数字的长度是不够的，由于空间复杂度也有要求，需要在原数组中通过移位等操作将重复元素删除。

在是已经排序了的数组的情况下，相同数字必然是连续的，只需要遍历一遍数字，遇到相同的数字，将下一数字之后的数组元素前移即可。

下一次循环开始的

## 解法

```java
public class Solution {
	public int removeDuplicates(int[] nums) {
        if (nums.length <= 1) {
            return nums.length;
        }

        int nowIdx = 1;

        for (int i = 1; i < nums.length; ++i) {
            if (nums[i] != nums[i - 1]) {
                nums[nowIdx] = nums[i];
                ++nowIdx;
            }
        }

        return nowIdx;
    }
}
```

# 27 Remove Element

## 概述

[Remove Element](https://leetcode.com/problems/remove-element/) 同样是remove，和26的区别是给定的数字进行去除。

## 分析

与 [Remove Duplicates from Sorted Array](https://leetcode.com/problems/remove-duplicates-from-sorted-array/) 的解法类似，不同的是考虑终止条件的时候需要注意删除了元素之后相当于产生了一个数组，长度是发生了变化，循环时需要考虑这一因素。

## 解法

```java
public class Solution {
    public int removeElement(int[] nums, int val) {
        int removed = 0;

        for (int i = 0; i < nums.length - removed; ++i) {
            if (nums[i] == val) {
                for (int j = i; j + 1 < nums.length; ++j) {
                    nums[j] = nums[j + 1];
                }

                ++removed;
                --i;
            }
        }

        return nums.length - removed;
    }
}
```

# 其他

26 与 27 逐个搬迁并不是最佳解法，还有优化的空间。