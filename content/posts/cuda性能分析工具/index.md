+++
date = '2026-08-08T13:27:12+08:00'
draft = false
title = 'Cuda性能分析工具'
+++
下面这份可以直接作为后续学习记录。

# CUDA 性能分析工具简要汇总

当前环境：

```text
GPU: NVIDIA GeForce RTX 3060 Laptop GPU
Compute Capability: 8.6
CUDA Toolkit: 12.8
Nsight Compute: 2025.1
Nsight Systems: 2024.6
```

## 1. 测试程序 `test_kernels`

作用：

- 检查计算结果是否正确。
- 输出输入尺寸。
- 测量整个 `rmsNorm()` 函数的端到端时间。
- 时间包含显存分配、数据拷贝、kernel 和显存释放。

查看详细测试信息：

```bash
SKIP_ATTENTION=1 ./test_kernels --verbose
```

主要输出：

```text
Rows              输入行数
Hidden Dim        每行元素数量
Warm-up Iters     预热次数
Profile Iters     正式计时次数
Avg Time          完整函数的平均时间
Max Diff          GPU结果与CPU参考结果的最大误差
Verification      正确性是否通过
```

当前测试器每个尺寸、每种数据类型执行：

```text
1次预热 + 10次正式测试 = 11次kernel launch
```

`Avg Time` 是完整函数时间，不是纯 kernel 时间。

---

## 2. Nsight Compute：`ncu`

### 主要用途

`ncu` 用于分析单个 CUDA kernel 的 GPU 执行情况，包括：

- kernel 执行时间
- Grid 和 Block 大小
- SM 利用率
- 显存吞吐率
- L1/L2 缓存命中率
- Occupancy
- 寄存器和共享内存使用量
- Warp 执行效率

它更适合回答：

> kernel 为什么慢？是计算、访存、并行度还是资源限制？

### 配置环境

```bash
export PATH=/usr/local/cuda-12.8/bin:$PATH
```

检查：

```bash
which ncu
ncu --version
```

### 只测 kernel 时间

以 Test #9 float 为例：

```bash
SKIP_ATTENTION=1 ncu \
  --kernel-name-base demangled \
  --kernel-name 'regex:.*rmsNorm_kernel<float>.*' \
  --metrics gpu__time_duration.sum \
  --launch-skip 89 \
  --launch-count 10 \
  --print-summary per-kernel \
  ./test_kernels
```

关键参数：

```text
--kernel-name-base demangled
    使用可读的C++函数名匹配kernel。

--kernel-name 'regex:...'
    只分析匹配的kernel。

--metrics gpu__time_duration.sum
    只收集GPU执行时间。

--launch-skip N
    跳过前N次匹配的kernel launch。

--launch-count N
    只收集N次匹配的kernel。

--print-summary per-kernel
    输出多次执行的最小、最大和平均值。
```

注意：`--launch-count 1` 只停止性能数据采集，测试程序仍会继续运行。

### 当前测试器的 skip 公式

当 kernel 名称已经限定为单独的 float 或 half 时：

```text
Test #N 的正式迭代：

launch-skip = (N - 1) × 11 + 1
launch-count = 10
```

例如 Test #9：

```text
(9 - 1) × 11 + 1 = 89
```

其中最后的 `+1` 是跳过该测试自己的 warm-up。

### 收集详细 kernel 指标

```bash
SKIP_ATTENTION=1 ncu \
  --kernel-name-base demangled \
  --kernel-name 'regex:.*rmsNorm_kernel<float>.*' \
  --launch-skip 89 \
  --launch-count 1 \
  --section SpeedOfLight \
  --section LaunchStats \
  --section Occupancy \
  --section MemoryWorkloadAnalysis \
  ./test_kernels
```

各 section 的作用：

| Section | 作用 |
|---|---|
| `SpeedOfLight` | 查看计算和内存吞吐是否接近硬件上限 |
| `LaunchStats` | 查看 Grid、Block、线程、寄存器和共享内存 |
| `Occupancy` | 查看理论和实际 Occupancy |
| `MemoryWorkloadAnalysis` | 查看显存吞吐、缓存命中率和访存压力 |

常见指标含义：

| 指标 | 含义 |
|---|---|
| `Duration` | kernel 的 GPU 执行时间 |
| `Compute Throughput` | SM 计算资源利用率 |
| `Memory Throughput` | GPU 内存系统利用率 |
| `DRAM Throughput` | 显存带宽利用率 |
| `L1/L2 Hit Rate` | 缓存命中率 |
| `Registers Per Thread` | 每个线程使用的寄存器 |
| `Achieved Occupancy` | 实际活跃 warp 比例 |
| `Waves Per SM` | Grid 能形成多少轮满载工作 |

### 保存输出

保存全部文本输出：

