---
title: 搭建基于 Kubernetes 生产可用的日志和监控系统
tags:
  - Kubernetes
  - Docker
categories: Kubernetes
keywords: 'blog,golang,php,k8s,kubernetes,probe'
description: 搭建基于 Kubernetes 生产可用的日志和监控系统
cover: >-
  https://graph.linganmin.cn/220529/b208e0a803bbec39eab9007b774f2d92?x-oss-process=image/format,webp/quality,q_10
top_img: >-
 https://graph.linganmin.cn/220529/b208e0a803bbec39eab9007b774f2d92?x-oss-process=image/format,webp/quality,q_40
abbrlink: k8s-sys
date: 2022-05-29 22:40:34
---

### 前言

本篇文章涉及很多 Kubernetes 基础知识，包括但不限于以下资源对象：

- Namespace
- Pod
- Deployment
- Service
  - Headless Service
- Volume
  - PersistenVolume
    - PersistenVolumeClaim
- StatefulSet
- DaemonSet
- Ingress
- Role
  - ClusterRole

如果这些东西你还很陌生，请转战[Kubernetes中文官方文档](https://kubernetes.io/zh/docs/home/)

### 基础设施

#### 安装工具

```bash
yum install bind-utils -y
```

#### 内网免密登录

```bash
ssh-keygen
```

master 节点的 pub key 加入各节点的 auth key

---

### 初始化集群

为了节省时间使用了 sealos 工具来搭建 K8S 集群，如果你想了解使用 Kubeadm 部署 K8S 集群可以参考笔者的另一篇文章[Kubeadm 安装 Kubernetes V1.22.2 踩坑手记](https://blog.linganmin.cn/posts/k8s-install-note.html)
#### 工具

```bash
wget -c https://sealyun-home.oss-cn-beijing.aliyuncs.com/sealos/latest/sealos && \
    chmod +x sealos && mv sealos /usr/bin 
```

#### 资源包

```bash
wget -c https://sealyun.oss-cn-beijing.aliyuncs.com/05a3db657821277f5f3b92d834bbaf98-v1.22.0/kube1.22.0.tar.gz

```

#### init

```bash
sealos init \ --user --pk /root/.ssh/id_rsa \
--master 172.16.168.2  \ # 更换ip
--node 172.16.168.3 \ # 更换 ip
--pkg-url /root/kube1.22.0.tar.gz \
--version v1.22.0
```

---

### 命令行工具

#### kubectl 补全

```bash

yum install bash-completion -y

source /usr/share/bash-completion/bash_completion

echo 'source <(kubectl completion bash)' >>~/.bashrc
```

#### helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

---

### 安装 traefik

```bash
git clone https://github.com/traefik/traefik-helm-chart

# 编辑
vim values-custom.yaml

```

#### 自定义 value

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

```bash

helm install -n traefik traefik ./traefik/ -f ./values-custom.yaml

```

#### dashboard 资源文件

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

---

### 部署 Nginx 服务

#### 生成资源文件

```bash

 kubectl create deployment nginx  --image=nginx --replicas=3 --port=80  -o=yaml --dry-run > deployment.yaml

 kubectl create svc clusterip nginx --tcp=80:80  -o=yaml --dry-run > service.yaml
```

#### 使用 traefik 转发 nginx

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: nginx

spec:
  entryPoints:
    - web
  routes:
  - match: PathPrefix(`/nginx`)
    kind: Rule
    services:
      - name: nginx
        port: 80
    middlewares:
      - name: stripprefix
        namespace: default
---

apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: stripprefix
  namespace: default

spec:
  stripPrefix:
    prefixes:
      - /nginx

```

---

### 安装 EFK

#### 部署 NFS 服务

因为 ES 需要部署有状态服务，存储需要共享磁盘，所以选用 NFS

```bash
# 安装
yum install nfs-utils

# 设为开机启动
systemctl enable rpcbind
systemctl enable nfs

# 启动

systemctl start rpcbind
systemctl start nfs


# 配置共享目录

mkdir /data
chmod 755 /data

# 配置文件
vi /etc/exports

# /data/     *(rw,sync,no_root_squash,no_all_squash)

# 编辑之后重启
systemctl restart nfs

```

##### 配置说明

- /data: 共享目录位置。
- 192.168.0.0/24: 客户端 IP 范围，* 代表所有，即没有限制。
- rw: 权限设置，可读可写。
- sync: 同步共享目录。
- no_root_squash: 可以使用 root 授权。
- no_all_squash: 可以使用普通用户授权。

##### 客户端安装

```bash
yum install nfs-utils

# 开机自启动
systemctl enable rpcbind

# 启动服务
systemctl start rpcbind

# 检查服务端
showmount -e 172.16.168.3

# 在客户端创建目录
mkdir /data

# 挂载
mount -t nfs 172.16.168.3:/data /data

# 挂载之后，可以使用 mount 命令查看一下
mount

# 如果看到类似 172.16.168.3:/data on /data type nfs4 (rw,relatime,vers=4.1,rsize=524288,wsize=524288,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=172.16.168.2,local_lock=none,addr=172.16.168.3) 说明挂载成功了

```

#### 部署 NFS provisioner

##### RBAC

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: loging
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: loging
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: loging
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: loging
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: loging
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io

```

##### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: logging
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: lank8s.cn/sig-storage/nfs-subdir-external-provisioner:v4.0.2
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME # 这个名字下面要用！！！
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: 172.16.168.3
            - name: NFS_PATH
              value: /data
      volumes:
        - name: nfs-client-root
          nfs:
            server: 172.16.168.3
            path: /data

```

##### StorageClass

```yaml

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client 
  namespace: logging
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner #  must match deployment's env PROVISIONER_NAME'
parameters:
  archiveOnDelete: "false"
```

#### 部署 ES 和 Kibana

##### ES 有状态 POD（StatefulSet）

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  namespace: logging
  name: es-cluster
spec:
  serviceName: elasticsearch
  replicas: 2
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: elasticsearch:7.5.0
        resources:
            limits:
              cpu: 1000m
            requests:
              cpu: 100m
        ports:
        - containerPort: 9200
          name: rest
          protocol: TCP
        - containerPort: 9300
          name: inter-node
          protocol: TCP
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
        env:
          - name: cluster.name
            value: k8s-logs
          - name: node.name
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: discovery.seed_hosts
            value: "es-cluster-0.elasticsearch,es-cluster-1.elasticsearch"
          - name: cluster.initial_master_nodes
            value: "es-cluster-0,es-cluster-1"
          - name: ES_JAVA_OPTS
            value: "-Xms512m -Xmx512m"
      initContainers:
      - name: fix-permissions
        image: busybox
        command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
        securityContext:
          privileged: true
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
      - name: increase-vm-max-map
        image: busybox
        command: ["sysctl", "-w", "vm.max_map_count=262144"]
        securityContext:
          privileged: true
      - name: increase-fd-ulimit
        image: busybox
        command: ["sh", "-c", "ulimit -n 65536"]
        securityContext:
          privileged: true
  volumeClaimTemplates: # pv 模板
  - metadata:
      name: data
      labels:
        app: elasticsearch
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "nfs-client" # 使用上面定义的 StroageClass
      resources:
        requests:
          storage: 10Gi # 申请的pvc大小
```

###### 无头服务（Headless Service）

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: logging
  name: elasticsearch
  labels:
    app: elasticsearch
spec:
  selector:
    app: elasticsearch
  clusterIP: None
  ports:
    - port: 9200
      name: rest
    - port: 9300
      name: inter-node

```

##### Kibana

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: logging
  labels:
    app: kibana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: kibana:7.5.0
        resources:
          limits:
            cpu: 1000m
          requests:
            cpu: 100m
        env:
          - name: ELASTICSEARCH_URL
            value: http://elasticsearch:9200
        ports:
        - containerPort: 5601

---

apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: logging
spec:
  selector:
    app: kibana
  type: NodePort  # 使用node port模式
  ports:
    - port: 8080
      targetPort: 5601
      nodePort: 30000


```


#### 使用 helm 部署 fluentd

```bash

helm repo add fluent https://fluent.github.io/helm-charts

helm search repo fluent

helm pull fluent/fluentd

```

然后解压下载来的压缩包，得到如下目录

```bash
fluentd
├── Chart.yaml
├── dashboards
├── README.md
├── templates
└── values.yaml
```

拷贝一份 values.yaml 用来自定义相关配置

```bash
cp values.yaml values-custom.yaml
```

编辑 values-custom.yaml，以下为修改的内容

```yaml
# 因为从 kubernetes 新版本默认使用 containerd 作为运行时，containerd/cri-o 使用不同的日志格式。要解析此类日志，需要改用cri解析器
# https://github.com/fluent/fluentd-kubernetes-daemonset#use-cri-parser-for-containerdcri-o-logs
fileConfigs:
  01_sources.conf: |-
    <source>
      @type tail
      @id in_tail_container_logs
      @label @KUBERNETES
      path /var/log/containers/*.log
      pos_file /var/log/fluentd-containers.log.pos
      tag kubernetes.*
      read_from_head true
      <parse>
        @type cri
      </parse>
      emit_unmatched_lines true
    </source>


# 此处修改 es 的地址，host 和 port
  04_outputs.conf: |-
    <label @OUTPUT>
      <match **>
        @type elasticsearch
        host "elasticsearch.logging.svc.cluster.local"
        port 9200
        path ""
        user elastic
        password changeme
      </match>
    </label>
```

安装

```bash
helm install fluentd fluentd/ -f fluentd/values-c.yaml -n logging

# 卸载
helm uninstall fluentd -n logging
```

![日志](https://graph.linganmin.cn/220529/08abd97688479e667a8a6350fdc1b774?x-oss-process=image/format,webp/quality,q_60)

--- 

### Prometheus + Grafana 监控

#### 使用 Helm 安装 Prometheus

Get Repo Info

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

下载 Charts，自定义 Values

```bash
helm pull  prometheus-community/prometheus

# 然后将对应压缩包解压，你将得到如下目录结构
prometheus/
├── Chart.lock
├── charts
│   └── kube-state-metrics
├── Chart.yaml
├── README.md
├── templates
│   ├── alertmanager
│   ├── _helpers.tpl
│   ├── node-exporter
│   ├── NOTES.txt
│   ├── pushgateway
│   └── server
└── values.yaml

# 我们需要将 values.yaml 复制一份，自定义部分数据，比如镜像（默认镜像地址可能国内无法下载）、StorageClass

cp values.yaml values-custom.yaml

# 然后编辑 values-custom.yaml
```

修改之后的 values-custom.yaml 完整内容参考：https://github.com/linganmin/charts-custom-value/blob/master/prometheus/custom-values.yaml ，下面只列出做了修改的地方

```yaml
# ...
### 镜像更换为了 RED HAT 的镜像地址
image:
  repository: quay.io/prometheus/alertmanager
  tag: v0.23.0
  pullPolicy: IfNotPresent

# ...
### 涉及到的存储都用了 nfs 的storageClass
storageClass: "nfs-client"
# ...

### configmapReload 的镜像更换为了我自己打的，原因是我用的 containerd 版本拉取官方镜像会失败，不是网络问题，是 containerd 的版本问题，较新版本无此问题
name: configmap-reload

## configmap-reload container image
##
image:
  repository: registry.cn-hangzhou.aliyuncs.com/lanni-base/configmap-reload
  tag: latest
  pullPolicy: IfNotPresent

# ...

### kubeStateMetrics 这里没有启用，因为如果这里启用会拉取官方镜像，国内会失败，所以单独部署
kubeStateMetrics:
  ## If false, kube-state-metrics sub-chart will not be installed
  ##
  enabled: false

# ...

### 镜像更换为了 RED HAT 的镜像地址
name: node-exporter

## node-exporter container image
##
image:
  repository: quay.io/prometheus/node-exporter
  tag: v1.3.0
  pullPolicy: IfNotPresent

# ...

### node-exporter 需要收集 master 节点的信息，所以要增加污点容忍

## Node tolerations for node-exporter scheduling to nodes with taints
## Ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/
##
tolerations: 
  - key: "node-role.kubernetes.io/master"
    operator: "Exists"
    effect: "NoSchedule"

# ...

### 镜像更换为了 RED HAT 的镜像地址

## Prometheus server container image
##
image:
  repository: quay.io/prometheus/prometheus
  tag: v2.34.0
  pullPolicy: IfNotPresent

# ...

```

##### 安装

```bash
helm install prometheus ./prometheus -f prometheus/values-custom.yaml -n monitoring
```

![prometheus](https://graph.linganmin.cn/220529/578095a010d9841d24e09074e4892458?x-oss-process=image/format,webp/quality,q_60)

#### 使用 Helm 安装 Grafana


TODO

![grafana](https://graph.linganmin.cn/220529/d7e657361eaed3c510a52a5c11d29b50?x-oss-process=image/format,webp/quality,q_60)

### 同步系统时间

ntpdate time.windows.cn

hwclock --systohc


### 代理

#### 替换 k8s.gcr.io

lank8s.cn替换k8s.gcr.io

### Ref

https://blog.51cto.com/u_11734401/4286237#efk%E6%97%A5%E5%BF%97%E7%B3%BB%E7%BB%9F
<https://devopscube.com/setup-efk-stack-on-kubernetes/>

<https://qizhanming.com/blog/2018/08/08/how-to-install-nfs-on-centos-7>
<https://www.cnblogs.com/panwenbin-logs/p/12196286.html>

[Kubernetes（k8s）helm 搭建 prometheus + Grafana 监控](https://www.akiraka.net/kubernetes/350.html)