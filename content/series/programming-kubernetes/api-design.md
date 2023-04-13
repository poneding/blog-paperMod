---
title: "Kubernetes API 设计"
date: 2022-10-02T01:28:03+08:00
draft: false
tags: 
  - Kubernetes
keywords:
  - Kubernetes
  - GVK
  - GVR
weight: 20
---



## 术语

**Group**

API 资源置于某个分组下，组作为相关功能的集合。一个组包含一个或多个版本。

**Version**

API 资源的版本，API 资源版本是会不断迭代的。

**Kind**

API 资源的的类型，用于存储 API 资源的描述信息或状态等。同一个 Kind 的 API 资源可以有多个版本，随着版本的不断迭代，Kind 代表的资源的会有字段内容的更改。

**GVK**

Group/Version/Kind，例如 Deployment：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  - name: deploy-1
...
```

>   上面的代码示例描述了一个 API 资源对象，这个资源对象：
>
>   -   Group 是 apps
>   -   Version 是 v1
>   -   Kind 是 Deployment。

**Resource**

代表 API 资源，与 GVK 一对一的关系。

**GVR**

可以将 GVK 比作是一个类，GVR 就是这个 GVK 类的实例。



当我们以 REST 的方式向发起 API 资源的请求是，请求 URL 格式一般类似这样：`/api/apps/v1/deployments`，里面就包含了三个上面提到的术语概念：

-   /apps：请求资源所在的组（Group）
-   /v1：请求资源的版本（Version）
-   /deployments：请求的资源的名称（Resource）
