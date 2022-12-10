---
title: 「深入学习 Golang」之 GPM
tags:
  - Golang
  - GMP
categories: 好好学习
keywords: 'blog,golang,php'
description: 一些 golang 的重点和难点
cover: >-
  https://graph.linganmin.cn/201007/87122d3084205a5821b1e39413e84349?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/201007/87122d3084205a5821b1e39413e84349?x-oss-process=image/format,webp/quality,q_40
abbrlink: 943537f7
date: 2020-10-03 15:40:34
---

## 相关术语

### runtime

`runtime`在 Golang 程序中很重要，`runtime`包含了调度、内存、垃圾回收、内部数据结构、定时器和各种系统调用的封装等。

### scheduler

`scheduler`是指调度器，主要工作是将准备好运行的`Goroutine`分散到工作线程中执行。

### TLS(thread local storage)

`TLS`代表每个线程中的本地数据，每个线程写入 TLS 的数据相互独立互不影响，Golang 的协程非常依赖 TLS 机制。TLS 里会存储当前线程中的`Goroutine`和其所属的`Machine`实例。

### spining

`spining`表示重复某块代码。

### systemstack、mcall 或 asmcgocall

TODO 这个没看懂。。。。。。

## 调度模型（GPM）

### 线程模型

{% note danger %}

Golang 中的`goroutine`和`go scheduler`的底层实现都是属于`两级线程模型`

{% endnote %}

> `KSE`是操作系统本身内核态的线程的简称（Kernel Scheduler Entities）。

我们现在的计算机语言，可以狭义的认为是一种“软件”，它们中所谓的“线程”，往往是用户态的线程，用户态线程和操作系统本身内核态线程是有区别的。

#### 内核级线程模型（1:1）

内核级线程是`用户态线程`和`内核态线程(KSE)`**一对一**的映射模型。也就是每一个用户态线程都会绑定且只会绑定一个真实的`内核线程`，<u>线程的调度完全交给操作系统去做，应用程序对线程的创建、终止以及同步都是基于内核提供的系统调用来完成的。</u>大部分编程语言的线程库，如：Linux 的 pthread，Java 的 java.lang.Thread 都是对操作系统系统的线程(内核态线程)的一层封装。这种方式实现简单，直接借助 OS 提供的线程能力，并且不同用户线程之间一般不会互相影响，但是<u>其创建、销毁以及多个线程之间的上下文切换等操作都需 OS 层面亲自操作，在线程急剧增多的场景下对 OS 的性能影响会很大。</u>

- 优点
  - 简单、直接并行
  - 在多核处理器的硬件的支持下，内核空间线程模型支持了真正的并行，当一个线程被阻塞后，允许另一个线程继续执行，所以并发能力较强。
- 缺点
  - 成本高
  - 每创建一个用户级线程都需要创建一个内核级线程与其对应，这样创建线程的开销比较大，会影响到应用程序的性能。

#### 用户级线程模型（M:1）

用户级线程是`用户态线程`和`内核态线程(KSE)`**多对一**的模型映射。也就是多个用户态线程一般从属单个进程。这种用户态线程的创建、销毁以及多个线程之间的协调等操作都是由用户自己实现的线程库来维护，一个进程中创建的线程都与同一个 KSE 在运行时动态关联。很多编程语言实现协程都属于这种方式，这种方式相比内核级线程可以做的很轻量级，对资源的消耗会小很多，因此可以创建的数量和上下文切换锁花费的成本也会小很多。

该模型的致命缺点是，如果某个用户态线程上调用阻塞式系统调用（如：阻塞式网络IO），一旦 KSE 因阻塞被内核调度出 CPU ，剩下所有对应的用户态线程都会变成阻塞状态（整个进程挂起）。

- 优点
  - 创建成本低
  - 这种模型的线程上下文切换都发生在用户空间，避免模态切换（mode switch)，从而对于性能有积极影响。
- 缺点
  - 并发性能不完全
  - 所有线程基于一个内核态线程（KSE），这意味着只会有一个处理器被利用，用户级线程模型只解决了并发问题，没有解决并行问题。如果线程因为 I/O 操作陷入了内核态，内核态线程阻塞等待 I/O 数据，则所有的线程都会被阻塞，用户空间也可以使用非阻塞 I/O ，但无法避免性能和复杂度问题。

