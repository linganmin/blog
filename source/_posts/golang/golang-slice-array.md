---
title: 「深入学习 Golang」之 切片（slice）、数组（array）
tags:
  - Golang
categories: Golang
keywords: 'blog,golang,php'
description: golang 切片（slice）和数组（array）
cover: >-
  https://graph.linganmin.cn/201007/87122d3084205a5821b1e39413e84349?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/201007/87122d3084205a5821b1e39413e84349?x-oss-process=image/format,webp/quality,q_40
abbrlink: golang-basic
date: 2020-10-07 05:30:20
---

## 数组

> 数组是由相同类型元素组合成的数据结构，计算机会为数组分配一块`连续的内存`来保存其中的元素。我们可以利用数组中元素的索引，快速访问元素对应的存储地址。

{% note info %}
1. Go 语言中数组在初始化之后大小是无法改变的。
2. 存储元素类型相同，但大小不同的数组在 GO 语言中是完全不同的
{% endnote %}

### 创建

```golang

arr1 := [3]uint64{1,2,3} // 显示指定数组大小
arr2 := [...]uint64{1,2,3} // 在编译期间通过源码对数组大小进行推导
```

如果数组中元素个数小于或等于4个，那么所有的变量都会直接在栈上初始化；如果数组元素大于4个，变量就会在静态存储区初始化然后拷贝到栈上。

### 访问和赋值

数组在内存中就是一连串的内存空间，表示数组的方法是一个指向数组开头的指针、数组中元素的数量以及数组中元素类型站的空间大小。

如果我们不知道数组中元素的数量，访问时就可能发生越界，数组访问越界是非常严重的错误。

对数组的访问和复制需要同时依赖编译器和运行时，它的大多数操作在编译期间都会转换成对内存的直接读写，在中间代码生产期间还会插入运行时方法`panicIndex`调用，防止发生越界错误。

## 切片

切片是动态数组，他的长度不固定，我们可以随意向切片中追加元素，切片也会在容量不足时自动扩容。

### 创建

#### 使用下标

```golang
sl1 := arr[0:3]
sl2 := slice[0:4] // 通过下标方式从数组或者切片中截取

```

使用下标方式初始化切片不会造成原始数组或者切片中数据的拷贝，他只会创建一个指向原始数组的切片值，所以修改切片的数据也会修改原始切片和数组。

```golang
arr := [10]int32{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
sl1 := arr[0:5]
fmt.Printf("arr.0.memory.address：%p,sl1.0.memory.address：%p\n", &arr[0], &sl1[0]) // arr.0.memory.address：0xc0000b6000,sl1.0.memory.address：0xc0000b6000

```

#### 使用字面量

```golang

slice := []int{1,2,3,4}

```

1. 根据切片中的元素数量对底层数组大小进行判断并创建一个数组
2. 将这些字面量元素存储到初始化的数组中
3. 创建一个同样指向初始化数组类型的数组指针
4. 将静态存储区的数组赋值给数组指针
5. 通过 [:] 操作，获取一个指针指向数组指针的切片

#### 使用关键字

```golang

slice := make([]int, 10,10)

```

如果使用字面量方式创建切片，大部分工作都会在编译期间完成。但当我们使用`make`关键字创建切片时，很多工作都需要运行时的参与。

### 访问元素

```golang

slice := []int{1,2,4}

len(slice) // 取长度
cap(slice) // 取容量

for _,_ range slice{ // 遍历切片

}

```

### 追加和扩容

```golang

slice := []int{1,2,3}

slice = append(slice, 4) // 使用 append 追加元素

```

扩容参考文章：https://juejin.im/post/6844903812331732999