---
title: "使用 Giscus 作为博客评论系统"
date: 2022-09-28T09:02:43+08:00
draft: false
tags:
  - Giscus
keywords:
  - Giscus
  - 评论系统
weight: 1
---

## Giscus
- 开源、无广告、永久免费
- 支持多语言
- 支持表情反馈
- 支持懒加载

## 必要条件
1. 你的博客所用的 GitHub 的仓库必须是 `Public`，并且开通了 `Dicussion` 功能；
2. 安装 [giscus.app](https://github.com/apps/giscus)，安装的时候，分配你的博客所用的 GitHub 仓库即可。
> 当然，如果你的博客没有托管在 Github 上，你也可以单独创建一个 Github 仓库作为开通 giscus 评论。

## 使用姿势
1. 在 [giscus.app](https://giscus.app/) 做自定义配置，填入你的仓库名称，选择主题等，Giscus 会自动帮你生成 javascript 脚本；
2. Hugo 博客目录下，创建 `layouts/partials/comments.html` 文件，写入获取的脚本：
```html
<script src="https://giscus.app/client.js"
        data-repo="[在此输入仓库]"
        data-repo-id="[在此输入仓库 ID]"
        data-category="[在此输入分类名]"
        data-category-id="[在此输入分类 ID]"
        data-mapping="pathname"
        data-strict="0"
        data-reactions-enabled="1"
        data-emit-metadata="0"
        data-input-position="bottom"
        data-theme="light"
        data-lang="zh-CN"
        crossorigin="anonymous"
        async>
</script>
```

>   ⚠️注意：为了使下面的 javascript 脚本生效，data-theme 选择 light；或者你可以根据你选择的主题修改下面的 javascript 脚本。

## 自动主题

1. 使用一个 div 作为评论区域的容器
```html
<div class="giscus_comments">
{{- partial "comments.html" . }}
</div>
```
2. 在该容器下方写入主题自动切换的语句
```html
<script>
    document.querySelector("div.giscus_comments > script")
        .setAttribute(
            "data-theme",
            localStorage.getItem("pref-theme") ? localStorage.getItem("pref-theme") : window.matchMedia("(prefers-color-scheme: dark)").matches ? "dark" : "light"),
        document.querySelector("#theme-toggle").addEventListener("click", () => {
            let e = document.querySelector("iframe.giscus-frame");
            e && e.contentWindow.postMessage({
                giscus: {
                    setConfig: {
                        theme: localStorage.getItem("pref-theme") ? localStorage.getItem("pref-theme") === "dark" ? "light" : "dark" : document.body.className.includes("dark") ? "light" : "dark"
                    }
                }
            }, "https://giscus.app")
        })
</script>
```

## 🔗 链接

- 应用：[https://github.com/apps/giscus](https://github.com/apps/giscus)
- 源码：[https://github.com/giscus/giscus](https://github.com/giscus/giscus)
- 使用：[https://giscus.app](https://giscus.app)

