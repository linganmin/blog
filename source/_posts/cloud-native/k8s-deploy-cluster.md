---
title: 使用 Kubeadm 部署 Kubernetes 集群
tags:
  - Kubernetes
categories: 云原生
keywords: 'blog,golang,php,k8s,kubernetes,probe'
description: 记录使用 kubeadm 部署一套 k8s 集群
cover: >-
  https://graph.linganmin.cn/200911/4043741d339ab2812a46836518975aa7?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/200911/4043741d339ab2812a46836518975aa7?x-oss-process=image/format,webp/quality,q_40
abbrlink: a145cb7
date: 2021-03-27 03:40:34
---

## 环境

大于等于两台`2C4G`的`CentOS7.8` Linux 服务器

> PS：如果本地资源不够，可以去阿里云开几台ECS，选按量付费。2c4g，5M带宽最低配大概每小时不到1块钱。

### 部署前准备

#### 关闭 firewalld 和 selinux

```bash
setenforce 0
sed -i '/^SELINUX=/cSELINUX=disabled' /etc/selinux/config

systemctl stop firewalld
systemctl disable firewalld

```

#### 安装 ipvs 内核

> todo ipvs 和 iptables 区别

```bash
yum -y install ipvsadm ipset sysstat conntrack libseccomp
```

设置开机默认加载配置文件

```bash
cat >>/etc/modules-load.d/ipvs.conf<<EOF
ip_vs_dh
ip_vs_ftp
ip_vs
ip_vs_lblc
ip_vs_lblcr
ip_vs_lc
ip_vs_nq
ip_vs_pe_sip
ip_vs_rr
ip_vs_sed
ip_vs_sh
ip_vs_wlc
ip_vs_wrr
nf_conntrack_ipv4
EOF

```

设置开机加载 ipvs

```bash
systemctl enable systemd-modules-load.service   # 设置开机加载内核模块
lsmod | grep -e ip_vs -e nf_conntrack_ipv4      # 重启后检查 ipvs 模块是否加载
```

#### 设置 Docker 源

```bash
curl -o /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

```

#### 设置 k8s 源

```bash
cat >>/etc/yum.repos.d/kuberetes.repo<<EOF
[kuberneres]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
enabled=1
EOF
```

#### 安装 docker-ce、kubelet、kubeadm、kubectl

```bash
yum install docker-ce kubelet kubectl kubeadm -y
```

#### 启动设置

```bash
systemctl start docker
systemctl enable docker
systemctl enable kubelet
```

#### 设置内核参数及 K8S 参数

```bash
cat >>/etc/sysctl.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

设置 kubelet 忽略 swap 使用 ipvs

```bash
cat >/etc/sysconfig/kubelet<<EOF
KUBELET_EXTRA_ARGS="--fail-swap-on=false"
KUBE_PROXY_MODE=ipvs
EOF
```

## 部署

### 部署 Master 节点

{% note warning %}
此步骤仅需在 Master 节点执行！！！
{% endnote %}

#### 提前拉取镜像

> 因为一些众所周知的原因，国内是访问不到 k8s 的官方镜像的。所以需要从国内的镜像源拉取然后通过tag方式命名成官方镜像(ps:当然如果服务器在国外是没有这个问题的)

##### 查看当前版本 kubeadm 所需的镜像

> 此处要根据自己实际依赖的镜像版本进行操作

```bash
> kubeadm config images list

k8s.gcr.io/kube-apiserver:v1.20.5
k8s.gcr.io/kube-controller-manager:v1.20.5
k8s.gcr.io/kube-scheduler:v1.20.5
k8s.gcr.io/kube-proxy:v1.20.5
k8s.gcr.io/pause:3.2
k8s.gcr.io/etcd:3.4.13-0
k8s.gcr.io/coredns:1.7.0
```

##### 从阿里镜像源拉取对应镜像

```bash

docker pull registry.aliyuncs.com/google_containers/kube-apiserver:v1.20.5

docker pull registry.aliyuncs.com/google_containers/kube-controller-manager:v1.20.5

docker pull registry.aliyuncs.com/google_containers/kube-scheduler:v1.20.5

docker pull registry.aliyuncs.com/google_containers/kube-proxy:v1.20.5

docker pull registry.aliyuncs.com/google_containers/pause:3.2

docker pull registry.aliyuncs.com/google_containers/etcd:3.4.13-0

docker pull registry.aliyuncs.com/google_containers/coredns:1.7.0

```

##### 修改tag

```bash
docker tag registry.aliyuncs.com/google_containers/kube-apiserver:v1.20.5 k8s.gcr.io/kube-apiserver:v1.20.5

docker tag registry.aliyuncs.com/google_containers/kube-controller-manager:v1.20.5 k8s.gcr.io/kube-controller-manager:v1.20.5

docker tag registry.aliyuncs.com/google_containers/kube-scheduler:v1.20.5 k8s.gcr.io/kube-scheduler:v1.20.5

docker tag registry.aliyuncs.com/google_containers/kube-proxy:v1.20.5 k8s.gcr.io/kube-proxy:v1.20.5
docker tag registry.aliyuncs.com/google_containers/pause:3.2 k8s.gcr.io/pause:3.2

docker tag registry.aliyuncs.com/google_containers/etcd:3.4.13-0 k8s.gcr.io/etcd:3.4.13-0

docker tag registry.aliyuncs.com/google_containers/coredns:1.7.0 k8s.gcr.io/coredns:1.7.0
```

##### 删除从阿里云拉取的镜像

```bash
docker rmi `docker images | grep aliyun | awk '{print $1":"$2}'`
```

#### 初始化 Master 节点

> 如果中途有中断可以使用`kubeadm reset`来恢复。如果初始化过程中有报错可以使用`journalctl -xeu kubelet`查看原因

```bash
kubeadm init --kubernetes-version=v1.20.5 --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=Swap
```

初始化成功后可以看到类型输出

```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

```

##### 查看集群状态

```bash
kubectl get ComponentStatus
```

这时我遇到了`controller-manager`和`scheduler`状态为`unhealthy`的情况。参考`https://my.oschina.net/u/4408611/blog/4660483`解决之。

##### 部署flannel网络插件

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

##### 从 master 节点取加入集群命令

```bash
kubeadm token create --print-join-command
```

#### 初始化 Node 节点

##### 拉取镜像

> 参考上文 初始化Master节点 相关内容

##### 加入集群

执行从 master 节点获取的加入集群命令

```bash
kubeadm join 172.16.168.68:6443 --token 29eeus.0g2ao4aieqywfjf7     --discovery-token-ca-cert-hash sha256:1464eadfbc8ece7378935096710b1d437134e6b626961c428f263ce1539ab332
```

成功后会有如下输出

```bash
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

##### 在 Master 节点查看集群信息

```bash
kubectl get nodes # 查看集群节点状态

kubectl get pods -n kube-system -o wide # 查看kube-system命名空间下组件运行状态
```
