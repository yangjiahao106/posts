---
title: Leetcode 115 Distinct Subsequences
date: 2019-02-25 16:05:59
tags:
---

# Leetcode 115 Distinct Subsequences
## 问题描述

给定一个字符串 S 和一个字符串 T，计算在 S 的子序列中 T 出现的个数。

一个字符串的一个子序列是指，通过删除一些（也可以不删除）字符且不干扰剩余字符相对位置所组成的新字符串。（例如，"ACE" 是 "ABCDE" 的一个子序列，而 "AEC" 不是）

## 例：

输入: S = "babgbag", T = "bag"
输出: 5
解释:
如下图所示, 有 5 种可以从 S 中得到 "bag" 的方案。 
(上箭头符号 ^ 表示选取的字母)
babgbag
^^ ^
babgbag
^^    ^
babgbag
^    ^^
babgbag
  ^  ^^
babgbag
    ^^^

## 使用动态规划求解

输入: S = "babgbag", T = "bag"

||||b|a|b|g|b|a|g|
|---|---|---|---|---|---|---|---|---|---|
||i/j|0|1|2|3|4|5|6|7|
||0  |1|1|1|1|1|1|1|1|
|b|1 |0|**1**|1|**2**|2|**3**|3|3|
|a|2 |0|0|**1**|1|1|1|**4**|4|
|g|3 |0|0|0|0|**1**|1|1|**5**|

动态规划将一个的问题分解成子的问题，然后从子问题一步步找到最优解。
如上图表所示，可以先求问题S="bababag", T = "b",
再求问题S="bababag",T="ba",
再求S="bababag",T="bag".
时间复杂度为O(mn)

### golang实现代码

```go
func numDistinct(s string, t string) int {
	dp := make([][]int, len(t)+1)
	for i := 0; i < len(t)+1; i++ {
		dp[i] = make([]int, len(s)+1)
	}

	// 将第一行初始化为1
	for i := 0; i < len(s)+1; i++ {
		dp[0][i] = 1
	}

	for i := 1; i < len(t)+1; i++ {
		for j := 1; j < len(s)+1; j++ {
			if s[j-1] == t[i-1] {
				dp[i][j] = dp[i-1][j-1] + dp[i][j-1]
			} else {
				dp[i][j] = dp[i][j-1]
			}
		}
	}

	return dp[len(t)][len(s)]
}


```


