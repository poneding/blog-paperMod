---
title: Golang 切片扩容
date: 2022-10-11T14:19:02+08:00
draft: false
tags:
  - Golang
keywords:
  - Golang
  - slice
  - 切片
weight: 1
lastmod: 2022-11-17T06:51:11.194Z
---



怎么理解切片 `s = append(s, item)` 需要使用 `s` 重新接收呢？

1.   在 golang 语言中所有的参数传递的方式都是值传递的，即便是指针，也是复制了一份指针传递；
2.   切片发生扩容后，底层的数组发生了变化，不再是原来的数组结构。

![image-20221011234826356](https://pding.oss-cn-hangzhou.aliyuncs.com/images/image-20221011234826356.png)
