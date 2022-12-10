---
title: 「深入学习 Golang」之 Goroutine
tags:
  - Golang
  - Goroutine
categories: 好好学习
keywords: 'blog,golang,php,闵令安'
description: 一些 golang 的重点和难点
cover: >-
  https://graph.linganmin.cn/201007/87122d3084205a5821b1e39413e84349?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/201007/87122d3084205a5821b1e39413e84349?x-oss-process=image/format,webp/quality,q_40
abbrlink: 3810a6eb
date: 2020-09-15 06:40:34
---

## 进程、线程、协程

### 进程

1. 进程是系统资源分配的最小单位
2. 进程包括`text region`，`data region`和`stack region`等
3. 进程的创建和销毁都是系统资源级别的，因此是一种比较昂贵的操作
4. 进程是抢占式调度，他有三个状态：等待态、就绪态、运行态
5. 进程之间是相互隔离的，每个进程有各自的系统资源，更加安全同时也带来了进程间通信不便的问题
6. 进程是线程的载体

### 线程

1. 同一个进程的多个线程共享进程的资源
2. 每个线程也拥有自己的一少部分独立的资源
3. 线程相比进程更加轻量，同一个进程内的多个线程通信比多个进程间通信容易，同时也带来了同步、互斥和线程安全的问题

### 协程

1. 协程调度不需要内核参与，是完全由用户态程序决定的
2. 协程不是抢占式调度，多个协程进行协作调度，避免了系统切换开销导致 CPU 使用率高

## goroutine

协程并不是 Go 语言的特有机制，Go 语言是在原生语言层面支持的的协程。<u>Go 语言可以通过通信实现 goroutine 之间的数据共享。</u>

> Golang 中，不要通过共享内存来通信，而应该通过通信来共享内存！

1. 创建一个 goroutine 不需要太多的内存，大概 2KB 左右栈空间
2. goroutine 的创建、调度和销毁是`runtime`完成的，`runtime`会被分配一些线程，用来运行所有的 goroutine 在任何时候每个线程都只会运行一个 goroutine，如果一个 goroutine 被阻塞会进入休眠状态，待操作完成后再恢复，另外一个 goroutine 会替换它到对应的线程上运行
3. goroutine 的调度是`协作式`的。当切换 goroutine 时，调度器只需要保存和恢复三个寄存器：程序指针、栈指针和DX，切换开销很小
4. goroutine 的退出机制是，goroutine 的退出只能由本身控制，不允许外部强制结束该 goroutine。两种例外：1.main函数结束，2.程序崩溃结束运行。要实现主 goroutine 控制子 goroutine 的开始和结束，需要借助其他方式，比如context、channel、全局变量

### goroutine 中的三个实体

- G
  - 代表一个 goroutine 对象，每次使用`go`调用的时候都会创建一个 G 对象，它包括`栈`,`指令指针`和调用 goroutine 重要的其他信息，比如阻塞它的 channel
- M
  - 代表一个线程，每次创建一个M的时候，都会用一个底层的线程被创建。所有的G任务最终还是在M上执行
  - M会从绑定的运行队列（P）中取出G，然后运行G，如果G运行完毕或进入休眠状态，则从运行队列中取出下一个G，周而复始
- P
  - 代表一个处理器，每一个运行的M都必须绑定一个P，就像线程必须在每一个cpu核上执行一样。P会调度G到M上运行。P的个数就是`GOMAXPROCS`（最大256），M的个数和P不一定一样多，每个P有自己单独的本地G任务队列，也会有一个全局的G任务队列
  - P可以理解为控制Go代码并行度的机制

相比较大多数并行设计模式，Golang 比较有优势的设计就是P上下文这个概念的出现，<u>如果只有G和M的对应关系，那么当G阻塞在I/O上的时候，M是没有实际工作的，这样造成了资源的浪费，没有了P那么所有的G的列表都放在了全局，会导致临界区太大，对多核调度造成极大影响</u>

### 调度器模型 GPM

相关 GPM 的内容参考：https://blog.linganmin.cn/posts/golang-gpm.html

## 并发控制

### 并行和并发

- 并发
  - 指在同一时刻只能有一条指令执行，但多个进程指令被快速轮换执行，使得在宏观上具有多个任务同时执行的效果，但在微观上并不是同时执行的，只是把时间分成了若干段，使得多个进程快速交替执行。
