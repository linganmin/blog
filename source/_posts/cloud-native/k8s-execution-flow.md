---
title: 「Kubernetes 拾遗」之 输入 kubectl run/create/apply 之后到底发生了什么
tags:
  - Kubernetes
categories: 云原生
keywords: 'blog,golang,php,k8s,kubernetes,probe'
description: 输入 kubectl run/create/apply 之后到底发生了什么？
cover: >-
  https://graph.linganmin.cn/200911/4043741d339ab2812a46836518975aa7?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/200911/4043741d339ab2812a46836518975aa7?x-oss-process=image/format,webp/quality,q_40
abbrlink: f902dddb
date: 2020-09-11 22:40:34
---

周四的时候参加我司容器组大佬的课，问到一个问题：

> `kubectl run nginx --image=nginx --replicals=3` 当这行命令在终端敲下回车键之后你会看到很快有三个`nginx pod`创建在集群的`worker`节点上，但当敲下回车键的时候`kubernetes`背后都做了什么呢？

自认为对 k8s 还算熟悉的在下一下子懵了，好像知道但又说不清楚。那今天就系统性的查资料+实践总结一下吧。

---

## kubectl

### 客户端验证

执行客户端验证，确保非法请求（创建的资源不存在、镜像格式不正确等）的快速失败，减少对`kuber-apiserver`的不必要请求提升系统性能。

### 生成运行时对象

使用`generator`根据需要创建`资源类型`来构建`运行时对象(runtime object)`

> 如果显示的设置了`--generator`参数，kubectl 将使用指定的生成器。如果没有指定生成器，kubectl 会根据其他参数自动选择要使用的生成器。具体优先级如下：

```bash
--schedule=<schedule>  CronJob
--restart=Always       Deployment
--restart=OnFailure    Job
--restart=Never        Pod
```

例如：当 kubectl 判断出要创建一个 Deployment 后，它将使用`DeploymentV1Beta1`的`generator`和我们提供的参数生成一个`Deployment 类型的 Runtime Object`

### 版本协商

#### API 版本

