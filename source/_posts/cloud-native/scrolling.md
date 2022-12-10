---
title: 使用 Docker Compose 实现滚动更新部署
tags:
  - Docker
  - DokerCompose
  - 滚动更新
categories: 云原生
keywords: 'blog,golang,php,k8s,kubernetes,probe'
description: 使用 Docker Compose 实现滚动更新部署
abbrlink: 9a11c27c
date: 2022-09-09 22:40:34
---

最近在写一个`Golang`的微服务项目，受限于测试和生产环境交付的时候都无法使用到`Kubernetes`，所以准备使用`Docker Compose`部署，记录一下实现方式。

## 思路

1. 变更容器编排文件`docker-compose.yaml`
2. 获取当前启动的容器id，标识为旧容器
3. 更新并调度服务比当前容器数量多的指定数量
4. 杀掉标识的旧容器
5. 清理杀掉的旧容器
6. 重新调度服务容器数量为期望数量

## shell 脚本实现

> 备注：此脚本需在 docker-compose.yaml 编排文件所在目录执行。或者自己改脚本，cd 进目标目录执行

```bash

# 1. 更新 docker-compose.yaml，有很多选择，你可以手动更新，也可以使用 ci 自动使用新的编排文件覆盖。总之就是把当前目录下的编排文件变成你期望的最新的，有可能是只改了镜像，或者其他

# 取到使用当前目录下的编排文件启动的服务容器id
previous_container=`docker-compose ps -q`

# 将容器数量调度到指定值，此值需要大于当前服务的容器数量
docker-compose up -d --no-deps --scale "$var"=3 --no-recreate

sleep 5

# 杀掉标识的旧容器
docker kill -s SIGTERM $previous_container

sleep 2

# 清理杀掉的旧容器
docker rm -f $previous_container

# 将新服务调度到指定数量
docker-compose up -d --no-deps --scale "$var"=2 --no-recreate "$var"

```

## 注意

使用此方式时，容器编排文件不能使用`ports`暴露端口，不然会提示端口占用，因为一个端口只能绑定一个端口，微服务的话你可以使用`consul/etcd`来实现服务注册发现，注册进去的是当前容器的ip。如果是http服务的话，可以在当前部署服务的网桥下启动一个`nginx`容器来暴露出指定端口做网关，nginx配置文件负载到服务容器。
