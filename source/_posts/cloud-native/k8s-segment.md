---
title: Kubernetes 集群部署之网段划分
tags:
  - Kubernetes
  - 网络
categories: 云原生
keywords: 'blog,golang,php,k8s,kubernetes,probe'
description: Kubernetes 集群部署的网络划分总结
cover: >-
  https://graph.linganmin.cn/230201/1121c70cca031f2591b48ce72261dedc?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/230201/1121c70cca031f2591b48ce72261dedc?x-oss-process=image/format,webp/quality,q_60
abbrlink: 4f614365
---

Kubernetes 集群部署会涉及到如下三个网段

- 宿主机网段
  - 准备安装 Kubernetes 集群的服务器的内网网段
- POD 网段
  - Kubernetes 中部署的 POD 的网段
- Service 网段
  - Kubernetes 中部署的 Service 的网段

需要注意的的是在部署`Kubernetes`的时候，这三个网段之间不能有任何的交叉。比如，如果宿主机的网段是`172.16.168.135/12`，那么`Service`或`Pod`的网段就不能在`172.16.0.1~172.31.255.254`之间。

所以一般的推荐是，三个网段的第一位就不要重复，比如宿主机的网段如果是`172`开头，那么`Service`和`Pod`就不要用`172`打头的网段，这样不近可以避免网段冲突，还可以减去计算的步骤。

![内网网段](https://graph.linganmin.cn/230201/f99145761826b3f4c838c32c4d67c71c?x-oss-process=image/format,webp/quality,q_60)

A 类地址子网掩码最小为8
B 类地址子网掩码最小为16
C 类地址子网掩码最小为24

为了方便`POD`数量的扩展，应该给`POD`设置B类或A类地址，因为C类最多只有254个可用IP。
