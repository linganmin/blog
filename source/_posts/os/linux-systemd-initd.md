---
title: Linux OS 的 Init 和 Systemd 
categories: 操作系统
keywords: 'linux,systemd'
description: Linux Init Service 和 Systemd
cover: >-
  https://graph.linganmin.cn/230202/45c4be216e488ead47b3373c4a04187f?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/230202/45c4be216e488ead47b3373c4a04187f?x-oss-process=image/format,webp/quality,q_60
abbrlink: 49a0b438
---

## 什么是 Init 守护进程

`Init`守护进程是`Linux`内核执行的第一个进程，它的进程`ID（PID）`始终是1，它的目的是初始化、管理和跟踪系统服务和守护进程，也就是说`init deamon(Init 守护进程)`是系统上所有进程的父进程。

## 什么是 Init

Init（也被称为`System V init`或者`SysVinit`），它是一个初始化守护进程，创建于 1980 年。它定义了6个运行级别（系统状态）并将所有的系统服务映射到这些运行级别。这使得所有服务都会以预定义的顺序启动，只有当顺序中的当前脚本执行完成或者卡主超时时才会执行下一个脚本。除了执行超时导致无法预期的等待之外，串行的启动服务也会使得系统初始化过程效率低下且相对较慢。

## 什么是 Systemd

`Systemd(system daemon)`是现代系统使用的`init`守护进程，它并行的启动系统服务，从而消除不必要的延迟并加快初始化进程。`systemd`通过使用`Unit Dependencies`来定义一个服务是否想要/需要依赖其他服务才能运行，使用`Unit Order`来定义一个服务是否需要在它之前/之后启动。可以使用`systemctl`进行对`system service`的管理。
