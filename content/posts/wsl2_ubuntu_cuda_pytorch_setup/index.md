+++
title = 'WSL2 + Ubuntu 24.04 + CUDA + PyTorch 环境搭建记录'
date = '2026-08-22T00:00:00+08:00'
draft = false
tags = ['WSL2', 'CUDA', 'PyTorch', 'Ubuntu']
+++

# WSL2 + Ubuntu 24.04 + CUDA + PyTorch 环境搭建记录

> 适用环境：Windows 11 + WSL2 + Ubuntu 24.04 + NVIDIA RTX 3060 Laptop GPU  
> 本文记录当前机器的实际安装过程，以及后续如何进入环境、编译 CUDA 程序、运行 PyTorch GPU 测试。

---

## 第一部分：安装过程

### 1. 安装 WSL2 和 Ubuntu 24.04

在 Windows PowerShell 中安装 WSL2 Ubuntu：

```powershell
wsl --install --web-download -d Ubuntu-24.04 --location D:\wsl\Ubuntu-24.04
```

如果已经安装，可以查看当前发行版：

```powershell
wsl -l -v
```

正常应该看到类似：

```text
NAME            STATE      VERSION
Ubuntu-24.04    Running    2
```

进入 Ubuntu：

```powershell
wsl -d Ubuntu-24.04
```

Ubuntu 的 Linux 主目录是：

```bash
/home/xu
```

物理上存放在 D 盘的 WSL 虚拟磁盘里：

```text
D:\wsl\Ubuntu-24.04\ext4.vhdx
```

不要直接双击或修改 `ext4.vhdx`。

---

### 2. Ubuntu 基础软件更新

进入 Ubuntu 后，建议先回到用户目录：

```bash
cd ~
```

更新软件包：

```bash
sudo apt update
sudo apt upgrade -y
```

安装基础开发工具：

```bash
sudo apt install -y build-essential gcc g++ make cmake git curl wget vim nano python3 python3-pip python3-venv pkg-config
```

---

### 3. 配置 Ubuntu 国内 apt 源，可选

Ubuntu 24.04 默认源配置文件是：

```bash
/etc/apt/sources.list.d/ubuntu.sources
```

备份原配置：

```bash
sudo cp /etc/apt/sources.list.d/ubuntu.sources /etc/apt/sources.list.d/ubuntu.sources.bak
```

换成清华源：

```bash
sudo tee /etc/apt/sources.list.d/ubuntu.sources > /dev/null <<'EOF'
Types: deb
URIs: https://mirrors.tuna.tsinghua.edu.cn/ubuntu/
Suites: noble noble-updates noble-backports
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

Types: deb
URIs: https://mirrors.tuna.tsinghua.edu.cn/ubuntu/
Suites: noble-security
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
EOF
```

更新索引：

```bash
sudo apt clean
sudo apt update
```

验证：

```bash
grep -R "URIs:" /etc/apt/sources.list.d/ubuntu.sources
```

---

### 4. 检查 WSL2 是否识别 NVIDIA GPU

在 WSL Ubuntu 中执行：

```bash
nvidia-smi
```

当前机器已识别到 NVIDIA GPU，示例输出中包含：

```text
NVIDIA GeForce RTX 3060 Laptop GPU
Driver Version: 591.74
CUDA Version: 13.1
```

说明 Windows 侧 NVIDIA 驱动正常，WSL2 可以访问 GPU。

注意：WSL2 中不要安装 Linux 版 NVIDIA Driver，例如不要执行：

```bash
sudo apt install nvidia-driver-xxx
sudo apt install cuda-drivers
```

WSL2 的 NVIDIA 驱动应安装在 Windows 宿主机上。

---

### 5. 安装 CUDA Toolkit 12.8

创建工具目录：

```bash
cd ~
mkdir -p tools
cd tools
```

下载 CUDA Toolkit runfile 后安装：

```bash
sudo sh cuda_12.8.0_570.86.10_linux.run
```

安装界面中只选择：

```text
CUDA Toolkit 12.8
```

不要选择 Driver。

安装完成后显示：

```text
Driver:   Not Selected
Toolkit:  Installed in /usr/local/cuda-12.8/
```

这个结果是正确的。WSL2 中不安装 Linux Driver，看到 “Incomplete installation / did not install CUDA Driver” 警告可以忽略。

---

