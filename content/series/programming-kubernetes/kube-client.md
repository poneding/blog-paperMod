---
title: 构造 Kubernetes 客户端实例
date: 2022-10-01T23:39:15+08:00
draft: false
tags: 
  - Kubernetes
keywords:
  - Kubernetes
  - kubeclient
  - clientset
weight: 10
---

本节介绍 Golang 程序如何通过 `rest.Config` 实例获取各种类型的 Kubernetes 客户端实例。
通过客户端访问 Kubernetes 中的 API 资源实例。

## Clientset

1. 获取 `*kubernetes.Clientset`  
    推荐使用该客户端实例去操作 K8s API 资源。

```go
package main

import (
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
)

func Clientset(config *rest.Config) *kubernetes.Clientset {
	client, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}
	return client
}
```

2. 获取 `*rest.RESTClient`  
    可以通过该客户端实例获取内置的以及自定义的 K8s API 资源。

```go
package main

import "k8s.io/client-go/rest"

func RESTClient(config *rest.Config) *rest.RESTClient {
	client, err := rest.RESTClientFor(config)
	if err != nil {
		panic(err)
	}
	return client
}
```

## DiscoveryClient

`DiscoveryClient` 动态客户端，通过动态指定 GVR 来操作任意的 Kubernetes 资源（内置资源 + CR）

- 使用嵌套的 `map[string]interface{}` 结构存储资源数据；
- 可以利用反射机制序列化成为特定资源实体；
- 灵活性更高，但无法做强数据类型检查和验证。

使用 DiscoveryClient 获取到的资源对象为 [`runtime.Object`](runtime-object.md)，可以通过 [``]

### 获取 DiscoveryClient

可以通过该客户端实例获取内置的以及自定义的 K8s API 资源。

```go
package main

import (
	"k8s.io/client-go/discovery"
	"k8s.io/client-go/rest"
)

func DiscoveryClient(config *rest.Config) *discovery.DiscoveryClient {
	client, err := discovery.NewDiscoveryClientForConfig(config)
	if err != nil {
		panic(err)
	}
	return client
}
```

### 使用 DiscoveryClient

```go

```

## DynamicClient

获取 `dynamic.Interface`

```go
package client

import (
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/rest"
)

func DynamicClient(config *rest.Config) dynamic.Interface {
	client, err := dynamic.NewForConfig(config)
	if err != nil {
		panic(err)
	}
	return client
}
```



4. 获取 `runtimecli.Client`  
    可以通过该客户端实例获取内置的以及自定义的 K8s API 资源。
      但是如果该客户端实例需要操作自定义 K8s API 资源，New 函数传入的参数 runtimecli.Options 中 Scheme 对象需要调整。

```go
package main

import (
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/client-go/kubernetes/scheme"
	"k8s.io/client-go/rest"
	runtimecli "sigs.k8s.io/controller-runtime/pkg/client"
	// 假设定义了一个 foo CRD，里面包含 register.go 中定义了 AddToScheme 实例
	foosv1 "pkg/apis/foos/v1"
)

func RuntimeClient(config *rest.Config) runtimecli.Client {
	client, err := runtimecli.New(config, runtimecli.Options{
		Scheme: scheme.Scheme,
	})
	if err != nil {
		panic(err)
	}
	return client
}

func RuntimeClientForCRD(config *rest.Config) runtimecli.Client {
	crScheme := runtime.NewScheme()
	foosv1.AddToScheme(scheme.Scheme)
	client, err := runtimecli.New(config, runtimecli.Options{
		Scheme: crScheme,
	})
	if err != nil {
		panic(err)
	}
	return client
}
```

5. 获取 `*http.Client`  
    最原生的客户端，使用该客户端实例操作 K8s API 资源就纯靠自己手工封装了，不太推荐。

```go
package main

import (
	"net/http"

	"k8s.io/client-go/rest"
)

func HTTPClient(config *rest.Config) *http.Client {
	client, err := rest.HTTPClientFor(config)
	if err != nil {
		panic(err)
	}
	return client
}
```

## 总结

http.Client => rest.RESTClient => discovery.DiscoveryClient => kubernetes.Clientset  
客户端的定制化越高，使用越高效。但同时对自定义 API 的资源操作的支持就越低。




