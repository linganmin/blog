---
title: 知识体系梳理
tags:
  - Golang
  - Kubernetes
  - PHP
  - MySQL
  - MQ
  - Redis
categories: Golang
keywords: 'blog,golang,php'
description: 知识体系梳理
cover: >-
  https://graph.linganmin.cn/200822/43c6bc33f3be83618d4f9cc69bf00fb0?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/200822/43c6bc33f3be83618d4f9cc69bf00fb0?x-oss-process=image/format,webp/quality,q_40
abbrlink: golang_knowledge
date: 2020-12-15 22:03:00
---

在微服务里面提倡，不同的模块单元用最实惠的语言。

## Golang

- Goroutine 生命周期
  - 野生goroutine
    - https://zhuanlan.zhihu.com/p/328591249
    - https://www.syncd.cn/go_advance_note2/
  - https://zhuanlan.zhihu.com/p/74090074
- Go 的内存模型
  - https://www.jianshu.com/p/5e44168f47a3
- 原子锁
  - atomic
  - data race
- cow copy on write（COW)写时复制
  - BGSave redis
- errgroup
- interface nil
  - https://www.cnblogs.com/52php/p/7363101.html
- context
  - 不要修改之前的值，生成新的context，copy on write 方式生成新的
  - 参考 rpc metadata 实现

## PHP

## Kubernetes
