---
title: 使用 Kubeadm 部署 Kubernetes 集群（基于 Containerd 运行时）
tags:
  - Kubernetes
  - Kubeadm
categories: 云原生
keywords: 'blog,golang,php,k8s,kubernetes,probe'
description: Kubeadm 安装 Kubernetes V1.22.2 踩坑手记，使用 Docker 作 Container Runtime
cover: >-
  https://graph.linganmin.cn/230201/236bd147e42b9a2fcd31784975f76399?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/230201/236bd147e42b9a2fcd31784975f76399?x-oss-process=image/format,webp/quality,q_60
abbrlink: c62b22dc
---

## 基础知识

### CRI、Containerd、OCI、runc

#### CRI

CRI(Container Runtime Interface)是`Kubernetes`定义的接口，定义了如何操作容器和镜像的统一规范。Kubernetes 尽量不关心您使用哪个容器运行时。 它只需要能够向它发送指令——为`Pod`创建容器、终止它们等等。 这就是`CRI`的用武之地。`CRI` 是对现在或将来可能存在的任何类型的容器运行时的抽象。 所以 `CRI` 让 Kubernetes 更容易使用不同的容器运行时。 `CRI` API描述了`Kubernetes`如何与任何运行时交互，而不是`Kubernetes`项目需要单独添加对每个运行时的支持。 只要给定的容器运行时实现了`CRI` API，运行时就可以随心所欲地创建和启动容器。