### 6. 配置 CUDA 环境变量

把 CUDA 路径写入 `~/.bashrc`：

```bash
echo 'export CUDA_HOME=/usr/local/cuda-12.8' >> ~/.bashrc
echo 'export PATH=$CUDA_HOME/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=$CUDA_HOME/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

验证 `nvcc`：

```bash
which nvcc
nvcc --version
```

正常输出类似：

```text
/usr/local/cuda-12.8/bin/nvcc
Cuda compilation tools, release 12.8
```

---

### 7. 安装 uv

在 WSL Ubuntu 中执行：

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

让当前终端立即识别 `uv`：

```bash
source $HOME/.local/bin/env
```

验证：

```bash
uv --version
which uv
```

把 uv 环境写入 `~/.bashrc`，以后自动生效：

```bash
echo 'source $HOME/.local/bin/env 2>/dev/null || true' >> ~/.bashrc
source ~/.bashrc
```

---

### 8. 配置 uv 国内 Python 包源

Ubuntu 的 apt 源和 Python 包源是分开的。  
apt 源只影响：

```bash
sudo apt install ...
```

uv / pip 源影响：

```bash
uv pip install ...
```

配置清华 PyPI 镜像：

```bash
echo 'export UV_DEFAULT_INDEX="https://pypi.tuna.tsinghua.edu.cn/simple"' >> ~/.bashrc
source ~/.bashrc
```

验证：

```bash
echo $UV_DEFAULT_INDEX
```

---

### 9. 创建 Python 虚拟环境

在项目目录中创建虚拟环境：

```bash
cd ~/projects
uv venv --python=3.14
```

激活环境：

```bash
source .venv/bin/activate
```

激活后命令行前面会出现：

```text
(projects)
```

查看 Python：

```bash
which python
python --version
```

---

### 10. 安装 PyTorch / torchvision / torchaudio / Triton

由于国内访问 PyTorch 官方源较慢，使用阿里云 PyTorch wheel 镜像安装：

```bash
UV_HTTP_TIMEOUT=300 uv pip install torch torchvision torchaudio \
  --index-url https://mirrors.aliyun.com/pytorch-wheels/cu128 \
  --extra-index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

安装结果中已包含：

```text
torch==2.12.0
torchvision==0.27.0
torchaudio==2.11.0
triton==3.7.0
```

同时也安装了 PyTorch 自带的 CUDA 运行时相关包，例如：

```text
nvidia-cublas
nvidia-cudnn
nvidia-cuda-runtime
nvidia-nccl
```

说明 PyTorch 使用的是 wheel 自带的 CUDA runtime；系统里的 `/usr/local/cuda-12.8` 主要用于 `nvcc` 编译 CUDA C/C++ 程序。

---

## 第二部分：环境进入、编译和运行代码

### 1. 进入 WSL Ubuntu

在 Windows PowerShell 中执行：

```powershell
wsl -d Ubuntu-24.04
```

进入后建议回到 Linux 用户目录：

```bash
cd ~
```

---

### 2. 进入项目目录

```bash
cd ~/projects
```

如果目录不存在：

```bash
mkdir -p ~/projects
cd ~/projects
```

---

### 3. 激活 Python 虚拟环境

当前虚拟环境位于：

```bash
~/projects/.venv
```

进入环境：

```bash
cd ~/projects
source .venv/bin/activate
```

激活后提示符类似：

```text
(projects) xu@DESKTOP-SSDJJDI:~/projects$
```

退出环境：

```bash
deactivate
```

---

### 4. 查看当前环境状态

查看 Python：

```bash
which python
python --version
```

查看 uv：

```bash
which uv
uv --version
```

查看 CUDA 编译器：

```bash
which nvcc
nvcc --version
```

查看 GPU：

```bash
nvidia-smi
```

---

### 5. 创建和编辑 CUDA 源文件

可以用 VS Code 打开当前目录：

```bash
cd ~/projects
code .
```

打开单个文件：

```bash
code hello.cu
```

如果没有配置 VS Code，也可以用 nano：

```bash
nano hello.cu
```

或者用 Windows 文件资源管理器打开当前 WSL 目录：

```bash
explorer.exe .
```

---

### 6. 最小 CUDA/C 程序测试

创建 `test.cu`：

```bash
cd ~/projects
code test.cu
```

写入：

