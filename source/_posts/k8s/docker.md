---
title: Docker 基础
tags:
	- Docker
categories: Kubernetes
keywords: 'blog,golang,php,k8s,kubernetes,probe'
description: 记录docker相关的基础知识
cover: >-
  https://graph.linganmin.cn/200911/4043741d339ab2812a46836518975aa7?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/200911/4043741d339ab2812a46836518975aa7?x-oss-process=image/format,webp/quality,q_40
abbrlink: docker-basic
date: 2020-07-11 22:40:34
---


## 实现原理

### Namespace

> `namespace`是`Linux`为我们提供的用于分离进程树、网络接口、挂载点以及进程间通信等资源的方法。

Docker 通过`namespace`完成了容器与宿主机、容器与容器间的隔离。

如果机器上运行了两个容器，对宿主机而言其实这两个容器就是两个特殊的进程

### Cgroups

> Linux Cgroups 全程是 Linux Control Group，主要的作用就是限制进程组使用的资源上限，包括CPU、内存、磁盘、网络带宽等。

### UnionFS 联合文件系统

Docker 中的每一个镜像都是由一系列只读的层组成，Dockerfile 中的每一个命令都会在已有的只读层上创建一个新的层

```bash
FROM redis:latest
COPY ./redis.conf /data
CMD redis-server /data/redis.conf
```

容器中的每一层都是只对当前容器进行了非常小的修改，上述的 Dockerfile 文件会构建一个拥有三层 layer 的镜像。当镜像被 docker run 命令创建时就会在最上层添加一个可写的层，也就是容器层，对以`对于运行时容器的修改其实都是对这个容器读写层的修改`。

容器和镜像的区别就在于，所有的镜像都是只读的，而每一个容器其实等于镜像加上一个可读的层，也就是同一个镜像可以对应多个容器。

> UnionFS 是一种为 Linux 操作系统设计的用于把多个文件系统联合到同一个挂载点的文件系统服务。

docker 的 每一个镜像层或者容器层都是`/var/lib/docker`目录下的一个子文件夹，UnionFS 能够将不同文件夹中的层联合到同一个文件夹中，这个联合的过程被称为`联合挂载`。


## 网络

Docker 虽然可以通过命名空间创建一个隔离的网络环境，但Docker服务仍然需要与外界连接才能发挥作用。Docker为我们提供了四种不同的网络模式。

- Bridge
  - 默认模式
- Host
- Container
- None

### bridge 模式

当 docker 服务在服务器上启动时，会创建一个新的虚拟网桥 docker0。

每一个容器在创建时都会创建一对虚拟网卡，两个虚拟网卡组成了数据的通道，其中一个会放在创建的容器中，另一个会加入到 docker0 网桥中。

当有 docker 容器需要将服务暴露（docker run xxx -p port:port）给宿主机时，docker0 网桥就会为该容器分配一个新的 ip 地址，并将 docker0 的ip 设置为默认的网关。网桥 docker0 通过 iptables 中的规则与宿主机上的网卡相连，所有符合条件的请求都会通过 iptables 转发到 docker0 并由网桥分发给对应容器。

## 解决的问题

1. 容器是如何进行隔离的？

在创建新进程时，通过 Namespace 技术实现隔离性，让运行后的容器仅仅只能看到自身的内容。比如在容器运行时，会默认加上PID、UTS、network、user、mount、IPC、Cgroup等 Namespace。

2. 容器是如何进行资源限制的？

通过 Linux CGroup 技术，可以为每个进程设定限制的 CPU，Memory 等资源，进而设置进程访问资源的上限。

3. 简述下 docker 的文件系统？

docker 的文件系统称为 rootfs，它的实现来自于 Linux unionFS，将不同的目录挂载到一起形成一个独立的视图。并且 docker 在此基础上引入了 层的概念，解决了可重复用的问题。

4. 容器的启动过程

  - 指定 Linux Namespace配置
  - 设置指定的 CGroups 参数
  - 切换进程的根目录
