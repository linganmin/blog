---
title: 使用 Opentracing 实现 Golang 链路追踪
tags:
  - Golang
  - Opentracing
  - Jeager
categories: Golang
keywords: 'blog,golang,opentracing,jeager'
description: 了解 Opentracing（分布式链路追踪）并尝试在 Golang 中使用
cover: >-
  https://graph.linganmin.cn/220217/66b241b8bb939fb3fbac1dc2956f3114?x-oss-process=image/format,webp/quality,q_60
top_img: >-
  https://graph.linganmin.cn/220217/66b241b8bb939fb3fbac1dc2956f3114?x-oss-process=image/format,webp/quality,q_60
abbrlink: opentracing
date: 2022-02-17 22:40:34
---

## 为什么需要链路追踪

在微服务架构系统中，请求在各服务之间流转，调用链路错综复杂，一旦出现了问题和异常，定位问题相当困难。链路追踪系统可以追踪并记录请求在系统中的调用顺序、调用时长等一系列关键信息，从而帮我们更简单的定位服务异常。

## Opentracing

Opentracing 是分布式链路追踪的一种规范标准，是 CNCF（云原生计算基金）孵化的项目之一。和一般规范标准不同，Opentracing 不是传输协议、也不是消息格式上的规范标准，而是一种语言层面上的 API 标准。只要在某链路追踪系统实现了 Opentracing 规定的接口，符合 Opentracing 定义的表现行为，那么就可以说该应用符合 Opentracing 标准。这意味着，开发者只需要修改少量的配置代码，就可以在符合 Opentracing 标准的链路追踪系统之间自由切换。

### 数据模型

#### Span

Span 是一条链路追踪的基本组成要素，一个 Span 表示一个独立的工作单元，比如一次函数调用，一次 RPC 请求，Span 会记录一些基本要素：

- 操作/行为名称
- 开始时间
- 结束时间
- Tags(一组零个或多个 key:value 的 Span Tag 键值对的标签，键必须是字符串，值可以是字符串、数字、布尔值类型)
- Logs(一组零个或多个 Span Log)
- 一个 SpanContext
- 对零个或多个的 Span 引用

#### Tags

Tags 以键值对的形式保存用户自定义标签，主要用于链路追踪结果的查询过滤。Span 的 Tag 仅自己可见，不会随着 SpanContext 传递给后续 Span

```golang

span.SetTag("rpc.grpc.status_code",0)

span.SetTag("rpc.systeam","grpc")

```

#### Logs

与 Tags 类似，也是键值对形式，不同的是 Logs 还会记录 Logs 的时间，因此 Logs 主要用于记录某些事件发生的时间。

```golang

span.LogFields(
  log.String("event", "message"),
  log.String("message.id", 1),
  log.String("message.type", "RECEIVED"),
)

```

#### SpanContext

SpanContext 携带着一些用于跨服务通信的数据

- 足够在系统中标识该 span 的信息，比如：span_id，trace_id
- Baggage Items
  - 键值对，但都只能是字符串格式
  - 不仅当前 Span 可见，他会随着 SpanContext 传递给后续所有的子 Span，需要谨慎使用，传递数据过多会有网络和 CPU 开销

#### References

Opentracing 定义了两种引关系，ChildOf 和 FollowFrom

- ChildOf
  - 父 span 的执行依赖子 span 的执行结果时，子 span 对父 span 的引用关系是 ChildOf。比如一次 RPC 调用，服务端的 span（子 span）与客户端发起请求时的 span（父 span）是 ChildOf 关系。
- FollowFrom
  - 父 span 的执行不依赖子 span 执行结果时，子 span 对父 span 的引用关系是 FollowFrom。

#### Trace

Trace 表示一次完整的链路追踪，Trace 由一个或多个 Span 组成，下图示例表示一个由 8 个 Span 组成的 Trace：

```bash

        [Span A]  ←←←(the root span)
            |
     +------+------+
     |             |
 [Span B]      [Span C] ←←←(Span C is a `ChildOf` Span A)
     |             |
 [Span D]      +---+-------+
               |           |
           [Span E]    [Span F] >>> [Span G] >>> [Span H]
                                       ↑
                                       ↑
                                       ↑
                         (Span G `FollowsFrom` Span F)


```

