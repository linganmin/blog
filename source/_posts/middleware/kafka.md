---
title: Kafka 知识梳理
tags:
  - Kafka
  - 中间件
categories: 消息队列
keywords: 'blog,golang,php,framework,架构,kafka,卡夫卡，MQ'
description: Kafka 知识梳理
cover: >-
  https://graph.linganmin.cn/210418/1ff08ddea3bcd783bd398cfa51aef556?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/210418/1ff08ddea3bcd783bd398cfa51aef556?x-oss-process=image/format,webp/quality,q_40
abbrlink: bc68ea96
date: 2021-04-18 07:30:20
---

kafka 是一种高吞吐量，分布式，基于发布/订阅的消息系统。

## 特点

- 吞吐量高、延迟低
  - 每个`topic`可以支持`consumer group(多个consumer)`对`partition`进行操作
- 扩展性好
  - 集群支持热扩展
- 持久性、可靠性好
  - 消息数据是被持久化到本地磁盘的，并且支持数据备份
- 高并发
  - 支持千级别客户端同时读写

## 核心概念

- Broker
  - kafka 服务器，负责消息存储和转发
- Topic
  - 消息主题，用于分类消息
- Partition
  - `topic`的分区（sharding）,每个`topic`可以有多个`partition`，对应`topic`的消息保存在这些`partition`上，单`partition`的消息是有序的。同一个`topic`的多个`partition`不保证有序。
- Offset
  - 消息在`partition`上的位置，也表示消息的唯一序号。
- Producer
  - 消息生产者
- Consumer Group
  - 消息消费者`consumer`的组

### Controller Broker

`Kafka`使用`Zookeeper`保存了集群的`broker`、`topic`、`partition`等元数据。另外，`Zookeeper`负责从`broker`中选举一个作为`Controller`，并确保唯一性，当`Controller`宕机时，会重新选举一个新的。

`Controller Broker`本身就是一个普通的`broker`，只不过它需要多负载一些额外的工作

- 创建、删除`topic`
- 增加`partition`，并分配`leader partition`
- 集群`broker`管理，包括：新增`broker`，`broker`主动关闭，`broker`故障

## 存储

todo

## 提交 offset

> `consumer`通过提交`offset`来记录当前消费的最后位置，以便`consumer`发生崩溃或者有新的`consumer`加入`consumer group`时触发分区再均衡操作，每个`consumer`重新分配新的分区可以获取到该分区当前消费的最后位置。

kafka 通过`enable.auto.commit`控制`offset`提交方式。

- enable.auto.commit
  - =true
    - 通过后台线程周期性提交，默认周期是5s，参数`auto.commit.interval.ms`
  - =false
    - 不自动提交

### 自动提交

大部分自动提交是定时任务，`pool()`和`close()`也会提交当前最大偏移量。

- 缺点
  - 有丢失消息风险
    - 当`consumer`取到消息处理比较耗时，此时自动提交了`offset`，若此时`consumer`或处理程序崩溃便会导致当前消息丢失，因为在`broker`已经将此消息标识为处理完成了。因为当前`consumer`崩溃，所以其负责的`partition`被重新负载到其他`consumer`时，将从最新偏移量开始消费消息。
  - 有重复消费风险
    - 当`consumer`取到消息处理到一半，因为`auto.commit.interval.ms`配置时间过长，若此时`consumer`或处理程序崩溃，由于未提交`offset`，此消息会在负载再平衡后被其他`consumer`消费处理。

### 手动提交

为了避免消息丢失、重复消费，可以使用手动提交。

## Consumer 数据重复场景及解决方案

### 原因

数据消费完，没有及时提交 offset 到 broker，consumer 异常重启了。

### 方案

业务`幂等性`设计

## 如何保证不丢消息

### Producer 丢失情况

#### 场景

当 Producer 调用`send`方法发送消息后，消息可能会因为网络问题发送失败。

#### 方案

为了防止此种情况出现，我们需要关注`send`的结果，判断结果如果失败了，需要重新发送。一般重试次数会设置一个比较合理的值，比如 3，但也要根据业务场景做取舍。

### Consumer 丢失情况

#### 场景

当`Consumer`拉取到消息后，`Consumer`自动提交了`Offset`。`Consumer`突然挂掉，但此时`Consumer`的业务处理逻辑尚未执行完成

#### 方案
cd 
关闭自动提交`offset`，处理完逻辑手动提交。

当处理完业务逻辑，`Consumer`突然挂掉，尚未手动提交`offset`，会导致消息被重复消费。需要`Consumer`的幂等性。

### Kafka 弄丢了消息

> kafka 分区为多副本机制，分区中的多个副本会有一个`leader`副本，其他副本统称为`follower`。当我们发送消息时，其实是发送到`leader` 副本，然后 `follower` 副本从 `leader` 副本拉取信息进行同步。`Producer`和`Consumer`只与`leader`副本进行交互。

#### 场景

当`leader`副本所在的`broker`突然挂掉，就会从`follower`副本中选出一个新的`leader`，但是如果挂掉的那个`broker`上的副本数据没有没`follower`完全同步的话，就会造成消息丢失。

#### 方案

- 设置 acks = all
  - acks 默认值为1。表示消息被`leader`副本接收之后就算成功，acks = all 表示所有副本都要接收到该消息之后才算成功。

- 设置 replication.factor >= 3
  - 保证每个分区的副本个数量，此做法虽然会造成数据冗余，但也带来了数据的安全性。

- 设置 min.insync.replicas > 1
  - 默认值为1，设置为大于1表示消息至少被写入2个副本才算发送成功
