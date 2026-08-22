+++
title = '将“在终端中打开”默认切换为 PowerShell 7'
date = '2026-08-22T00:00:00+08:00'
draft = false
tags = ['Windows', 'PowerShell', '终端']
+++

# 将“在终端中打开”默认切换为 PowerShell 7

本文记录如何把 Windows 右键菜单中的 **“在终端中打开”** 默认终端配置改为 **PowerShell 7**。

## 适用场景

你已经安装了 **PowerShell 7**，并且在开始菜单中可以看到它，但在文件夹空白处右键点击 **“在终端中打开”** 时，打开的不是 PowerShell 7。

## 操作步骤

### 1. 打开 Windows Terminal

可以通过开始菜单搜索：

```text
Windows Terminal
```

或使用运行窗口：

1. 按 `Win + R`
2. 输入：

```text
wt
```

3. 回车打开 Windows Terminal

---

### 2. 打开 Windows Terminal 设置

在 Windows Terminal 窗口中按快捷键：

```text
Ctrl + ,
```

这会打开 Windows Terminal 的设置页面。

---

### 3. 修改默认配置文件

在设置页面中找到 **默认配置文件** 选项。

将默认配置文件改为：

```text
PowerShell
```

或：

```text
PowerShell 7
```

通常它对应的程序路径是：

```text
C:\Program Files\PowerShell\7\pwsh.exe
```

---

### 4. 保存设置

点击 **保存**。

之后在资源管理器中右键点击 **“在终端中打开”**，默认就会使用 PowerShell 7 打开。

## 验证方法

在任意文件夹空白处右键，点击：

```text
在终端中打开
```

打开后执行：

```powershell
$PSVersionTable.PSVersion
```

如果显示主版本号为 `7`，说明已经切换成功。

示例：

```text
Major  Minor  Patch
-----  -----  -----
7      x      x
```

## 备注

如果设置页面里没有看到 PowerShell 7，可以先确认 PowerShell 7 是否安装成功，并检查 Windows Terminal 的配置文件列表中是否存在 PowerShell 7。

一般情况下，只要按 `Ctrl + ,` 进入 Windows Terminal 设置页面，就可以直接选择默认配置文件，不需要手动编辑 `settings.json`。
