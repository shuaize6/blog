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

---

## 三、WSL + VS Code C++ 开发环境

### 1. VS Code 打开 WSL 项目

在 WSL 终端中进入项目目录后执行：

```bash
cd ~/playground
code .
```

注意是 `code .`（code + 空格 + 点），不是 `.code`。

第一次以 WSL 模式打开时，VS Code 会自动下载并安装 **VS Code Server** 到 WSL 中，后续打开通常不需要重复下载。

### 2. 判断是否进入 WSL 模式

VS Code 左下角或顶部标题栏应显示：

```text
WSL: Ubuntu-24.04
```

这表示编辑器、终端、扩展、编译器等都在 WSL 中运行。Windows 本地的 MinGW / MSVC 可以保留，但和 WSL 中的编译环境是两套独立环境。

### 3. 安装 C++ 编译调试环境

```bash
sudo apt update
sudo apt install build-essential gdb clang-format
```

验证安装：

```bash
g++ --version
gdb --version
clang-format --version
```

各组件的用途：

| 包名 | 作用 |
|------|------|
| `build-essential` | 基础编译工具集合（含 gcc、g++、make 等） |
| `g++` | 编译 C++ 源文件 |
| `gdb` | 调试 C++ 程序 |
| `clang-format` | 格式化 C++ 代码 |

### 4. VS Code 扩展

在扩展面板中，确保安装到 **WSL: Ubuntu-24.04** 的是：

```text
C/C++
Publisher: Microsoft
Extension ID: ms-vscode.cpptools
```

> 不要只安装 `C/C++ Themes`，它只是主题，不提供编译、补全或格式化功能。

### 5. 编译、运行与格式化

编译并运行：

```bash
g++ main.cpp -o main
./main
```

格式化代码快捷键：`Shift + Alt + F`

也可以在 VS Code 设置中开启保存时自动格式化：

```json
{
    "[cpp]": {
        "editor.formatOnSave": true
    }
}
```

之后按 `Ctrl + S` 保存时就会自动格式化。

---

## 四、补充命令

### 删除文件与目录

```bash
rm 文件名          # 删除文件
rm -r 文件夹名      # 递归删除文件夹及其内容
rm -rf 文件夹名     # 强制删除，跳过确认（小心使用）
```

### 查看命令帮助

```bash
命令 --help         # 简洁帮助
man 命令            # 详细手册，按 q 退出
```

### 文件内容查看

```bash
cat 文件名          # 一次性显示全部内容
less 文件名         # 分页查看，上下翻页，按 q 退出
head -n 20 文件名   # 查看前 20 行
tail -n 20 文件名   # 查看末尾 20 行
tail -f 文件名      # 实时追踪文件末尾（适合看日志）
```

### 磁盘与空间

```bash
df -h               # 查看磁盘分区使用情况
du -sh 目录名       # 统计目录总大小
du -h --max-depth=1 # 查看当前目录下各子目录大小
```

### 进程管理

```bash
ps aux              # 列出所有进程
kill PID            # 结束指定 PID 的进程
kill -9 PID         # 强制结束进程
htop                # 交互式进程管理器（比 btop 更轻量）
```

### 快捷操作速查表

| 快捷键 | 作用 |
|--------|------|
| `Ctrl + C` | 终止当前正在运行的命令 |
| `Ctrl + D` | 发送 EOF（退出终端 / Python 交互环境） |
| `Ctrl + L` | 清屏（等价于 `clear`） |
| `Ctrl + R` | 反向搜索历史命令 |
| `Ctrl + A` | 光标跳到行首 |
| `Ctrl + E` | 光标跳到行尾 |
| `Ctrl + U` | 删除光标前的所有内容 |
| `Ctrl + K` | 删除光标后的所有内容 |
| `Tab` | 自动补全文件名 / 命令 |

---

## 小结

这篇笔记把终端操作、WSL C++ 开发环境配置和博客发布流程汇总到一起，以后直接在 VSCode 里打开这个文件就能查到，不用再翻教程。
