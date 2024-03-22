---
title: 使用 Patch 策略更新 Kubernetes 资源对象
description: ""
date: 2023-04-26T08:10:14.335Z
lastmod: 2023-04-26T08:10:14.336Z
slug: how-to-update-kubernetes-resource-object-with-patch-strategy
tags:
  - Kubernetes
weight: 0
cover:
  image: ""
---

## 什么是 Patch 策略

Kubernetes API Server 提供了三种 Patch 策略，分别是：

- `strategic-merge`：使用 [strategic-merge patch](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/#use-a-strategic-merge-patch-to-update-a-deployment) 策略，这是默认的 Patch 策略。

## 参考

- [使用 kubectl patch 更新 API 对象 | Kubernetes](https://kubernetes.io/zh-cn/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/)
- [JSON Patch](https://jsonpatch.com/)
- [mattbaird/jsonpatch](https://github.com/mattbaird/jsonpatch)
