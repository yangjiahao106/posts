---
title: 使用快速排序解决TopK问题和数据流的中位数
date: 2019-01-19 21:57:31
tags:
---

# 使用快排思想解决TopK问题和数据流的中位数

## 快速排序

快速排序这里就不多介绍了，Partition函数为快速排序的核心函数，负责选择一个数字将比这个数字小的数字放到左边，大的数字放到右边，最后返回这个数字的下标，下面为使用golang 实现的代码。

```go
// 快速排序
func QuickSort(nums []int, start, end int) {
	if start == end {
		return
	}
	index := Partition(nums, start, end)
	if index > start {
		QuickSort(nums, start, index-1)
	}
	if index < end {
		QuickSort(nums, index+1, end)
	}
}

// Partition 函数用于快速排序
func Partition(nums []int, start, end int) int {
	//随机生成一个下标，此下标中的值作为中间值
	if start >= end {
		return start
	}
	index := rand.Intn(end-start) + start
	nums[index], nums[end] = nums[end], nums[index]
	small := start
	for index = start; index < end; index++ {
		if nums[index] > nums[end] {
			if small != index {
				nums[index], nums[small] = nums[small], nums[index]
			}
			small++
		}
	}
	//small++
	nums[small], nums[end] = nums[end], nums[small]
	// 返回中间值的下标
	return small
}
```

## Top K 问题：最大的K个数字

使用快速排序的分治思想可以将这个问题的时间复杂度降低到O(n)

**减治法** : 减治法比分治法效率更高，相比于分治法，减治法只需要解决分治之后其中一个**子问题**，而分治法需要解决分治后的所有子问题，所以它的时间复杂度为O(n)

``` go
func TopK(nums []int, k int) []int {
	if nums == nil || len(nums) ==0{
		return nil
	} 
	start := 0
	end := len(nums) -1
	index := Partition(nums, start, end)
	for index != k-1 {
		if index > k-1 {
			end = index-1
			index = Partition(nums, start, end)
		} else {
			start = index + 1
			index = Partition(nums, start, end)
		}
	}
	return nums[0:k]
}

```

## 无序数组的中位数

如果数组的个数为奇数则中位数为排序后位于中间的那个数字， 如果个数为偶数，则中位数为排序后中间两个数字的平均值，所以要分两种情况。

此问题同样可以使用快排的partition函数解决，时间复杂度为O(n)

```go
func GetMedian(nums []int) float64 {
	if nums == nil || len(nums) == 0 {
		return 0
	}
	if len(nums)&1 == 1 { // 长度为奇数
		mid := len(nums) / 2
		index := Partition(nums, 0, len(nums)-1)
		for index != mid {
			if index < mid {
				index = Partition(nums, index+1, len(nums)-1)
			} else {
				index = Partition(nums, 0, index-1)
			}
		}
        return float64(nums[mid])
        
	} else { // 长度为偶数
		midR := len(nums) / 2
		midL := (len(nums) / 2) - 1
		res := 0
		findL, findR := false, false
		start := 0
		end := len(nums) - 1
		index := Partition(nums, 0, len(nums)-1)
		for !findL || !findR {
			if index < midL {
				start = index + 1
				index = Partition(nums, start, end)
			} else if index > midR {
				end = index - 1
				index = Partition(nums, start, end)
			}
			if index == midL {
				res += nums[midL]
				findL = true
				start = index + 1
				fmt.Println(start, end, midL)
				index = Partition(nums, start, end)
			} else if index == midR {
				res += nums[midR]
				findR = true
				end = index - 1
				fmt.Println(start, end)
				index = Partition(nums, start, end)
			}
		}
		return float64(res) / 2
	}
}
```



def NumberOf1Between1AndN_Solution(n):
    ones,m =0,1
    while m<=n:
        ones += (n//m+8)//10*m + (n//m%10==1)*(n%m+1)
        m *= 10

--------------------- 
作者：daisyyyyyyyy 
来源：CSDN 
原文：https://blog.csdn.net/u013129109/article/details/79765776 
版权声明：本文为博主原创文章，转载请附上博文链接！