---
title: "部署个人博客：Hugo + GitHub Pages"
date: 2025-04-18
tags: ["Hugo", "个人博客"]
draft: false
---

这是一篇介绍 Hugo 博客的文章。在这里我们会介绍如何安装和使用 Hugo。

<!--more-->

## 🚀 一、安装 Hugo

首先，使用 `winget` 在 Windows 上安装 Hugo：

```bash
winget install Hugo.Hugo.Extended
```

安装完成后，验证是否成功安装 Hugo：

```bash
hugo version
```

如果命令行中显示版本信息，说明安装成功。

## 🏗️ 二、创建 Hugo 网站

接下来，创建一个新的 Hugo 网站：

```bash
hugo new site my-blog
```

进入站点目录：

```bash
cd my-blog
```

## 🎨 三、安装主题

您可以访问 [Hugo Themes](https://themes.gohugo.io/) 选择喜欢的主题，或者使用以下命令直接安装 `ananke` 主题：

```bash
git submodule add https://github.com/theNewDynamic/gohugo-theme-ananke.git themes/ananke
```

然后，在 `config.toml` 文件中设置主题：

```toml
theme = "ananke"
```

## ✍️ 四、创建文章

您可以通过以下命令创建一篇新的文章：

```bash
hugo new posts/my-first-post.md
```

在 `content/posts/my-first-post.md` 文件中，使用 Markdown 编辑您的文章内容。

## 🌐 五、预览本地站点

通过启动本地服务器，您可以在浏览器中查看站点效果：

```bash
hugo server -D
```

在浏览器中访问 `http://localhost:1313/` 进行预览。

## 📦 六、部署到 GitHub Pages

### 1. 创建 GitHub 仓库

在 GitHub 上创建一个新的仓库，并命名为 `username.github.io`，这是 GitHub Pages 的标准仓库命名方式。

### 2. 初始化 Git 仓库

在站点根目录下，初始化 Git 仓库并添加远程仓库：

```bash
git init
git remote add origin https://github.com/username/username.github.io.git
```

### 3. 添加并提交文件

添加所有文件并提交：

```bash
git add .
git commit -m "Initial commit"
```

### 4. 推送到 GitHub

将本地仓库推送到 GitHub：

```bash
git push -u origin master
```

### 5. 启用 GitHub Pages

在 GitHub 仓库的设置中，找到 GitHub Pages 部分，选择 `master` 或 `gh-pages` 分支作为发布源。

## 🔄 七、自动化部署（可选）

为了实现自动化部署，您可以使用 GitHub Actions。以下是一个简单的工作流示例：

```yaml
name: Deploy Hugo site to GitHub Pages

on:
  push:
    branches:
      - master
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          extended: true

      - name: Build
        run: hugo --minify

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

将此文件保存为 `.github/workflows/deploy.yml`，并推送到 GitHub 仓库。每次推送到 `master` 分支时，GitHub Actions 将自动构建并部署您的博客。

---

按照以上步骤，您就可以成功地部署您的个人博客，并享受 GitHub Pages 自动化部署带来的便捷体验！🎉