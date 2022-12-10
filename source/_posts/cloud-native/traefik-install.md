---
title: 使用 Helm 在 Kubernetes 集群安装最新版 Traefik
tags:
  - Kubernetes
  - Helm
categories: 云原生
keywords: 'traefik,blog,golang,php,k8s,kubernetes,probe'
description: 使用 Helm 在 Kubernetes 集群安装最新版 Traefik 作反向代理和负载均衡器
cover: >-
  https://graph.linganmin.cn/210921/4ae58f0d7d8d8a33e91b55adb822dbdb?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/210921/4ae58f0d7d8d8a33e91b55adb822dbdb?x-oss-process=image/format,webp/quality,q_40
abbrlink: 2d15ed93
date: 2021-09-21 00:40:34
---

## 简单介绍

> Traefik 是一个开源的边缘路由器，他负责接收系统的请求，然后使用合适的组件对这些请求进行处理。

下图为 traefik 的架构图，traefik 的官方文档地址是：[https://doc.traefik.io/traefik/](https://doc.traefik.io/traefik/)

![traefik](https://graph.linganmin.cn/210921/a5f69f09c3290cb13e2d5417055e28a6?x-oss-process=image/format,webp/quality,q_60)

## 核心概念

### providers

Traefik 中的配置发现是通过 Providers 实现的，Provider 可以是编排工具、容器引擎、或者 KV 存储等。

![providers](https://graph.linganmin.cn/210921/00134790cdc2773e5024934607e1dad4?x-oss-process=image/format,webp/quality,q_60)

支持的 Providers 文档：[https://doc.traefik.io/traefik/providers/overview/#supported-providers](https://doc.traefik.io/traefik/providers/overview/#supported-providers)

### entrypoints

entrypoints 是 Traefik 的网络入口点，他们定义了将接收数据包的端口，以及侦听 TCP 还是 UDP。

![entrypoints](https://graph.linganmin.cn/210921/a7b0bbb78bbd11de88bb932b03664df1?x-oss-process=image/format,webp/quality,q_60)

配置文档参考：[https://doc.traefik.io/traefik/routing/entrypoints/](https://doc.traefik.io/traefik/routing/entrypoints/)

### routers

路由器负责将传入请求连接到可以处理他们的服务，在这个过程中可能会使用中间件来更新请求，或者在将请求转发给服务之前做一些处理。

![routers](https://graph.linganmin.cn/210921/b80dce3cf24250b8d67951e8cec2aeba?x-oss-process=image/format,webp/quality,q_60)

配置文档参考：[https://doc.traefik.io/traefik/routing/routers/#rule](https://doc.traefik.io/traefik/routing/routers/#rule)

### middleware

附加到路由器的中间件是一种在请求发送到服务之前（或咋服务的响应发送到客户端之前）调整请求的方法。

Traefik 中有几个可用的中间件，可以修改 request、header、redirect或者进行身份认证（authentication）。

![middleware](https://graph.linganmin.cn/210921/89281fdb61066fef9ed1c58bc2748675?x-oss-process=image/format,webp/quality,q_60)

配置文档参考：https://doc.traefik.io/traefik/middlewares/overview/

### services

负责配置如何到达实际服务，最终将处理传入的请求。

![](https://graph.linganmin.cn/210921/06068790260da9c9679cabdb3c1b43e3?x-oss-process=image/format,webp/quality,q_60)

配置文档参考：[https://doc.traefik.io/traefik/routing/services/](https://doc.traefik.io/traefik/routing/services/)

## 安装

使用 Helm 快速安装 Traefik。

确保满足以下要求:

- Kubernetes 1.14+
- Helm version 3.x is installed

因为官方在 Helm 镜像仓库的 Traefik 版本并不是最新的，所以我们要拉下官方仓库本地执行`helm install`。

![heml-chart](https://graph.linganmin.cn/210921/013c783eeed051f786f6c43cccf09515?x-oss-process=image/format,webp/quality,q_60)

```bash
git clone https://github.com/traefik/traefik-helm-chart
```

在`traefik-helm-chart`目录创建一个定制的 values 配置文件（备注：如果不熟悉配置可以使用官方默认的配置文件）。

```bash
vim values-custom.yaml
```

写入以下内容

```yaml
# values-custom.yaml
# Create an IngressRoute for the dashboard
ingressRoute:
  dashboard:
    enabled: false # 禁用 helm 中渲染的 dashboard，后面手动创建

# Configure ports
ports:
  web:
    port: 8000
    nodePort: 31800 # 使用 NodePort 模式
  websecure:
    port: 8443
    nodePort: 31443

# Service
service:
  enabled: true
  type: NodePort # 使用 NodePort 模式

# Logs
logs:
  general:
    level: DEBUG
```

为 Traefik 创建独立的 Namespace

```bash
kubectl create ns traefik
```

安装 traefik

```bash
helm install -n traefik traefik ./traefik/ -f ./values-custom.yaml
```

查看 traefik 的运行状态

```bash
# pod
➜ kubectl get pods -n traefik
NAME                       READY   STATUS    RESTARTS   AGE
traefik-86f478fb6f-xghzs   1/1     Running   0          23s

# svc

➜ kubectl get svc -n traefik
NAME      TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
traefik   NodePort   10.96.76.244   <none>        80:31800/TCP,443:31443/TCP   2m5s

```

查看 POD 的资源清单来了解 Traefik 的运行方式

```bash
➜ kubectl get pods traefik-86f478fb6f-xghzs -n traefik -o yaml

apiVersion: v1
kind: Pod
metadata:
  ...... # metadata 内容过多，省略之
spec:
  containers:
  - args:
    - --global.checknewversion
    - --global.sendanonymoususage
    - --entryPoints.metrics.address=:9100/tcp
    - --entryPoints.traefik.address=:9000/tcp
    - --entryPoints.web.address=:8000/tcp
    - --entryPoints.websecure.address=:8443/tcp
    - --api.dashboard=true
    - --ping=true
    - --metrics.prometheus=true
    - --metrics.prometheus.entrypoint=metrics
    - --providers.kubernetescrd
    - --providers.kubernetesingress
    - --log.level=DEBUG

...... # 后面关于探针、镜像等内容配置过多，省略之
```

从容器参数上可以看到`entryPoints`定义了`web`和`websecure`两个入口，并且`provider`开启了`kubernetescrd`和`kubernetesingress`，也就是我们既可以使用 Kubernetes 自身的`Ingress`资源对象也可以使用 Traefik 自己扩展的`IngressRoute`类型的 CRD 资源对象来定义入口。

### 使用 IngressRoute 资源清单部署一个 Traefik-Dashboard 入口

创建 dashboard.yaml 文件

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: dashboard
  namespace: traefik
spec:
  entryPoints: # 入口点使用 web
    - web
  routes:
    - match:  (PathPrefix(`/dashboard`) || PathPrefix(`/api`)) # 路由匹配规则，/dashboard 对静态页，/api 对接口
      kind: Rule
      services:
        - name: api@internal
          kind: TraefikService # Helm 安装 Traefik 时创建的自定义Service
```

部署

```bash
➜ kubectl apply -f dashboard.yaml
ingressroute.traefik.containo.us/dashboard created
```

部署完成后如果你是在云厂商的机器上就可以使用 NODE 节点的公网 IP + Traefik 的 Service 暴露出的端口号进行访问，如果是本地环境使用内网 IP 也是如此。

> http://your-ip:31800/dashboard/#/

![dashboard](https://graph.linganmin.cn/210921/394ccaebb904ab846872648edf03ef64?x-oss-process=image/format,webp/quality,q_60)

至此，我们使用 Helm 在 Kubernetes 集群部署最新版 Traefik 便高一段落。

---

TODO List：

- 研究 Traefik CRD
- 研究 IngressRoute + Let’s Encrypt 自动生成 Https
- 研究 Traefik 中间件
- 研究 Traefik 灰度发布
