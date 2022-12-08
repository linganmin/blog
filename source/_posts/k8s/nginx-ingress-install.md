---
title: Kubernetes 集群部署最新版 Ingress-Nginx
tags:
  - Kubernetes
  - Docker
categories: Kubernetes
keywords: 'traefik,blog,golang,php,k8s,kubernetes,probe'
description: Kubernetes 集群部署 Ingress-Nginx
cover: >-
  https://graph.linganmin.cn/210922/88f2bb6a988224d79d9f6cdf1edce170?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/210922/88f2bb6a988224d79d9f6cdf1edce170?x-oss-process=image/format,webp/quality,q_40
abbrlink: nginx-ingress-install
date: 2021-09-22 22:40:34
---

## 简介

> ingress-nginx 是 Kubernetes 的入口控制器，使用 Nginx 作为反向代理和负责均衡器。

如引用所写的那样，ingress-nginx 其实就是使用 Nginx 作为 Kubernetes 集群入口的反向代理和均衡器。对于 Nginx 我们应该都很熟悉，或者说至少不陌生。那么什么是`ingress`呢。

## 什么是 Ingress

> Ingress 是对集群中服务的外部访问进行管理的 API 对象，典型的访问方式是 HTTP。Ingress 公开了从集群外部到集群内部服务的 HTTP 和 HTTPS 路由。流量路由由 Ingress 资源定义的规则控制。

下图是一个将所有流量发送到同一个 Service 的简单 Ingress 示例：

