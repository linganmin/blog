---
title: 基于 Consul 实现 gRPC 服务注册与发现
tags:
  - 中间件
  - Consul
categories: Middleware
keywords: 'blog,golang,consul,etcd'
description: 学习 Consul 基础知识，并基于 Consul 实现 gRPC 服务注册发现。
cover: >-
  https://graph.linganmin.cn/220219/f7972adc36f4c23da00f92f05b0b6be7?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/220219/f7972adc36f4c23da00f92f05b0b6be7?x-oss-process=image/format,webp/quality,q_60
abbrlink: consul
date: 2022-02-19 03:30:20
---


## 什么是服务注册与发现，为什么需要服务注册发现

- 服务注册
  - 服务进程在注册中心注册自己的元数据，一般包括 Host 和 Port，有时候还有身份验证信息，协议，版本号以及运行环境等信息。
- 服务发现
  - 客户端进程向服务注册中心发起查询，来获取依赖的服务的信息，然后向其发送请求。服务发现一个重要的作用是提供给客户端进程一个可用的服务列表。

简单的说，当`服务A`需要依赖`服务B`时，服务发现中间件需要告诉`服务A`哪些地址是`服务B`的可用地址，这就是服务注册发现需要解决的问题。

![service-register](https://graph.linganmin.cn/220219/a0f05cf50814c0ac0077aa80e3bb962f?x-oss-process=image/format,webp/quality,q_60)

### 服务注册

- 客户端注册
  - 服务自身负责注册与注销工作，当服务启动后注册线程向注册中心注册，当服务停止时注销自己。
- 代理注册
  - 当服务启动后以某种方式通知代理服务，代理服务向注册中心发起注册工作。

#### 健康检测

- 被动检测
  - 服务主动向注册中心发送心跳消息，时间间隔可以自定义。注册中心如果在指定周期内未收到服务节点的心跳消息，则将其从该服务可用节点列表中移除
- 主动检测
  - 服务注册中心指定时间间隔内向所有列表中的服务节点发送心跳检测，如果指定周期内未成功则主动移除该节点。

### 服务发现

- 客户端发现
  - 客户端向注册中心发起请求查询指定服务的可用节点地址列表，客户端根据负载均衡算法选择一个节点发起调用请求。
- 代理发现
  - 通过路由服务转发客户端请求，负载均衡算法只需要在路由服务中实现。

## Consul

Consul 是分布式的、高可用的、可横向扩展的用于实现分布式系统的服务发现与配置的系统。

[Consul 官方文档](https://www.consul.io/docs)

### 原理

在`Consul`集群的节点（Agent）中分为`Server`和`Client`两种。

- Server
  - 保存数据，处理 Client 节点的请求，包含 Client 节点的所有功能。
  - Server 节点由一个 Leader 节点和多个 Follower 节点，Leader 节点会将数据同步到 Follower 节点，在 Leader 节点挂掉后，会启动选举机制产生一个新的 Leader。
- Client
  - Client 节点很轻量，它以 RPC 的方式向 Server 节点做读写请求转发。

### 架构图

![架构图](https://graph.linganmin.cn/220219/9e0b93fbc153bb64876dff1fed4e9267?x-oss-process=image/format,webp/quality,q_60)

- Consul 支持多数据中心，如上图所示。多数据中心通过 Internet 互联，为了提高通信效率，只有 Server 节点才可以跨数据中心通信
- Server 节点数量推荐是3个或5个（奇数个），对选举友好
- 集群内部节点通过`gossip`协议维护成员关系，每个节点都知道当前集群还有哪些节点，以及这些节点对应的类型（是 Client 还是 Server）。
- 对于集群内数据的操作（读写请求）可以直接发送到 Server 节点也可以通过 Client 节点使用 RPC 转发到 Server 节点。

### 常用启动参数

- -bootstrap
  - 该标志用于控制服务器是否处于“引导”模式。每个数据中心最多只能运行一个服务器，这一点很重要。从技术上讲，一个处于引导模式的服务器可以自我选择为Raft领导者。只有一个节点处于这种模式非常重要; 否则，一致性不能保证，因为多个节点能够自我选择。不建议在引导群集后使用此标志。
- -bind
  - 应为内部集群通信绑定的地址。这是集群中所有其他节点都应该可以访问的IP地址。默认情况下，这是“0.0.0.0”，这意味着 Consul 将绑定到本地计算机上的所有地址，并将 第一个可用的私有IPv4地址通告给群集的其余部分。
- -client
  - Consul 将绑定客户端接口的地址，包括HTTP和DNS服务器。默认情况下，这是“127.0.0.1”，只允许回送连接。
- -join
  - 启动时加入的另一位代理的地址。这可以指定多次以指定多个代理加入。如果Consul无法加入任何指定的地址，代理启动将失败。默认情况下，代理在启动时不会加入任何节点。
- -retry-join
  - 类似于-join第一次尝试失败时允许重试连接。这对于知道地址最终可用的情况很有用。
- -server
  - 此标志用于控制代理是否处于服务器或客户端模式。使用时，节点将作为 Server 服务器。
- -ui
  - 启用内置的Web UI服务器和所需的HTTP路由。

### 使用 docker-compose 部署 consul 集群

```yaml
version: '3'

services:
  consul1:
    image: consul:latest
    restart: always
    container_name: consul-node1
    command: agent -server -bootstrap -client 0.0.0.0 # -bootstrap 标识该服务是否处于”引导“模式，每个数据中心最多只能运行一台有该配置的服务，有该标识的服务可以自我选举为 leader

  consul2:
    image: consul:latest
    restart: always
    container_name: consul-node2
    command: agent -server -client 0.0.0.0 -retry-join consul-node1
    depends_on:
      - consul1

  consul3:
    image: consul:latest
    restart: always
    container_name: consul-node3
    command: agent -server  -client 0.0.0.0 -retry-join consul-node1
    depends_on:
      - consul1

  consul4: # client
    image: consul:latest
    restart: always
    container_name: consul-node4
    command: agent -ui -client 0.0.0.0 -retry-join consul-node1
    ports:
      - 8500:8500
    depends_on:
      - consul2
      - consul3

```

上面的 docker-compose 配置文件会启动一个有 3 个 Server 节点，一个 Client 节点的 Consul 集群。

![consul-ui](https://graph.linganmin.cn/220219/35878d9a941b348cff6592e30b4d8551?x-oss-process=image/format,webp/quality,q_60)

![consul-node](https://graph.linganmin.cn/220219/58f2c21284678b7ab3088cb38cff63b4?x-oss-process=image/format,webp/quality,q_60)

## gRPC

gRPC 是一个现代的开源的高性能的远程过程调用框架。

[gRPC官方文档](https://grpc.io/docs/)

### 编写一个 Golang gRPC 服务

目录结构

```bash
.
├── go.mod
├── go.sum
├── greeter_client
│   └── main.go
├── greeter_server
│   ├── main.go
│   └── services
│       └── services.go
├── main.go
├── pb
│   └── helloworld
│       └── helloworld.pb.go
└── protos
    └── helloworld.proto

```

#### 创建项目目录&初始化项目

  ```bash
  mkdir grpcdemo && cd grpcdemo && mkdir greeter_client greeter_server protos

  go mod init grpcdemo

  ```

#### 编写 proto 文件

```bash
vim protos/helloworld.proto
```

写入以下内容

```proto
syntax = "proto3";


option go_package = "pb/helloworld";


package helloworld;


service Greeter {

  rpc SayHello (HelloReq) returns (HelloResp);
}

message HelloReq {
  string name = 1;
}

message HelloResp {

  string message = 1;
}
```

#### 编译 proto 文件

```bash
 protoc --go_out=plugins=grpc:./  protos/helloworld.proto
```

#### 整理依赖包

```bash
go mod tidy
```

#### 编写 grpc server 代码

greeter_server/services/helloservice.go 写入以下内容

```golang
package services

import (
    "context"

    pb "grpcdemo/pb/helloworld"
)

type HelloService struct {
    pb.UnimplementedGreeterServer
}

func (h *HelloService) SayHello(ctx context.Context, req *pb.HelloReq) (*pb.HelloResp, error) {

    return &pb.HelloResp{Message: "saboran"}, nil
}

```

greeter_server/main.go 写入以下内容

```golang
package main

import (
    "flag"
    "fmt"
    "log"
    "net"

    "grpcdemo/greeter_server/services"
    pb "grpcdemo/pb/helloworld"

    "google.golang.org/grpc"
)

var port = flag.Int("port", 12123, "The server port")

func main() {

    flag.Parse()

    lis, err := net.Listen("tcp", fmt.Sprintf("localhost:%d", *port))

    if err != nil {
        log.Fatalf("failed to listen :%v", err)
    }

    s := grpc.NewServer()

    // 注册service
    //pb.RegisterGreeterServer(s, new(services.HelloService))
    pb.RegisterGreeterServer(s, &services.HelloService{})

    log.Printf("server listen at %v", lis.Addr())

    if err := s.Serve(lis); err != nil {
        log.Fatalf("failed to serve:%v", err)
    }
}

```

#### 编写 grpc client 代码

greeter_client/main 写入以下代码

```golang
package main

import (
    "context"
    "flag"
    "log"
    "time"

    pb "grpcdemo/pb/helloworld"

    "google.golang.org/grpc"
)

const defaultName = "world"

var (
    addr = flag.String("addr", "localhost:12123", "server addr")
    name = flag.String("name", defaultName, "name to reply")
)

func main() {

    flag.Parse()

    conn, err := grpc.Dial(*addr, grpc.WithInsecure())

    if err != nil {
        log.Fatalf("did not connect: %v", err)
    }

    defer conn.Close()

    c := pb.NewGreeterClient(conn)

    ctx, cancel := context.WithTimeout(context.Background(), time.Second)

    defer cancel()

    r, err := c.SayHello(ctx, &pb.HelloReq{Name: *name})

    if err != nil {
        log.Fatalf("could not greet:%v ", err)
    }

    log.Printf("Greeting: %s", r.Message)
}

```

#### 运行

```bash
# 一个终端启动 server
go run greeter_server/main.go

# 另一个终端执行 client 请求

☁ go run greeter_client/main.go

2022/02/20 20:09:18 Greeting: saboran

```

至此我们的 grpc 服务算是实现完成了

### gRPC 的 LB

在 greeter_client/main.go 中，我们是通过启动命令指定 server 地址的方式来实现访问到目标服务的，试想一下，如果此时 greeter_server 服务变更了端口号或者当前 client 执行命令传入的地址的 server 挂掉了，我们 client 端便会一直访问失败。所以这种方式在生产环境是不可行的。

gRPC 提供了关于 gRPC 负载均衡方案[Load Balancing in gRPC](https://github.com/grpc/grpc/blob/master/doc/load-balancing.md) 的定义，此方案是为 gRPC 设计实现的。

> gRPC 中负载均衡是基于每次 gRPC 调用，而不是基于每个客户端连接，也就是说即使请求都来自一个客户端，系统仍然希望在所有服务器之间进行负载均衡。

gRPC-GO 内置了`pick_first`，`round_robin`两种负载均衡策略。

- pick_first
  - 尝试逐个连接客户端地址，如果某一地址连接成功，则将其用于所有 RPC ，如果所有的失败，则报告错误
  - 默认策略
- round_robin
  - 连接所有地址，并依次向每个可用的后端发送 RPC 请求


### gRPC 的 Name Resolution

gRPC 中的默认 name-system 是 DNS，在客户端以插件形式提供了自定义 name-system 机制。

gRPC NameResolver 会根据 name-system 选择对应的解析器，用以解析用户提供的服务器名称，最终返回服务的地址列表（IP:Port）


[gRPC 名称解析文档](https://github.com/grpc/grpc/blob/master/doc/naming.md)


#### resolver 源码

```go
// https://github.com/grpc/grpc-go/blob/master/resolver/resolver.go

// Package resolver defines APIs for name resolution in gRPC.
// All APIs in this package are experimental.
package resolver

import (
  "context"
  "net"
  "net/url"

  "google.golang.org/grpc/attributes"
  "google.golang.org/grpc/credentials"
  "google.golang.org/grpc/serviceconfig"
)

var (
  // m is a map from scheme to resolver builder.
  m = make(map[string]Builder)
  // defaultScheme is the default scheme to use.
  defaultScheme = "passthrough"
)

// Register 注册 resolver builder 到 m 中，在初始化的时候使用，线程不安全
func Register(b Builder) {
  m[b.Scheme()] = b
}

// Get returns the resolver builder registered with the given scheme.
//
// If no builder is register with the scheme, nil will be returned.
func Get(scheme string) Builder {
  if b, ok := m[scheme]; ok {
    return b
  }
  return nil
}

// SetDefaultScheme sets the default scheme that will be used. The default
// default scheme is "passthrough".
func SetDefaultScheme(scheme string) {
  defaultScheme = scheme
}

// GetDefaultScheme gets the default scheme that will be used.
func GetDefaultScheme() string {
  return defaultScheme
}

// Address 描述一个服务的地址信息
type Address struct {
  Addr string

  ServerName string

  // 包含了关于这个地址用于任意数据
  Attributes         *attributes.Attributes
  BalancerAttributes *attributes.Attributes
}

// BuildOptions 创建解析器的额外信息
type BuildOptions struct {
  // DisableServiceConfig indicates whether a resolver implementation should
  // fetch service config data.
  DisableServiceConfig bool
  DialCreds            credentials.TransportCredentials
  Dialer               func(context.Context, string) (net.Conn, error)
}

// State 与 ClientConn 相关的当前 Resolver 状态。
type State struct {
  // 最新的 target 解析出来的可用节点地址集
  Addresses []Address

  ServiceConfig *serviceconfig.ParseResult

  Attributes *attributes.Attributes
}

// ClientConn 用于通知服务信息更新的 callback 
type ClientConn interface {
  // UpdateState updates the state of the ClientConn appropriately.
  UpdateState(State) error
  // ReportError notifies the ClientConn that the Resolver encountered an
  // error.  The ClientConn will notify the load balancer and begin calling
  // ResolveNow on the Resolver with exponential backoff.
  ReportError(error)

  // ParseServiceConfig parses the provided service config and returns an
  // object that provides the parsed config.
  ParseServiceConfig(serviceConfigJSON string) *serviceconfig.ParseResult
}

// Target represents a target for gRPC, as specified in:
// https://github.com/grpc/grpc/blob/master/doc/naming.md.
// It is parsed from the target string that gets passed into Dial or DialContext
// by the user. And gRPC passes it to the resolver and the balancer.
//
// If the target follows the naming spec, and the parsed scheme is registered
// with gRPC, we will parse the target string according to the spec. If the
// target does not contain a scheme or if the parsed scheme is not registered
// (i.e. no corresponding resolver available to resolve the endpoint), we will
// apply the default scheme, and will attempt to reparse it.

// Target 请求目标地址解析出的对象
type Target struct {

  // URL contains the parsed dial target with an optional default scheme added
  // to it if the original dial target contained no scheme or contained an
  // unregistered scheme. Any query params specified in the original dial
  // target can be accessed from here.
  URL url.URL
}

// Builder 创建一个 resolver 并监听更新
type Builder interface {
  // Build creates a new resolver for the given target.
  //
  // gRPC dial calls Build synchronously, and fails if the returned error is
  // not nil.
  Build(target Target, cc ClientConn, opts BuildOptions) (Resolver, error)
  // Scheme returns the scheme supported by this resolver.
  // Scheme is defined at https://github.com/grpc/grpc/blob/master/doc/naming.md.
  Scheme() string
}

// ResolveNowOptions includes additional information for ResolveNow.
type ResolveNowOptions struct{}

// Resolver 解析器监视指定目标的更新，包括地址更新和服务配置更新。
type Resolver interface {
  // ResolveNow will be called by gRPC to try to resolve the target name
  // again. It's just a hint, resolver can ignore this if it's not necessary.
  //
  // It could be called multiple times concurrently.
  ResolveNow(ResolveNowOptions)
  // Close closes the resolver.
  Close()
}

// UnregisterForTesting removes the resolver builder with the given scheme from the
// resolver map.
// This function is for testing only.
func UnregisterForTesting(scheme string) {
  delete(m, scheme)
}

```

### resolver 做的事情

- 解析 target 获取 scheme
- 调用 resolver.Get 根据 scheme 拿到对应的 Builder
- 调用 Builder.Build 方法
  - 解析 target
  - 获取服务地址的信息
  - 调用 ClientConn.UpdateState  callback 把服务信息传递给上层的调用方
  - 返回 Resolver 接口实例给上层
- 上层可以通过 Resolver.ResolveNow 方法主动刷新服务信息

#### 参考官方 dns_resolver 实现 consul_resolver

```go
package lb

import (
    "fmt"
    "log"
    "net/url"
    "strings"
    "sync"

    "github.com/hashicorp/consul/api"
    "google.golang.org/grpc/resolver"
)

type consulResolver struct {
    address              string
    tag                  string
    wg                   sync.WaitGroup
    cc                   resolver.ClientConn
    name                 string
    disableServiceConfig bool
    lastIndex            uint64
}

// ResolveNow 更新逻辑在 watcher 里处理掉了
func (c *consulResolver) ResolveNow(o resolver.ResolveNowOptions) {

}

// Close 暂时不处理
func (c *consulResolver) Close() {

}

// 实现了调用 consul 接口获取指定服务的可用节点
// WaitIndex 用于阻塞，直到有新的可用节点，避免重复刷新
// 将获取到的可用节点更新 c.cc.UpdateState
// 支持了 consul 的 tag 过滤，在 target 通过 query 参数传递
func (c *consulResolver) watcher() {

    defer c.wg.Done()

    conf := api.DefaultConfig()

    conf.Address = c.address

    client, err := api.NewClient(conf)

    if err != nil {
        log.Fatalf("create consul client err:%+v", err)
    }

    for {

        services, meta, err := client.Health().Service(c.name, c.tag, true, &api.QueryOptions{WaitIndex: c.lastIndex})

        if len(services) == 0 {
            panic(fmt.Sprintf("no available endpoints for server:%s,tag:%s", c.name, c.tag))
        }
        if err != nil {
            fmt.Printf("retrieving instances from consul err: %+v", err)
            continue
        }
        c.lastIndex = meta.LastIndex

        var endpoints []resolver.Address

        for _, service := range services {
            endpoints = append(endpoints, resolver.Address{
                Addr: fmt.Sprintf("%v:%v", service.Service.Address, service.Service.Port),
            })
        }

        _ = c.cc.UpdateState(resolver.State{
            Addresses: endpoints,
        })
    }
}

// ------------

const (
    schemeName = "consul"
)

type consulBuilder struct {
}

func Init() {
    resolver.Register(NewBuilder())
}

func NewBuilder() resolver.Builder {
    return &consulBuilder{}
}

func (b *consulBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {

    // 解析 target 获取 consul 地址，服务名，服务tag
    host, _, name, tag, err := parseTarget(target.URL)

    if err != nil {
        return nil, err
    }

    c := &consulResolver{
        address:              host,
        tag:                  tag,
        cc:                   cc,
        name:                 name,
        disableServiceConfig: opts.DisableServiceConfig,
        lastIndex:            0,
    }

    c.wg.Add(1)
    go c.watcher()

    return c, nil
}

func (b *consulBuilder) Scheme() string {
    return schemeName
}

func parseTarget(target url.URL) (host, port, name string, tag string, err error) {

    tag = target.Query().Get("tag")

    return target.Host, target.Port(), strings.Replace(target.Path, "/", "", -1), tag, err
}

```

## 实现 Demo 代码地址

[基于 Consul 作为 NameResolver 解析器实现 gRPC 服务发现的 Demo](https://github.com/linganmin/grpc-service-discovery-consul-demo)

``

### grpc server

```go
// gRPC服务是使用Protobuf(PB)协议的，而PB提供了在运行时获取Proto定义信息的反射功能。
// grpc-go中的"google.golang.org/grpc/reflection"包就对这个反射功能提供了支持。
// 通过该反射我们就可以使用类似 grpcurl 的终端工具测试 rpc 接口了
reflection.Register(s)

// 健康检查
// 官方文档：https://github.com/grpc/grpc/blob/master/doc/health-checking.md
// gRPC-go 提供了健康检测库：https://pkg.go.dev/google.golang.org/grpc/health?tab=doc 把上面的文档接口化了。
grpc_health_v1.RegisterHealthServer(s, health.NewServer())

```

## grpc client

```go
  target := "consul://localhost:8500/hello.service" // schema:[//authority/]host[:port]/service[?query] 参考文档：https://github.com/grpc/grpc/blob/master/doc/naming.md
  ctx, cancel := context.WithTimeout(context.Background(), time.Second*10)
  defer cancel()

  conn, err := grpc.DialContext(ctx,
      target,
      grpc.WithTransportCredentials(insecure.NewCredentials()),
      grpc.WithDefaultServiceConfig(fmt.Sprintf(`{"loadBalancingConfig": [{"%s":{}}]}`, roundrobin.Name))) // 负载均衡策略，默认 pick_first ,文档：https://github.com/grpc/grpc/blob/master/doc/load-balancing.md

```
## Run

### 环境依赖

1. 和本地环境相通的 consul ，例如在本机使用 docker 启动一个 consul 节点
2. 将代码中的 consul 地址`localhost:8500`替换为可用地址
3. 执行`go mod tidy`处理依赖
4. `go run greeter_server/main.go` 启动服务，也可指定端口，例如：`go run greeter_server/main.go  -port 12124`, 可以去 consul dashboard 查看服务注册及健康检查状态，可以指定端口多启动几个节点
5. `go run greeter_client/main.go -name 小下` 发起客户端请求

## 参考

- [聊聊微服务的服务注册与发现](https://www.cnblogs.com/ExMan/p/11869742.html)
- [Consul源码分析——Raft实现](https://www.jianshu.com/p/307be3bd3b61)
- [由 Consul 谈到 Raft](https://toutiao.io/posts/js549a/preview)