> 为了更容易地消除字段或者重新组织资源结构，Kubernetes 支持多个 API 版本，每个版本都放在不同的 API 路径下。如：`/api/v1`、`/apis/extensions/v1beta1`，不同的 API 版本表明不同的稳定性和支持级别。具体更详细的请参考[Kubernetes API 概述](https://k8smeetup.github.io/docs/reference/api-overview/)

#### API 组

> API 组旨在对相似的资源进行分类，以便后期更容易扩展。API 的组别在 REST 路径或者序列化对象的`apiVersion`字段中指定。比如我们常用到的`Deployment`的`apiVersion`为`apiVersion: apps/v1`,`apps`就是 Deployment 的 API 的组名，`v1`则是 Deployment 的版本号

kubectl 在生成运行时对象后，开始为它找到合适的`API组和API版本`，然后组装一个知道该资源各种 REST 语义的版本化客户端。这个阶段被称为`版本协商`，为了提升性能，kubectl 会将`OpenAPI scheme`缓存到`~/.kube/cache`目录。

### 客户端身份认证

为了能够成功的发送请求，kubectl 需要先进行身份认证。用户凭证一般保存在`kubeconfig`文件中，kubectl 通过以下顺序找寻配置文件：

- 执行 kubectl 命令时，如果携带了`--kubeconfig`参数，则会使用该参数对应值的路径文件作为`kubeconfig`
- 如果没有携带`--kubeconfig`参数，但是系统的环境变量设置了`$KUBECONFIG`，则使用该环境变量的值的路径文件作为`kubeconfig`
- 如果没有携带`--kubeconfig`参数系统内也没有`$KUBECONFIG`环境变量，kubectl 则会使用默认的配置文件，默认配置文件路径`$HOME/.kube/config`

解析完配置文件`kubeconfig`后，kubectl 会确定当前要使用的上下文，当前配置指向的集群以及当前配置用户关联的认证信息。如果执行 kubectl 时提供了额外的参数，例如：`--username` 则会优先使用这些参数覆盖`kubeconfig`中的对应值。当获取到这些数据之后，kubectl 就会把这些信息填充到将要发送的 HTTP 请求头中：

- x509 证书使用`tls.RLSConfig`发送，其中包括 CA 证书
- `bearer tokens` 放在 HTTP 请求头`Authorization`中
- 用户名和密码通过 HTTP 基本认证发送
- `OpenID` 认证是由用户事先处理的，产生一个类似`bearer tokens`
  的 token，放在请求头里发送

## kube-apiserver

当 kubectl 发送请求成功后，这时候就该`kube-apiserver`登场了。

`kube-apiserver`是客户端和系统组件用来保存和检索集群状态的主要接口。

### 身份认证(Authentication)

为了执行请求带来的相应功能，`kube-apiserver`需要验证请求者是合法的，这个过程被称为`认证`。

**TODO**
为了验证请求，当`kube-apiserver`第一次启动时，它会查看用户提供的所有`cli参数`并组合成一个合适的令牌列表。
**TODO 这里没太懂，需要后面折腾集群的时候理解下**

当每个请求到`kube-apiserver`时，它都会通过令牌链进行认证，直到某一个认证成功。
**注意：这里说的是某个认证成功，并不是所有，详见下面的代码**

```golang
func (authHandler *unionAuthRequestHandler) AuthenticateRequest(req *http.Request) (user.Info, bool, error) {
	var errlist []error
	for _, currAuthRequestHandler := range authHandler.Handlers {
		info, ok, err := currAuthRequestHandler.AuthenticateRequest(req)
		if err != nil {
			if authHandler.FailOnError {
				return info, ok, err
			}
			errlist = append(errlist, err)
			continue
		}

		if ok {
			return info, ok, err
		}
	}

	return nil, false, utilerrors.NewAggregate(errlist)
}

```

如果认证失败，则将返回相应的错误信息。如果验证成功，则将请求中的`Authorization`请求头删除，并将用户信息添加到上下文，给后续的授权和准入控制器提供访问前的建立用户身份的能力。

### 用户鉴权（Authorization）

身份认证结束仅仅代表我们的请求是合法的，并不代表当前`kubeconfig`的所属用户具有执行当前操作的权限。

{% note info %}
`身份认证(identity)`和`权限校验(permission)`不是一回事
{% endnote %}

`kube-apiserver`处理授权的方式和处理身份认证方式相似：基于`cli参数`输入，组合一系列授权者，这些授权者将针对每个传入的请求进行授权，如果所有授权者都拒绝了该请求，则该请求返回一个`Forbidden`响应并通知执行后续操作，如果某个授权者被批准了该请求，则请求继续。

```golang

// WithAuthorizationCheck passes all authorized requests on to handler, and returns a forbidden error otherwise.
func WithAuthorization(handler http.Handler, requestContextMapper request.RequestContextMapper, a authorizer.Authorizer, s runtime.NegotiatedSerializer) http.Handler {
	if a == nil {
		glog.Warningf("Authorization is disabled")
		return handler
	}
	return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
		ctx, ok := requestContextMapper.Get(req)
		if !ok {
			responsewriters.InternalError(w, req, errors.New("no context found for request"))
			return
		}

		attributes, err := GetAuthorizerAttributes(ctx)
		if err != nil {
			responsewriters.InternalError(w, req, err)
			return
		}

		// 鉴权
		authorized, reason, err := a.Authorize(attributes)
		if authorized {
			handler.ServeHTTP(w, req)
			return
		}
		if err != nil {
			responsewriters.InternalError(w, req, err)
			return
		}

		glog.V(4).Infof("Forbidden: %#v, Reason: %q", req.RequestURI, reason)
		responsewriters.Forbidden(ctx, attributes, w, req, reason, s)
	})
}

func GetAuthorizerAttributes(ctx request.Context) (authorizer.Attributes, error) {
	attribs := authorizer.AttributesRecord{}

	user, ok := request.UserFrom(ctx)
	if ok {
		attribs.User = user
	}

	requestInfo, found := request.RequestInfoFrom(ctx)
	if !found {
		return nil, errors.New("no RequestInfo found in the context")
	}

	// Start with common attributes that apply to resource and non-resource requests
	attribs.ResourceRequest = requestInfo.IsResourceRequest
	attribs.Path = requestInfo.Path
	attribs.Verb = requestInfo.Verb

	attribs.APIGroup = requestInfo.APIGroup
	attribs.APIVersion = requestInfo.APIVersion
	attribs.Resource = requestInfo.Resource
	attribs.Subresource = requestInfo.Subresource
	attribs.Namespace = requestInfo.Namespace
	attribs.Name = requestInfo.Name

	return &attribs, nil
}
```

kube-apiserver 目前支持的授权方式：

- webhook
- ABAC
- RBAC
- Node

### 准入控制(Admission Controller)

当完成了`身份认证`和`授权`之后，便进入了`Admission Controller`。

> 身份认证是建立用户身份，授权是校验用户是否有权限，而`Admission Controller`则是 Kubernetes 其他组件对该请求的校验，`Admission Controller`会校验当前请求是否满足个组件的`准入控制规则`，官方默认有十几个规则，且该规则是可以自定义的。这也是当前请求所携带的资源对象保存到`ETCD`之前的最后一个堡垒。

`Admission Controller`的规则是封装的一些列额外的检查，以确保当前请求所执行的操作不会对集群产生意外和负面结果。

和身份认证、授权不同的是，身份认证和授权只关心请求的用户和操作，准入控制还处理请求的内容，且该操作只对创建、更新、删除或连接等有效，对读操作无效。

和身份认证、授权另一个不同的是，身份认证和授权只要满足一种方式即可通过，而`Admission Controller`的判断是`且/&&`逻辑，即如果某个准入控制器校验不通过，则整个`Admission Controller`校验中断，请求失败。

{% note info %}
`Admission Controller`的设计重点在于提高可扩展性，可以根据不同需求自定义准入规则。
{% endnote %}
Kubernetes 自带的准入控制器见代码：[源码](https://github.com/kubernetes/kubernetes/tree/master/plugin/pkg/admission)

## etcd

> TODO kube-apiserver 启动&工作流程

到此刻，Kubernetes 就完成了对请求的所有审查，并允许进入下一步。在下一步中， kuber-apiserver 会反序列化 HTTP 请求，构造`运行时对象(runtime object)`(类似 kubectl generator 的逆过程)，并将其通过`storage provider`持久化到`etcd`中，默认情况下存储的键的格式为`<namespace>/<name>`例如：`default/nginx`，也可以自定义。

在资源创建过程中如果出现任何错误都会被捕获，最后`service provider`会执行`get`操作来确认该资源是否被创建成功。

然后构造 HTTP 响应并返回给客户端。

{% note info %}
注：在 Kubernetes v1.14 之前，这往后还有 Initializer 的步骤，该步骤在 v1.14 被 [webhook admission 取代](https://github.com/kubernetes/kubernetes/issues/67113)。
{% endnote %}

## 控制循环（Control loops）

截止到目前，我们的 Deployment 已经存储在 etcd 中，且所有初始化逻辑已经完成。在接下来的阶段将涉及 Deployment 所依赖的资源拓扑结构。

在 Kubernetes 中，Deployment 是 ReplicaSet 的集合，而 ReplicaSet 是 Pod 的集合。那么 Kubernetes 是如何从一个 HTTP 请求中按照层级结构依次穿件这些资源呢？其实这些工作都是由 Kubernetes 内置的 `Controller` 完成的。

在 Kubernetes 整个系统中使用了大量的 Controller ，Controller 是一个用于将系统状态从`当前状态`修正到`期望状态`的异步脚本。所有 Controller 都通过`kube-controller-manager`组件并行运行，每种 Controller 都负责一种具体的控制流程。

{% note success %}

(notes: 不要忘记 Kubernetes 是`声明式`)

所谓`声明式`,指的就是我只需要提交一个定义好的 API 对象来`声明`我所期望的状态是什么样子。

Kubernetes 基于对 API 对象的增、删、改、查，在完全无需外界干预的情况下，完成对`实际状态`和`期望状态`的调谐（Reconcile）过程便是控制循环做的事。

声明式 API，是 Kubernetes 项目编排能力的核心。
{% endnote %}

### Deployments Controller

在上一步，假如将反序列化后的 HTTP 请求存储到 etcd 的是 Deployment 资源，此时 kube-apiserver 可以使其可见，`Deployment Controller`会检测到它。

> Deployment Controller 的哦工作就是负责监听 Deployment 记录的变化





### ReplicaSets Controller

### informers

### scheduler

## kubelet

### pod 同步

### CRI 和 pause 容器

### CNI 和 pod 网络

### 跨主机容器网络

### 容器启动

## 总结

---

太困了，睡觉去了。。。

> TODO
