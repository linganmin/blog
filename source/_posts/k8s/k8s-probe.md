---
title: 「Kubernetes 拾遗」之 Probe（探针）
tags:
  - Kubernetes
  - Docker
	- 拾遗
categories: Kubernetes
keywords: 'blog,golang,php,k8s,kubernetes,probe'
description: Kubernetes ( k8s ) 存活检查 ( livenessProbe ) 与 就绪检查 ( readinessProbe )
cover: >-
  https://graph.linganmin.cn/200823/cedb311450bb165121fd1f1a6d49fa13?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/200823/cedb311450bb165121fd1f1a6d49fa13?x-oss-process=image/format,webp/quality,q_40
abbrlink: k8s-probe
date: 2020-08-10 22:40:34
---

Kubernetes 中的健康检查是使用`存活性探针`和`就绪性探针`来实现的，基于这两种探测机制，实现了 k8s 的自愈能力。可以做到：

1. 异常节点(pod)自动剔除，并重建
2. 安全的滚动升级策略

## 存活性探针

`livenessProbe`用于判断当前容器`是否存活(running)状态`，如果`livenessProbe`探测到容器不健康，则`kubectl`会杀掉该容器，并根据容器的重启策略做相应重启操作。

> 如果一个容器不包含`livenessProbe`，那么`kubectl`会始终认为该容器的存活性探针返回的永远是`success`状态。

## 就绪性探针

`readinessProbe`用于判断当前容器`是否完成启动(read状态)`，如果`readinessProbe`探测到容器不健康，则判定该容器不可接收流量，`Endpoint Controller`将从该服务对应的`Service`的`endpoints`中移除该容器的`endpoint`。

## 探测方式

- HTTP
- TCP
- Exec 命令

## 实际在生产环境使用

### livenessProbe

```yaml
livenessProbe:
    httpGet:
      path: /ping
      port: 9501
    initialDelaySeconds: 30 # 延迟探测时间, 等待容器初始化
    periodSeconds: 10 # 执行探测的频率（多少秒执行一次）。默认为10秒。最小值为1。
    timeoutSeconds: 1 # 探测的超时时间，默认 1s，最小 1s
    successThreshold: 1 # 健康阈值，最少连续成功多少次才视为成功。默认值为1。最小值为1。
    failureThreshold: 3 # 不健康阈值，最少连续多少次失败才视为失败。默认值为3。最小值为1。

```

### readinessProbe

```yaml
readinessProbe:
  httpGet:
    path: /ping
    port: 9501
  initialDelaySeconds: 30 
  periodSeconds: 3
  successThreshold: 1
  timeoutSeconds: 1
  failureThreshold: 3

```

----

探针相关文档：[https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)