![ingress-demo](https://graph.linganmin.cn/210922/cc29e19116ea7ab1642988c40785c788?x-oss-process=image/format,webp/quality,q_60)

### Ingress 资源

和所有其他 Kubernetes 资源一样，Ingress 需要使用`apiVersion`、`kind`、`metadata`字段，Ingress 通常使用注解（annotations）来配置一些选项，具体取决于 Ingress 控制器，不同的 Ingress 控制器支持不同的注解。

Ingress 规约 提供了配置负载均衡器或者代理服务器所需的所有信息。 最重要的是，其中包含与所有传入请求匹配的规则列表。 Ingress 资源仅支持用于转发 HTTP 流量的规则。

详细 Ingress 资源定义参考文档：[https://kubernetes.io/zh/docs/concepts/services-networking/ingress/#ingress-rules](https://kubernetes.io/zh/docs/concepts/services-networking/ingress/#ingress-rules)

```yaml
# 一个 Ingress 资源的示例。后面我们也会用到这个例子用来将 whoami 这个后端服务暴露到集群外部
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  creationTimestamp: null
  name: whoami
  annotations:
    kubernetes.io/ingress.class: "nginx" # 
spec:
  rules:
  - http:
      paths:
      - backend:
          service:
            name: whoami
            port:
              number: 80
        path: /proxy/nginx/whoami
        pathType: Exact
status:
  loadBalancer: {}
```

## 部署 Ingress-Nginx

### 手动通过 kubectl 部署（后文有使用 helm 部署方式）

下载官方提供的 deploy.yaml 文件

```bash

wget  https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.0.1/deploy/static/provider/cloud/deploy.yaml

```

修改 service

```yaml
# Source: ingress-nginx/templates/controller-service.yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
  labels:
    ..... # 过多，省略之
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  type: NodePort # 使用节点端口的方式供外部访问，修改为 NodePort
  externalTrafficPolicy: Local
  ports:
    - name: http
      port: 80
      nodePort: 30800 # 增加节点端口号，端口号可用区间 [30000-32767]
      protocol: TCP
      targetPort: http
      appProtocol: http
    - name: https
      port: 443
      nodePort: 30443 # 增加节点端口号，端口号可用区间 [30000-32767]
      protocol: TCP
      targetPort: https
      appProtocol: https
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/component: controller
```

#### 部署

```bash

➜ kubectl apply -f deploy.yaml

namespace/ingress-nginx created
serviceaccount/ingress-nginx created
configmap/ingress-nginx-controller created
clusterrole.rbac.authorization.k8s.io/ingress-nginx unchanged
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx unchanged
role.rbac.authorization.k8s.io/ingress-nginx created
rolebinding.rbac.authorization.k8s.io/ingress-nginx created
service/ingress-nginx-controller-admission created
service/ingress-nginx-controller created
deployment.apps/ingress-nginx-controller created
ingressclass.networking.k8s.io/nginx unchanged
validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission configured
serviceaccount/ingress-nginx-admission created
clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission unchanged
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission unchanged
role.rbac.authorization.k8s.io/ingress-nginx-admission created
rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
job.batch/ingress-nginx-admission-create created
job.batch/ingress-nginx-admission-patch created

```

查看 pods

```yaml
➜ kubectl get pods -n ingress-nginx

NAME                                       READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create--1-9wfnj    0/1     Completed   0          2m26s
ingress-nginx-admission-patch--1-fb7rh     0/1     Completed   1          2m26s
ingress-nginx-controller-fd7bb8d66-kmt6n   1/1     Running     0          2m26s

```

查看 svc

```yaml
➜ kubectl get svc -n ingress-nginx

NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.100.59.116   <none>        80:30800/TCP,443:30443/TCP   95s
ingress-nginx-controller-admission   ClusterIP   10.111.89.174   <none>        443/TCP                      95s

```

可以看到`ingress-nginx-controller`的`service`通过`NodePort`的方式，将入口端口暴露出去`80:30800/TCP,443:30443/TCP`。此时通过云主机的外网IP或者你Node节点的IP便可以请求`http://your-ip:30800`可以得到一个`404 Not Found`的返回，是不是很熟悉。

![404](https://graph.linganmin.cn/210922/063af1b5092df33cf7fda04b2e346f73?x-oss-process=image/format,webp/quality,q_60)

至此，我们的`ingress-nginx`边算是部署完成了。

#### 部署一个 whoami 的应用，使用 Ingress 将服务暴露给外部访问

编写 Deployment 资源文件

```yaml
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: whoami
  labels:
    app: whoami
spec:
  replicas: 2
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
        - name: whoami
          image: containous/whoami
          ports:
            - name: web
              containerPort: 80
```

编写 Service 资源文件

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: whoami
spec:
  ports:
    - protocol: TCP
      name: web
      port: 80
  selector: # 选择器，匹配满足条件的 Pods
    app: whoami
```

编写 Ingress 资源文件

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  creationTimestamp: null
  name: whoami
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - http:
      paths:
      - backend:
          service: # Service 匹配
            name: whoami
            port:
              number: 80
        path: /proxy/nginx/whoami # 路由匹配
        pathType: Exact
status:
  loadBalancer: {}
```

依次部署三个资源

```bash
➜ kubectl apply -f deploy.yaml

deployment.apps/whoami created

➜ kubectl apply -f service.yaml

service/whoami created

➜ kubectl apply -f ingress-nginx.yaml

ingress.networking.k8s.io/whoami created

```

查看部署的 Pods、Service、Ingress 状态

```bash
➜ kubectl get pods | grep whoami

whoami-658d568b94-6wcfh   1/1     Running     0              2m31s
whoami-658d568b94-kvtxx   1/1     Running     0              2m32s

➜ kubectl get deployments.apps | grep whoami

whoami   2/2     2            2           2m52s

➜ kubectl get svc | grep whoami

whoami       ClusterIP   10.100.95.204   <none>        80/TCP    2m28s

➜ kubectl get ingress | grep whoami

whoami   <none>   *       10.100.59.116   80      2m23s

```

可以看到针对`whoami`这个服务部署的各资源均运行正常，那么便可以使用`Ingress`里定义的路由`path: /proxy/nginx/whoami`进行访问了。在浏览器输入`http://your-ip:30800/proxy/nginx/whoami`试一下。

![whoami](https://graph.linganmin.cn/210922/f5ae7718cdf378f7cb81a798580a1bcc?x-oss-process=image/format,webp/quality,q_60)

### 使用 helm 安装（仅做简单介绍，很简单，不做具体演示）

NGINX Ingress controller can be installed via `Helm` using the chart from the project repository. To install the chart with the release name ingress-nginx:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx
```

---

备注: 关于 IngressClass 及常见问题，参考文档：[https://kubernetes.github.io/ingress-nginx/](https://kubernetes.github.io/ingress-nginx/)
