title: LeetCode-19-20-21
date: 2016-04-30 10:06:06
categories: Try
tags: LeetCode
---

这三题均为 `Easy` ，合并到一起写写解题思路。

# 19 Remove Nth Node From End of List

## 概述

[Remove Nth Node From End of List](https://leetcode.com/problems/remove-nth-node-from-end-of-list/) 即删除单向链表中从链表尾部开始的第N个节点。

## 分析

单向链表只能通过逐步前进才能到达指定节点，是无法在 `O(1)` 的时间复杂度下拿到链表的长度的，可以考虑通过一次遍历将链表明确节点节点位置与值的对应关系，之后通过链表的节点操作完成删除节点的效果。

节点位置与值的关系可以通过Hash进行存储，由于链表长度不定，预先留出数组空间的方式个人认为并不是好的选择。

## 解法

```
public class Solution {
    public ListNode removeNthFromEnd(ListNode head, int n) {
        HashMap<Integer, ListNode> idxListMap = new HashMap<Integer, ListNode>();

        ListNode it = head;
        int i = 1;

        for(; it != null; it = it.next, ++i) {
            idxListMap.put(i, it);
        }

        if (idxListMap.size() == 1) {
            return null;
        }

        if (n == 1) {
            idxListMap.get(idxListMap.size() - 1).next = null;
            return head;
        }

        if (n == idxListMap.size()) {
            head = head.next;
            return head;
        }

        if (idxListMap.containsKey(idxListMap.size() - n) && idxListMap.containsKey(idxListMap.size() - n + 2)) {
            idxListMap.get(idxListMap.size() - n).next = idxListMap.get(idxListMap.size() - n + 2);
        }

        return head;
    }
}
```

