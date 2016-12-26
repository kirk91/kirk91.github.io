title: divide two integers
date: 2016-12-26 23:47:29
tags:
  - leetcode
  - algorithm
---

> 最近脑子越来越不听使唤，很多时候都会hang住30s秒，真要了老命。从小被教育好记性不如烂笔头，
> 就索性重新拾起好久未动的blog，anyway起码记录下当前的自己。

## 问题

Divide two integers without using multiplication, division and mod operator.

If it is overflow, return MAX_INT.

> 不用乘法、除法和取模算术符来计算两个整数的商

## 想法

除法其实是加减法的升级版，可以使用加法或者减法来求商。使用加法的话，可以一直累加除数，直到其
大于被除数, 商即为循环次数； 减法的话，则一直递减被除数直到其小于除数，商即为循环次数。

使用普通的加法或者减法，在一些极端的情况下效率会非常低，e.g. 100 divide 1, 1000 divide 1, 10000 divide 1;
为此我们需要提高收敛速度，在乘法和除法被限制使用的情况下，只能使用移位操作。 另外在数学上，任何一个整数n
都可以表示成 `n = a0 * 2^0 + a1 * 2^1 + a2 * 2^2 + ... + an * 2^n`, so问题迎刃而解.