```cpp
#include <stdio.h>

int main() {
    printf("hello cuda\n");
    return 0;
}
```

编译：

```bash
nvcc test.cu -o test
```

运行：

```bash
./test
```

预期输出：

```text
hello cuda
```

---

### 7. GPU Kernel 测试程序

创建 `hello.cu`：

```bash
cd ~/projects
code hello.cu
```

写入：

```cpp
#include <cstdio>
#include <cuda_runtime.h>

__global__ void hello() {
    printf("Hello from GPU block %d thread %d\n", blockIdx.x, threadIdx.x);
}

int main() {
    hello<<<1, 4>>>();
    cudaDeviceSynchronize();

    int count = 0;
    cudaGetDeviceCount(&count);
    printf("CUDA device count: %d\n", count);

    return 0;
}
```

编译并指定输出文件名：

```bash
nvcc hello.cu -o hello
```

运行：

```bash
./hello
```

预期输出类似：

```text
Hello from GPU block 0 thread 0
Hello from GPU block 0 thread 1
Hello from GPU block 0 thread 2
Hello from GPU block 0 thread 3
CUDA device count: 1
```

如果只执行：

```bash
nvcc hello.cu
```

没有指定 `-o`，默认会生成：

```bash
a.out
```

运行：

```bash
./a.out
```

---

### 8. 关于 nvcc warning

编译时可能看到：

```text
nvcc warning : Support for offline compilation for architectures prior to '<compute/sm/lto>_75' will be removed in a future release
```

这是警告，不是错误。程序仍然已经编译成功。

如果想隐藏该警告：

```bash
nvcc hello.cu -o hello -Wno-deprecated-gpu-targets
```

---

### 9. 删除编译结果

查看当前目录：

```bash
ls
```

删除默认编译结果：

```bash
rm a.out
```

删除指定输出文件：

```bash
rm test
rm hello
```

一次性删除多个编译结果：

```bash
rm -f a.out test hello
```

注意不要误删源码：

```text
hello.cu
test.cu
```

---

### 10. PyTorch GPU 测试

进入虚拟环境：

```bash
cd ~/projects
source .venv/bin/activate
```

运行测试：

```bash
python - <<'PY'
import torch

print("torch:", torch.__version__)
print("torch cuda:", torch.version.cuda)
print("cuda available:", torch.cuda.is_available())

if torch.cuda.is_available():
    print("gpu:", torch.cuda.get_device_name(0))
    x = torch.randn(4096, 4096, device="cuda")
    y = x @ x
    print("test result:", y.mean().item())
PY
```

如果输出：

```text
cuda available: True
gpu: NVIDIA GeForce RTX 3060 ...
```

说明 PyTorch 可以正常调用 GPU。

---

### 11. 查看 PyTorch / torchvision / torchaudio / triton 版本

```bash
python - <<'PY'
import torch
import torchvision
import torchaudio
import triton

print("torch:", torch.__version__)
print("torchvision:", torchvision.__version__)
print("torchaudio:", torchaudio.__version__)
print("triton:", triton.__version__)
PY
```

---

### 12. 常用命令速查

进入 Ubuntu：

```powershell
wsl -d Ubuntu-24.04
```

进入项目：

```bash
cd ~/projects
```

激活 Python 环境：

```bash
source .venv/bin/activate
```

退出 Python 环境：

```bash
deactivate
```

检查 GPU：

```bash
nvidia-smi
```

检查 CUDA 编译器：

```bash
nvcc --version
```

编译 CUDA 文件：

```bash
nvcc hello.cu -o hello
```

运行：

```bash
./hello
```

删除编译结果：

```bash
rm -f a.out hello test
```

打开当前目录到 Windows 文件资源管理器：

```bash
explorer.exe .
```

用 VS Code 打开当前目录：

```bash
code .
```

---

## 当前环境完成情况

- WSL2 Ubuntu 24.04：已完成
- Ubuntu 安装位置：`D:\wsl\Ubuntu-24.04`
- GPU 识别：已完成，RTX 3060 Laptop GPU
- CUDA Toolkit：已安装，`/usr/local/cuda-12.8`
- nvcc 编译：已通过
- uv：已安装
- Python 虚拟环境：已创建，`~/projects/.venv`
- PyTorch / torchvision / torchaudio / Triton：已安装
- PyTorch GPU 测试：需要执行验证脚本确认
