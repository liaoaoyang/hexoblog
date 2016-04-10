title: leetcode-3
date: 2016-04-10 23:41:39
categories: Try
tags: LeetCode
---

# 概述

今天份的LeetCode题目 [Longest Substring Without Repeating Characters][1] 也是一道 `Medium` 的题目。

大意即找出给定字符串中的连续最长非重复子序列的长度。

# 分析

首先，字符串连续的前提之下，若在位置 `n` 与 `m` (`m`>`n`)出现了重复的字符，`n`位置以及之前的字符是不能再继续算入连续的非重复字符串的，所以，这时候统计的字符串的起始位置要变为 `n + 1` 位置下的字符。

在出现重复时，判断当前已经连续的非重复字符的个数是否已经大于已记录的最长字符串的长度，如大于则替换之。

最后返回计算出的结果即可。

这一方法只需一次遍历字符串即可得知最大的长度，由于使用了HashMap存储最长子串中的字符与下标的对应关系，时间复杂度为*O(n)*。


# 解法

```java
public class Solution {
        public int lengthOfLongestSubstring(String s) {
        int longest = 0;
        int startIdx = 0;

        for (int i = 0; i < s.length(); ++i) {
            String atI = s.substring(i, i + 1);
            String currentLongest = s.substring(startIdx, i);
            int repeatIdx = currentLongest.indexOf(atI);

            if (repeatIdx >= 0) {
                if (currentLongest.length() > longest) {
                    longest = currentLongest.length();
                }

                startIdx += (repeatIdx + 1);
            }
        }

        if (s.substring(startIdx).length() > longest) {
            longest = s.substring(startIdx).length();
        }

        return longest;
    }
}
```

[1]: https://leetcode.com/problems/longest-substring-without-repeating-characters/
