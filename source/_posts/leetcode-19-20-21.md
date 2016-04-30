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

```java
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

# 20 Valid Parentheses

## 概述

[Valid Parentheses](https://leetcode.com/problems/valid-parentheses/) 即判断括号是否匹配。

## 分析

括号需要一一对应，即一个左括号，需要对应一个右括号，简单来说，在字符串遍历过程中，右括号出现之后，需要判断上一个括号是不是一个左括号，如果是一个左括号，那么说明这个括号完成了匹配。否则，说明没有完成匹配。

从数据结构上来说，`栈`是一个非常适合的数据结构，左括号直接入栈，出现右括号则比较栈顶元素，如果相同，则出栈，继续比较，否则则说明不匹配。完成比较之后，如果栈已空，则说明括号完成了完全的匹配。

但是，使用字符串操作其实也能做到这一效果，完全不需要使用 Java 内置的栈。即栈顶元素可以看成是字符串的最后一个元素，出栈则是删除最后一个字符串。

## 解法

```java
public class Solution {
	public boolean isValid(String s) {
        String p = "";

        for (int i = 0; i < s.length(); ++i) {
            String c = s.substring(i, i + 1);

            if ("([{".contains(c)) {
                p = p + c;
                continue;
            }

            if (")]}".contains(c)) {
                if (p.length() == 0) {
                    return false;
                }

                String topOfStack = p.substring(p.length() - 1, p.length());

                if (c.equals(")") && !topOfStack.equals("(")) {
                    return false;
                }

                if (c.equals("]") && !topOfStack.equals("[")) {
                    return false;
                }

                if (c.equals("}") && !topOfStack.equals("{")) {
                    return false;
                }

                p = p.substring(0, p.length() - 1);
            }
        }

        return p.length() == 0;
    }
}
```

# 21 Merge Two Sorted Lists

## 概述

[Merge Two Sorted Lists](https://leetcode.com/problems/merge-two-sorted-lists/) 即单向链表归并。

## 分析

归并已排序链表，首先需要判断两个链表是否已经到达链表终点，达到链表终点之后只需要处理另一个没有到达终点的链表。

在两个链表没有到达终点时，比较二者的首节点的值，在升序的情况下，较小的值添加到结果链表中，较小值所在的链表向前前进一个节点，之后周而复始的比较。

## 解法

```java
public class Solution {
    public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
        if (l1 == null && l2 == null) {
            return null;
        }

        ListNode head = new ListNode(Integer.MIN_VALUE);
        ListNode it = head;

        while (l1 != null || l2 != null) {
            int val = 0;

            if (l1 == null) {
                val = l2.val;
                l2 = l2.next;
            } else if (l2 == null) {
                val = l1.val;
                l1 = l1.next;
            } else if (l1.val < l2.val) {
                val = l1.val;
                l1 = l1.next;
            } else {
                val = l2.val;
                l2 = l2.next;
            }

            if (it.val == Integer.MIN_VALUE) {
                it.val = val;
                continue;
            }

            it.next = new ListNode(val);
            it = it.next;
        }

        return head;
    }
}
```


