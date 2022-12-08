---
title: Redis 知识梳理
tags:
  - Cache
categories: Frchitecture
keywords: 'blog,golang,php,framework,架构,缓存,redis'
description: Redis 知识梳理
cover: >-
  https://graph.linganmin.cn/210501/f9d5bf728f79544fde5f1e7afe4c0d91?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/210501/f9d5bf728f79544fde5f1e7afe4c0d91?x-oss-process=image/format,webp/quality,q_60
abbrlink: redis
date: 2021-04-21 07:30:20
---

## 数据结构

常用的五种类型

- String
  - 常用命令
    - get、set、incr、decr、mget
  - 使用场景
    - `key-value`缓存
    - 计数
- Hash
  - redis hash 是一个`String`类型的`field`和`value`映射表。
  - 常用命令
    - hget、hset、hgetall
  - 使用场景
    - 用户对象信息
- List
  - list 是简单的字符串列表，我们可以在列表的头部或尾部插入元素。
  - 实现方式
    - 双向链表，redis的list的每个元素都是`String`类型的双向链表
  - 常用命令
    - lpush（左进）、rpop（右出）、rpush（右进）、lpop（左出）、lrange（获取列表片段`lrange key start end`）
  - 使用场景
    - 粉丝列表
    - 消息队列
- Set
  - set 是`String`类型的无序集合，集合是通过`hashtable`实现的。与数学中的集合类似，redis 中的集合可以取`交集、并集、差集`等。因为`set`中的元素是没有顺序的，所以对 set 的添加、删除、查找的复杂度都是O(1)
  - 实现方式
    - `Set`内部实现是一个 value 永远为 null 的`HashMap`。
  - 常用命令
    - sadd、spop、smembers、sunion
  - 使用场景
    - 和`List`结构一样，`Set`可以提供列表功能，特殊之处在于`Set 可以自动去重`。当需要存储一个列表数据，又不希望出现重复数据时，`Set`是一个很好的选择。
  - 案例
    - 新浪微博使用`Set`存储用户关注和粉丝数据，可以使用`Set`的求交集、并集、差集等操作实现比如互相关注、共同喜好、二度好友等功能。
- SortedSet（zset）
  - zset 和 set 一样也是`String`元素的集合，且不允许重复的元素，zset 需要在集合中给每个元素设置对应的`score`。通过`score`就可以保证元素在集合中的顺序。
  - 实现方式
    - zset 内部使用`HashMap`和`SkipList（跳表）`保证数据的存储和有序。`HashMap`里存的是集合中成员和`score`的映射，跳表里存的是所有成员。
  - 常用命令
    - zadd、zrange、zrem、zcard
  - 使用场景
    - 有序列表/排名
    - 微博 feed 流
      - `zadd feed_uid 1620104824 content_id` 1620104824 = createtime
    - 延迟队列
      - 将需要执行的时间戳作为`score`写入集合，程序指定周期取指定时间段的数据消费
    - 排行榜
  
除了以上五种还支持以下类型

- HyperLogLog
- Geo
- Pub/Sub

## SkipList(跳表)在 Redis 中的使用

参考文章：[Redis 为什么用跳表而不用平衡树？](https://juejin.cn/post/6844903446475177998)

## 布隆过滤

> todo

## 管道(pipeline)

> todo


## 缓存问题

### 缓存雪崩

#### 原因

同一时间，大面积缓存失效，大量流量直接打到`DB`。

#### 解决

1. 设置每个`key`的失效时间都带一个随机值，保证数据不会在统一时间大面积失效。
2. 对于`热点数据`（比如：首页数据）设置永不过期，当管理后台有操作时，主动刷一下缓存。

### 缓存穿透

#### 原因

用户请求的数据，在系统`Cache`和`DB`中都没有（比如：我们用户表主键id是从1开始自增，有人拿-1的uid疯狂刷接口），当这种请求量很大时，会导致数据库压力过大。

#### 解决

1. 参数校验，参数大小限制，用户鉴权
2. 设置`DB`中无数据的`key`的缓存为`null`
3. 布隆过滤

### 缓存击穿

#### 原因

热点`key`失效瞬间，大量请求涌入`DB`。

#### 解决

1. 热点`key`不过期
2. 互斥锁

  ```golang
  package main

  import (
      "context"
      "errors"
      "fmt"
      "sync"
      "time"

      "github.com/go-redis/redis/v8"
  )

    var rdb *redis.Client

    func main() {
      ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
      defer cancel()

      err := initClient(ctx)

      if err != nil {
        fmt.Printf("initClient.error:%+v\n", err)
        return
      }

      key := "aa"
      num := 1000

      var r []string
      var wg sync.WaitGroup
      wg.Add(num)

      for i := 0; i < num; i++ {
        time.Sleep(time.Millisecond * 10)

        go func() {

          defer wg.Done()
          val, err := getData(ctx, key)
          if err != nil {
            fmt.Printf("getData.error:%+v\n", err)
            return
          }
          r = append(r, val)

        }()
      }

      wg.Wait()
      fmt.Printf("result.len:%+v", len(r))
    }

    func initClient(ctx context.Context) (err error) {

      rdb = redis.NewClient(&redis.Options{
        Addr:     "localhost:6379",
        Password: "",
        DB:       0,
        PoolSize: 1000,
      })

      if rdb == nil {
        return errors.New("redis.connected.faild")
      }

      _, err = rdb.Ping(ctx).Result()

      return err
    }

    // mockGetDataFromDB
    func mockGetDataFromDB(ctx context.Context, key string) (string, error) {

      fmt.Println("cache.miss")
      time.Sleep(time.Second * 2)
      return "hello", nil
    }

    // get Data
    func getData(ctx context.Context, key string) (string, error) {

      val, err := rdb.Get(ctx, key).Result()

      if err == nil {
        return val, nil
      }

      if err.Error() == redis.Nil.Error() { // 处理 cache miss
        // get lock
        if ok, err := rdb.SetNX(ctx, fmt.Sprintf("%s_lock", key), 1, time.Second*5).Result(); err == nil && ok {

          val, err := mockGetDataFromDB(ctx, key)
          if err != nil {
            return "", err
          }
          // cache
          _, err = rdb.SetEX(ctx, key, val, time.Second*5).Result()

          // del lock
          rdb.Del(ctx, fmt.Sprintf("%s_lock", key))

          if err == nil {
            return val, nil
          }

          return "", err

        } else {
          // sleep 1s
          time.Sleep(time.Second)
          return getData(ctx, key)
        }

      } else {
        return "", err
      }

    }
  ```

