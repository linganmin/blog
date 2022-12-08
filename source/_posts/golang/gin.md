---
title: 浅读 Go 优秀开源项目源码— Gin框架
tags:
  - Gin
categories: Golang
keywords: 'blog,golang,gin'
description: 浅读 Gin 框架源码，学习优秀项目的设计
cover: >-
  https://graph.linganmin.cn/220306/e6d48282ac21a3cc02dd7b6a2b571e81?x-oss-process=image/format,webp/quality,q_60
top_img: >-
  https://graph.linganmin.cn/220306/e6d48282ac21a3cc02dd7b6a2b571e81?x-oss-process=image/format,webp/quality,q_60
abbrlink: gin
date: 2022-03-06 22:30:20
---

> Gin 是一个用 Go (Golang) 编写的 HTTP web 框架。它是一个类似于 martini 但拥有更好性能的 API 框架, 由于 httprouter，速度提高了近 40 倍。

Gin 是目前 Go 里面使用最广泛的框架之一了，弄清楚 Gin 框架的原理，有助于我们更好的使用 Gin。通过浅读 Gin 框架源码，大致总结一下它的一些核心实现。

## 从一个 Demo 入手

下面这段代码便是使用 Gin 编写一个简单的只有一个接口的 Go server 端程序，详细使用方式参考官方文档：[Gin Web Frameword](https://gin-gonic.com/zh-cn/docs/introduction/) 

```GO
package main

import (
    "fmt"
    "log"
    "time"

    "github.com/gin-gonic/gin"
)


func main() {
    r := gin.Default()

    // 注册路由
    r.GET("/ping", func(c *gin.Context) {
        // 处理返回
        c.JSON(200, gin.H{ 
            "message": "pong",
        })
    })

    // 启动 server
    log.Fatal(r.Run(":8989"))
}


```

## 路由

Gin框架使用的是定制版本的[httprouter](https://github.com/julienschmidt/httprouter)，其路由的原理是大量使用公共前缀的树结构，它基本上是一个紧凑的前缀树（Trie tree）或者只是基数树(Radix Tree）

> 基数树（Radix Tree）又称为PAT位树（Patricia Trie or crit bit tree），是一种更节省空间的前缀树（Trie Tree）。对于基数树的每个节点，如果该节点是唯一的子树的话，就和父节点合并。下图就是一个基数树的实例：

![基数树](https://graph.linganmin.cn/220306/106bd33d205c255aefab6ea04bbdce2e?x-oss-process=image/format,webp/quality,q_60)


基数树可以被认为是简洁版的前缀树，在 Gin 框架中，我们注册路由的过程就是构造前缀树的过程，在路由匹配的过程就是查找前缀树的过程。具有公共前缀树的节点会共享一个公共父节点，假设我们注册以下路由信息：

```Go
r := gin.Default()

r.GET("/", func1)
r.GET("/search/", func2)
r.GET("/support/", func3)
r.GET("/blog/", func4)
r.GET("/blog/:post/", func5)
r.GET("/about-us/", func6)
r.GET("/about-us/team/", func7)
r.GET("/contact/", func8)

```

我们实际就构造了一个`GET`类型请求对应的路由树如下图，从根节点遍历到叶子节点我们就能得到完整的路由表。

```bash
Priority   Path             Handle
9          \                *<1>
3          ├s               nil
2          |├earch\         *<2>
1          |└upport\        *<3>
2          ├blog\           *<4>
1          |    └:post      nil
1          |         └\     *<5>
2          ├about-us\       *<6>
1          |        └team\  *<7>
1          └contact\        *<8>

```

通过阅读源码发现，路由实现为每一种请求方法都单独管理了一个前缀树，这样处理的好处是可以节省更多空间，而且在查找过程中逻辑更加简单。

```Go
  func (engine *Engine) addRoute(method, path string, handlers HandlersChain) {
    assert1(path[0] == '/', "path must begin with '/'")
    assert1(method != "", "HTTP method can not be empty")
    assert1(len(handlers) > 0, "there must be at least one handler")

    debugPrintRoute(method, path, handlers)

    // linganmin
    // 为每种请求方式生成单独的前缀树
    root := engine.trees.get(method)
    if root == nil {
      root = new(node)
      root.fullPath = "/"
      engine.trees = append(engine.trees, methodTree{method: method, root: root})
    }
    root.addRoute(path, handlers)

    // Update maxParams
    if paramsCount := countParams(path); paramsCount > engine.maxParams {
      engine.maxParams = paramsCount
    }

    if sectionsCount := countSections(path); sectionsCount > engine.maxSections {
      engine.maxSections = sectionsCount
    }
  }
```

为了获得更好的可伸缩性，每个树级别上的子节点都按照`优先级`排序，其中层级越深的路由优先级越高。

```Go
// addRoute adds a node with the given handle to the path.
// Not concurrency-safe!
func (n *node) addRoute(path string, handlers HandlersChain) {
	fullPath := path
	n.priority++

	// Empty tree
	if len(n.path) == 0 && len(n.children) == 0 {
		n.insertChild(path, fullPath, handlers)
		n.nType = root
		return
	}

// .....

```

更多细节参见源码：https://github.com/gin-gonic/gin/blob/master/tree.go

## 中间件

```Go
// Use adds middleware to the group, see example code in GitHub.
func (group *RouterGroup) Use(middleware ...HandlerFunc) IRoutes {
	group.Handlers = append(group.Handlers, middleware...)
	return group.returnObj()
}

func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
	absolutePath := group.calculateAbsolutePath(relativePath)
	handlers = group.combineHandlers(handlers) // 将处理请求的函数与中间件结合
	group.engine.addRoute(httpMethod, absolutePath, handlers)
	return group.returnObj()
}

func (group *RouterGroup) combineHandlers(handlers HandlersChain) HandlersChain {
	finalSize := len(group.Handlers) + len(handlers)
	if finalSize >= int(abortIndex) {
		panic("too many handlers")
	}
	mergedHandlers := make(HandlersChain, finalSize)
	copy(mergedHandlers, group.Handlers)
	copy(mergedHandlers[len(group.Handlers):], handlers)
	return mergedHandlers
}
```


中间件和处理请求的函数合并成一个 slice（handlersChain）,遍历该链条，依次执行该路由的每一个函数，可以在中间件中 使用 c.Next() 实现嵌套调用。

![mid](https://graph.linganmin.cn/220306/2424ea5907a7c727af04748b17ce736f?x-oss-process=image/format,webp/quality,q_60)

Gin.Default() 会使用默认中间件`debugPrintWARNINGDefault()`，Gin.New() 不会使用

## Context

Gin 的 context 实现了 context.Context Interface，除此之外又对 request 和 response 进行了封装。
