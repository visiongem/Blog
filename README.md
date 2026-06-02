# Yane's Blog

基于 [Hugo](https://gohugo.io/) + [PaperMod](https://github.com/adityatelange/hugo-PaperMod) 主题的个人技术博客。

## 写文章

```bash
# 创建新文章
hugo new content posts/my-new-post.md

# 本地预览
hugo server -D
```

## 发布

推送到 `main` 分支，GitHub Actions 自动构建并部署到 GitHub Pages。

## 主题

PaperMod，以 git submodule 方式引入：

```bash
git clone --recurse-submodules git@github.com:visiongem/Blog.git
```
