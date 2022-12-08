---
title: 使用 log-pilot + elk 搭建 docker stdout 日志解决方案
tags:
	- Docker
categories: Kubernetes
keywords: 'blog,golang,php,k8s,kubernetes,probe'
description: 使用 log-pilot + elk 搭建 docker stdout 日志解决方案
cover: >-
    https://graph.linganmin.cn/211107/bf81612871634692c7c423dcc5aef292?x-oss-process=image/format,webp/quality,q_60
abbrlink: 13167fc8
date: 2022-08-08 22:40:34
---

最近在写一个`Golang`的微服务项目，受限于企业内部网络等各种问题，日志无法上云只好自建日志收集管理平台了，记录一下实现方式。

> 整个环境从`服务`到`elk`到`log-pilot`均使用`docker-compose`部署

## log-pilot 介绍

log-Pilot 是一个阿里云开源的一款智能容器日志采集工具，它不仅能够高效便捷地将容器日志采集输出到多种存储日志后端，同时还能够动态地发现和采集容器内部的日志文件。log-pilot 通过声明式配置实现强大的容器事件管理，可同时获取容器标准输出和内部文件日志，解决了动态伸缩问题，此外，log-pilot 具有自动发现机制，CheckPoint 及句柄保持的机制，自动日志数据打标，有效应对动态配置、日志重复和丢失以及日志源标记等问题。

