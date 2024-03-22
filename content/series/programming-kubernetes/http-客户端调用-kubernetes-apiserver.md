---
title: HTTP 客户端调用 Kubernetes APIServer
description: ""
date: 2023-05-04T07:09:50.614Z
lastmod: 2023-05-04T07:09:50.614Z
slug: http-client-call-kubernetes-apiserver
tags: []
weight: 0
cover:
  image: "https://fs.poneding.com/images/202305041515191.png"
---

本篇介绍 HTTP 客户端调用 Kubernetes APIServer 的方法。

## 如何获取 Kubernetes api-server 地址

查看 api-server 的几种方式：

```bash
# 1. 直接查看 kubeconfig 文件
$ cat ~/.kube/config 
apiVersion: v1
clusters:
- cluster:
    server: https://192.168.58.2:8443
...

# 2. kubectl 查看集群信息
$ kubectl cluster-info
Kubernetes control plane is running at https://192.168.58.2:8443
...

# 3. kubectl 查看集群配置
$ kubectl config view
clusters:
- cluster:
    ...
    server: https://192.168.58.2:8443
...
```

配置环境变量

```bash
KUBE_API=$(kubectl config view -o jsonpath='{.clusters[0].cluster.server}')

echo $KUBE_API
```

## api-server 如何给客户端授权

### 使用证书授权

直接调用：

```bash
$ curl $KUBE_API/version
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.

# 忽略证书验证
$ curl $KUBE_API/version --insecure
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
}
```

结果是我们无法顺利调用接口。

获取证书：

```bash
# 服务端证书
$ kubectl config view --raw \
  -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' \
  | base64 --decode > ./server-ca.crt

# 服务端证书
$ kubectl config view --raw \
  -o jsonpath='{.users[0].user.client-certificate-data}' \
  | base64 --decode > ./client-ca.crt

# 服务端证书密钥
$ kubectl config view --raw \
  -o jsonpath='{.users[0].user.client-key-data}' \
  | base64 --decode > ./client-ca.key
```

我们这次尝试使用证书调用 API：

```bash
$ curl $KUBE_API/version --cacert ./server-ca.crt \
  --cert ./client-ca.crt \
  --key ./client-ca.key
{
  "major": "1",
  "minor": "22",
  "gitVersion": "v1.22.6+k3s1",
  "gitCommit": "3228d9cb9a4727d48f60de4f1ab472f7c50df904",
  "gitTreeState": "clean",
  "buildDate": "2022-01-25T01:27:44Z",
  "goVersion": "go1.16.10",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

此时，可以正常调用 API 了，我们继续调用其他接口：

```bash
$ curl $KUBE_API/apis/apps/v1/deployments --cacert ./server-ca.crt \
  --cert ./client-ca.crt \
  --key ./client-ca.key
{
  "kind": "DeploymentList",
  "apiVersion": "apps/v1",
  "metadata": {
    "resourceVersion": "654514"
  },
  "items": [...]
}
```

也 OK 了。

### 使用 token 授权

token 指的是 ServiceAccount 的 Token，每个命名空间内都会存在一个默认的 sa：default。

我们尝试来生成 sa token，我们首先生成一个 default 命名空间下 default sa 的 token：

```bash
# 获取 sa 采用的 secret
$ kubectl get sa default -o jsonpath='{.secrets[0].name}'

# 获取 token
DEFAULT_DEFAULT_TOKEN=$(kubectl get secrets \
  $(kubectl get sa default -o jsonpath='{.secrets[0].name}') \
  -o jsonpath='{.data.token}' | base64 --decode)

$ echo $DEFAULT_DEFAULT_TOKEN
eyJhbGciOiJSUzI1NiIsImtpZCI6IjFJdzhRZlV3TkpRRm5aSzU4b2hWTWI3YmdPUTlyLTBWSU9OQmNlN3cwdGsifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlZmF1bHQtdG9rZW4tcW50bGsiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGVmYXVsdCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjE0OGJmM2U0LTkzY2EtNGJiMS04MDczLWMzZmIzM2NiYmJmNiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.ivtk9eRH-35wIaPk4CtPSBoBkuiMxQyta17qMxNjhgedyV5i1QGYty36k0ufbOpBMXr2DsdRy8yQTx2qnH-AcduyxaoCxX-SQ4yGUsKSHCTipktcWqFi-CFzNo6EMCZiX8zAmeXjYOMmF8kh2T6wkHmjERDYsqWPaftasTUrKEYpcawFCMnv0QTpDe-okr6vQx6t7pJ5fx_PCw-GEEZUKQZn1tHIStd77eZd546--rrS6nPczKc3GnVFsDTcPM5HI7T_hXnId1TEnOYM8H5ornJ6uDP2oN_niwV41qOXMM52Bep0cvnikG-kUklLpmZxkwAtQCHDDh36A5JX_oaK5w
```

💡 如果你的 kubectl 版本大于 1.24，那么你可以直接使用以下命令获取 token：

```bash
$ DEFAULT_DEFAULT_TOKEN=$(kubectl create token default -n default)

