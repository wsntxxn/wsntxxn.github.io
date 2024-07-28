---
title: 'Maximum Subarray Series'
date: 2019-08-24
permalink: /posts/2019/08/maximum-subarray-series/
tags:
  - leetcode
  - dynamic programming
---

> Maximum Subarray 是非常经典的算法题，最开始学数据结构的时候就讲了从 $$O(N^3)$$ 到 $$O(N^2)$$ 再到 $$O(N)$$ 的优化过程，在此基础上还有其它变种题目。实际上，这一系列题都可以用动态规划来解决。

# Maximum Subarray

## Source

* LeetCode: [(53) Maximum Subarray](https://leetcode-cn.com/problems/maximum-subarray/)
* LintCode: [(41) Maximum Subarray](https://www.lintcode.com/problem/maximum-subarray/)

## Description

>
Given an array of integers, find a contiguous subarray which has the largest sum.<br>
**Example**<br>
**Example1:**<br>
Input: [-2, 2, -3, 4, -1, 2, 1, -5, 3]<br>
Output: 6<br>
Explanation: the contiguous subarray [4,-1,2,1] has the largest sum = 6.<br>
**Challenge**<br>
Can you do it in time complexity O(n)?


## Analysis & Code

### Solution 1 - Two Pointers

这是笔者初学数据结构时书上的经典算法。既然是 subarray，就可以用两个指针 (记为 `start` 和 `end`)分别指向起始和结束位置。一个值得注意的性质是，如果有一个子数组 $$[a_i, a_{i+1}, ..., a_j]$$ 的和小于0，那么这个子数组肯定不会出现在结果的两边，因为去掉这部分仍是满足条件的子数组，而且和一定会变大。于是，一开始让 `start` 和 `end` 都指向第一个数，`end` 不断往后移，同时更新 `sum`，一旦 `sum` 小于0就舍弃这部分 (end = start = end + 1)，直到 `end` 达到最后一个数。

```java
/**
 * @param nums: A list of integers
 * @return: A integer indicate the sum of max subarray
 */
public int maxSubArray(int[] nums) {
    // write your code here
    if (nums == null || nums.length == 0)
        return -1;
  
    int res = Integer.MIN_VALUE;

    int start = 0;
    int end = 0;
    int candidate = 0;

    while (end < nums.length) {
        candidate += nums[end];
        if (candidate > res)
            res = candidate;
        if (candidate < 0) {
            start = end + 1;
            end = start;
            candidate = 0;
        }
        else
            end++;
    }

    return res;
}
```

当然，这道题不需要返回子数组的起始和结束位置，其实不需要保存`start`和`end`，直接更新`candidate`就可以，在遍历每个数时：

```java
candidate = Math.max(candidate, 0);
candidate += num;
```

### Solution 2 - DP (global & local)

从动态规划的角度来思考，那么 `dp[i]` 存的是 `nums[0...i]` 子数组的最大子数组和，如何更新 `dp[i+1]` 呢？

因为我们也不知道 `dp[i]` 表示的最大子数组和包不包括 `nums[i]`，所以光凭 `dp[i]` 是没法得到增加 `nums[i+1]` 对 `dp[i+1]` 的影响的，于是还要维护一个 `local` 数组，表示 `nums[0...i]` 子数组中，包含 `nums[i]` 的最大子数组和，这样增加了 `nums[i+1]`，就能更新 `local[i+1]` 的值了，从而更新`dp[i+1]`。

这里把 dp 换成 global，那么状态转移方程就是：

$$local[i+1] = max(local[i] + nums[i+1], nums[i+1])$$

$$global[i+1] = max(local[i+1], global[i])$$

第二个式子这么理解，`global[i+1]`取值来源有以下两种，取最大即可：

* 包含 `nums[i+1]` 的子数组和 (`local[i+1]`)
* 不包含 `nums[i+1]` 的子数组和 (`global[i]`)

```java
public int maxSubarray(int[] nums) {
    if (nums == null || nums.length == 0)
        return -1;
    int[] local = new int[nums.length + 1];
    int[] global = new int[nums.length + 1];
    global[0] = Integer.MIN_VALUE;
    
    for (int i = 1; i <= nums.length; ++i) {
        local[i] = Math.max(local[i - 1] + nums[i - 1], nums[i - 1]);
        global[i] = Math.max(global[i - 1], local[i]);
    }
    
    return global[nums.length];
}
```

### Solution 3 - DP (sum(nums[0...m]) - sum(nums[0...n]))

这种方法不直接存 `nums[0...i]` 的最大子数组和，存的是 `nums[0...i]` 子数组的最小子数组和 (存在`minSum`数组中)，因为子数组 `nums[m+1...n]` 的和可以看成两个从第一个数开始的子数组 (`nums[0...m]` 和 `nums[0...n]`) 的差。这个思想也是[Range Sum Query](https://leetcode-cn.com/problems/range-sum-query-immutable/)的关键，还在[买卖股票](https://leetcode-cn.com/problems/best-time-to-buy-and-sell-stock/)那道题中出现过，也是一种经典思想了。

```java
public int maxSubArray(int[] nums) {
    // the sum of subarray equals the subtraction of two arrays, both starting from index 0
    // so it is max(nums[0: m]) - min(nums[0: n]) where n < m
    int sum = 0;
    int minSum = 0;
    int res = Integer.MIN_VALUE;

    for (int num : nums) {
        // minSum denotes the minimum sum in subarray nums[0: n-1] (excluding nums[n])
        minSum = Math.min(minSum, sum);
        sum += num;
        res = Math.max(res, sum - minSum);
    }

    return res;
}
```

# Maximum Subarray II

## Source

- LintCode: [(42) Maximum Subarray II](https://www.lintcode.com/problem/maximum-subarray-ii/)

## Description

> Given an array of integers, find two non-overlapping subarrays which have the largest sum.<br>
> The number in each subarray should be contiguous.<br>
> Return the largest sum.<br>
> **Example**<br>
> **Example:**<br>
> Input:<br>
> [1, 3, -1, 2, -1, 2]<br>
> Output:<br>
> 7<br>
> Explanation:<br>
> the two subarrays are [1, 3] and [2, -1, 2] or [1, 3, -1, 2] and [2]<br>
> **Challenge**<br>
>Can you do it in time complexity O(n) ?

## Analysis & Code

由于两个子数组是 non-overlap 的，比较自然的思路就是以某个数的分界线，将数组一分为二，分别求左边和右边子数组的 maximum subarray，相加就得到了这个分界线下的最大和。遍历这个分界线的所有位置求最大，就得到了结果。所以这里需要求一个 `forward` 数组表示前向的最大子数组和，具体就是 `forward[i]` 表示 `nums[0...i]` 的最大子数组和；相对地，还需要一个 `backward` 数组，`backward[i]` 表示 `nums[i+1...n-1]`的最大子数组和。求这两个数组直接用`Maximum Subarray I` 的方法即可，只是 `backward` 要从后往前遍历。

```java
public int maxTwoSubArrays(List<Integer> nums) {
    // write your code here
  	int len = nums.size();
    int[] forward = new int[len - 1];
    int[] backward = new int[len - 1];

    // forward
    forward[0] = nums.get(0);
    int forward_candidate = nums.get(0);
    for (int i = 1; i < len - 1; ++i) {
        forward_candidate = Math.max(forward_candidate, 0) + nums.get(i);
        forward[i] = Math.max(forward[i - 1], forward_candidate);
    }

    // backward
    backward[len - 2] = nums.get(len - 1);
    int backward_candidate = nums.get(len - 1);
    for (int i = len - 3; i >= 0; --i) {
      	backward_candidate = Math.max(backward_candidate, 0) + nums.get(i + 1);
        backward[i] = Math.max(backward[i + 1], backward_candidate);
    }

    int res = Integer.MIN_VALUE;
    for (int i = 0; i < len - 1; ++i)
      	res = Math.max(res, forward[i] + backward[i]);
    return res;
}
```

这里用的是 localMax & globalMax 的方法 (其实和双指针思路是一样的)，注意 `forward` 和 `backward` 因为下标的关系和上面的叙述略有不同，自己写的时候搞清楚下标代表啥，不要重复算某个数。

# Maximum Subarray III

## Source

* LintCode: [(43) Maximum Subarray III](https://www.lintcode.com/problem/maximum-subarray-iii/description)

## Description

>Given an array of integers and a number *k*, find k **non-overlapping** subarrays which have the largest sum.<br>
>The number in each subarray should be **contiguous**.<br>
>Return the largest sum.<br>
>
>**Example**<br>
>Input:<br>
>List = [-1, 4, -2, 3, -2, 3]<br>
>k = 2<br>
>Output: 8<br>
>Explanation: 4 + (3 + -2 + 3) = 8

## Analysis & Code

这道题难了很多，不过变化并不大，只是将两个 non-overlap 的子数组换成了`k`个子数组，因此 DP 不仅要沿着数组长度的方向进行，还要沿着`k`的方向进行，即先求 1 个、2 个、3个 non-overlap 子数组的最大和并保存起来，逐渐增大子数组的个数直到`k`即可得到答案。

由于固定`k`求 maximum subarray 时沿着数组长度 DP 的过程中要根据新遍历到的数 `nums[i]` 更新结果，所以还是用一个 `localMax` 表示包含 `nums[i]` 的 (k个) 最大子数组和，`globalMax` 表示总的 nums[0...i] 的 (k个) 最大子数组和，更新两个数组的值。

新加入 `nums[i-1]` 时，`localMax[i][j]` 的更新公式为：

$$localMax[i][j] = max(nums[i-1] + globalMax[i-1][j-1], nums[i-1] + localMax[i-1][j])$$

分两种情况：
1. `nums[i-1]` 单独作为一个 subarray，需要从前 i-1 个数中选出最大的 j-1 个子数组
2. `nums[i-1]` 和 `nums[i-2]`及之前的数一起组成 subarray，需要从前 i-1 个数中选出包括 `nums[i-2]` 的最大的 j 个子数组

`globalMax[i][j]`的更新就简单了：

$$globalMax[i][j] = max(globalMax[i-1][j], localMax[i][j])$$

和 maximum subarray I 一样，从包含 `nums[i-1]` 和不包含 `nums[i-1]` 的子数组中选较大的：

```java
public int maxSubArray(int[] nums, int k) {
    int len = nums.length;
    // globalMax[i][j]: the maximum sum of j non-overlapping subarrays in the first i elements, nums[i-1] not necessarily included
    int[][] globalMax = new int[len + 1][k + 1];
    // localMax[i][j]: the maximum sum of j non-overlapping subarrays in the first i elements, nums[i-1] included
    int[][] localMax = new int[len + 1][k + 1];

    for (int j = 1; j <= k; ++j) {
        // when i < j, the values are nonsense so set them as MIN_VALUE to not influence following updates
        localMax[j - 1][j] = globalMax[j - 1][j] = Integer.MIN_VALUE;
        for (int i = j; i <= len; ++i) {
            // 2 conditions:
            // 1. nums[i-1] seen as an independent subarray, need j-1 subarrays in the first i-1 elements
            // 2. nums[i-1] is included in the previous subarray, then j subarrays is needed in the first i-1 elements and nums[i-2] must be included
            localMax[i][j] = Math.max(localMax[i - 1][j], globalMax[i - 1][j - 1]) + nums[i - 1];

            // 2 potential updates: nums[i-1] included; nums[i-1] not included
            globalMax[i][j] = Math.max(localMax[i][j], globalMax[i - 1][j]);
        }
    }

    return globalMax[len][k];
}
```

这道题实现还是有一点挑战，主要是边界值的问题，i 或者 j 为 0 时初值为 0 没有问题，不影响结果；但是 i < j 的部分是无用的且不应更新的，而 dp\[j-1\]\[j\] 会在更新对角线元素的时候起作用，所以要将这些值设定为 `Integer.MIN_VALUE`，使之不影响对角线元素。

# Maximum Product Subarray

## Source

* LeetCode: [(152) Maximum Product Subarray](https://leetcode-cn.com/problems/maximum-product-subarray/)

## Description

>Given an integer array nums, find the contiguous subarray within an array (containing at least one number) which has the largest product.<br>
>
>**Example 1**:<br>
>Input: [2,3,-2,4]<br>
>Output: 6<br>
>Explanation: [2,3] has the largest product 6.<br>
>
>**Example 2**:<br>
>Input: [-2,0,-1]<br>
>Output: 0<br>
>Explanation: The result cannot be 2, because [-2,-1] is not a subarray.

## Analysis & Code

严格来说这题不属于 Maximum Subarray，因为求的是最大积子数列，但是和前面的思路是一脉相承的，所以这里放在一起了。

依然是在遍历数组的过程中更新最终的结果 `res`，和前面一样，要考虑 `nums[i]` 的影响，就要同时更新一个 `localMax` 数组表示包含 `nums[i-1]` 的最大乘积子数组。`nums[i]` 大于0时的更新策略和前面一模一样，变成乘积稍有不同的是，`nums[i]` 小于 0 时，带有 `nums[i-1]` 的 `local[i-1]` 子数组乘积越小，`local[i]` 越大，所以还要记录并更新一个 `localMin` 数组表示包含 `nums[i-1]` 的最小乘积子数组。

```java
public int maxProduct(int[] nums) {
    int len = nums.length;
    int[] globalMax = new int[len + 1];
    int[] localMax = new int[len + 1];
    int[] localMin = new int[len + 1];

    globalMax[1] = localMax[1] = localMin[1] = nums[0];
    int res = nums[0];
    for (int i = 2; i <= len; ++i) {
        if (nums[i - 1] > 0){
            localMax[i] = Math.max(nums[i - 1], nums[i - 1] * localMax[i - 1]);
            localMin[i] = Math.min(nums[i - 1], nums[i - 1] * localMin[i - 1]);
        }
        else {
            localMax[i] = Math.max(nums[i - 1], nums[i - 1] * localMin[i - 1]);
            localMin[i] = Math.min(nums[i - 1], nums[i - 1] * localMax[i - 1]);
        }
        globalMax[i] = Math.max(globalMax[i - 1], localMax[i]);
        res = Math.max(res, globalMax[i]);
    }
    return res;
}
```



# References

1. https://www.kancloud.cn/kancloud/data-structure-and-algorithm-notes/73086
2. https://zhengyang2015.gitbooks.io/lintcode/maximum_subarray_iii_43.html