### 两级线程模型(M:N)

两级线程是`用户态线程`和`内核态线程(KSE)`**多对多**的关系。这种实现综合了前两种模型的优点，为一个进程中创建多个 KSE ,并且用户态线程可以和不同的 KSE 在运行时进行动态关联，当某个 KSE 有雨其上下文的线程阻塞操作被内核调度出 CPU 时，当前与其关联的其他用户态线程可以重新与其他 KSE 建立关联关系。

Golang 的并发是使用的这种实现方式，Golang 为了实现该模型，自己实现了一个运行时调度器来维护 Golang 中的 `用户态线程`和`KSE`的动态关联。

这种模型也被成为`混合线程模型`，用户调度器实现用户态线程到 KSE 的调度，内核调度器实现了 KSE 到 CPU 的调度。

> 线程相关知识参考自：https://www.jianshu.com/p/397ade19ad38

### GPM 模型

每一个`goroutine`是一个独立的执行单元，`goroutine`的栈采用动态扩容的内存模式，初始化时仅为2KB，随着任务执行按需增长，且完全由 Golang 自己的调度器`Go Scheduler`来调度。

此外，GC 还会周期性的将不在使用的内存回收，收缩栈空间。因此 Golang 程序可以同时并发成千上万个`goroutine`是得益于它强劲的调度器和高效的内存模型。

#### G

1. G 表示`goroutine`，每个`goroutine`对应一个 G 结构体，<u>G 存储`goroutine`的运行堆栈、状态以及任务函数</u>。

2. G 并非执行体，每个 G都需要绑定到 P 才能被调度执行。

```golang
package main

import (
	"fmt"
	"time"
)

func f() {
	fmt.Println("hello 小下")
}

func main() {

 // go 关键字创建了一个 goroutine，此时这个 goroutine 的任务函数就是 f()
	go f()

	time.Sleep(time.Second)
}


```

#### P

1. P(Processor)，表示逻辑处理器，代表线程 M 执行的上下文，P 不执行任何代码。

2. 对 G 来说 P 相当于 CPU 核，G 只有绑定到 P （在 P 的 local runq中）才能被调度。

3. 对 M 来说 P 提供了相关的执行环境（Context），如内存分配状态(mcache)、任务队列（G）等。

4. P 的数量决定了系统内最大可并行 G 的数量。P 的数量由用户设置的 GOMAXPROCS 决定，但是不论 GOMAXPROCS 设置为多大，P 的数量最大为 256。

#### M

M(Machine)，OS 线程抽象，代表真正执行计算的资源，可以认为它就是`系统线程（os thread）`。在绑定有效的 P 后，进入调度循环(schedule)。

M 是有线程栈的，如果不对该线程栈提供内存的话，系统会给该线程提供内存，当指定了线程栈，则 M.stack -> G.stack, M 的 PC 寄存器指向 G 提供的函数，然后去执行。

调度循环的机制大致是从`Global队列`、P 的`Local 队列`以及`wait 队列`中获取 G，切换到 G 的执行栈上，并执行 G 的函数，然后调用 goexit 做清理工作并回到 M，如此反复。

M 并不会保留 G 的状态，这是 G 可以跨 M 调度的基础，M 的数量是不定的，由 Go Runtime 调整，为了防止创建过多 OS 线程导致系统调度出现问题，目前默认限制为 10000 个。

### 调度过程

> 每个`Processor`都维护一个本地队列

当一个`Goroutine`被创建出来时，会优先将其放在`Processor`的本地队列，如果当前`Processor`的本地队列满了则会放进`Global`队列。

`Machine`绑定一个`Processor`后，会启动一个 OS 线程，循环从`Processor`的本地队列里取出一个`Goroutine`并执行。

{% note info %}
调度算法：
当`Machine`执行完当前绑定的`Processor`本地队列中所有的`Goroutine`后，`Processor`会尝试从`Global队列`中取`Goroutine`来执行，如果`Global队列`也为空，`Processor`会随机从另一个`Processor`中的本地队列里取一半的`Goroutine`到自己的队列。
{% endnote %}

当一个`Goroutine`在`Machine`执行结束后，`Processor`会从本地队列将其取出，如果此时`Processor`的本地队列为空，