- 并行
  - 指在同一时刻有多条指令在多个处理器上同时运行。

并发编程在 CPU 密集型的程序中无法提升程序性能，更适用在在 I/O 密集型的程序中。

### 并发实现模型

- 多进程
  - 优点
    - 每个进程独立，都有自己单独的内存空间和上下文信息。
  - 缺点
    - 开销大
    - 进程间通信难
    - 进程间内存共享不方便
- 多线程
  - 优点
    - 开销小
    - 轻量，创建和销毁成本低
    - 线程之间通信和共享内存方便
  - 缺点
    - 频繁创建和销毁会导致开销大，可使用线程池
    - 数据紊乱、死锁
- 协程
  - 优点
    - 不基于操作系统，基于程序，开销小

### Golang 中的 CSP 并发模型

我们常见的多线程模型一般是通过共享内存实现的，但共享内存就会出现如：资源抢占、一致性的问题。为了解决这些问题，需要引入多线程锁、原子操作等限制来保证程序执行结果的正确性。

除了共享内存模型外，还有一个经典的模型就是 CSP 模型。CSP 全拼是 Communicating Sequential Processes，大致意思是 通信顺序进程。CSP 描述了并发系统中的交互模式，是一种面向并发的的语言的源头。

Golang 只使用了CSP中的`Process/Channel`的部分，简单的说就是`Process` 映射=> `Goroutine`，`Channel` 映射=> `Channel`。Goroutine 之间是没有任何耦合，可以完全并发执行，Channel 用于给 Goroutine 传递消息，保持数据同步。

### 控制并发的方法

#### 并发数据安全

- 锁
  - sync.Mutex
    - 互斥锁
  - sync.RWMutex
    - 读写锁
- 原子操作
  - sync
  - atomic

#### 并发行为控制

开发 go 程序时，经常要使用 goroutine 并发处理任务，有时候这些 goroutine 是相互独立的，有时候多个 goroutine 之间往往是需要同步数据通信的，还有一种情况是，主 goroutine 需要控制所属于它的所有子 goroutine 。

实现 goroutine 间通信的方式大致如下

- 全局变量
  - 实现
    - 声明一个全局变量
    - 所有子 goroutine 共享该全局变量，并不断轮训检测变量是否有更新
    - 主 goroutine 更新该变量
    - 子 goroutine 检测到全局变量更新，执行相应逻辑
- Channel 通信
  - 场景
    - 某个任务中的某一个或多个 goroutine 依赖另一个 goroutine 中的条件或产生的结果
  - 实现
    - channel + select
      ```golang
      package main

      import (
        "fmt"
        "time"
      )

      func main() {

        finish := make(chan bool)

        go func() {

          for true {
            fmt.Println("hello 小下")
            // 监听通知
            select {
            case <-finish:
              return
            default:
              time.Sleep(time.Millisecond * 500)
            }
          }

        }()

        go func() {
          // 使用 sleep 模拟耗时
          time.Sleep(time.Second * 2)
          // 通知完成
          finish <- true
        }()

        // 防止 main 过早退出
        time.Sleep(time.Second * 5)
      }

      ```
- Context
  - 场景
    - 多层级的 goroutine 之间的信号传递
  - 实现
    ```golang
    package main

    import (
      "context"
      "fmt"
      "time"
    )

    func foo(ctx context.Context) {

      go bar(ctx)
      for true {
        fmt.Println("hello 小下")

        select {
        case <-ctx.Done():
          fmt.Println("foo exit.....")
          return
        default:
          time.Sleep(time.Millisecond * 400)
        }
      }

    }

    func bar(ctx context.Context) {
      for true {
        fmt.Println("hello world")

        select {
        case <-ctx.Done():
          fmt.Println("bar exit.....")
          return
        default:
          time.Sleep(time.Millisecond * 600)
        }
      }

    }
    func main() {

      ctx, cancel := context.WithCancel(context.Background())

      go foo(ctx)

      // 防止过早发送退出信号
      time.Sleep(time.Second * 2)
      cancel()
      // 防止过早退出
      time.Sleep(time.Second * 5)

    }
    ```