---
title: Go1.18 - 工作区模式
date: 2022-09-28T18:25:05+08:00
draft: false
tags:
  - Golang
keywords:
  - Golang
  - 工作区模式
weight: 1
lastmod: 2022-11-17T06:49:04.133Z
---

安装的 Go1.18 或更新版本，它为你提供了工作区模式（Workspace mode），帮助你更好做 go 模块之间依赖的管理。


例如，你开发一个新项目，分了两个 go module，分别为 `service-a` 和 `service-b`，`service-a` 依赖了`service-b` ，现在项目还处于开发阶段，我们都是这么处理的。

创建项目目录：

```bash
mkdir service
cd service
```

创建 `service-b` 模块:

```bash
mkdir service-b
cd service-b
go mod init github.com/poneding/service-b
```

编写 `service-b` 模块代码：

```bash
mkdir greeting
vim greeting/hello.go
```

```go
package greeting

import "fmt"

func Hello(name string){
    fmt.Println("Hello, " + name) 
}
```

继续创建 `service-a` 模块：

```bash
cd ..
mkdir service-a
cd service-a
go mod init github.com/poneding/service-a
```

因为 `service-a` 需要依赖本地开发的 `service-b` 类库，所以我们需要在 go.mod 中引入 `service-a` ：

```bash
vim go.mod
```

```go
module github.com/poneding/service-a

go 1.18

require(
    github.com/poneding/service-b v1.0.0
)

replace(
    github.com/poneding/service-a v1.0.0 => ../service-b
)
```

然后，编写主函数代码：

```bash
vim main.go
```

```go
package main
import(
    "github.com/poneding/service-b/greeting"
)

func main(){
    greeting.Hello()
}
```

在上述的整个过程中，你会发现，我们引用了本地的代码类库 `github.com/poneding/service-b v1.0.0 => ../service-b`。而且，如果此时别的开发者和你一起协作开发 `service-a` module的时候，他是无法引用到这个类库的（除非他本地也同步了`../service-b`）。

现在你就可以通过 `go work` 来解决这种烦恼了。

首先，你要做的是回到module的外面，然后执行 `go work init` 命令：

```bash
cd ..
go work init service-a
go work init service-b
```

此时，你会发现，目录下多了个`go.work`文件，查看该文件内容：

```bash
$ cat go.work
go 1.18

use(
    ./service-a
    ./service-b
)
```

module 依赖的类库目录就在 `use` 块中，此时，你可以删除 `service-a`目录下 `go.mod` 中的 `replace` 块，然后运行 main 函数了：

```bash
cd service-a && go run main.go
# 或者你可以直接在工作区目录运行 go run service-a/main.go
```

如果你的主项目依赖多个本地类库，那么你可以使用如下命令添加

```bash
go work use service-c
go work use service-d
```

最后，你可以通过 `go help work`，了解更多。
