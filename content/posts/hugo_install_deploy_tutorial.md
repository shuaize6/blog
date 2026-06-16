+++
title = 'Windows 安装 Hugo 并部署到 GitHub Pages：从零到上线'
date = 2026-06-16T00:00:00+09:00
draft = false
+++

## 前言

这篇文章记录一次从零开始安装 Hugo、创建博客、写文章，并最终部署到 GitHub Pages 的完整过程。

使用环境：

- Windows
- PowerShell 7.6.2
- Hugo Extended
- Git
- GitHub Pages
- GitHub Actions

最终目标是让博客可以通过类似下面的地址访问：

```text
https://shuaize6.github.io/blog/
```

---

## 一、安装 Hugo 和 Git

### 1. 安装 Hugo Extended

在 PowerShell 中执行：

```powershell
winget install Hugo.Hugo.Extended
```

安装完成后检查版本：

```powershell
hugo version
```

如果能看到类似下面的输出，说明 Hugo 安装成功：

```text
hugo v0.163.2-19a5cec0b9618163bb519487382e861d29edf383+extended windows/amd64
```

这里的关键是 `extended`，说明安装的是 Hugo Extended 版本。

---

### 2. 安装 Git

执行：

```powershell
winget install Git.Git
```

安装完成后检查：

```powershell
git --version
```

如果看到类似下面的输出，说明 Git 安装成功：

```text
git version 2.52.0.windows.1
```

---

## 二、创建 Hugo 博客项目

假设博客项目放在：

```text
E:\hugo
```

进入目录：

```powershell
cd E:\hugo
```

创建 Hugo 项目：

```powershell
hugo new project myblog
```

进入项目根目录：

```powershell
cd E:\hugo\myblog
```

项目根目录一般会包含这些内容：

```text
hugo.toml
content
themes
archetypes
```

以后所有 Hugo 相关操作，基本都在这个项目根目录中完成。

---

## 三、初始化 Git 仓库

在项目根目录执行：

```powershell
git init
```

这一步的作用是把当前 Hugo 项目变成一个 Git 仓库，后面才能提交代码并推送到 GitHub。

---

## 四、安装 Hugo 主题

这里使用 Ananke 主题作为入门主题。

执行：

```powershell
git submodule add https://github.com/gohugo-ananke/ananke themes/ananke
```

然后把主题写入 `hugo.toml`：

```powershell
Add-Content hugo.toml "theme = 'ananke'"
```

也可以手动打开 `hugo.toml`，保证里面有这一行：

```toml
theme = 'ananke'
```

---

## 五、配置 hugo.toml

打开项目：

```powershell
code .
```

编辑 `hugo.toml`，可以配置成类似下面这样：

```toml
baseURL = 'https://shuaize6.github.io/blog/'
languageCode = 'zh-cn'
title = 'Xu 的技术博客'
theme = 'ananke'

[caches]
  [caches.images]
    dir = ':cacheDir/images'
```

这里需要注意：

- 如果仓库名是 `blog`
- GitHub 用户名是 `shuaize6`

那么 `baseURL` 应该是：

```toml
baseURL = 'https://shuaize6.github.io/blog/'
```

最后的 `/blog/` 不要漏掉。

---

## 六、创建第一篇文章

执行：

```powershell
hugo new content content/posts/my-first-post.md
```

然后打开文章文件：

```powershell
code .\content\posts\my-first-post.md
```

文章开头一般会有 front matter：

```toml
+++
title = 'My First Post'
date = 2026-06-16T00:00:00+09:00
draft = true
+++
```

如果要正式发布文章，需要把：

```toml
draft = true
```

改成：

```toml
draft = false
```

否则本地预览可能能看到，但是部署到 GitHub Pages 后看不到。

示例文章内容：

```markdown
+++
title = '我的第一篇博客'
date = 2026-06-16T00:00:00+09:00
draft = false
+++

## 前言

这是我的第一篇 Hugo 博客。

我准备用这个博客记录自己的学习过程，包括 AI Infra、CUDA 算子优化、nano-vLLM 源码阅读，以及水声定位相关内容。

## 为什么写博客

学习技术时，如果只是看视频和代码，很容易忘。

写博客可以把问题、思路、踩坑和解决方法记录下来，也方便以后复习。

## 下一步计划

接下来我会继续完善博客主题，并尝试把网站部署到 GitHub Pages。
```

---

## 七、本地预览博客

在项目根目录执行：

```powershell
hugo server -D
```

然后浏览器打开终端里显示的地址，一般是：

```text
http://localhost:1313/
```

`-D` 的作用是显示草稿文章。

如果文章已经设置为：

```toml
draft = false
```

也可以直接执行：

```powershell
hugo server
```

停止本地服务器：

```text
Ctrl + C
```

---

## 八、创建 .gitignore

Hugo 构建会生成 `public/` 目录，这个目录不建议提交到 GitHub，因为 GitHub Actions 会自动构建。

创建 `.gitignore` 文件：

```powershell
notepad .gitignore
```

写入：

```gitignore
# Hugo build output
.hugo_build.lock
public/
```

---

## 九、提交到 Git

查看状态：

```powershell
git status
```

添加文件：

```powershell
git add .
```

提交：

```powershell
git commit -m "Initial commit: Hugo blog site"
```

这里解释一下：

```powershell
git add .
```

表示把当前目录下的修改加入“待提交区”。

```powershell
git commit -m "Initial commit: Hugo blog site"
```

表示把这些修改保存成一次 Git 提交记录。

---

## 十、推送到 GitHub

在 GitHub 上创建一个仓库，比如：

```text
blog
```

