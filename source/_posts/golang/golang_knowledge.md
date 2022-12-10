---
title: Golang 知识点梳理
tags:
  - Golang
categories: 好好学习
keywords: 'blog,golang,php'
description: 学习 Golang 的知识点梳理
cover: >-
  https://graph.linganmin.cn/200822/43c6bc33f3be83618d4f9cc69bf00fb0?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/200822/43c6bc33f3be83618d4f9cc69bf00fb0?x-oss-process=image/format,webp/quality,q_40
abbrlink: d90f8fc7
date: 2020-03-10 12:40:34
---

## `new`和`make`的区别

```golang
package main

import "fmt"

// make 和 new 的区别

// new(T) 是为一个 T 类型的变量分配内存空间并将此空间初始化为 T 的零值。返回的是新值的地址，也就是 T 类型的指针 *T，该指针指向 T 的新分配的零值。很少使用 new

// make 只能用于 slice、map、channel 这三种类型。
// make(T, args...) 返回的是初始化之后的 T 类型的值，这个值并不是 T 类型的零值，也不是 *T ，是经过初始化之后的 T 的引用。

func main() {

	_new()

	_make()
}

func _new() {
	a := new(int)

	fmt.Printf("a.value.pointer:%+v\n", a) // a.value.pointer:0xc000014090
	fmt.Printf("a.value:%+v\n", *a) // a.value:0

	c := new([]string)
	fmt.Printf("c.value.pointer:%+v\n", c) // c.value.pointer:&[]
	fmt.Printf("c.value:%+v\n", *c) // c.value:[]
}

func _make() {

	b := make([]string, 0)

	fmt.Printf("b.value:%+v\n", b) // b.value:[]

}

```

## 数组&切片

> 长度是数组类型的一部分，不同长度的数组是不能比较的。

> 数组作为参数传递给函数时，是值传递（不是引用传递）。所以在函数内对数组做任何修改，不会影响原数组。

### 切片
#### 扩容机制

当切片长度小于1024时，双倍扩容。当切片长度大于1024时，每次扩大1/4，直到满足位置。


`当切片扩容后，会被分配新的数组存储空间，如果扩容之前某个指针指向了切片中的某个元素，那么扩容后，无论如何修改切片内容，之前的指针永远指向的是扩容之前的旧值。`

```golang
	s1 := []int{0, 1, 2, 3}

	fmt.Printf("s1.pointer:%p,s1.length:%+v,s1.cap:%+v\n", &s1, len(s1), cap(s1)) // // a.value:0xc0000b8000,a.value.v:0

	// a 指向扩容前的元素
	a := &s1[0]
	fmt.Printf("a.value:%+v,a.value.v:%+v\n", a, *a) // a.value:0xc0000b8000,a.value.v:0

	// 扩容后
	s1 = append(s1, 4, 5)
	fmt.Printf("s1.pointer:%p,s1.length:%+v,s1.cap:%+v\n", &s1, len(s1), cap(s1)) // s1.pointer:0xc0000a6020,s1.length:5,s1.cap:8

	// 修改 第一个元素的值
	s1[0] = 10
	fmt.Printf("s1[0].poniter:%+v,s1[0].value:%+v\n", &s1[0], s1[0])
	fmt.Printf("a.value:%+v,a.value.v:%+v\n", a, *a) // a.value:0xc0000b8000,a.value.v:0

```

#### 切片指针和指针切片

## 函数

### 参数传递是传值还是传引用

在 Go 语言中，`所有的传参都是值传递`，都是传的一个副本、一个拷贝。因为拷贝的内容有时候是`非引用类型（int、string、struct等）`，这种类型的传递在函数中是无法修改内容数据的。有的拷贝的则是`引用类型（指针、map、slice、chan等）`，这种就可以在函数找那个修改内容数据。

#### 传值

传值的意思是，函数传递的总是原来这个东西的一个副本（拷贝），比如传递一个`int`类型的参数，传递的其实就是这个参数的一个拷贝；`传递一个指针类型的参数,其实传递的是这个指针的一份拷贝，而不是这个指针指向的值。`

