---
title: 如何写一个 Admission Webhook
description: ""
date: 2023-04-26T07:40:56.946Z
lastmod: 2023-04-26T07:40:56.946Z
slug: how-to-write-an-admission-webhook
tags:
  - Kubernetes
  - Admission Webhook
weight: 0
cover:
  image: "https://fs.poneding.com/images/202305041519080.png"
---

## 准入控制器（Admission Controllers）

Admission Controllers 是 Kubernets API Server 中的一种插件机制，它可以在 API 资源对象创建、更新、删除的请求通过认证和鉴权之后，持久化到 etcd 之前，对资源对象进行额外的变更（Mutating）或者校验（Validating），准入控制器不会拦截对资源的读请求（Get，Watch，List）。

准入控制分为两个阶段：第一阶段，运行变更准入控制器；第二阶段，运行验证准入控制器。变更准入控制器可以变更资源，也可以对请求进行验证；但是验证准入控制器只能对请求进行验证，不能变更资源。

## 准入 Webhook（Admission Webhook）

![202305041453105.png](https://fs.poneding.com/images/202305041453105.png)

Admission Webhook 是准入控制器的一种实现方式，它是一个 HTTP Server，它监听一个端口，等待 API Server 的准入请求，请求体是一个 `AdmissionReview` 对象，它包含了 `AdmissionRequest` 和 `AdmissionResponse`，`AdmissionRequest` 里面包含了请求的资源对象信息，`AdmissionResponse` 里面包含了准入控制器对请求的处理结果。

![202305041439868.png](https://fs.poneding.com/images/202305041439868.png)

上图展示了 APIServer 在接收到请求之后，所有的处理流程。我们集中关注 Admission 部分，在这部分里面，首先会调用 Admission Mutating Webhook，然后调用 Admission Validating Webhook。

## 写一个 Admission 控制器

我们以 MutatingAdmissionWebhook 为例，来写一个 Admission 控制器。  
[查看源码](https://github.com/poneding/programming-kubernetes/tree/master/examples)

Admission Webhook Server 是一个 HTTP Server，它需要监听一个端口，等待 API Server 发送 AdmissionRequest 请求，然后对 AdmissionRequest 进行处理，最后返回 AdmissionResponse。

先来看一下 AdmissionReview 的结构：

```go
type AdmissionReview struct {
	metav1.TypeMeta `json:",inline"`
	Request  *AdmissionRequest `json:"request,omitempty" protobuf:"bytes,1,opt,name=request"`
	Response *AdmissionResponse `json:"response,omitempty" protobuf:"bytes,2,opt,name=response"`
}
```

`AdmissionReview` 里面包含了 `AdmissionRequest` 和 `AdmissionResponse`，我们先来看一下 `AdmissionRequest` 的结构：

```go
type AdmissionRequest struct {
	UID types.UID `json:"uid" protobuf:"bytes,1,opt,name=uid"`
	Kind metav1.GroupVersionKind `json:"kind" protobuf:"bytes,2,opt,name=kind"`
	Resource metav1.GroupVersionResource `json:"resource" protobuf:"bytes,3,opt,name=resource"`
	SubResource string `json:"subResource,omitempty" protobuf:"bytes,4,opt,name=subResource"`

	RequestKind *metav1.GroupVersionKind `json:"requestKind,omitempty" protobuf:"bytes,13,opt,name=requestKind"`
	RequestResource *metav1.GroupVersionResource `json:"requestResource,omitempty" protobuf:"bytes,14,opt,name=requestResource"`
	RequestSubResource string `json:"requestSubResource,omitempty" protobuf:"bytes,15,opt,name=requestSubResource"`

	Name string `json:"name,omitempty" protobuf:"bytes,5,opt,name=name"`
	Namespace string `json:"namespace,omitempty" protobuf:"bytes,6,opt,name=namespace"`
	Operation Operation `json:"operation" protobuf:"bytes,7,opt,name=operation"`
	UserInfo authenticationv1.UserInfo `json:"userInfo" protobuf:"bytes,8,opt,name=userInfo"`
	Object runtime.RawExtension `json:"object,omitempty" protobuf:"bytes,9,opt,name=object"`
	OldObject runtime.RawExtension `json:"oldObject,omitempty" protobuf:"bytes,10,opt,name=oldObject"`
	DryRun *bool `json:"dryRun,omitempty" protobuf:"varint,11,opt,name=dryRun"`
	Options runtime.RawExtension `json:"options,omitempty" protobuf:"bytes,12,opt,name=options"`
}
```

### 场景

某项目团队，为了提高镜像访问速度，决定将镜像仓库从 Docker Hub 迁移到阿里云镜像仓库。但是，由于项目中有很多 Pod 使用了 Docker Hub 的镜像，如果手动修改，工作量太大，所以决定写一个 Admission 控制器，将 Pod 中的镜像仓库地址修改为阿里云镜像仓库。

### 1. 编写 Admission Webhook

首先，我们需要引入需要的包

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"os"
	"strings"

	v1 "k8s.io/api/admission/v1"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/klog/v2"
)
```

由于 Admission Webhook Server 需要使用 HTTPS，所以我们需要事先准备好证书和私钥，这里我们使用了 [cert-manager](https://cert-manager.io/) 生成的证书和私钥。

安装 cert-manager，参考官方文档：[https://cert-manager.io/docs/installation/](https://cert-manager.io/docs/installation/)

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.yaml
```

创建证书和私钥，以下是示例：
*webhook-cert.yaml*

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-webhook-cert
spec:
  secretName: example-admission-webhook-cert-secret
  duration: 2160h # 90 days
  renewBefore: 360h # 15 days
  commonName: example-admission-webhook.default.svc
  dnsNames:
    - default-admission-webhook.default.svc
  issuerRef:
    name: default-admission-webhook-issuer
    kind: ClusterIssuer
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: default-admission-webhook-issuer
spec:
  selfSigned: {}
```

执行以下命令，创建证书和 Issuer：

```bash
kubectl apply -f webhook-cert.yaml
```

在 Admission 控制器中，我们需要使用到证书和私钥，所以我们需要将证书和私钥保存到 Secret 中，以下是示例：

```go
// AdmissionResponse contains the fields extracted from an AdmissionReview response
type AdmissionResponse struct {
	AuditAnnotations map[string]string
	Allowed          bool
	Patch            []byte
	PatchType        admissionv1.PatchType
	Result           *metav1.Status
	Warnings         []string
}
```

## 注册 Admission 控制器

将 Admission 控制器注册到 API Server，需要创建一个 MutatingWebhookConfiguration/ValidatingWebhookConfiguration 对象，然后将该对象提交到 API Server。以下是示例：

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: example-admission-webhook
  annotations:
    cert-manager.io/inject-ca-from: default/example-admission-webhook-cert
webhooks:
  - admissionReviewVersions:
      - v1
    clientConfig:
      caBundle: ""
      service:
        name: example-admission-webhook
        namespace: default
        port: 443
        path: /mutating
    failurePolicy: Ignore
    matchPolicy: Exact
    name: example-admission-webhook.default.svc
    rules:
      - apiGroups:
          - ""
        apiVersions:
          - v1
        operations:
          - CREATE
          - UPDATE
        resources:
          - pods
        scope: "*"
    objectSelector:
      matchExpressions:
        - key: example-admission-webhook-test
          operator: Exists
    sideEffects: None
    timeoutSeconds: 3
```

> ⚠️ 注意：  
> 1、`cert-manager.io/inject-ca-from` 注解指定了证书的 Secret，这里使用了 cert-manager 生成的证书和私钥；  
> 2、`clientConfig.service` 字段指定了 Admission Webhook Server 的地址，这里使用了本测试 Admission 控制器的 Kubernetes Service 的地址；  
> 3、为了测试不影响其他 Pod 的正常验证逻辑，`objectSelector` 字段指定包含了 `example-admission-webhook-test` 标签的 Pod 才会被本测试 Admission 控制器处理。

## 测试

### 1. 部署 Admission 控制器

```bash
kubectl apply -f deployment.yaml
```

## 参考资料

- [准入控制器参考 | Kubernetes](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/admission-controllers/)
- [动态准入控制 | Kubernetes](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/extensible-admission-controllers/)
- [Kubernetes admission webhook server 开发教程](https://www.zeng.dev/post/2021-denyenv-validating-admission-webhook/)
