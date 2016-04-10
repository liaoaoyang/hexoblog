title: LeetCode-2
date: 2016-04-10 13:28:41
categories: Try
tags: LeetCode
---

# 概述

[Add Two Numbers][1] 难度为`Medium`，这一题目简单来说即给出两个序列，代表两个整数，模拟加法运算。

# 分析

这一题难度个人认为依然不大，即把正常加法的算式过程用程序表示。需要特殊处理的是进位这一操作。

正常的加法竖式运算，需要将最低位对其，然后开始从低位向高位计算。

再具体一些，考虑一个加法运算，两个数相加，一个数字长度为n，一个数字为m，m>n，那么，和的字符个数至多为m+1。

对于进位值的计算，其实就是将当前位数的值以及前期的进位值相加，除以10即为进位制。

这一解法可以在*O(n)*的时间复杂度上解决问题。

# 解法

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
public class Solution {
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        Map<Integer, Integer> lm1 = new HashMap<>();
        Map<Integer, Integer> lm2 = new HashMap<>();

        ListNode l = l1;
        int idx = 0;

        while (l != null) {
            lm1.put(idx, l.val);
            ++idx;
            l = l.next;
        }

        l = l2;
        idx = 0;

        while (l != null) {
            lm2.put(idx, l.val);
            ++idx;
            l = l.next;
        }

        Map<Integer, Integer> longer = (lm1.size() > lm2.size() ? lm1 : lm2);
        Map<Integer, Integer> shorter = (lm1.size() <= lm2.size() ? lm1 : lm2);
        int lengthGap = longer.size() - shorter.size();

        ListNode result = new ListNode(0);
        ListNode resultRef = result;
        int carryVal = 0;

        for (int i = 0; i < longer.size(); ++i) {
            int nowVal = 0;

            if (i >= shorter.size()) {
                nowVal = longer.get(i) + carryVal;
            } else {
                nowVal = longer.get(i) + shorter.get(i) + carryVal;
            }

            carryVal = (nowVal / 10);
            resultRef.val = (nowVal >= 10 ? nowVal % 10 : nowVal);

            if (i != longer.size() - 1) {
                resultRef.next = new ListNode(carryVal);
                resultRef = resultRef.next;
            } else {
                if (carryVal > 0) {
                    resultRef.next = new ListNode(carryVal);
                    resultRef = resultRef.next;
                }

                resultRef.next = null;
            }
        }

        return result;
    }
}
```



[1]: https://leetcode.com/problems/add-two-numbers/
