---
title: 类微博/微信朋友圈 feed 流系统设计
tags:
  - Feed流
categories: 系统架构
keywords: 'blog,golang,php,framework,架构'
description: 类微博/微信朋友圈 feed 流系统设计
cover: >-
  https://graph.linganmin.cn/210410/1e1052bedd6dfa45b8c8626bec224ed2?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/210410/1e1052bedd6dfa45b8c8626bec224ed2?x-oss-process=image/format,webp/quality,q_40
abbrlink: 330d259a
date: 2021-04-07 07:30:20
---


- 写扩散(push)
  
  为每个用户维护一个订阅列表，记录该用订阅/关注的用户的消息的索引(一般为消息ID、类型、发布时间等基础数据)。

- 读扩散 (pull)

  为每个用户维护一个发送列表，记录这个用户所有发布的消息索引。

- 混合模式 (push + pull)

![时序图](https://graph.linganmin.cn/210410/1e1052bedd6dfa45b8c8626bec224ed2?x-oss-process=image/format,webp/quality,q_60)

### 使用场景

写扩展更适用于粉丝/好友低于某个阈值的场景，比如刘亦菲子微博有将近7000万粉丝，如果采用写扩散的方式来做，那么每当她发布一条微博就会产生将近7000万条队列消息，试想一下如果有100个相同体量的微博博主，每天每个人发布一条微博就会产生7亿条队列消息，对消息系统的压力是巨大的。

同样的，读扩散则更适用于粉丝/好友高于某个阈值的场景。

在一个庞大的类微博 feed 流系统中更合适的场景应该是基于某个阈值，同时使用读扩散和写扩散来实现内容的发布和拉取。

### 缓存

feed list 缓存可以使用 redis 的 zset 结构存储

```bash
zadd user_id 1618035919 creator_id:feed_id
```