[log-Pilot代码仓库](https://github.com/AliyunContainerService/log-pilot?spm=a2c4g.11186623.0.0.496d2874ZxMmdE)

### log-pilot 特性

- 一个单独的 log 进程收集机器上所有容器的日志。不需要为每个容器启动一个 log 进程。
- 支持文件日志和 stdout
  - docker log dirver 亦或 logspout 只能处理 stdout，log-pilot 不仅支持收集 stdout 日志，还可以收集文件日志。
- 声明式配置
  - 当你的容器有日志要收集，只要通过 label 声明要收集的日志文件的路径，无需改动其他任何配置，log-pilot 就会自动收集新容器的日志。
- 支持多种日志存储方式
  - 无论是强大的阿里云日志服务，还是比较流行的 elasticsearch 组合，甚至是 graylog，log-pilot 都能把日志投递到正确的地点。

## 部署

### 创建共用 docker 网桥

```bash
docker network create tools 
```

### ELK

先在宿主机上执行如下命令，防止报错`max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]`

> max_map_count文件包含限制一个进程可以拥有的VMA(虚拟内存区域)的数量

```bash
sudo sysctl -w vm.max_map_count=262144
```

#### ekl 容器编排文件

```yaml
version: "3"

services:
  elk:
    image: sebp/elk:latest
    restart: always
    volumes:
      # 因为使用到的这个 ekl 镜像 默认开启了安全模式，如果不想使用可以参考下面文件内容，将 xpack.security.enabled 设为 false，配置文件挂载进容器
      - ./config/elasticsearch.yml:/etc/elasticsearch/elasticsearch.yml
    ports:
      - "5601:5601"
      - "9200:9200"
      - "5044:5044"

networks:
  default:
    external:
      name: tools
```

#### elasticsearch 配置文件关闭 security

```yaml
node.name: elk

path.repo: /var/backups

network.host: 0.0.0.0

cluster.initial_master_nodes: ["elk"]

xpack.security.enabled: false
```

#### 启动

```bash
# 启动
docker-compose up -d

# 查看容器状态
docker-compose ps
  Name              Command           State                              Ports
---------------------------------------------------------------------------------------------------------
elk_elk_1   /usr/local/bin/start.sh   Up      0.0.0.0:5044->5044/tcp,:::5044->5044/tcp,
                                              0.0.0.0:5601->5601/tcp,:::5601->5601/tcp,
                                              0.0.0.0:9200->9200/tcp,:::9200->9200/tcp, 9300/tcp,
                                              9600/tcp
```

#### 访问

从上面我们就可以看到elk容器暴露出的几个端口，通过`http://yourhost:5601`和`http://yourhost:9200`访问 dashboard 和 ES

```bash
curl 127.0.0.1:9200

{
  "name" : "elk",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "CAKwVjUERpWGpUbxdL2N7A",
  "version" : {
    "number" : "7.15.0",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "79d65f6e357953a5b3cbcc5e2c7c21073d89aa29",
    "build_date" : "2021-09-16T03:05:29.143308416Z",
    "build_snapshot" : false,
    "lucene_version" : "8.9.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

![dashboard](https://graph.linganmin.cn/211107/56b85f51c02e044cb31f9c30c5fc92db?x-oss-process=image/format,webp/quality,q_60)

### log-pilot

#### 编排文件

```yaml
version: '3'

services:
  logpilot:
    image: registry.cn-hangzhou.aliyuncs.com/acs/log-pilot:0.9.7-fluentd
    restart: always
    privileged: true # 这里要注意，我就踩了坑。不加容器会启动失败，提示权限受限
    command:
      --privileged
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /:/host # 将宿主机根目录挂载到容器的host目录下，容器内log-pilot程序是采集的/host下的日志
    environment:
      FLUENTD_OUTPUT: elasticsearch # 按需替换，把日志发送到 ElasticSearch，可以参考仓库代码将载体更换为你需要的
      ELASTICSEARCH_HOST: elk # 按需替换，ElasticSearch 的域名
      ELASTICSEARCH_PORT: 9200 # ElasticSearch 的端口号


networks:
  default:
    external:
      name: tools
```

#### 启动

```bash
docker-compose up -d 

docker-compose ps
       Name                      Command               State   Ports
--------------------------------------------------------------------
logpoilt_logpilot_1   /pilot/entrypoint --privileged   Up
```

### 自己的容器服务

我这里启动的是自己`golang`项目的服务，如果你参考此文档，这里需要部署自己的容器服务，并且服务的日志要采用`stdout`输出。

#### 编排文件

```yaml

version: "3"

services:
  gate:
    image: "换成你自己的镜像地址" # 要更换为自己的镜像哈！！！！
    restart: always
    labels:
      - "aliyun.logs.det=stdout" # 下面会有关于此label的说明哦，这也是logpilot收集日志的重要标识。
```

#### 启动

```bash
docker-compose up -d

docker-compose ps

   Name                  Command              State   Ports
-----------------------------------------------------------
gate_gate_1   /bin/sh -c ./app -f conf.yaml   Up

```

此时我们访问 ES 就可以看到我们日志的 index 了

```bash
curl 127.0.0.1:9200/_cat/indices?

......
yellow open det-2021.11.07                  JS3Y2zuSTt2VhVSwsfNgPg 1 1   191     0 142.3kb 142.3kb
```

### 配置 Kibana

浏览器打开`http://yourhost:5601/app/management`创建 Index patterns。

![kibana](https://graph.linganmin.cn/211107/5250e52e6d21639106bd6314b1a7e371?x-oss-process=image/format,webp/quality,q_60)

完成之后访问`http://yourhost:5601/app/discover`就可以查看日志了

![discover](https://graph.linganmin.cn/211107/c87ccdd23346c385cf77df0f82d60330?x-oss-process=image/format,webp/quality,q_60)

## 关于服务容器 label 的说明

上面启动自己服务时，我声明了下一个`label`来告诉`log-pilot`这个容器的日志位置。

```yaml
labels:
  - "aliyun.logs.det=stdout"
```

你可以在自己应用容器上添加更多标签，参考官方文档[容器日志采集利器Log-Pilot
](https://developer.aliyun.com/article/674327)

至此，便实现了使用 log-pilot + elk 搭建 docker stdout 日志解决方案

## 备注

此方案同样适用于采集容器内部生成的文件日志，但更推荐服务日志使用`stdout`方式输出。
