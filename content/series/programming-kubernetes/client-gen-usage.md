---
title: "使用 client-gen 生成 clientset 代码"
date: 2022-10-08T17:37:07+08:00
draft: false
tags: 
  - Kubernetes
keywords:
  - Kubernetes
  - client-gen
weight: 40
---

*本页是[这篇Kubernetes 文档](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-api-machinery/generating-clientset.md#using-client-gen)中一些内容摘要。*

大致分为 3 个步骤：

1. 为 API 类型结构做 tag 标签
    在 API 类型例如 `pkg/apis/${Group}/${Version}/types.go` 中的 `Pod` 结构体上打标签，支持的标签：

- `// +genclient`：生成客户端函数（包括 Create, Update, Delete, DeleteCollection, Get, List, Update, Patch, Watch，如果 API 类型结构中包括 `.Status` 字段，还会额外生成 `UpdateStatus` 函数）；
- `// +genclient:nonNamespaced`：指定 API 类型是集群级别而不是命名空间级别的，生成的客户端函数都没有命名空间；
- `// +genclient:onlyVerbs=create,get`：只生成 Create, Get 客户端函数；
- `// +genclient:skipVerbs=watch`：生成除了 Watch 之外的所有其他客户端函数；
- `// +genclient:noStatus`：即使 API 类型结构包含 `.Status` 字段，也不生成 `UpdateStatus` 客户端函数；
    有些情况下，可能你想要额外生成非标准的客户端函数，例如子资源函数，那么你需要使用下列这些 tag 标签：
- `// +genclient:method=Scale,verb=update,subresource=scale,input=k8s.io/api/extensions/v1beta1.Scale,result=k8s.io/api/extensions/v1beta1.Scale`：该例中使用标签，将会自动生成 `Scale(string, *v1beta.Scale) *v1beta.Scale` 客户端函数，里面配置了子资源函数的输入和输出参数。
    另外，以下的 tag 标签也影响着客户端代码的生成：
- `// +groupName=policy.authorization.k8s.io`：使用该 group 名称生成到客户端代码中（默认使用 package 名）；
- `// +groupGoName=AuthorizationPolicy`：驼峰 Golang 标识符避免带有非唯一前缀例如 `policy.authorization.k8s.io` 和 `policy.k8s.io` 的组名冲突。这将可能导致两个 Policy() 函数生成到 clientset 代码中。

2. 如果你是参与开发 `k8s.io/kubernetes` 项目，你只需要执行 `./hack/update-group.sh` 脚本即可生成或更新代码；
    如果你开发自己的项目，你需要使用 `client-gen` 命令以及其命令参数：

```bash
client-gen --input="pkg/apis/${group1}/${version1},pkg/apis/${group2}/${version2}" --clientset-name="my_clientset"
```

> 使用 `client-gen -h` 查看更多命令使用姿势。

3. 除了 client-gen 自动生成的客户端代码外，你可以手动添加额外的代码，[参考这里](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/client-go/kubernetes/typed/core/v1/pod_expansion.go)。