然后添加远程仓库地址：

```powershell
git remote add origin https://github.com/shuaize6/blog.git
```

推送到 GitHub：

```powershell
git branch -M main
git push -u origin main
```

以后再提交新文章时，只需要：

```powershell
git add .
git commit -m "Add new post"
git push
```

---

## 十一、配置 GitHub Pages

进入 GitHub 仓库：

```text
https://github.com/shuaize6/blog
```

进入：

```text
Settings → Pages
```

在 `Build and deployment` 中，把 `Source` 设置为：

```text
GitHub Actions
```

不要选择 Jekyll，也不要选择 Static HTML。

---

## 十二、创建 GitHub Actions 工作流

在项目根目录创建目录：

```powershell
New-Item -ItemType Directory -Force .github\workflows
```

创建文件：

```powershell
notepad .github\workflows\hugo.yaml
```

写入下面内容：

```yaml
name: Build and deploy Hugo site

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

defaults:
  run:
    shell: bash

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v6
        with:
          submodules: recursive
          fetch-depth: 0

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v6
        with:
          enablement: true

      - name: Install Hugo
        run: |
          HUGO_VERSION=0.163.2
          curl -sLJO "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
          mkdir -p "${HOME}/.local/hugo"
          tar -C "${HOME}/.local/hugo" -xf "hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
          echo "${HOME}/.local/hugo" >> "${GITHUB_PATH}"

      - name: Verify Hugo
        run: hugo version

      - name: Build
        run: hugo --gc --minify --baseURL "${{ steps.pages.outputs.base_url }}/"

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v5
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}

    runs-on: ubuntu-latest
    needs: build

    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v5
```

这个工作流的作用是：

1. 每次推送到 `main` 分支时自动运行
2. 拉取 Hugo 博客源码
3. 安装 Hugo
4. 构建静态网站
5. 上传 `public/` 构建产物
6. 部署到 GitHub Pages

---

## 十三、提交 GitHub Actions 配置

执行：

```powershell
git add .github\workflows\hugo.yaml hugo.toml .gitignore
git commit -m "Configure GitHub Pages deployment"
git push
```

推送后，进入 GitHub 仓库的：

```text
Actions
```

查看 workflow 是否运行成功。

绿色勾表示成功。

红色叉表示失败，需要点进去查看错误日志。

---

## 十四、常见问题

### 1. 首页没有文章

最常见原因是文章还是草稿。

检查文章开头：

```toml
draft = true
```

改成：

```toml
draft = false
```

然后提交：

```powershell
git add content\posts\my-first-post.md
git commit -m "Publish first post"
git push
```

---

### 2. GitHub Actions 中旧的红色失败记录有影响吗？

没有影响。

Actions 页面里的红色记录只是历史失败记录。

只要最新一次运行是绿色勾，说明当前部署成功。

---

### 3. 出现 LF will be replaced by CRLF 警告

例如：

```text
warning: in the working copy of 'content/posts/nanovllm-learning-notes.md', LF will be replaced by CRLF the next time Git touches it
```

这不是错误，只是 Windows 换行符提醒。

可以先忽略，不影响 Hugo，也不影响 GitHub Pages。

---

### 4. Setup Pages 报错

如果 GitHub Actions 报错：

```text
Get Pages site failed. Please verify that the repository has Pages enabled and configured to build using GitHub Actions
```

可以在 workflow 中给 `actions/configure-pages` 加上：

```yaml
with:
  enablement: true
```

完整形式：

```yaml
- name: Setup Pages
  id: pages
  uses: actions/configure-pages@v6
  with:
    enablement: true
```

---

### 5. Install Hugo 报错 mkdir cannot create directory

如果报错：

```text
mkdir: cannot create directory '/home/runner/.local/hugo': No such file or directory
```

说明父目录不存在。

把：

```bash
mkdir "${HOME}/.local/hugo"
```

改成：

```bash
mkdir -p "${HOME}/.local/hugo"
```

---

## 十五、以后如何写新文章

创建新文章：

```powershell
hugo new content content/posts/nanovllm-learning-notes.md
```

打开编辑：

```powershell
code .\content\posts\nanovllm-learning-notes.md
```

确认文章开头：

```toml
+++
title = 'Nanovllm Learning Notes'
date = 2026-06-16T00:00:00+09:00
draft = false
+++
```

本地预览：

```powershell
hugo server -D
```

提交并发布：

```powershell
git add content\posts\nanovllm-learning-notes.md
git commit -m "Publish nano-vLLM learning notes"
git push
```

等待 GitHub Actions 变成绿色勾，文章就会自动上线。

---

## 十六、常用命令速查

### 查看 Hugo 版本

```powershell
hugo version
```

### 查看 Git 版本

```powershell
git --version
```

### 本地预览

```powershell
hugo server -D
```

### 正式构建

```powershell
hugo --gc --minify
```

### 查看 Git 状态

```powershell
git status
```

### 添加所有修改

```powershell
git add .
```

### 提交修改

```powershell
git commit -m "Your commit message"
```

### 推送到 GitHub

```powershell
git push
```

---

## 总结

这次 Hugo 博客搭建流程可以概括为：

```text
安装 Hugo 和 Git
创建 Hugo 项目
安装主题
写文章
本地预览
推送到 GitHub
配置 GitHub Pages
配置 GitHub Actions
自动部署上线
```

Hugo 的核心使用方式并不复杂。

以后真正重要的是持续写文章，把学习 AI Infra、CUDA 算子、nano-vLLM、水声定位等过程记录下来。

博客搭好只是第一步，长期输出才是它真正的价值。
