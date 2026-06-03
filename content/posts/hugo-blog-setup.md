---
title: "从 Hexo 到 Hugo：一个极简技术博客的搭建笔记"
date: 2026-06-03T10:00:00+08:00
draft: false
categories: ["工具"]
tags: ["Hugo", "GitHub Pages", "博客"]
---

![封面图](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/other/2026/hugo0.png)

## 一、为什么折腾博客

2021 年用 Hexo 搭过一个博客，写了篇 Hello World 就再也没碰过。三年后打开仓库，发现几个问题：

- 源文件丢了——仓库里只有生成的 HTML，原始 Markdown 不知所踪
- Node.js 依赖全部过期，`npm install` 一片红
- 站点标题还是默认的 **Hexo**，作者是 **John Doe**

简单说：这个博客**没法维护了**。于是重新整理了一套方案——Hugo + GitHub Pages + GitHub Actions。这篇记录一下搭建过程。

## 二、方案选择：为什么是 Hugo

静态博客的选项不少，先拉个表对比一下几个主流方案：

| | Hugo | Hexo | VitePress |
|---|---|---|---|
| 语言/运行时 | Go 编译的二进制 | Node.js | Node.js |
| 安装 | `brew install hugo` | `npm install -g hexo-cli` | `npm init vitepress` |
| 构建速度 | 毫秒级 | 秒级 | 秒级 |
| 依赖腐烂风险 | **零**（单文件） | 高（node_modules） | 高（node_modules） |
| 中文支持 | 好，主题生态丰富 | 好，中文用户多 | 偏文档风格 |
| 主题数量 | 300+ | 300+ | 少 |

核心考量就一条：**半年不碰还能直接用**。Hugo 就一个二进制文件，不存在 `node_modules` 腐烂的问题。

托管选 GitHub Pages，免费、不需要买服务器、和仓库天然集成。部署用 GitHub Actions，push 代码自动构建发布，本地连 Hugo 都不用装也能更新博客。

## 三、搭建流程

### 3.1 安装 Hugo

```bash
brew install hugo
```

一行就够了。不用配 Node 版本、不用装全局包。装完验证：

```bash
hugo version
# hugo v0.162.1+extended+withdeploy darwin/arm64
```

### 3.2 初始化项目

```bash
hugo new site Blog
cd Blog
```

生成的目录结构：

```
Blog/
├── hugo.toml           # 站点配置
├── content/            # 文章放这里
├── archetypes/         # 新文章模板
├── themes/             # 主题
└── public/             # 构建产物（不提交到 git）
```

### 3.3 装主题

选的是 **PaperMod**，Hugo 社区最流行的博客主题。支持深色模式、搜索、代码高亮、多语言。

用 git submodule 引入（不复制代码到仓库里）：

```bash
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

submodule 的好处：主题和博客代码分开维护，以后想更新主题一行命令搞定：

```bash
git submodule update --remote
```

### 3.4 配置 hugo.toml

这是最核心的一步，完整配置大概长这样：

```toml
baseURL = 'https://visiongem.github.io/Blog/'
locale = 'zh-cn'
title = "Yane's Blog"
theme = 'PaperMod'

# 菜单
[[menus.main]]
  name = '归档'
  url = '/archives/'
  weight = 10

[[menus.main]]
  name = '标签'
  url = '/tags/'
  weight = 20

# PaperMod 参数
[params]
  author = 'Yane'
  defaultTheme = 'auto'
  ShowReadingTime = true
  ShowCodeCopyButtons = true
  ShowToc = true
```

配置好之后本地跑一下预览：

```bash
hugo server -D
```

浏览器打开 `http://localhost:1313` 就能看到效果了。

### 3.5 设置自动部署

![部署流程图](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/other/2026/hugo1.png)

在 `.github/workflows/deploy.yml` 里写一个 Actions 流程：

