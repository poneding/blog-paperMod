---
title: Go 包 - errors
date: 2022-10-19T14:44:06+08:00
lastmod: 2022-11-03T03:32:31.646Z
tags:
  - Golang
  - gopkg
keywords:
  - Golang
  - gopkg
  - errors
weight: 1
---

`errors` 包为你的 Go 程序提供一种对程序员调试、查看日志更友好的错误处理方式。

Go 程序中传统的错误处理方法：

```go
if err != nil {
    return err
}
```

递归的向上传递错误，这种方式有一个缺陷：最终处理错误的位置无法获取错误的调用上下文信息。

`errors` 包以不破坏错误的原始值的方式向错误中的添加调用上下文信息。

## 获取包

```go
go get github.com/pkg/errors
```

## 错误添加上下文

The errors.Wrap function returns a new error that adds context to the original error. For example

```go
_, err := ioutil.ReadAll(r)
if err != nil {
    return errors.Wrap(err, "read failed")
}
```

## Retrieving the cause of an error

Using `errors.Wrap` constructs a stack of errors, adding context to the preceding error. Depending on the nature of the error it may be necessary to reverse the operation of errors.Wrap to retrieve the original error for inspection. Any error value which implements this interface can be inspected by `errors.Cause`.

```go
type causer interface {
    Cause() error
}
```

`errors.Cause` will recursively retrieve the topmost error which does not implement `causer`, which is assumed to be the original cause. For example:

```go
switch err := errors.Cause(err).(type) {
case *MyError:
    // handle specifically
default:
    // unknown error
}
```

[Read the package documentation for more information](https://godoc.org/github.com/pkg/errors).

## Roadmap

With the upcoming [Go2 error proposals](https://go.googlesource.com/proposal/+/master/design/go2draft.md) this package is moving into maintenance mode. The roadmap for a 1.0 release is as follows:

- 0.9. Remove pre Go 1.9 and Go 1.10 support, address outstanding pull requests (if possible)
- 1.0. Final release.

## Contributing

Because of the Go2 errors changes, this package is not accepting proposals for new functionality. With that said, we welcome pull requests, bug fixes and issue reports. 

Before sending a PR, please discuss your change by raising an issue.

## License

BSD-2-Clause