![You can choose your own container runtime for Kubernetes](https://graph.linganmin.cn/230201/ea90a93e75b45ff4dde97f1a4d96fdca?x-oss-process=image/format,webp/quality,q_60)

#### Containerd

Containerd 脱胎于`Docker`的高级容器运行时。它实现了`CRI`规范。它从镜像仓库中拉取镜像、管理它们，然后交给较低级别的运行时（`runc`），它使用`Linux`内核的特性来创建我们称为“容器”的进程。

#### OCI

Open Container Initiative 是一个开放的治理结构，其明确目的是围绕容器格式和运行时创建开放的行业标准

#### Runc

runc 是一个兼容`OCI`标准的容器运行时。它实现`OCI`规范并运行容器进程。
runc 为容器提供所有低级功能，与现有的低级`Linux`功能（如`namespaces`和`control groups`）交互。它使用这些功能来创建和运行容器进程。

![The relationship between Docker, CRI-O, containerd and runc – in a nutshell](https://graph.linganmin.cn/230201/e886b156ff4803fc27a44b382ee0fb1d?x-oss-process=image/format,webp/quality,q_60)

### ipvs

todo

## 环境

### 主机

- 系统版本
  - CentOS7.9
  - Linux 内核
    - 3.10.0-1160.el7.x86_64
- Master 节点
  - 172.16.168.135
  - 2C/4G
- Node 节点
  - 172.16.168.136
  - 2C/4G

### 软件版本

- Kubernetes
  - 1.25.3
- Containerd
  - 1.6.16

### 网段划分

参考另一篇关于网段划分的文章：[Kubernetes 集群部署之网段划分](https://blog.linganmin.cn/posts/4f614365/)

- 宿主机 CIDR
  - 172.16.168.0/24
- Kubernetes Service CIDR
  - 10.96.0.0/12
    - 10.96.0.1 ~ 10.111.255.254
- Kubernetes POD CIDR
  - 10.16.0.0/12
    - 10.16.0.1 ~ 10.31.255.254

### 初始化基础环境

>除`修改hostname`在各自节点执行，其余步骤均需在所有节点执行！！！

#### 修改hostname

此步骤在各自节点执行！！！

```bash
hostnamectl set-hostname master

hostnamectl set-hostname node1
```

#### yum update

```bash

yum update

```

#### 修改hosts文件，使机器可以使用hostname互通

```bash
cat >> /etc/hosts <<-EOF
172.16.168.135   master
172.16.168.136   node1
EOF
```

#### 关闭防火墙

```bash

systemctl stop firewalld && systemctl disable firewalld && systemctl status firewalld

```

#### 关闭 selinux

```bash

setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config

```

#### 关闭 swap

```bash

swapoff -a
sed -i '/swap/s/^\(.*\)$/#\1/g' /etc/fstab

```

#### 配置 iptables 的 ACCEPT 规则

```bash

iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat && iptables -P FORWARD ACCEPT

```

#### 转发 IPv4 并让 iptables 看到桥接流量

```bash

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 设置所需的 sysctl 参数，参数在重新启动后保持不变
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 应用 sysctl 参数而不重新启动
sudo sysctl --system


# 通过运行以下指令确认 br_netfilter 和 overlay 模块被加载
lsmod | grep br_netfilter
lsmod | grep overlay


# 通过运行以下指令确认 net.bridge.bridge-nf-call-iptables、net.bridge.bridge-nf-call-ip6tables 和 net.ipv4.ip_forward 系统变量在你的 sysctl 配置中被设置为 1：

sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward

```

#### 安装 ipvs

```bash

cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF

chmod 755 /etc/sysconfig/modules/ipvs.modules 

bash /etc/sysconfig/modules/ipvs.modules 

# 上面脚本创建了的/etc/sysconfig/modules/ipvs.modules 文件，保证在节点重启后能自动加载所需模块。 使用 lsmod | grep -e ip_vs -e nf_conntrack_ipv4 命令查看是否已经正确加载所需的内核模块。 

lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```

#### 安装 ipset & ipvsadm

```bash

yum install ipset -y

# 为了便于查看 ipvs 的代理规则，最好安装一下管理工具 ipvsadm
yum install ipvsadm -y

```

### 同步服务器时间

```bash

yum install chrony -y
systemctl enable chronyd
systemctl start chronyd
chronyc sources

```

### 安装 containerd

```bash

yum install -y yum-utils

yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

yum install -y containerd.io

```

#### 创建 containerd 配置文件

```bash

# 创建目录
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
# 替换配置文件的国内镜像
sed -i 's#sandbox_image = "registry.k8s.io/pause:3.6"#sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.6"#' /etc/containerd/config.toml
```

> 对于使用`systemd`作为`init system`的`Linux`发行版，使用`systemd`作为容器的`cgroup driver`可以确保节点在资源紧张的情况更加稳定，所以推荐将`containerd`的`cgroup driver`配置为`systemd`。[官网文档](https://kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/#cgroup-drivers）

修改前面生成的配置文件`/etc/containerd/config.toml`，在`plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options`配置块下面将`SystemdCgroup`设置为`true`，并且`kubelet`的`cgroupDriver`也要使用`systemd`以保持一致。

```bash

sed -i 's#SystemdCgroup = false#SystemdCgroup = true#' /etc/containerd/config.toml

```

#### 启动 containerd

```bash
systemctl enable containerd
systemctl start containerd
systemctl status containerd

# 验证
ctr version

```

### 安装三大件

#### Kubernetes repo

```bash

cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

```

#### 安装 kubeadm、kubelet、kubectl（我安装的指定版本 1.25.3，有版本要求自己设定版本）

```bash

yum install -y kubelet-1.25.3 kubeadm-1.25.3 kubectl-1.25.3

```

#### 设置运行时

```bash

crictl config runtime-endpoint /run/containerd/containerd.sock

```

#### 配置 kubelet 的 cgroup

`kubeadm`支持在执行`kubeadm init`时，传递一个`KubeletConfiguration`结构体。`KubeletConfiguration`包含`cgroupDriver`字段，可用于控制`kubelet`的`kubelet`驱动。

***在`kubeadm`1.22版本后，如果没有在`KubeletConfiguration`中设置`cgroupDriver`，`kubeadm`会将其默认设置为`systemd`***

下面是一个最小化示例，显式的配置了此字段：

```yaml
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
```

因为我们使用`kubeadm`版本较高，所以不用在`kubeadm config print init-defaults > kubeadm.yaml`生成的声明文件中配置此项。

#### 将 kubelet 设置成开机启动

> 此时如果启动`kubelet`将会失败，因为`kube-apiserver`还没有启动，`kubelet`无法建立连接。

```bash

systemctl daemon-reload

systemctl enable kubelet

```

## 初始化集群 Master 节点

通过如下命令导出默认的初始化配置

```bash

kubeadm config print init-defaults > kubeadm.yaml

```

### 修改配置文件

以下仅列出修改的点

```yaml

advertiseAddress: 1.2.3.4 # 修改为自己的 master 节点 IP
name: master # 修改为 master
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers  # 因为网络问题拉去不到官方镜像，修改为阿里云镜像地址
kubernetesVersion: 1.25.3 #确认是否为要安装版本，版本根据执行：kubelet --version 得来
podSubnet: 10.16.0.0/12　　　# networking: 下添加 pod 网段，注意该网段不能和主机在同一网段下
serviceSubnet: 10.96.0.0/12 # 修改 Service 网段

---
#  添加 proxy 模式使用 ipvs
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
 
```

### 初始化集群

```bash

kubeadm init --config=kubeadm.yaml

```

执行拷贝 kubeconfig 文件

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

添加节点（node1

```bash

kubeadm join 172.16.168.135:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:b4b8f89f3bf52cee593e3f385902fff5001b8c87ca2212527725d513dbc15ea2 
```

查看集群状态

```bash

[root@master ~]# kubectl get nodes
NAME     STATUS     ROLES           AGE    VERSION
master   NotReady   control-plane   6m7s   v1.25.3
node     NotReady   <none>          10s    v1.25.3

```

## 安装网络插件

 可以看到是``状态，这是因为还没有安装网络插件，必须部署一个容器网络接口 `(CNI)` 基于`Pod`网络附加组件，以便您的`Pod`可以相互通信。在安装网络之前，集群`DNS (CoreDNS)` 不会启动，这里我们安装`calico`，[calico文档](https://projectcalico.docs.tigera.io/about/about-calico)

```bash
# 下载calico 资源清单文件
wget https://projectcalico.docs.tigera.io/archive/v3.25/manifests/calico.yaml

# 安装
kubectl apply -f calico.yaml

# 查看 POD 运行状态
kubectl get pods -n kube-system
# NAME                                       READY   STATUS    RESTARTS   AGE
# calico-kube-controllers-74677b4c5f-tg9cm   1/1     Running   0          107s
# calico-node-8z52z                          1/1     Running   0          107s
# calico-node-wk8ww                          1/1     Running   0          107s
# coredns-7f8cbcb969-pnckr                   1/1     Running   0          10m
# coredns-7f8cbcb969-pqhkk                   1/1     Running   0          10m
# etcd-master                                1/1     Running   2          10m
# kube-apiserver-master                      1/1     Running   2          10m
# kube-controller-manager-master             1/1     Running   0          10m
# kube-proxy-7zx75                           1/1     Running   0          9m48s
# kube-proxy-f82rl                           1/1     Running   0          10m
# kube-scheduler-master                      1/1     Running   6          10m

```

至此`Kubernetes`安装完成。

## 部署一个 Deployment 资源

```bash

# 创建一个名叫 nginx 的 deployment pod 副本数为3
kubectl create deployment nginx --image=nginx:latest -r=3

# 查看容器状态
kubectl get pods
# NAME                     READY   STATUS    RESTARTS   AGE
# nginx-6d666844f6-hrlxv   1/1     Running   0          106s
# nginx-6d666844f6-n5tjz   1/1     Running   0          106s
# nginx-6d666844f6-t7865   1/1     Running   0          106s

# 查看 deployment
kubectl get deploy
# NAME    READY   UP-TO-DATE   AVAILABLE   AGE
# nginx   3/3     3            3           2m42s

```
