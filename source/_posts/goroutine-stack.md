---
title: 聊一聊goroutine stack
tags:
  - go
  - goroutine
  - stack
categories:
  - go
abbrlink: 2d571d09
date: 2017-07-29 17:48:51
---

> 推送在外卖订餐中扮演着重要的角色，为商家实时接单、骑手实时派单提供基础的数据通道。早期推送是由第三方服务商提供
的，随着业务复杂度的提升、订单量和用户数的持续增长，之前的系统已经远远不能满足需求，构建一个高性能、高可用的推送系统
势在必行。上半年我们用go开发了一个hybrid push服务，用户在线通过长连接下发消息，不在线借助厂商或第三方下发消息，在
构建过程中遇到了些与routine stack相关的问题，这里就和大家扯一扯。

带着问题阅读，才能让阅读更加高效，首先让我们看下问题:

1. *goroutine stack多大呢？是固定的还是动态变化的呢?*
2. *stack动态变化的话，什么时候扩容和缩容呢?*
3. *扩容和缩容是如何实现的呢? 对服务有什么影响吗?*

问题明确了，我们就开始往下扯呗。

## 栈大小

为了能够更清楚地描述goroutine stack，我们先看下Linux进程内存布局:

![](/images/linux-process-memory-layout.png)

user stack的大小是固定的，Linux中默认为8192KB，运行时内存占用超过上限程序会崩溃掉并报告segment错误。
为了修复这个问题，我们可以增大内核参数中的stack size, 或者在创建线程时显式地传入所需要大小的内存块。 这两种方案
都有自己的优缺点, 前者比较简单但会影响到系统内所有的thread，后者需要开发者精确计算每个thread的大小负担比较高。

有没有办法既不影响所有thread又不会给开发者增加太多的负担呢? 答案当然是有的，比如: 我们可以在函数调用处插桩，每次调用的时候检查
当前栈的空间是否能够满足新函数的执行，满足的话直接执行，否则创建新的栈空间并将老的栈拷贝到新的栈然后再执行。 这个想法听起来很fancy & simple, 但
当前的linux thread model却不能满足，实现的话只能在用户空间且有不小的难度。

go作为一门21世纪的现代语言，定位于简单高效，自然不能够少了这个特性，它使用内置的运行时runtime优雅地解决了这个问题，每个routine在初始化
时stack大小都为2KB, 在运行过程中会根据不同的场景做动态的调整。

## 栈扩容和缩容

go在1.3之前栈是分段栈Segmented Stack, 栈空间不够用的时候申请一块新的空间用于被调函数的执行，执行后销毁新申请的空间并返回到老的栈空间继续执行，
在函数频繁调用的时候可能会引发hot split问题；为了避免这个问题，1.3之后栈的管理改为了连续栈Contiguous Stack, 在栈不够用的时候申请一个2X大小的
新栈，并把数据拷贝过去并替换到新栈, 接下来所有的执行都发生在新栈上。

在了解栈管理之前，我们先看下栈的内存布局和一些基本的数据结构

![](/images/linux-goroutine-stack-layout.png)

stack.lo和stack.hi分别为栈的低地址和高地址，StackGuard为保护区的大小，StackSmall小函数调用的优化。
在发生函数调用时，根据被调用函数的栈帧大小可以分为三种情况

1. 小于StackSmall

    SP小于stackguard0, 执行栈扩增，否则直接执行。

2. 大于StackSamll, 小于StackBig

    SP - Function's Stack Frame Size + StackSmall 小于stackguard0, 执行栈扩增，否则直接执行。

3. 大于StackBig

    执行栈扩增


下面我们通过一个简单的函数调用，来观察下栈的情况。

```go
package main

func main() {
	a, b := 1, 2
	_ = add(a, b)
}

func add(x, y int) int {
	_ = make([]byte, 200)
	return x + y
}
```

编译(禁用优化和内敛) `go tool compile -N -l -S stack.go > stack.s` , 部分汇编码如下:

