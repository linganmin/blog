---
title: 「深入学习 Golang」之 Channel
tags:
  - Golang
  - Channel
categories: 好好学习
keywords: 'blog,golang,php'
description: golang 的 Channel
cover: >-
  https://graph.linganmin.cn/201007/87122d3084205a5821b1e39413e84349?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/201007/87122d3084205a5821b1e39413e84349?x-oss-process=image/format,webp/quality,q_40
abbrlink: 93eb20a1
date: 2020-08-07 05:30:20
---

## 设计原理

> 不要通过共享内存的方式进行通信，而应该通过通信的方式共享内存。

在其他语言中，多个线程传递数据的方式一般是共享内存，为了解决线程的冲突就需要限制同一时间能够读写这些变量的线程数，在 go 语言中并不需要这么做，因为 Golang 提供了一种不同的并发模型，也就是`通信顺序进程（Communicating sequential process, CSP）`。

### Golang 中的 CSP 实现

`Goroutine`和`Channel`分别对应 CSP 中的`实体`和`传递信息的媒介`。Go 语言中的`Goroutine`可以通过`Channel`传递数据。

#### 先入先出

目前的 Channel 收发操作遵循了`先入先出(FIFO)`的设计，具体规则如下：

- 先从 Channel 读取数据的 Goroutine 会先接收到数据
- 先向 Channel 发送数据的 Goroutine 会得到先发送数据的权力

## 数据结构

Go 语言的 Channel 在运行时使用`runtime.hchan`结构体表示，我们在 Go 语言创建新的 Channel 时，实际上创建的都是如下所示的结构体：

```golang

type chan struct {
  qcount uint        // Channel 中元素个数
  dadaqsiz uint      // Channel 中环形队列的长度
  buf unsafe.Pointer // Channel 中缓冲区的数据指针,指向一个长度为 dataqsiz 的数组
  elemsize uint16    // Channel 中能收发元素的大小
  elemtype *_type    // Channel 中能收发元素的类型
  closed uint32
  sendx uint         // Channel 中发送操作处理到的位置
  recvx uint         // Channel 中接收操作处理到的位置
  recvq waitq
  sendq waitq

  lock mutex

}

```

`recvq`和`sendq`存储了当前 Channel 中由于缓冲区空间不足而阻塞的`Goroutine`列表，这些等待列表使用双向链表`runtime.waitq`表示，链表中所有元素都是`runtime.sudog`结构

```golang
type waitq struct {
  first *sudog
  last *sudog

}

```

`runtime.sudog`表示一个在等待列表中的 Goroutine，该结构中存储阻塞的相关信息以及两个分别指向前后`runtime.sudog`的指针。

## 创建 Channel

Go 语言中所有的 Channel 的创建都需要使用`make`关键字。

make 函数在创建 channel 的时候回在该进程申请一块内存，创建一个 hchan 的结构体，返回执行该内存的指针，所以获取的 ch 变量本身就是一个指针。

hchan 用一个环形队列保存 Goroutine 之间传递的数据（如果当前创建的 Channel 有缓冲区），使用两个list保存向该 ch 发送和从该 ch 接收数据的 Goroutine，还有一个 mutex 来保证操作这些结构的安全。

> TODO 有时间和精力了需要看这一块的源码

## 发送数据

`ch <- i`表达式想 Channel 发送数据时遇到的几种情况

- 如果当前的 Channel 的`recvq(消费队列)`上存在已被阻塞的 Goroutine ，那么会直接将数据发送给当前的 Goroutine，并将其设成下一个运行的协程
- 如果 Channel 存在缓冲区且其中还有空闲量，会将数据直接存储到当前缓冲区`sendx`所在位置
- 如果都不满足以上两种情况，就会创建一个`sudog`结构，并加入到 Channel 的`sendq`队列中，当前 Goroutine 也会陷入等待其他协程从 Channel 接收数据

## 接收数据

`<- ch`或`i,ok <- ch`表达式想 Channel 发送数据时遇到的几种情况

- 如果 Channel 为空，那么就会直接调用`runtime.gopark`挂起当前 Goroutine
- 如果 Channel 已经关闭并且缓冲区没有任何数据，`runtime.chanrecv`函数会直接返回
- 如果 Channel 的`sendq`队列中存在挂起的 Goroutine，就会将`recvx`索引所在的数据拷贝到接收变量所在的内存空间上，并将`sendq`队列中的 Goroutine 的数据拷贝到缓冲区
- 如果 Channel 的缓冲区中包含数据就会直接读取`recvx`索引对应的数据
- 在默认情况下会挂起当前的 Goroutine，将`runtime.sudog`结构加入`recvq`队列并陷入休眠等待调度器唤醒

## 关闭管道

> TODO 需要读源码:smile