---
title: median of two sorted arrays
tags:
  - leetcode
  - algorithm
categories:
  - algorithm
abbrlink: d67af418
date: 2016-12-30 18:53:33
---

## [问题](https://leetcode.com/problems/median-of-two-sorted-arrays/)

There are two sorted arrays nums1 and nums2 of size m and n respectively.

Find the median of the two sorted arrays. The overall run time complexity should be O(log (m+n)).

> 求两个有序数组的中位数, 时间复杂度需要满足O(log(m+n))

Example 1:
nums1 = [1, 3]
nums2 = [2]

The median is 2.0
Example 2:
nums1 = [1, 2]
nums2 = [3, 4]

The median is (2 + 3)/2 = 2.5

## 想法

将两个数组合并成一个临时数组，然后在新数组中求其中位数, 如果新数组长度为基数, 中位数为`arr[(n+1)/2]`,
否则为`(arr[(n-1)/2] + arr[n/2])/2`。 该算法的时间复杂度为O(max(m,n)),
空间复杂度为O(m+n), 时间复杂度不能满足题目要求，需要另寻他法。

思前想后没有想到好的算法，最后查看了leetcode中的top solution, 算法部分分析和处理
的很是精妙, 时间复杂度为O(log(min(m,n)))。 下面是原文的翻译，
如有需要可以点击[*这里*](https://discuss.leetcode.com/topic/4996/share-my-o-log-min-m-n-solution-with-explanation)

> 为了解决这个问题，我们需要理解什么是中位数。在统计学上，中位数用于将一个集合拆分成
两个同等大小的子集合，其中一个子集的任何一个元素都大于另外一个子集。如果我们明白了
中位数是用于划分相同子集的，就能够很容易地想出解决方案。

> 首先我们选取任意的位置*i*把数组*A*划分成两个部分
```
      left_A             |        right_A
A[0], A[1], ..., A[i-1]  |  A[i], A[i+1], ..., A[m-1]
```
>假设**A**有**m**个元素，则存在**m+1**中切分的方法(i取0~m之间的任意值), 并且需要满足: len(left_A) = i,
len(right_A) = m -i(当i=0的时候，left_A即为空; i=m的时候, right_A即为空)
按照同样的方法，我们可以选取任意的位置j把B分为两个部分:
```
      left_B             |        right_B
B[0], B[1], ..., B[j-1]  |  B[j], B[j+1], ..., B[n-1]
```
>把left_A和left_B放在一起组成一个集合left_part，把right_A和right_B也放在一起组成另外一个集合right_part:
```
      left_part          |        right_part
A[0], A[1], ..., A[i-1]  |  A[i], A[i+1], ..., A[m-1]
B[0], B[1], ..., B[j-1]  |  B[j], B[j+1], ..., B[n-1]
```

> 如果我们可以保证:
```
1) len(left_part) == len(right_part)
2) max(left_part) <= min(right_part)
```
>那么我们就可以把{A,B}里面的所有元素分成两个有着相同长度的部分,  并且其中一个部分永远大于另外一部分。
那么中位数就等于`(max(left_part) + min(right_part))/2`
为了确保这两个条件，我们只需要保证:
```
(1) i + j = m - i + n - j (偶数)  or  i + j = m - i + n - j + 1 (基数)
    if n >= m 只需要保证: i在0~m之间, j = (m+n+1)/2 -i (偶数加1除2保持值不变)
(2) B[j-1] <= A[i] and A[i-1] <= B[j]
```

>注释1: 为了简化处理，这里假设A[i-1], B[j-1], A[i], B[j]都是有效的，后面我们会详细谈到边界的处理

>注释2: 为什么n>=m? 因为需要确保当0 <= i <= m 并且 j = (m+n+1)/2 -i 时，j是非负数。 如果 n < m,
j可能会出现负数的情况，会造成数组越界的情况。

> 因此，我们需要做的是:
```
在[0, m]寻找i, 使其能够满足:
    B[j-1] <= A[i] and A[i-1] <= B[j], ( where j = (m + n + 1)/2 - i )
```

>我们可以使用二分法来处理，具体流程如下:
```
<1> 设置 imin = 0, imax = m, 然后开始在[imin, imax]中寻找i

<2> 设置 i = (imin + imax)/2, j = (m+n+1)/2 -i

<3> 让左边部分的长度等于右边，会遇到下面三种情况:
    <a> B[j-1] <= A[i] and A[i-1] <= B[j]
        意味着我们找到到目标i, 可以停止寻找
    <b> B[j-1] > A[i]
        意味着 A[i]的太小了，我们需要调整i的值使其满足B[j-1] <= A[i]
        A是升序数组，如果要增大A[i], 需要增大i值。即我们需要调整寻找区间为
        [i+1, imax], 因此我们让imin等于i+1, 并跳转至<2>继续执行
    <c> A[i-1] > B[j]
        意味着A[i-1]的值太大了，需要减少i的值使其满足A[i-1] <= B[j]。即我们需要
        调整寻找区间为[imin, imax-1], 因此我们让imax等于i-1, 并跳转至<2>继续执行
```

>当对象i寻找到的时候，中位数即为:
```
max(A[i-1], B[j-1]) (when m + n is odd)
or (max(A[i-1], B[j-1]) + min(A[i], B[j]))/2 (when m + n is even)
```

>现在让我们来考虑下边界值 i=0, i=m, j=0, j=n, 这些边界值会导致 A[i-1], B[j-1], A[i], B[j]
一些值不存在。事实上边界值的情况会比我们想的更简单些, 如果A[i-1], B[j-1], A[i], B[j]中的
一些不存在，我们就没必要去完全检查所需要的条件(B[j-1] <= A[i] and A[i-1] <= B[j])。例如,
如果i=0, 则A[i-1]就不存在，我们就不需要检查A[i-1] <= B[j]。因此，我们只需要做:
```
在[0, m]寻找i, 使其能够满足:
    (j == 0 or i == m or B[j-1] <= A[i]) and
    (i == 0 or j == n or A[i-1] <= B[j])
    where j = (m + n + 1)/2 - i>)>)
```

>在一个循环中，我们会遇到一下三种情况:
```
<a> (j == 0 or i == m or B[j-1] <= A[i]) and
    (i == 0 or j = n or A[i-1] <= B[j])
    Means i is perfect, we can stop searching.

<b> j > 0 and i < m and B[j - 1] > A[i]
    Means i is too small, we must increase it.

<c> i > 0 and j < n and A[i - 1] > B[j]
    Means i is too big, we must decrease it.
```

## 算法

### 描述

上面的想法中详细描述了算法的步骤，这里不再赘述

### codes

- [golang](https://github.com/kirk91/leetcode/blob/master/algorithms/median_of_two_sorted_arrays/main.go)
```golang
func findMedianSortedArrays(a, b []int) float64 {
	m, n := len(a), len(b)
	if m > n {
		a, b, m, n = b, a, n, m
	}
	l, r := 0, m
	halfLen := (m + n + 1) / 2

	var i, j int
	var median float64
	// binary search
	for l <= r {
		i = (l + r) / 2
		j = halfLen - i

		if i < m && b[j-1] > a[i] {
			l++
		} else if i > 0 && a[i-1] > b[j] {
			r--
		} else {
			var maxLeft int
			if i == 0 {
				maxLeft = b[j-1]
			} else if j == 0 {
				maxLeft = a[i-1]
			} else {
				maxLeft = max(a[i-1], b[j-1])
			}

			if (m+n)%2 == 1 {
				median = float64(maxLeft)
				break
			}

			var minRight int
			if i == m {
				minRight = b[j]
			} else if j == n {
				minRight = a[i]
			} else {
				minRight = min(a[i], b[j])
			}

			median = float64(maxLeft+minRight) / 2
			break
		}
	}
	return median
}

func max(a, b int) int {
	if a > b {
		return a
	}
	return b
}

func min(a, b int) int {
	if a > b {
		return b
	}
	return a
}
```