```
push 到 master
    ↓
① checkout 代码 + submodule
    ↓
② 安装 Hugo
    ↓
③ hugo --minify 构建
    ↓
④ 部署到 GitHub Pages
    ↓
网站更新 ✅
```

关键配置：

```yaml
on:
  push:
    branches:
      - master
  workflow_dispatch:  # 允许手动触发

jobs:
  build:
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: 'latest'
          extended: true

      - run: hugo --minify

      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    needs: build
    steps:
      - uses: actions/deploy-pages@v4
```

然后去 GitHub 仓库的 Settings → Pages，把 Source 选为 **GitHub Actions**。

从此以后，写完文章只要推送代码，等一分钟网站就自动更新了。

### 3.6 写第一篇文章

```bash
hugo new content posts/hello-blog.md
```

生成的文件：

```markdown
+++
date = '2026-06-02T21:00:00+08:00'
draft = true
title = '欢迎来到我的博客'
tags = ['博客']
categories = ['杂谈']
+++

正文在这里...
```

把 `draft: true` 改成 `false`（或删掉这行），提交推送，文章就上线了。

## 四、文章分类怎么管

在每篇文章的头部用 `categories` 和 `tags` 两个字段管理：

| | categories | tags |
|---|---|---|
| 类比 | 书的章节 | 关键词索引 |
| 数量 | 通常 1 个 | 可以多个 |
| 示例 | Android、Kotlin、杂谈 | Compose、协程、性能 |

```markdown
categories: ["Android"]
tags: ["Compose", "性能优化"]
```

Hugo 会自动生成 `/categories/` 和 `/tags/` 页面，每个分类/标签都有独立的文章列表，不需要手动维护索引页。

当前采用的分类体系：

```
categories:
├── Android   — Jetpack、Compose、性能
├── Kotlin    — 语言特性、协程、KMP
├── 工具      — 开发工具、效率
└── 杂谈      — 非代码类技术话题

tags（自由组合）:
Compose、协程、KMP、源码分析、踩坑记录...
```

## 五、踩过的坑

**坑 1：自定义域名重定向**

之前给 `visiongem.github.io` 配过自定义域名 `v2yn.com`。这个域名过期后，访问 `visiongem.github.io/Blog/` 还是会被 301 跳转到 `v2yn.com`。

原因：GitHub Pages 里，如果 `用户名.github.io` 这个用户级仓库配了自定义域名，**所有**项目 Pages 都会被重定向。在 `visiongem/visiongem.github.io` 仓库的 Settings → Pages 里把 Custom domain 清掉就恢复了。

**坑 2：GitHub Actions 不触发**

workflow 里触发条件写的 `branches: [main]`，但仓库默认分支叫 `master`。push 了代码 CI 没任何反应。改成 `master` 就好了。

**坑 3：浏览器缓存 301**

自定义域名去掉之后，`curl` 已经返回 200，但浏览器还是跳旧域名。301 重定向会被浏览器**永久缓存**，开无痕窗口或手动清理缓存才能解决。

## 六、总结

这套方案的核心思路是**极简 + 零维护**：

- **极简**：安装只要一个二进制文件，没有 node_modules
- **零维护**：写完 Markdown 推送代码，CI 自动完成构建和部署
- **可迁移**：主题用 submodule 引入，和内容完全分离

日常写作流程就三步：

```bash
hugo new content posts/新文章.md   # 创建
# 编辑文章，draft 改成 false
git add . && git commit -m "新文章" && git push   # 发布
```

> 本文基于个人搭建经验整理，水平有限，难免有遗漏或不准确的地方，欢迎指正。

---

**参考来源**

- Hugo 官方文档：https://gohugo.io/documentation/
- PaperMod 主题：https://github.com/adityatelange/hugo-PaperMod
- GitHub Pages 文档：https://docs.github.com/en/pages
- peaceiris/actions-hugo：https://github.com/peaceiris/actions-hugo