$ echo $DEFAULT_DEFAULT_TOKEN
eyJhbGciOiJSUzI1NiIsImtpZCI6IjFJdzhRZlV3TkpRRm5aSzU4b2hWTWI3YmdPUTlyLTBWSU9OQmNlN3cwdGsifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiLCJrM3MiXSwiZXhwIjoxNjc2NDUwODg2LCJpYXQiOjE2NzY0NDcyODYsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0Iiwic2VydmljZWFjY291bnQiOnsibmFtZSI6ImRlZmF1bHQiLCJ1aWQiOiIxNDhiZjNlNC05M2NhLTRiYjEtODA3My1jM2ZiMzNjYmJiZjYifX0sIm5iZiI6MTY3NjQ0NzI4Niwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRlZmF1bHQ6ZGVmYXVsdCJ9.FC6SZ3fKhKML76AYnfaq4Y74mlRMBUxgfsEocrczzoN_NvDbqzZ_0sCAvA_ZdcVXv74hXTTeO1_DLoXZE_aLmGIxH1ImfbCDbxZH1xvNbE-7oozKmWBjYM7VRnNVvNC8EiRmcSEMttnQxgnBqUDZCyU8VA_pujld_RsB4SiD8tpXN5PaSaEx6vz6AWYWtW8wqwcAlIWTGk4hae090a0sLplyB4xx-7SiYjmkM9tVXFz5WWdUYSfyQeM-EfDpH4fNsvefWtW_KeJ5Wg28RuhiLbUv9_UV1RGt11Wh7lf0nNmxobqB8j-PEnphiECMKDv29x5KtQDU1wSgbSMI-_eTlQ
```

使用 token 调用 API：

```bash
$ curl $KUBE_API/apis/apps/v1/namespaces/default/deployments --cacert ./server-ca.crt \
  -H "Authorization: Bearer $DEFAULT_DEFAULT_TOKEN"
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  }
  "status": "Failure",
  "message": "deployments.apps is forbidden: User \"system:serviceaccount:default:default\" cannot list resource \"deployments\" in API group \"apps\" at the cluster scope",
  "reason": "Forbidden",
  "details": {
    "group": "apps",
    "kind": "deployments"
  },
  "code": 403
}
```

可以看到用户 `system:serviceaccount:default:default` 并没有权限获取自身命名空间内的 Kubernetes 对象列表。

接下来让我们尝试给这个 sa 绑定一个 `cluster-admin` 的 ClusterRole，然后我们再次生成 token，并调用接口：

```bash
$ kubectl create clusterrolebinding default-sa-with-cluster-admin-role \
  --clusterrole cluster-admin --serviceaccount default:default

$ DEFAULT_DEFAULT_TOKEN=$(kubectl create token default -n default)

$ curl $KUBE_API/apis/apps/v1/namespaces/default/deployments --cacert ./server-ca.crt \
  -H "Authorization: Bearer $DEFAULT_DEFAULT_TOKEN"
{
  "kind": "DeploymentList",
  "apiVersion": "apps/v1",
  "metadata": {
    "resourceVersion": "2824281"
  },
  "items": []
}
```

这完美生效，因为 defualt sa 有了一个超级大的 admin 角色。（另外，你可以单独创建一个特定资源权限的 role，在绑定到一个 sa，来满足最小权限原则）

## 在 Pod 内访问 api-server

在 pod 内部，默认会生成 Kubernetes 服务地址相关的环境变量，并且会在特殊目录下保存证书以及 token，当然这个token 是根据 pod 所使用的 sa 生成的。

证书文件地址：/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

token 文件地址：/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

```bash
$ kubectl run -it --image curlimages/curl --restart=Never mypod -- sh
$ env | grep KUBERNETES
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.43.0.1:443
KUBERNETES_PORT_443_TCP_ADDR=10.43.0.1
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP=tcp://10.43.0.1:443
KUBERNETES_SERVICE_HOST=10.43.0.1

$ TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
$ curl https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}/apis/apps/v1 \
  --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  --header "Authorization: Bearer $TOKEN"