## Jaeger

Jaeger 是一款开源的端到端的分布式链路追踪系统,Jaeger 遵循 Opentracing 规范。

[Jaeger 官方文档](https://www.jaegertracing.io/docs/1.31/)

### 使用 Jaeger 对 Golang 项目做分布式跟踪

#### Jaeger 使用 docker-compose 部署

```yaml
version: '3'

services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - '14268:14268' # 服务上报 trace 端口
      - '16686:16686' # 用于暴露 Jaeger UI
```

服务启动后台可以通过 http://localhost:16686 打开 Jaeger UI

#### 代码

```golang

package main

import (
	"context"
	"errors"
	"fmt"
	"io"
	"time"

	"github.com/opentracing/opentracing-go"
	"github.com/opentracing/opentracing-go/log"
	"github.com/uber/jaeger-client-go"
	"github.com/uber/jaeger-client-go/config"
)

// 初始化 Jaeger ，将 Jaeger trace 设置为全局 tracer
func initJaeger(serviceName string, endpoint string) io.Closer {

	conf := config.Configuration{
		ServiceName: serviceName,
		Sampler: &config.SamplerConfig{
			Type:  jaeger.SamplerTypeConst,
			Param: 1, // 采样频率设为1
		},
		Reporter: &config.ReporterConfig{
			LogSpans:          true,
			CollectorEndpoint: endpoint, // jaeger 的 http 收集地址
		},
	}
	closer, err := conf.InitGlobalTracer(serviceName, config.Logger(jaeger.StdLogger))

	if err != nil {
		panic(fmt.Sprintf("init.jaeger.err:%+v", err))
	}

	return closer
}

func main() {

	closer := initJaeger("traceDemo", "http://101.35.174.236:14268/api/traces")

	defer closer.Close()

	// 获取 Jaeger tracer

	tracer := opentracing.GlobalTracer()

	// 创建 root span
	span := tracer.StartSpan("root")
	defer span.Finish()

	span.SetTag("type", "demo")
	span.LogFields(
		log.String("demo.log", "this is tracing demo"),
	)

	// 将 span 传递给 demoFun

	ctx := opentracing.ContextWithSpan(context.Background(), span)

	demoFun(ctx)

}

func demoFun(ctx context.Context) {
	span, ctx := opentracing.StartSpanFromContext(ctx, "demoFun")
	defer span.Finish()

	// 假设出错
	err := errors.New("do something erro")
	span.SetTag("error", true)
	span.LogFields(
		log.String("event", "error"),
		log.String("message", err.Error()),
	)

	// 将  ctx 传递给 demoFoo
	demoFoo(ctx)

	//  模拟耗时
	time.Sleep(time.Second * 1)
}

func demoFoo(ctx context.Context) {
	span, _ := opentracing.StartSpanFromContext(ctx, "demoFoo")
	defer span.Finish()

	//  模拟耗时
	time.Sleep(time.Second * 2)
}

```

#### Jaeger UI 查看链路信息

![Jaeger-list](https://graph.linganmin.cn/220218/342c15dd6449292e2591a2babdf744b1?x-oss-process=image/format,webp/quality,q_60)

![Jaeger-detail](https://graph.linganmin.cn/220218/6e2d387b0758477d4ad631c79aa30c03?x-oss-process=image/format,webp/quality,q_60)


## 备注

当前 Opentracing 的 Spec 没有提供直接获取 TraceID 的方法和标准，不过在2.0版本中即将加入 ToTraceID 和 ToSpanID 方案以简化 TraceID 的使用。

当前可以通过断言方式获取 TraceID

```golang

	if sc, ok := span.Context().(jaeger.SpanContext); ok {
		fmt.Println("traceId", sc.SpanID())
	}

```

## 参考

[Opentracing Go Tutorial](https://github.com/yurishkuro/opentracing-tutorial/tree/master/go)
[Opentracing 规范文档 v1.1](https://github.com/opentracing/specification/blob/master/specification.md)
[Opentracing 语义习惯](https://github.com/opentracing/specification/blob/master/semantic_conventions.md)
