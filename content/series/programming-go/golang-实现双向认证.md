---
title: Golang 实现双向认证
date: 2022-10-02T01:49:13+08:00
draft: false
tags:
  - Golang
  - tls
keywords:
  - Golang
  - tls
  - mtls
weight: 1
lastmod: 2022-11-17T06:50:37.545Z
---

## TLS

传输层安全协议（TLS），在互联网上，通常是由服务器单向的向客户端提供证书，以证明其身份。

## mTLS

双向 TLS 认证，是指在客户端和服务器之间使用双行加密通道，mTLS 是云原生应用中常用的通信安全协议。

使用双向TLS连接的主要目的是当服务器应该只接受来自有限的允许的客户端的 TLS 连接时。例如，一个组织希望将服务器的 TLS 连接限制为只来自该组织的合法合作伙伴或客户。显然，为客户端添加IP白名单不是一个好的安全实践，因为IP可能被欺骗。

为了简化 mTLS 握手的过程，我们这样简单梳理：

1.   客户端发送访问服务器上受保护信息的请求；

2.   服务器向客户端提供公钥证书；

3.   客户端通过使用 CA 的公钥来验证服务器公钥证书的数字签名，以验证服务器的证书；
4.   如果步骤 3 成功，客户机将其客户端公钥证书发送到服务器；
5.   服务器使用步骤 3 中相同的方法验证客户机的证书；
6.   如果成功，服务器将对受保护信息的访问权授予客户机。

## 代码实现

需要实现客户端验证服务端的公钥证书，服务端验证客户端的公钥证书。

### 生成证书

```bash
echo '清理并生成目录'

OUT=./certs
DAYS=365
RSALEN=2048
CN=poneding

rm -rf ${OUT}/*
mkdir ${OUT}  >> /dev/null 2>&1

cd ${OUT}


echo '生成CA的私钥'
openssl genrsa -out ca.key ${RSALEN} >> /dev/null 2>&1

echo '生成CA的签名证书'
openssl req -new \
-x509 \
-key ca.key \
-subj "/CN=${CN}" \
-out ca.crt

echo ''
echo '生成server端私钥'
openssl genrsa -out server.key ${RSALEN} >> /dev/null 2>&1

echo '生成server端自签名'
openssl req -new \
-key server.key \
-subj "/CN=${CN}" \
-out server.csr

echo '签发server端证书'
openssl x509 -req  -sha256 \
-in server.csr \
-CA ca.crt -CAkey ca.key -CAcreateserial \
-out server.crt -text >> /dev/null 2>&1

echo '删除server端自签名证书'
rm server.csr

echo ''
echo '生成client私钥'
openssl genrsa -out client.key ${RSALEN}  >> /dev/null 2>&1

echo '生成client自签名'
openssl req -new \
    -subj "/CN=${CN}" \
    -key client.key \
    -out client.csr
echo '签发client证书'
openssl x509 -req -sha256\
 -CA ca.crt -CAkey ca.key -CAcreateserial\
 -days  ${DAYS}\
 -in client.csr\
 -out client.crt\
 -text >> /dev/null 2>&1
echo '删除client端自签名'
rm client.csr

echo ''
echo '删除临时文件'
rm ca.srl

echo ''
echo '完成'%  
```

### 服务端

```go
package main

import (
	"crypto/tls"
	"crypto/x509"
	"fmt"
	"log"
	"net/http"
	"os"
	"time"
)

var (
	caCert     = "../../certs/ca.crt"
	serverCert = "../../certs/server.crt"
	serverKey  = "../../certs/server.key"
)

type mtlsHandler struct {
}

func (m *mtlsHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "Hello World! ", time.Now())
}

func main() {
	pool := x509.NewCertPool()

	caCertBytes, err := os.ReadFile(caCert)
	if err != nil {
		panic(err)
	}
	pool.AppendCertsFromPEM(caCertBytes)

	server := &http.Server{
		Addr:    ":8443",
		Handler: &mtlsHandler{},
		TLSConfig: &tls.Config{
			ClientCAs:  pool,
			ClientAuth: tls.RequireAndVerifyClientCert, // 需要客户端证书
		},
	}

	log.Println("server started...")
	log.Fatalln(server.ListenAndServeTLS(serverCert, serverKey))
}
```

### 客户端

```go
package main

import (
	"crypto/tls"
	"crypto/x509"
	"fmt"
	"io"
	"net/http"
	"os"
)

var (
	caCert     = "../../certs/ca.crt"
	clientCert = "../../certs/client.crt"
	clientKey  = "../../certs/client.key"
)

func main() {
	pool := x509.NewCertPool()

	caCertBytes, err := os.ReadFile(caCert)
	if err != nil {
		panic(err)
	}
	pool.AppendCertsFromPEM(caCertBytes)

	clientCertBytes, err := tls.LoadX509KeyPair(clientCert, clientKey)
	if err != nil {
		panic(err)
	}

	tr := &http.Transport{
		TLSClientConfig: &tls.Config{
			RootCAs:            pool,
			Certificates:       []tls.Certificate{clientCertBytes},
			InsecureSkipVerify: true,
		},
	}

	client := http.Client{
		Transport: tr,
	}

	r, err := client.Get("https://127.0.0.1:8443") // server
	if err != nil {
		panic(err)
	}
	defer r.Body.Close()

	b, err := io.ReadAll(r.Body)
	if err != nil {
		panic(err)
	}

	fmt.Println(string(b))
}
```

