title: LeetCode-9
date: 2016-04-20 23:13:59
categories: Try
tags: LeetCode
---

# 概述

[Palindrome Number](https://leetcode.com/problems/palindrome-number/) 判断文字是否是回文，输出布尔值。

# 分析

作为一道 `Easy` 的题目，看起来并不是特别麻烦，不过这题要求无需增加附加的空间复杂度。

题目要求判断是否为回文数，那么负数肯定不是回文数，因为负数带有符号位。

从字符串的角度来看，回文只需判断反转之后是否相同即可，那么针对数字亦可以使用这一方法。

考虑到数字反转之后会出现溢出的情况，而正数溢出之后在 Java 中的处理方式则是变为一个负数（参见 SO 上的问题 [How does Java handle integer underflows and overflows and how would you check for it?](http://stackoverflow.com/questions/3001836/how-does-java-handle-integer-underflows-and-overflows-and-how-would-you-check-fo)），那么反转后变为负数的数字自然不是一个回文数了，因为反转之后在非负整数的范围内显然这个数字不存在。

不新增变量的情况下，只能通过递归来解决问题了。

# 解法

```
public class Solution {
    public static int len(int v) {
        return (v == 0 ? 0 : 1 + len(v / 10));
    }

    public static int reverse(int x) {
        if (x < 10) {
            return x;
        }

        return reverse(x / 10) + (x % 10) * (int)Math.pow(10, len(x) - 1);
    }

    public boolean isPalindrome(int x) {
        if (x < 0) {
            return false;
        }

        if (x == 0) {
            return true;
        }

        return x == reverse(x);
    }
}
```