```assembly
"".main t=1 size=88 args=0x0 locals=0x30
	0x0000 00000 (stack.go:3)	TEXT	"".main(SB), $48-0
	0x0000 00000 (stack.go:3)	MOVQ	(TLS), CX
	0x0009 00009 (stack.go:3)	CMPQ	SP, 16(CX)
	0x000d 00013 (stack.go:3)	JLS	81
	0x000f 00015 (stack.go:3)	SUBQ	$48, SP
	0x0013 00019 (stack.go:3)	MOVQ	BP, 40(SP)
	0x0018 00024 (stack.go:3)	LEAQ	40(SP), BP
	0x001d 00029 (stack.go:3)	FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x001d 00029 (stack.go:3)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x001d 00029 (stack.go:4)	MOVQ	$1, "".a+32(SP)    ;参数压栈
	0x0026 00038 (stack.go:4)	MOVQ	$2, "".b+24(SP)
	0x002f 00047 (stack.go:5)	MOVQ	"".a+32(SP), AX
	0x0034 00052 (stack.go:5)	MOVQ	AX, (SP)
	0x0038 00056 (stack.go:5)	MOVQ	"".b+24(SP), AX
	0x003d 00061 (stack.go:5)	MOVQ	AX, 8(SP)
	0x0042 00066 (stack.go:5)	PCDATA	$0, $0
	0x0042 00066 (stack.go:5)	CALL	"".add(SB)         ;调用add函数
	0x0047 00071 (stack.go:6)	MOVQ	40(SP), BP
	0x004c 00076 (stack.go:6)	ADDQ	$48, SP
	0x0050 00080 (stack.go:6)	RET
	0x0051 00081 (stack.go:6)	NOP
	0x0051 00081 (stack.go:3)	PCDATA	$0, $-1
	0x0051 00081 (stack.go:3)	CALL	runtime.morestack_noctxt(SB)
	0x0056 00086 (stack.go:3)	JMP	0
    ...
"".add t=1 size=151 args=0x18 locals=0xd0
	0x0000 00000 (stack.go:8)	TEXT	"".add(SB), $208-24
	0x0000 00000 (stack.go:8)	MOVQ	(TLS), CX     ;获取线程local storage地址并放入CX寄存器
	0x0009 00009 (stack.go:8)	LEAQ	-80(SP), AX   ;计算SP - CurrentFrameSize + StackSmall并放入AX寄存器
	0x000e 00014 (stack.go:8)	CMPQ	AX, 16(CX)    ;与当前routine StackGuard0地址比较
	0x0012 00018 (stack.go:8)	JLS	141               ;地址小于StackGuard0，栈空间不够用跳转到141， 否则继续执行
	0x0014 00020 (stack.go:8)	SUBQ	$208, SP
	0x001b 00027 (stack.go:8)	MOVQ	BP, 200(SP)
	0x0023 00035 (stack.go:8)	LEAQ	200(SP), BP
	0x002b 00043 (stack.go:8)	FUNCDATA	$0, gclocals·54241e171da8af6ae173d69da0236748(SB)
	0x002b 00043 (stack.go:8)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x002b 00043 (stack.go:8)	MOVQ	$0, "".~r2+232(FP)
	0x0037 00055 (stack.go:9)	MOVQ	$0, ""..autotmp_0(SP)
	0x003f 00063 (stack.go:9)	LEAQ	""..autotmp_0+8(SP), DI
	0x0044 00068 (stack.go:9)	XORPS	X0, X0
	0x0047 00071 (stack.go:9)	DUFFZERO	$247
	0x005a 00090 (stack.go:9)	LEAQ	""..autotmp_0(SP), AX
	0x005e 00094 (stack.go:9)	TESTB	AL, (AX)
	0x0060 00096 (stack.go:9)	JMP	98
	0x0062 00098 (stack.go:10)	MOVQ	"".x+216(FP), AX
	0x006a 00106 (stack.go:10)	MOVQ	"".y+224(FP), CX
	0x0072 00114 (stack.go:10)	ADDQ	CX, AX
	0x0075 00117 (stack.go:10)	MOVQ	AX, "".~r2+232(FP)
	0x007d 00125 (stack.go:10)	MOVQ	200(SP), BP
	0x0085 00133 (stack.go:10)	ADDQ	$208, SP
	0x008c 00140 (stack.go:10)	RET
	0x008d 00141 (stack.go:10)	NOP
	0x008d 00141 (stack.go:8)	PCDATA	$0, $-1
	0x008d 00141 (stack.go:8)	CALL	runtime.morestack_noctxt(SB)  ;调用morestack_noctxt扩展栈空间
	0x0092 00146 (stack.go:8)	JMP	0                                 ;扩展后跳转到0，继续执行本函数
```