```golang
func main() {

	s := "string"
	ip := &s

	fmt.Printf("原数据的内存地址:%v\n", ip)  // 原数据的内存地址:0xc000010200
	fmt.Printf("原指针的内存地址:%p\n", &ip) // 原指针的内存地址:0xc00000e028
	modify(ip)
	fmt.Printf("值被修改了:%+v\n", s) // 值被修改了:pointer

}

func modify(ip *string) {
	fmt.Printf("函数接收到的参数的指针的内存地址:%p\n", &ip) // 函数接收到的参数的指针的内存地址:0xc00000e038
	fmt.Printf("函数接收到的参数的指针的值:%v\n", ip)     // 函数接收到的参数的指针的值:0xc000010200
	*ip = "pointer"
}
```

通过输出可以看到，这是一个指针的拷贝，因为存放的两个指针的内存地址不同，只是这两个指针的值相同（值都是同一个内存地址）。
这个例子中不管是`0xc00000e028`还是`0xc00000e038`，我们都可以称之为指针的指针，他们指向的是同一个指针`0xc000010200`,当我们改变这个指针的值时也就意味着改变了指向这个指针的变量的值。

#### 传引用（引用传值）

Golang 中不支持传引用。

#### Map

在使用`make`函数创建`map`的时候，`make`函数返回的是一个`hmap`类型的指针`*hmap`，只是使用`map`字面量将其包装为我们省去了指针的操作可以更容易使用map，所以map是引用类型，但不是传引用。

#### Chan

`chan`也是一个引用类型，`make`返回的是一个 `*hchan`

#### Slice

`slice` 也是引用类型

### 匿名函数和闭包

- 匿名函数
  - 匿名函数即没有函数名的函数
  - 因为没有函数名，所以不能像普通函数那样调用，所以匿名函数需要保存到某个变量中或者昨晚立即执行的函数。
  - 匿名函数多用于实现回调函数或闭包

  ```golang
  func main() {

    func(x, y int){
      fmt.Println(x + y)
    }(1, 3)

  }

  ```

- 闭包
  - 闭包是一个函数和其相关的引用环境组合而成的实体，即`闭包=函数+引用环境`
  - **闭包捕获的变量和常量是引用传递，不是值传递**

### defer

Go 语言中`defer`语句会将其后面跟随的语句进行延迟处理，在`defer`归属的函数即将返回时。将延迟的语句按`defer`定义的顺序进行`逆序`执行。

由于`defer`语句延迟调用的特性，所以`defer`语句能非常方便的处理资源释放的问题。比如：资源清理，文件关闭，解锁，记录时间等。

### defer 的执行时机

在 Go 语言的函数中 `return` 语句在底层并不是原子操作，它分为给返回值`赋值`和`RET`指令两步。

