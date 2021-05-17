---
title: LeetCode 之数组
date: 2020-05-11 08:25:05
tags: LeetCode
categories: 编程
mathjax: true
---

## 移动零

### 题目

[移动零](https://leetcode-cn.com/problems/move-zeroes/description/)

给定一个数组 `nums`，编写一个函数将所有 `0` 移动到数组的末尾，同时保持非零元素的相对顺序。

<!-- more -->

**示例:**

```
输入: [0,1,0,3,12]
输出: [1,3,12,0,0]
```

**说明**:

1. 必须在原数组上操作，不能拷贝额外的数组。
2. 尽量减少操作次数。

### 解法

一维数组的坐标变换

- j 指针指向零元素的位置，只要是非零就移动指针指向下一个，如果是零就停下来不再移动，然后 i 指针继续向后走，遇到非零元素，交换两者位置，实现了把 0 向后移动

```go
func moveZeroes(nums []int) {
    j := 0
	for i, n := range nums {
		if n != 0 {
			nums[i], nums[j] = nums[j], nums[i]
			j++
		}
	}
}
```

## 盛最多水的容器

### 题目

[盛最多水的容器](https://leetcode-cn.com/problems/container-with-most-water/description/)

**说明：**你不能倾斜容器，且 *n* 的值至少为 2。

 

![](https://aliyun-lc-upload.oss-cn-hangzhou.aliyuncs.com/aliyun-lc-upload/uploads/2018/07/25/question_11.jpg)

图中垂直线代表输入数组 [1,8,6,2,5,4,8,3,7]。在此情况下，容器能够容纳水（表示为蓝色部分）的最大值为 49。

**示例：**

```
输入：[1,8,6,2,5,4,8,3,7]
输出：49
```

### 解法

1. 枚举：左右的边界，$O(n^2)$
2. 两个下标，左右边界，向中间收敛，左右夹逼，$O(n)$

#### 暴力求解

```go
func maxArea(height []int) int {
	var maxArea int
	for i := 0; i < len(height); i++ {
		for j := i + 1; j < len(height); j++ {
			area := (j - i) * min(height[i], height[j])
			maxArea = max(maxArea, area)
		}
	}
	return maxArea
}
func max(a, b int) int {
	if a < b {
		return b
	}
	return a
}
func min(a, b int) int {
	if a < b {
		return a
	}
	return b
}
```

#### 双指针法

```go
func maxArea(height []int) int {
	maxArea, i, j := 0, 0, len(height)-1
	for i < j {
		area := (j - i) * min(height[i], height[j])
		if height[i] < height[j] {
			i++
		} else {
			j--
		}
		maxArea = max(area, maxArea)
	}
	return maxArea
}
func max(a, b int) int {
	if a < b {
		return b
	}
	return a
}
func min(a, b int) int {
	if a < b {
		return a
	}
	return b
}
```

## 爬楼梯

[爬楼梯](https://leetcode-cn.com/problems/climbing-stairs/description/)

假设你正在爬楼梯。需要 *n* 阶你才能到达楼顶。

每次你可以爬 1 或 2 个台阶。你有多少种不同的方法可以爬到楼顶呢？

**注意：**给定 *n* 是一个正整数。

**示例 1：**

```
输入： 2
输出： 2
解释： 有两种方法可以爬到楼顶。
1.  1 阶 + 1 阶
2.  2 阶
```

**示例 2：**

```
输入： 3
输出： 3
解释： 有三种方法可以爬到楼顶。
1.  1 阶 + 1 阶 + 1 阶
2.  1 阶 + 2 阶
3.  2 阶 + 1 阶
```

### 斐波那契数列法

```go
func climbStairs(n int) int {
	if n < 4 {
		return n
	}
	f1, f2, f3 := 2, 3, 0
	for i := 4; i < n+1; i++ {
		f3 = f1 + f2
		f1 = f2
		f2 = f3
	}
	return f3
}
```

## 两数之和

给定一个整数数组 `nums` 和一个目标值 `target`，请你在该数组中找出和为目标值的那 **两个** 整数，并返回他们的数组下标。

你可以假设每种输入只会对应一个答案。但是，数组中同一个元素不能使用两遍。

**示例:**

```
给定 nums = [2, 7, 11, 15], target = 9

因为 nums[0] + nums[1] = 2 + 7 = 9
所以返回 [0, 1]
```

### 解法

1. 暴力法  $O(n^2)$
2. 哈希表 $O(n)$

```go
func twoSum(nums []int, target int) []int {
    for i := 0; i < len(nums); i++ {
		for j := i + 1; j < len(nums); j++ {
			if nums[i]+nums[j] == target {
				return []int{i,j}
			}
		}
	}
	return []int{}
}
```

- 两遍循环

```go
func twoSum(nums []int, target int) []int {
	indexMap := make(map[int]int)
	for i, v := range nums {
		indexMap[v] = i
	}
	for i, v := range nums {
		value := target - v
		index, ok := indexMap[value]
		if ok && index != i {
			return []int{i, index}
		}
	}
	return []int{}
}
```

- 一遍循环

```go
func twoSum(nums []int, target int) []int {
	indexMap := make(map[int]int)
	for i, v := range nums {
		if index, ok := indexMap[target-v]; ok {
			return []int{index, i}
		}
		indexMap[v] = i
	}
	return []int{}
}
```

## 三数之和

给你一个包含 *n* 个整数的数组 `nums`，判断 `nums` 中是否存在三个元素 *a，b，c ，*使得 *a + b + c =* 0 ？请你找出所有满足条件且不重复的三元组。

**注意：**答案中不可以包含重复的三元组。

**示例：**

```
给定数组 nums = [-1, 0, 1, 2, -1, -4]，

满足要求的三元组集合为：
[
  [-1, 0, 1],
  [-1, -1, 2]
]
```

### 解法

三数之和可以转化为两数之和，要求解的其实是 a + b = -c

1. 暴力求解：三重循环 $O(n^3)$
2. 哈希表 $O(n^2)$
3. 双指针法

- 暴力求解

```go
func threeSum(nums []int) [][]int {
	var (
		result [][]int
	)
	for i, n := range nums {
		for j := i + 1; j < len(nums); j++ {
			for k := j + 1; k < len(nums); k++ {
				if nums[j]+nums[k]+n == 0 {
					result = append(result, []int{n, nums[j], nums[k]})
				}
			}
		}
	}
	return result
}
```

存在的问题：去重比较麻烦

- 暴力+哈希表

```go
func threeSum(nums []int) [][]int {
	var (
		result [][]int
	)
	for i, n := range nums {
		indexMap := make(map[int]int)
		for j := i + 1; j < len(nums); j++ {
			if k, ok := indexMap[-n-nums[j]]; ok {
				result = append(result, []int{n, nums[j], nums[k]})
			} else {
				indexMap[nums[j]] = j
			}
		}
	}
	return result
}
```

存在的问题：去重比较麻烦

- 双指针法

```go
func threeSum(nums []int) [][]int {
	var (
		i, j   int
		result [][]int
	)
	sort.Ints(nums)
	for k := 0; k < len(nums)-2; k++ {
		if nums[k] > 0 {
			break
		}
		if k > 0 && nums[k] == nums[k-1] {
			continue
		}
		i, j = k+1, len(nums)-1
		for i < j {
			s := nums[k] + nums[i] + nums[j]
			if s < 0 {
				i++
			} else if s > 0 {
				j--
			} else {
				result = append(result, []int{nums[k], nums[i], nums[j]})
				i++
				j--
				for i < j && nums[i] == nums[i-1] {
					i++
				}
				for i < j && nums[j] == nums[j+1] {
					j--
				}
			}
		}

	}
	return result
}
```

