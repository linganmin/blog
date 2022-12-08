---
title: Kubeadm 安装 Kubernetes V1.22.2 踩坑手记
tags:
  - Kubernetes
  - Docker
categories: Kubernetes
keywords: 'blog,golang,php,k8s,kubernetes,probe'
description: Kubeadm 安装 Kubernetes V1.22.2 踩坑手记，使用 Docker 作 Container Runtime
cover: >-
  https://graph.linganmin.cn/210919/4e6c850efdc628e06f037a82d5d7b185?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/210919/4e6c850efdc628e06f037a82d5d7b185?x-oss-process=image/format,webp/quality,q_40
abbrlink: k8s-install-note
date: 2021-09-20 00:40:34
---

记录一下最近在云服务器上折腾最新版 Kubernetes 遇到的一些坑。

## 使用 Docker 作为 Container Runtime

### cgroup 驱动

> kubeadm 支持在执行`kubeadm init`时，传递一个`KubeletConfiguration`结构体，`KubeletConfiguration`结构体包含`cgroupDriver`字段，用于控制 kubelet 的 cgroup 驱动。

{% note note %}

注意：如果没有在`KubeletConfiguration`中设置`cgroupDriver`字段，`kubeadm init`会将它设为默认值`systemd`。

Kubernetes 官方推荐使用`systemd`设为默认 cgroup 驱动。

{% endnote %}

{% note warning %}

特别注意：如果使用`Docker`作为容器运行时，Docker 默认的驱动类型为`cgroupfs`，需要修改成`native.cgroupdriver=systemd`。

{% endnote %}

编辑`/etc/docker/daemon.json`文件，修改 docker 的 cgroup 驱动类型


```json

{
  "exec-opts": ["native.cgroupdriver=systemd"]
}

```

### 设置 docker 代理

因为 kubeadm 默认为 google 的镜像仓库地址k8s.gcr.io，国内无法访问，所以需要使用代理（如果你没有的话，可以搜索使用`私有镜像仓库`的方式拉取）。

```bash
mkdir -p /etc/systemd/system/docker.service.d

vim /etc/systemd/system/docker.service.d/proxy.conf

# 重启之
systemctl daemon-reload && systemctl restart docker

```

增加以下配置（ps：以下端口是我自己的代理配置，按需更改！）

```conf
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:1087/"
Environment="HTTPS_PROXY=http://127.0.0.1:1087/"
Environment="NO_PROXY=localhost,127.0.0.1,.example.com"
```

### 修改 docker0 网桥默认网段

Docker 服务启动后会创建一个 docker0 网桥，他在内核层连通了其他的物理或虚拟网卡，以此实现将所有容器和本地主机都放在同一个物理网络。

Docker 默认指定了 docker0 接口的 IP 和子网掩码，让主机和容器之间以通过网桥互相通信。

为了防止 docker0 网桥网段与宿主机或 VPC 网络网段冲突，建议修改之。

编辑`/etc/docker/daemon.json`文件，增加如下配置，然后重启docker`systemctl restart docker`

```json
# 以下为笔者配置，按需修改
{
  "bip": "192.168.100.1/24"
}
```

----
下面是 TODO

---

- openssl 自签名证书 for kubernetes dashboard
- 安装 Nginx-ingress
- 安装 Traefik-ingress
- 部署 StatefulSet 服务
