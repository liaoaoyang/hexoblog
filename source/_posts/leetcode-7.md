title: LeetCode-7
date: 2016-04-16 14:15:09
categories: Try
tags: LeetCode
---

# 概述

[Reverse Integer](https://leetcode.com/problems/reverse-integer/) 简单题，要求将不带符号位的整数反转，如果遇到溢出问题，则输出0。

<!-- more -->

# 分析

题目的特别之处在于溢出的处理，数字反转之后很可能超过了 `Integer.MAX_VALUE` 或者`Integer.MIN_VALUE` 的范围，应对这种情况，可以将溢出整数范围的的结果认定为0。

参见 SO 上的问题 [How does Java handle integer underflows and overflows and how would you check for it?](http://stackoverflow.com/questions/3001836/how-does-java-handle-integer-underflows-and-overflows-and-how-would-you-check-fo), Java的处理方式是在两个极值之间轮转，超过最大值（`2147483647`），则从最小值开始逐步趋近最小值(` -2147483648`)，如：

```java
Long l = Long.parseLong("2147483648");
System.out.println(l.intValue());
l = Long.parseLong("2147483649");
System.out.println(l.intValue());
l = Long.parseLong("2147483650");
System.out.println(l.intValue());
```

输出结果为：

```java
-2147483648
-2147483647
-2147483646
```

反之亦然。

解法先用字符串操作的方式进行，还需要探究更高效的算法。

# 解法


```java
public class Solution {
    public int reverse(int x) {
        StringBuffer sb = new StringBuffer(x + "");
        Long xL;
        int factor = 1;

        if (sb.substring(0, 1).equals("-")) {
            factor = -1;
            xL = Long.parseLong(sb.replace(0, 1, "").reverse().toString());
        } else {
            xL = Long.parseLong(sb.reverse().toString());
        }

        if (factor > 0 && xL > Integer.MAX_VALUE) {
            return 0;
        }

        if (factor < 0 && (-1 * xL) < Integer.MIN_VALUE) {
            return 0;
        }


        return xL.intValue() * factor;
    }
}
```