```

## 使用 curl 执行基本的 CRUD 操作

创建资源：

```bash
$ curl $KUBE_API/apis/apps/v1/namespaces/default/deployments \
  --cacert ./server-ca.crt \
  --cert ./client-ca.crt \
  --key ./client-ca.key \
  -X POST \
  -H 'Content-Type: application/yaml' \
  -d '---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sleep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sleep
  template:
    metadata:
      labels:
        app: sleep
    spec:
      containers:
      - name: sleep
        image: curlimages/curl
        command: ["/bin/sleep", "365d"]
'

$ kubectl get deploy
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
sleep   1/1     1            1           8s
```

获取资源：

```bash
$ curl $KUBE_API/apis/apps/v1/namespaces/default/deployments \
  --cacert ./server-ca.crt \
  --cert ./client-ca.crt \
  --key ./client-ca.key

# 给定特定的资源名
$ curl $KUBE_API/apis/apps/v1/namespaces/default/deployments/sleep \
  --cacert ./server-ca.crt \
  --cert ./client-ca.crt \
  --key ./client-ca.key

# watch
$ curl $KUBE_API'/apis/apps/v1/namespaces/default/deployments?watch=true' \              
  --cacert ./server-ca.crt \
  --cert ./client-ca.crt \
  --key ./client-ca.key
```

更新资源：

```bash
curl $KUBE_API/apis/apps/v1/namespaces/default/deployments/sleep \
  --cacert ./server-ca.crt \
  --cert ./client-ca.crt \
  --key ./client-ca.key \
  -X PUT \
  -H 'Content-Type: application/yaml' \
  -d '---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sleep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sleep
  template:
    metadata:
      labels:
        app: sleep
    spec:
      containers:
      - name: sleep
        image: curlimages/curl
        command: ["/bin/sleep", "730d"]  # <-- Making it sleep twice longer
'

# patch 更新
curl $KUBE_API/apis/apps/v1/namespaces/default/deployments/sleep \
  --cacert ./server-ca.crt \
  --cert ./client-ca.crt \
  --key ./client-ca.key \
  -X PATCH \
  -H 'Content-Type: application/merge-patch+json' \
  -d '{
  "spec": {
    "template": {
      "spec": {
        "containers": [
          {
            "name": "sleep",
            "image": "curlimages/curl",
            "command": ["/bin/sleep", "1d"]
          }
        ]
      }
    }
  }
}'
```

删除资源：

```bash
$ curl $KUBE_API/apis/apps/v1/namespaces/default/deployments \
  --cacert ./server-ca.crt \
  --cert ./client-ca.crt \
  --key ./client-ca.key \
  -X DELETE

# 删除单个资源
$ curl $KUBE_API/apis/apps/v1/namespaces/default/deployments/sleep \
  --cacert ./server-ca.crt \
  --cert ./client-ca.crt \
  --key ./client-ca.key \
  -X DELETE
```

## 如何使用 kubectl 调用 API

### 使用 kubectl 代理 Kubernetes api-server

```bash
$ kubectl proxy --port 8080

# 启动了代理服务后，调用 Kubernetes api-server 变得更简单
$ curl localhost:8080/apis/apps/v1/deployments
{
  "kind": "DeploymentList",
  "apiVersion": "apps/v1",
  "metadata": {
    "resourceVersion": "660883"
  },
  "items": [...]
}
```

### kubectl 使用原始模式调用 API

```bash
# 获取
$ kubectl get --raw /api/v1/namespaces/default/pods

# 创建
$ kubectl create --raw /api/v1/namespaces/default/pods -f file.yaml

# 更新
$ kubectl replace --raw /api/v1/namespaces/default/pods/mypod -f file.json

# 删除
$ kubectl delete --raw /api/v1/namespaces/default/pods
```

## 如何查看 kubectl 命令（如 apply）发送的 API 请求

```bash
$ kubectl create deployment nginx --image nginx -v 6
I0215 17:07:35.188480   43870 loader.go:372] Config loaded from file:  /Users/dp/.kube/config
I0215 17:07:35.362580   43870 round_trippers.go:553] POST https://192.168.58.2:8443/apis/apps/v1/namespaces/default/deployments?fieldManager=kubectl-create&fieldValidation=Strict 201 Created in 167 milliseconds
deployment.apps/nginx created
```

## 参考

- [How To Call Kubernetes API using Simple HTTP Client](https://iximiuz.com/en/posts/kubernetes-api-call-simple-http-client/)
- [使用 Kubernetes API 访问集群 | Kubernetes](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/access-cluster-api/)
- [从 Pod 中访问 Kubernetes API | Kuberentes](https://kubernetes.io/zh-cn/docs/tasks/run-application/access-api-from-pod/)
