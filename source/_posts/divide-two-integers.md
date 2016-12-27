title: divide two integers
date: 2016-12-26 23:47:29
tags:
  - leetcode
  - algorithm
---

> 最近大脑越来越不听使唤，很多想法到嘴边了却突然忘了表达不出, 真要了老命, 忧伤30s...
  都说好记性不如烂笔头，索性就重新拾起好久未动的blog, 无论是否对记忆有帮助, 起码记录下当前的自己。
  Anyway, let's start it.

## 问题

Divide two integers without using multiplication, division and mod operator.
If it is overflow, return MAX_INT.

> 不用乘法、除法和取模算术符来计算两个整数的商

## 想法

乘除法其实是加减法的变种，理论上来说完全可以用加减法来代替乘除法来进行计算(不考虑时间复杂度),
因此我们可以用加减法来计算两个整数相除, 大概思路如下:

- 加法
一直累加除数，直至其大于被除数，此时的循环次数即为所求的结果, 伪代码如下:
```
tmp = divisor
shift = 0
while dividend >= tmp
    tmp += divisor
    shift++
```

- 减法
一直递减被除数，直至其小于除数，此时的循环次数即为所求的结果, 伪代码如下:
```
tmp = dividend
shift = 0
while tmp >= divisor
    tmp -= divisor
    shift++
```

上述的加减法的确能够解决问题，并且在一些情况下效率还蛮高，但在一些极端的情况下时间复杂度却为O(n),
因此我们需要想办法提高收敛速度，减少循环次数降低时间复杂度。在乘法和除法被限制使用的情况下，我们
只能放出大杀器移位操作(左移一位相当于乘以2， 右移一位相当于除以2)。
另外在数学上，任何一个整数n都可以表示成`n = a0*2^0 + a1*2^1 + ... + an*2^n`, 间接说明了移位操作的可行性。

## 算法

### 描述

1. 初始化变量result
2. 如果被除数dividend大于等于除数divisor, 执行3, 否则执行6
3. 初始化临时变量shift
4. 被除数左移位至`dividend < (divisor << shift)`, 然后让dividend等于`dividend - (divisor << (shift -1))`
5. 让result加上左移的值, 跳转到2继续执行
6. 输出result

### codes

- [golang](https://github.com/kirk91/leetcode/blob/master/algorithms/divide_two_integers/main.go):
```golang
func divide(dividend int, divisor int) int {
	if divisor == 0 {
		return math.MaxInt64
	}
	if dividend == 0 {
		return 0
	}

	sign := 1
	if (dividend > 0 && divisor < 0) || (dividend < 0 && divisor > 0) {
		sign = -1
	}
	if dividend < 0 {
		dividend = -dividend
	}
	if divisor < 0 {
		divisor = -divisor
	}

	var result int
	for dividend >= divisor {
		var shift uint
		for dividend >= divisor<<shift {
			shift++
		}
		dividend -= divisor << (shift - 1)
		result += 1 << (shift - 1)
	}

	if sign == -1 {
		return -result
	}
	return result
}
```

## Q&A
写的比较仓促，如有错误或者未考虑到的问题，欢迎指出😊
