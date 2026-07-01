+++
title = '终端常用命令 & Hugo 发布流程速查'
date = '2026-07-01T11:31:42+08:00'
draft = false
tags = ['Linux', '终端', '命令', 'Hugo', 'Blog']
+++

## 前言

记录日常高频使用的终端命令和 Hugo 博客发布流程，方便查阅，减少重复搜索。

---

## 一、终端常用命令

### 1. 文件与目录

#### `ls`

列出当前目录下的普通文件和文件夹。

```bash
ls
```

#### `la`

显示主目录下的全部文件，包括隐藏文件。通常是 `ls -a` 或 `ls -al` 的别名。

```bash
la
# 等价于
ls -a
```

---

### 2. 目录导航

#### `cd ~`

回到当前用户的主目录，即 `/home/xu`。

```bash
cd ~
```

#### `cd <项目目录>`

进入指定的项目目录。

```bash
cd nanovllm/
```

---

### 3. Python 虚拟环境

#### `source .venv/bin/activate`

激活项目内的 Python 虚拟环境 `.venv`。激活后终端前缀出现 `(nanovllm)`，说明后续 `python`、`pip` 会使用这个项目专属环境。

```bash
source .venv/bin/activate
```

**激活后效果：**

```text
(nanovllm) xu@machine:~$
```

#### `deactivate`

退出当前 Python 虚拟环境，恢复系统默认 Python 环境。

```bash
deactivate
```

---

### 4. 系统监控

#### `btop`

打开终端资源监控工具，可查看 CPU、内存、进程、磁盘、网络等状态。按 `q` 退出。

```bash
btop
```

安装：

```bash
# Ubuntu / Debian
sudo apt install btop

# macOS
brew install btop
```

---

### 终端命令速查表

| 命令 | 作用 |
|------|------|
| `ls` | 列出目录内容 |
| `la` | 列出全部文件（含隐藏） |
| `cd ~` | 回到用户主目录 |
| `cd <dir>` | 进入指定目录 |
| `source .venv/bin/activate` | 激活 Python 虚拟环境 |
| `deactivate` | 退出虚拟环境 |
| `btop` | 系统资源监控 |

---

## 二、Hugo 博客发布流程

每次写新文章的标准流程（所有命令在 `myblog/` 目录下执行）。

### 1. 创建文章

```powershell
mkdir content\posts\你的文章名
hugo new content content/posts/你的文章名/index.md
```

### 2. 编辑文章

```powershell
code .\content\posts\你的文章名\index.md
```

### 3. 修改 front matter

- 把 `draft` 改成 `false`
- 写上文章标题和正文

### 4. 本地预览

```powershell
hugo server -D
```

浏览器打开 `http://localhost:1313/`，确认效果没问题后 `Ctrl + C` 停止。

### 5. 发布到 GitHub Pages

```powershell
git add -A
git commit -m "Add new post: 文章标题"
git push
```

推送后等待 GitHub Actions 变绿色勾，文章自动上线。

---

### 发布流程速查表

| 步骤 | 命令 |
|------|------|
| 创建文章 | `mkdir content\posts\xxx` → `hugo new content content/posts/xxx/index.md` |
| 编辑 | `code .\content\posts\xxx\index.md` |
| 预览 | `hugo server -D` |
| 提交 | `git add -A` |
| 推送 | `git commit -m "..."` → `git push` |

---

## 小结

这篇笔记把终端操作和博客发布流程汇总到一起，以后直接在 VSCode 里打开这个文件就能查到，不用再翻教程。
