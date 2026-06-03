# Yane's Blog

> 🏠 博客地址：**[visiongem.github.io/Blog](https://visiongem.github.io/Blog/)**

基于 [Hugo](https://gohugo.io/) + [PaperMod](https://github.com/adityatelange/hugo-PaperMod) 的个人技术博客，托管在 GitHub Pages，通过 GitHub Actions 自动部署。

## 本地运行

```bash
# 克隆（含主题 submodule）
git clone --recurse-submodules git@github.com:visiongem/Blog.git
cd Blog

# 安装 Hugo（macOS）
brew install hugo

# 启动本地预览
hugo server -D
```

浏览器打开 `http://localhost:1313` 即可预览。

## 写文章

```bash
# 创建新文章
hugo new content posts/文章名.md

# 编辑 content/posts/文章名.md
# 把 draft: true 删掉或改成 false

# 本地预览
hugo server -D
```

## 发布

推送到 `master` 分支，GitHub Actions 会自动构建并部署到 GitHub Pages。

```bash
git add . && git commit -m "new post" && git push
```

等一分钟左右，文章就会出现在 [visiongem.github.io/Blog](https://visiongem.github.io/Blog/)。

## 技术栈

| | |
|---|---|
| 静态生成 | [Hugo](https://gohugo.io/) |
| 主题 | [PaperMod](https://github.com/adityatelange/hugo-PaperMod) |
| 托管 | [GitHub Pages](https://pages.github.com/) |
| CI/CD | [GitHub Actions](.github/workflows/deploy.yml) |
| 图床 | [jsDelivr](https://www.jsdelivr.com/) + GitHub |
