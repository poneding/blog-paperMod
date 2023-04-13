---
title: "构造 rest.Config 实例"
date: 2022-10-01T23:28:02+08:00
draft: false
tags: 
  - Kubernetes
keywords:
  - Kubernetes
  - kubeconfig
weight: 1
---

本节介绍几种构造 `rest.Config` 实例的场景或者方法。

`rest.Config` 可以帮助我们构建各种类型的 Kubernetes 客户端实例，从而访问 Kubernetes APIServer。

## 通过 kubeconfig 文件构造

程序通过读取 kubeconfig 文件来构造一个 `rest.Config` 对象。

```go
package main

import (
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
)

func KubeConfig() *rest.Config {
	config, err := clientcmd.BuildConfigFromFlags("", clientcmd.RecommendedHomeFile)
	if err != nil {
		panic(err)
	}
	return config
}
```

## 通过 Secret 资源构造

通过将程序部署在 Kubernetes 集群中，使用 Pod 所配置的 ServiceAccount（默认：default）账号构造 `rest.Config` 对象。

>   运行的 Pod 内都会存储一个
>
>   每个 ServiceAccount 都有一个对应的 Secret，这个 Secret 包含了对集群的操作权限。

```go
package main

import (
	"k8s.io/client-go/rest"
)

func KubeConfig() *rest.Config {
	config, err := rest.InClusterConfig()
	if err != nil {
		panic(err)
	}
	return config
}
```

## 通过 controller-runtime 快速获取

使用 `sigs.k8s.io/controller-runtime` 包快速获取 `rest.Config` 对象。  
优先级：

- 从 `--kubeconfig` 指定的文件获取
- 从 `KUBECONFIG` 环境变量配置的文件获取
- 运行在集群中，以 In-cluster 方式获取
- 从 `$HOME/.kube/config` 文件获取

> 优点：灵活，对配置友好，推荐使用该种方式来获取 rest.Config 对象。

```go
package main

import (
	"k8s.io/client-go/rest"
	ctrl "sigs.k8s.io/controller-runtime"
)

func KubeConfig() *rest.Config {
	config, err := ctrl.GetConfig()
	if err != nil {
		panic(err)
	}
	return config
}
```

## 通过文本构造

从 kubeconfig 文本获取 `rest.Config` 对象。
例如可以用于多集群管理平台，都过租户上传的 kubeconfig 来实例化 `rest.Config` 对象。

```go
package main

import (
	"io/ioutil"

	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
)

var fakeConfig = `---
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: cert-xxx
    server: https://127.0.0.1:6443
  name: cluster-t2ktl
contexts:
- context:
    cluster: cluster-t2ktl
    namespace: default
    user: user-7k89b
  name: context-ctc98
current-context: context-ctc98
kind: Config
preferences: {}
users:
- name: user-7k89b
  user:
    token: token-xxx
`

func KubeConfig() *rest.Config {
	realConfig, _ := ioutil.ReadFile(clientcmd.RecommendedHomeFile)
	fakeConfig = string(realConfig)

	client, err := clientcmd.RESTConfigFromKubeConfig([]byte(fakeConfig))
	if err != nil {
		panic(err)
	}
	return client
}
```

