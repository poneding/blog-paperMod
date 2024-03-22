---
title: Golang 父子节点数组转树结构
date: 2022-10-18T18:00:26+08:00
draft: false
tags:
  - Golang
keywords:
  - 数组
  - 父子
  - 树
weight: 1
lastmod: 2022-11-21T10:09:54.522Z
---

## 场景介绍

输入：一组父子关系节点的数组，元素具有 ID，PID 字段，ID 为元素的唯一标识，PID 为父节点的 ID。

输出：树形结构的数据，根节点为 PID 为 0 的节点，子节点为 PID 为父节点 ID 的节点。

## 代码实现

```go
package main

import (
	"encoding/json"
	"fmt"
)

type Node struct {
	ID       int
	PID      int
	Children []*Node
}

func main() {
	var nodes = []*Node{
		{ID: 1, PID: 0},
		{ID: 2, PID: 1},
		{ID: 3, PID: 1},
		{ID: 4, PID: 2},
		{ID: 5, PID: 2},
		{ID: 6, PID: 3},
	}

	tree := buildTree(nodes)

	b, err := json.Marshal(tree)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(b))
}

func buildTree(nodes []*Node) *Node {
	if len(nodes) == 0 {
		return nil
	}

	nodeMap := make(map[int]*Node)
	for _, node := range nodes {
		nodeMap[node.ID] = node
	}

	var root *Node
	for _, node := range nodes {
		if node.PID == 0 {
			root = node
		} else {
			parent, ok := nodeMap[node.PID]
			if !ok {
				continue
			}
			parent.Children = append(parent.Children, node)
		}
	}

	return root
}
```