- `defer`语句执行的时机就在返回赋值操作之后，`RET`指令执行之前。
  ![defer](https://graph.linganmin.cn/200825/6206c2863a4421f7cededc222247f12d?x-oss-process=image/format,webp/quality,q_60)
- `defer`注册要延迟执行的函数时该函数所有的参数都需要确定其值

每次 defer 语句执行的时候，会把函数`压栈`，**函数参数会被拷贝下来**；当外层函数（非代码块，如一个 for 循环）退出时，defer 函数按照定义的`逆序执行`；如果 defer 执行的函数为 nil, 那么会在最终调用函数的产生 panic.

defer 语句并不会马上执行，而是会进入一个栈，函数 return 前，会按先进后出的顺序执行。也说是说最先被定义的 defer 语句最后执行。先进后出的原因是后面定义的函数可能会依赖前面的资源，自然要先执行；否则，如果前面先执行，那后面函数的依赖就没有了。

在 defer 函数定义时，对外部变量的引用是有两种方式的，分别是作为函数参数和作为闭包引用。**作为函数参数，则在 defer 定义时就把值传递给 defer，并被 cache 起来**；作为闭包引用的话，则会在 defer 函数真正调用时根据整个上下文确定当前的值。

**defer 后面的语句在执行的时候，函数调用的参数会被保存起来，也就是复制了一份。真正执行的时候，实际上用到的是这个复制的变量，因此如果此变量是一个“值”，那么就和定义的时候是一致的。如果此变量是一个“引用”，那么就可能和定义的时候不一致。**

## 结构体

- 结构体指针

  > 当结构体开销比较大的时候尽量使用结构体指针，减少程序的内存开销

  ```golang
  type person struct {
    name string
    age  int
  }
  user := &person{
    name: "xiaoxia",
    age:  18,
  }
  user.age = 20
  fmt.Println(user)
  (*user).age = 21
  fmt.Println(user)

  ```

- 结构体构造函数

  > 构造函数一般使用`new`开头（约定俗成）

  ```golang
  type student struct {
    name  string
    class string
    sno   int
  }

  func newStudent(name string, class string, sno int) *student {
    return &student{
      name:  name,
      class: class,
      sno:   sno,
    }
  }

  student1 := newStudent("xiaoxia", "毕业班", 20200912029)
  fmt.Println(student1)
  ```

### 方法和接受者

```golang

func (接受者变量 接收者类型) 方法名(方法参数) (返回参数) {
  // 函数体
}
```

`方法`是作用于特定类型的函数

> 只能给当前包里的类型加方法

`接受者`表示的是调用该方法的具体类型,多用类型首字母小写表示

```golang

type dog struct {
  name string
}

func main() {

  d := &dog{
    name: "xia",
  }

  d.bark()
}

// bark is method for dog type ,  dog type is receiver, d is dog type shorthand
func (d dog) bark() {

  fmt.Printf("%s: wangwang~", d.name)

}

```

### 值接收者&指针接收者

```golang

type person struct {
  name string
  age  int
}

func main() {

  p := &person{
    name: "xia",
    age:  18,
  }

  fmt.Println(p.age) // 18

  p.birthday()

  fmt.Println(p.age) // 18

  p.birthday2()

  fmt.Println(p.age) // 19

}

// 值接受者，传拷贝值
func (p person) birthday() {
  p.age++
}

// 指针接受者, 传内存地址
func (p *person) birthday2() {
  p.age++
}

```

### 指针接收者使用场景

- 当需要修改接收者中的值
- 当接收者作为拷贝值代价比较大的对象（占内存）
- 保证一致性，如果某个地方方法使用了指针接收者，那么其他地方也需要使用指针接收者

### 匿名字段

## 接口

在 Go 语言中，`接口(interface)`是一种抽象的类型，当看到一个接口类型时，唯一能知道的是通过他能做什么。

`interface`是一组`method`的集合，他做的事就像是定义了一个协议。

### 接口定义

```golang
type 接口类型名 interface{
  方法名(参数列表) 返回值列表
}

```

- 接口名
  - 一般会在单词后面添加`er`，接口名最好突出该接口的类型含义
- 方法名

  - 当方法名首字母是大写，且该接口类型名首字母也是大写时，这个方法可以被接口所在包之外的代码调用

```golang

type hander interface{
  write(content string) bool
  cook()
}

```

> 接口是一个`需要实现的方法的列表`，只要实现了接口中的所有方法，就实现了这个接口。

### 空接口

`空接口`是指没有定义任何方法的接口，因此，任何类型都实现了`空接口`。

`空接口`可以存储任意类型的变量。

- 空接口作为函数的参数，可以接收任意类型的函数参数

  ```golang
  func show(a interface{}) {
    fmt.Printf("type: %T ", a)
  }
  ```

- 空接口作为 map 的值，可以保存任意值的字典

  ```golang
  student := make(map[string]interface{})
  student["name"] = "xia"
  student["age"] = 19
  student["is_rich"] = false

  ```

## 文件读写

### 读

### 写

## 时间

## 协程

> ing

### `goroutine`调度模型

`GMP`

## GC

## 锁

## map 并发安全

## context

## 进程

### 进程的本质

### 进程间通讯方式有多少种

## 协程暴增有哪些原因

## 实现斐波那契数列

> 使用动态规划和递归两种方式

## 数组实现一个最大堆

## duck typing(鸭子类型)相关

## Golang 断言
