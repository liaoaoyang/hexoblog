title: LeetCode-1
date: 2016-04-09 23:03:39
categories: Try
tags: LeetCode
---

# 概述

作为简单题的第一题([Two Sum][1])，简答来说就是给出一个序列以及目标值，求出能加和出目标值的两个数字在数组中的下标。

# 分析

这个题目可以说是一目了然，直接能够想到的解法就是遍历这一个序列，然后遍历下标大于当前遍历到的数字的元素，如果有匹配的数字就直接返回结果即可。

如果单纯遍历，时间复杂度高达*O(n<sup>2</sup>)*，这个复杂度不太令人满意。

那么是否可以通过空间换时间呢？自然是没问题的。

后续考虑通过使用HashMap的方式记录每个值对应的元素的下标，每次检查是否有当前值相加得到目标值的的key存在，将时间复杂度降低到了*O(n)*，不过空间复杂度也上升到了*O(n)*。

# 解法

```java
public class Solution {
    public int[] twoSum(int[] nums, int target) {
                Map<Integer, Integer> numIdxMap = new HashMap<Integer, Integer>();

        for (int i = 0; i < nums.length; ++i) {
            numIdxMap.put(nums[i], i);
        }

        int[] idx = new int[2];

        for (int i = 0; i < nums.length; ++i) {
            if (numIdxMap.containsKey(target - nums[i])) {
                int j = numIdxMap.get(target - nums[i]);

                if (i == j) {
                    continue;
                }

                idx[0] = i;
                idx[1] = j;

                if (idx[0] > idx[1]) {
                    int tmp = idx[1];
                    idx[1] = idx[0];
                    idx[0] = tmp;
                }

                break;
            }
        }

        return idx;
    }
}
```

[1]: https://leetcode.com/problems/two-sum/
