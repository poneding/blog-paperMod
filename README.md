# 博客 · 丁鹏

博客主页：

👉🏻 [博客 · 丁鹏](https://blog.poneding.com)

博客技术栈：

- 博客系统：[hugo](https://gohugo.io)
- 主题定制：[PaperMod](https://github.com/adityatelange/hugo-PaperMod)
- 评论系统：[giscus](https://giscus.app)
- 页面统计：[busuanzi](https://busuanzi.ibruce.info/)
- 自动发布：[actions-hugo](https://github.com/peaceiris/actions-hugo)
- 静态部署：[netlify](https://www.netlify.com/)

写博客：

1. 安装 hugo

```bash
brew install hugo
hugo version
```

2. 克隆博客仓库

```bash
git clone https://github.com/poneding/blog.git
cd blog
```

3. 新建

```bash
# 新建博客
hugo new -k posts content/posts/my-post.md

# 新建系列
hugo new -k series content/series/my-series.md
```

4. 发布博客

```bash
./release.sh
```

5. 本地运行

```bash
hugo server -D
```

6. 本地编译

```bash
hugo -D
```