```bash
SKIP_ATTENTION=1 ncu ... ./test_kernels \
  > ncu_result.txt 2>&1
```

查看：

```bash
less ncu_result.txt
```

也可以边看边保存：

```bash
SKIP_ATTENTION=1 ncu ... ./test_kernels 2>&1 |
  tee ncu_result.txt
```

### NCU 注意事项

- `Duration` 只包含 kernel，不包含 `cudaMalloc` 和 `cudaMemcpy`。
- 收集多个 section 时，NCU 会重放 kernel 多次。
- 输出中的 `7 passes` 表示为收集指标重放了7遍。
- 不应使用整个 NCU 命令的墙钟时间评价性能。
- `--set full` 会收集大量指标，速度很慢，通常先用少量 section。
- 比较优化前后时应保持尺寸、编译参数和 GPU 环境一致。

---

## 3. Nsight Systems：`nsys`

### 主要用途

`nsys` 用于分析整个程序的时间线，包括：

- CPU 调用了哪些 CUDA API
- `cudaMalloc` 花费多少时间
- Host↔Device 拷贝花费多少时间
- kernel 何时启动
- CPU 和 GPU 是否并行
- 是否存在同步和空闲间隔

它更适合回答：

> 完整程序为什么慢？时间究竟花在 kernel、内存拷贝还是显存分配上？

### 采集报告

```bash
SKIP_ATTENTION=1 nsys profile \
  --trace=cuda,osrt \
  --output=rmsnorm_systems \
  --force-overwrite=true \
  ./test_kernels
```

生成：

```text
rmsnorm_systems.nsys-rep
```

参数含义：

```text
--trace=cuda,osrt
    跟踪CUDA API和操作系统运行时调用。

--output=rmsnorm_systems
    设置报告名称。

--force-overwrite=true
    允许覆盖同名旧报告。
```

### 生成文本统计

```bash
nsys stats \
  --report cuda_api_sum,cuda_gpu_kern_sum,cuda_gpu_mem_time_sum \
  rmsnorm_systems.nsys-rep \
  > rmsnorm_systems_stats.txt
```

报告作用：

| Report | 内容 |
|---|---|
| `cuda_api_sum` | CUDA API 调用次数和耗时 |
| `cuda_gpu_kern_sum` | GPU kernel 汇总 |
| `cuda_gpu_mem_time_sum` | GPU 内存传输汇总 |

当前环境的报告没有记录到 GPU kernel/memory 时间线，因此后两项显示 `SKIPPED`，但 CUDA API 统计仍然有效；纯 kernel 时间由 NCU 提供。

### 当前分析得到的例子

一共有286次 RMSNorm 调用：

```text
cudaMalloc:       858次
cudaMemcpy:       858次
cudaFree:         858次
cudaLaunchKernel: 286次
```

CUDA API 时间占比：

```text
cudaMalloc     39.5%
cudaMemcpy     32.3%
cudaFree       24.1%
kernel launch   2.7%
```

这说明当前完整函数的主要瓶颈是显存分配、拷贝和释放，而不是 RMSNorm kernel。

---

## 4. NCU 和 NSYS 的区别

| 工具 | 分析范围 | 主要问题 |
|---|---|---|
| 测试器 `--verbose` | 完整函数 | 正确吗？总共多长时间？ |
| `ncu` | 单个 GPU kernel | kernel 内部为什么慢？ |
| `nsys` | 整个应用时间线 | 时间花在分配、拷贝、同步还是 kernel？ |

可以这样理解：

```text
测试器发现“慢”
       ↓
Nsight Systems 定位慢在哪个阶段
       ↓
Nsight Compute 深入分析具体kernel
       ↓
修改和优化
       ↓
重新运行相同测试验证
```

## 5. 推荐的性能分析流程

1. 先确认正确性：

```bash
SKIP_ATTENTION=1 ./test_kernels --verbose
```

2. 用 NCU 测量代表性尺寸的 kernel 时间：

```bash
ncu --metrics gpu__time_duration.sum ...
```

3. 用 NCU 收集详细指标：

```bash
ncu --section SpeedOfLight \
    --section LaunchStats \
    --section Occupancy \
    --section MemoryWorkloadAnalysis ...
```

4. 用 NSYS 分析端到端开销：

```bash
nsys profile --trace=cuda,osrt ...
```

5. 根据目标选择优化方向：

- 完整函数慢：优先减少分配、拷贝和同步。
- kernel 慢：分析访存、归约、occupancy、block 配置。
- 修改后使用完全相同的测试重新测量。

最重要的是始终区分：

```text
Kernel Duration ≠ 完整函数 Avg Time
```

在当前 Test #9 中：

```text
纯kernel：约8 μs
完整函数：约1.1～1.3 ms
```

两者相差约两个数量级。