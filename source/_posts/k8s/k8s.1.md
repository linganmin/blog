---
title: Kubernetes 暴力入门(一)
tags:
  - Kubernetes
  - Docker
categories: Kubernetes
keywords: 'Kubernetes,K8S,docker'
description: 学习 Kubernetes 的总结第一篇
cover: >-
  https://graph.linganmin.cn/200118/3c4cf20b664e660235099a4262185023?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/200118/3c4cf20b664e660235099a4262185023?x-oss-process=image/format,webp/quality,q_40
abbrlink: 6c891c2
date: 2020-01-18 20:24:51
---

> 这是折腾 `Kubernetes` 的一个系列笔记 + 总结，文中可能会有一些个人理解的错误，请大佬们指正，希望我们都能成为越来越厉害的人。

## K8S 简介

`Kubernetes` 是 Google 团队发起的一个开源项目，旨在更简单方便的自动部署、扩展和管理跨多个主机的容器化应用程序。

`Kubernetes` 的组件和架构相对比较复杂，我的学习路径是先体验他的一些概念并试着去折腾一下后，再去理解 `Kubernetes` 的组件和架构，个人觉得这样会更好理解一些。

> Tips: 折腾 `Kubernetes` 必须要掌握的一些基础的 `Docker` 和 `计算机网络知识` 哦。

### 为什么要用 K8S

1. 可以“轻装上阵”地开发复杂系统，以前需要大量人力才能实现的分布式系统，在采用`K8S`后，只需要一个小团队即可轻松应对。
2. 使用`K8S`可以更方便的拥抱`微服务`架构。
3. 超强的横向扩展能力。
4. 更自动化的故障迁移能力：当某个`Node`节点挂掉或者宕机后，该`Node`节点上的服务会自动转移到另一个节点上，这个过程所有服务都不会中断。
5. 滚动发布/升级。
6. 资源隔离。

## 一些必须的概念

### Node

Node（节点）是 K8S 集群中相对于 Master 而言的工作主机，Node 可以是一台物理机，也可以是一台虚拟机。在每个 Node 上运行用于启动和管理 pid 的服务`kubelet`，并且能够被 Master 管理。

### Pod

#### 概念

Pod 是 K8S 的最基本单元，也是应用运行的载体，整个 K8S 系统都是围绕着 Pod 展开的，它内部应该包含一个或多个紧密相关的容器，就像它的名字`Pod(豆荚)`。

一个 Pod 可以被一个容器化的环境看做是应用层的逻辑宿主机。

一个 Pod 中的多个容器应用通常是高耦合、高依赖的。

如果多个容器在同一 Pod 下他们公用一个 IP 所以不能出现重复的端口号,比如在一个 Pod 下运行两个 nginx 就会有一个容器异常,一个 Pod 下的多个容器可以使用 localhost 来访问对方端口

Pod 在 Node 上被创建、启动和销毁。

#### 对 Pod 的定义

通常我们通过 Yaml 或 JSON 格式的文件来定义 Pod，下面的配置文件定义了一个名为：nginx-test 的 Pod，其中 kind 为 Pod。在 spec 中主要包含了对容器（container）的定义。

```yaml

apiVersion: v1
kind: Pod
metadata:
  name: nginx-test
  labels:
    name: nginx-test
spec:
  containers:
    - name: nginx-test
      image: nginx
      imagePullPolicy: "Always"
      ports:
        - containerPort: 80

```

Pod 的状态：

- Pending
  - Pod 定义正确，提交到 Master，但其包含的容器还未创建完成。
- ContainerCreating
  - Pod 中包含的容器正在构建中。
- Running
  - Pod 已经被分配到某个 Node 上，且其所有报=包含的容器都已创建完成，并且运行起来了。
- Terminating
  - Pod 中包含的容器正在终止中。
- Successed
  - Pod 中所包含的所有容器都成功结束，且不会被重启，这是 Pod 的最终状态。
- Failed
  - Pod 中所有容器都结束了，但至少一个容器是以失败状态结束的，这也是 Pod 的一种最终状态。

> K8S 为 Pod 设计了一套独特的网络配置，包括：为每个 Pod 分配一个IP地址，使用Pod名作为容器间通信的主机名等。

Pod 生命周期：

- 初始化容器
- 启动后钩子
- 探测
  - liveness probe
    - 存活性探测
  - readiness probe
    - 就绪性探测
  - 探针类型
    - exec
    - httpGet
    - tcpSocket

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: liveness-http-test
    namespace: default
  spec:
    containers:
      - name: liveness-http-test-containers
        image: nginx
        imagePullPolicy: IfNotPresent
        ports:
          - name: http
            containerPort: 80
        livenessProbe:
          httpGet:
            port: http
            path: /index.html
          initialDelaySeconds: 1
          periodSeconds: 3

  ```

- 停止前钩子

### Label

Label 是 K8S 系统中一个重要的概念，Label 以键值对(key/value)的形式附加在各种对象上，比如：Pod、Service 等。Label 定义了这些对象的可识别属性，用来对他们进行管理和选择，通常我们在创建对象的时候都会附加一些区分度高的 Label 到对应对象上，当然也可以在对象创建后通过 API 对其进行管理。

### Namespace

> TODO

### Deployment

> TODO

### Service

> TODO
