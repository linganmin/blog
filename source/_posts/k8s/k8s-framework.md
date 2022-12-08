---
title: 「Kubernetes 拾遗」之 K8S 基础知识
tags:
	- Kubernetes
	- Docker
	- 拾遗
categories: Kubernetes
keywords: 'blog,golang,php,k8s,kubernetes,probe'
description: 输入 kubectl run/create/apply 之后到底发生了什么？
cover: >-
  https://graph.linganmin.cn/200911/4043741d339ab2812a46836518975aa7?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/200911/4043741d339ab2812a46836518975aa7?x-oss-process=image/format,webp/quality,q_40
abbrlink: k8s-framework
date: 2020-09-11 05:40:34
---

## 架构图

![架构图](https://graph.linganmin.cn/210403/902edee9c3af4fc63804dc26d586499f?x-oss-process=image/format,webp/quality,q_60)

## 核心组件

- API Server

  - 用户唯一可以直接进行交互的 k8s 组件
  - 提供了认证、授权、访问控制、api 注册和发现等机制

- ETCD

  - 集群数据库
  - 存储了整个集群的配置和状态

- Controller Manager

  - 负责维护集群的状态，比如故障检测、自动扩展、滚动更新等

- Scheduler

  - 负责资源的调度。按照预定的调度策略将 pod 调度到相应机器上

- Kubelet

  - 负责维护容器的生命周期、同时也负责 Volume(CSI)和网络(CNI)的管理

- Container Runtime

  - 负责镜像管理以及 pod 和容器的真正运行 (CRI)

- Kube-Proxy

  - 负责为 service 提供 cluster 内部的服务发现和负载均衡

## 工作流
