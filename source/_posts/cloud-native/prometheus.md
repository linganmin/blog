---
title: kube-prometheus-stack Usage
tags:
  - Kubernetes
  - Prometheus
categories: 云原生
keywords: 'blog,golang,php,k8s,kubernetes,prometheus'
description: kube-prometheus-stack 部署使用
cover: >-
  https://graph.linganmin.cn/230501/b1800426df2eeba8c39b3e1afcb43802?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/230501/b1800426df2eeba8c39b3e1afcb43802?x-oss-process=image/format,webp/quality,q_10
abbrlink: 57dfec22
---

## Environment

- Kubernetes 集群
  - 需要一个已经部署完成且可用的`Kubernetes 1.16+`集群。
- Helm
  - helm version v3+

## Steps

- 添加 helm 仓库

  ```bash
  helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
  helm repo update
  ```

- 将仓库拉取到本地

  ```bash
  helm pull prometheus-community/kube-prometheus-stack
  ```

- 将拉取到的压缩包解压

  ```bash
  tar -zxvf kube-prometheus-stack-45.8.1.tgz 
  
  # kube-prometheus-stack/Chart.yaml
  # kube-prometheus-stack/Chart.lock
  # kube-prometheus-stack/values.yaml
  # kube-prometheus-stack/templates/NOTES.txt
  # kube-prometheus-stack/templates/_helpers.tpl
  # kube-prometheus-stack/templates/alertmanager/alertmanager.yaml
  # kube-prometheus-stack/templates/alertmanager/extrasecret.yaml
  # ...
  ```

- 修改`values.yaml`
  - 需要修改的点
    - 因为默认`Chart`中有一些`k8s.gcr.io`的镜像国内拉取会失败，所以要换成`docker.io`的镜像
    - 开启你需要的配置比如`grafana`和`prometheus`的`ingress`等，参考如下图
      - ![grafana](https://graph.linganmin.cn/230430/9177626d299b676416087770a842706d?x-oss-process=image/format,webp/quality,q_60)

- 去`Kubernetes`的 Master 节点部署

  ```bash
  # 创建目标 namespace
  kubectl create ns monitoring

  cd kube-prometheus-stack

  helm install -n monitoring prometheus-stack . -f ./values.yaml

  ```

  在另外一个终端查看一下 pod 启动情况

  ```bash
  watch kubectl get pods -n monitoring
  ```

  > 如果有 POD 启动失败或者报`ImagePullError`类似的错误，基本就是因为默认镜像拉不到，需要修改当前目录下的`values.yaml`或对应服务下的`values.yaml`的镜像地址。

- 查看服务状态

  ```bash
  kubectl get pods -n monitoring
  
  # NAME                                                     READY   STATUS    RESTARTS        AGE
  # alertmanager-prometheus-kube-prometheus-alertmanager-0   2/2     Running   1 (2d13h ago)   2d13h
  # prometheus-grafana-76767f949d-m4phk                      3/3     Running   0               4d1h
  # prometheus-kube-prometheus-operator-588b89d9bc-kscrt     1/1     Running   0               6d1h
  # prometheus-kube-state-metrics-6f7c84c8d6-fkdss           1/1     Running   0               6d1h
  # prometheus-prometheus-kube-prometheus-prometheus-0       2/2     Running   0               43h
  # prometheus-prometheus-node-exporter-2rfg2                1/1     Running   0               6d1h
  # prometheus-prometheus-node-exporter-6mphj                1/1     Running   0               6d1h
  # prometheus-prometheus-node-exporter-gvd78                1/1     Running   0               6d1h
  # prometheus-prometheus-node-exporter-tcncb                1/1     Running   0               6d1h

  ```

## Prometheus Usage

> Prometheus通过使用平台提供的API就可以找到所有需要监控的云主机。在Kubernetes这类容器管理平台中，Kubernetes掌握并管理着所有的容器以及服务信息，那此时Prometheus只需要与Kubernetes打交道就可以找到所有需要监控的容器以及服务对象。Prometheus还可以直接与一些开源的服务发现工具进行集成，例如在微服务架构的应用程序中，经常会使用到例如Consul这样的服务发现注册软件，Promethues也可以与其集成从而动态的发现需要监控的应用服务实例。除了与这些平台级的公有云、私有云、容器云以及专门的服务发现注册中心集成以外，Prometheus还支持基于DNS以及文件的方式动态发现监控目标，从而大大的减少了在云原生，微服务以及云模式下监控实施难度。

- 基于 Kubernetes 的服务发现示例（kubernetes_sd_config）
  
  ```yaml
      - job_name: 'k8s-pod-demo'
        kubernetes_sd_configs:
        - role: pod
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
          target_label: __address__
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: kubernetes_pod_name
  ```

- 基于静态配置的实例

  ```yaml
      - job_name: 'static-demo'
        scheme: https
        tls_config:
          insecure_skip_verify: true
        metrics_path: '/metrics'
        static_configs:
        - targets: ['www.demo.local']
          labels:
            instance: demo
  ```

详情参考官方文档[Prometheusg 官方文档](https://prometheus.io/docs/prometheus/latest/getting_started/)

## Grafana Usage

- 登录
  - Grafana 默认账号为`admin`，默认密码可以进入`prometheus-grafana`的 POD 通过环境变量获取。

    ```bash
    # 进入自己 Grafana POD 
    kubectl exec -it  prometheus-grafana-76767f949d-m4phk -n monitoring  -- sh

    # 从环境变量获取默认密码
    env | grep PASS

    # REQ_PASSWORD=prom-operator

    ```

- 监控自己的指标
  - 添加一个面板
  
  ![pannel](https://graph.linganmin.cn/230501/45f46be8315d73acb9db1322cc639c9d?x-oss-process=image/format,webp/quality,q_60)

  - 写 promQL

    ![promQL](https://graph.linganmin.cn/230501/a70a6fc907183105b8f247267765900d?x-oss-process=image/format,webp/quality,q_60)


> 详细参考官方文档[Gafana官方文档](https://grafana.com/docs/grafana/latest/dashboards/use-dashboards/)
