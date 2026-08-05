+++
date = '2026-07-08T17:33:53+08:00'
draft = false
title = '逐元素操作算子'
+++

## 1. 当前线程在一维线程队列里的全局下标

```cpp
int idx = blockIdx.x * blockDim.x + threadIdx.x;
```

对于 block 0，`blockIdx.x = 0`，公式含义：

> **全局编号 = 前面所有 block 的线程数 + 当前 block 内的线程编号**

---

## 2. Kernel 实现：从 Naive 到向量化

### 2.1 v1：Naive 版本

每个线程只处理一个元素。当 `N` 超过 `gridDim.x * blockDim.x` 时，超出部分的元素不会被处理。

```cpp
__global__ void add_kernel_v1(const float* A, const float* B, float* C, int N) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    if (idx < N) {
        C[idx] = A[idx] + B[idx];
    }
}
```

**问题：** 线程总数固定，无法处理任意大小的 N。

### 2.2 v2：Grid-Stride Loop

用固定数量的 block 处理任意 N：每个线程以 `stride` 为步长跳跃处理多个元素。

```cpp
__global__ void add_kernel_v2(const float* A, const float* B, float* C, int N) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    int stride = gridDim.x * blockDim.x;  // 所有线程总数

    for (int i = idx; i < N; i += stride) {
        C[i] = A[i] + B[i];
    }
}
```

**核心思想：**

```
线程 0: 处理元素 0, 4, 8, 12, ...
线程 1: 处理元素 1, 5, 9, 13, ...
线程 2: 处理元素 2, 6, 10, 14, ...
线程 3: 处理元素 3, 7, 11, 15, ...
```

> **Grid-Stride Loop 的价值**不是让单个 add 更快，而是让 kernel 写法更通用——可以用固定数量的 block 处理任意 N，同时保证访存连续。

### 2.3 v3：float4 向量化加载与存储

即使使用了 Grid-Stride Loop，每个线程每次循环依然只处理 **1 个 float**。GPU 显存总线位宽通常是 **32 字节**（或更高），硬件有能力在一次事务中搬运更多数据。

CUDA 提供内置向量类型：`float2`、`float4`、`double2` 等。一个 `float4` 包含 4 个连续的 float，共 16 字节，可将 **4 次 32-bit 访存合并为 1 次 128-bit 访存**。

**注意事项：**
- 原本 `idx` 的计算方式需要换算：以 `float4` 为单位，元素数量变为 `N / 4`
- 需要处理尾部不足 4 的剩余元素

```cpp
__global__ void add_kernel_v3(const float* A, const float* B, float* C, int N) {
    // 向量化：将 float* 按 float4 重新解释
    const float4* A4 = reinterpret_cast<const float4*>(A);
    const float4* B4 = reinterpret_cast<const float4*>(B);
    float4* C4 = reinterpret_cast<float4*>(C);

    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    int stride = gridDim.x * blockDim.x;

    // 以 float4 为单位循环
    int N4 = N / 4;
    for (int i = idx; i < N4; i += stride) {
        float4 a = A4[i];
        float4 b = B4[i];
        float4 c;
        c.x = a.x + b.x;
        c.y = a.y + b.y;
        c.z = a.z + b.z;
        c.w = a.w + b.w;
        C4[i] = c;
    }

    // 处理尾部不足 4 的剩余元素
    int tail_start = N4 * 4;
    for (int i = tail_start + idx; i < N; i += stride) {
        C[i] = A[i] + B[i];
    }
}
```

**float4 的价值：** 提高每个线程的处理粒度，减少循环次数和访存指令数量。

---

## 3. Memory-Bound 特性

GPU 计算能力很强（每秒可达数 TFLOPs），但这个算子的瓶颈不在计算：

- 每做 **1 次加法**，就要从显存搬 **12 字节**数据（读 A、读 B、写 C，各 4 字节）
- 计算本身太简单，GPU 算力根本吃不满
- 真正拖慢速度的是：**数据从显存读入、再写回显存**

因此它是典型的 **memory-bound 算子**（受显存带宽限制的算子）。

衡量 Element-wise kernel 好坏的指标不是 TFLOPs，而是**有没有跑满显存带宽**。

### 性能指标计算

```cpp
bytes_total = 3 * N * sizeof(float);  // 读 A + 读 B + 写 C
bandwidth = bytes_total / time;        // 单位：GB/s
```

---

## 4. 编译

```bash
nvcc add_naive.cu -O3 -arch=sm_86 -o add_naive
```

| 参数 | 含义 |
|------|------|
| `nvcc` | CUDA 编译器 |
| `add_naive.cu` | 源文件 |
| `-O3` | 开启较高优化 |
| `-arch=sm_86` | 针对 RTX 3060 / Ampere 架构编译 |
| `-o add_naive` | 输出可执行文件名 |

---

## 5. 用 nsys 分析 CUDA API 行为

### 5.1 生成 `.nsys-rep`

```bash
nsys profile \
  --trace=cuda,nvtx,osrt \
  --sample=none \
  --force-overwrite=true \
  --stats=true \
  -o add_naive_bench_profile \
  ./add_naive_bench
```

| 参数 | 含义 |
|------|------|
| `nsys profile` | 启动 Nsight Systems profiling |
| `--trace=cuda,nvtx,osrt` | 采集 CUDA API、NVTX、OS runtime |
| `--sample=none` | 不做 CPU 采样，减少干扰 |
| `--force-overwrite=true` | 允许覆盖已有结果 |
| `--stats=true` | 采集后直接生成统计信息 |
| `-o add_naive_bench_profile` | 输出文件名前缀 |
| `./add_naive_bench` | 被分析的程序 |

**生成文件：**

- `add_naive_bench_profile.nsys-rep`
- `add_naive_bench_profile.sqlite`

### 5.2 分析 `.nsys-rep`

查看总体统计：

```bash
nsys stats add_naive_bench_profile.nsys-rep
```

---

## 6. 用 ncu 分析单个 Kernel 性能

`ncu`（Nsight Compute）比 `nsys` 更适合分析单个 CUDA kernel。

### 6.1 查看可用的 metric set

```bash
ncu --list-sets
```

可用的 set：

| Set | 用途 |
|-----|------|
| `basic` | 基础指标 |
| `detailed` | 详细指标 |
| `full` | 完整指标 |
| `nvlink` | NVLink 相关 |
| `pmsampling` | PM 采样 |
| `roofline` | Roofline 分析 |

最常用：`ncu --set basic ...`

### 6.2 基础 Profiling 命令

```bash
ncu --set basic \
    --kernel-name regex:".*add_kernel.*" \
    --launch-skip 1 \
    --launch-count 1 \
    ./add_naive_bench
```

| 参数 | 含义 |
|------|------|
| `--set basic` | 采集基础指标：LaunchStats、Occupancy、SpeedOfLight 等 |
| `--kernel-name regex:".*add_kernel.*"` | 只分析名字包含 `add_kernel` 的 kernel |
| `--launch-skip 1` | 跳过第 1 次 kernel（warm-up） |
| `--launch-count 1` | 只分析 1 次正式 kernel，避免输出太长 |
| `./add_naive_bench` | 被分析程序 |

### 6.3 关键结果（naive kernel 实测）

| 指标 | 数值 |
|------|------|
| Duration | 2.55 ms |
| Memory Throughput | 95.47% |
| DRAM Throughput | 95.47% |
| Compute (SM) Throughput | 24.35% |
| Achieved Occupancy | 75.90% |
| Registers Per Thread | 16 |
| Static Shared Memory | 0 |
| Dynamic Shared Memory | 0 |

---

## 7. 总结

### 性能分析结论

- **DRAM Throughput** 很高（95.47%），接近打满显存带宽
- **SM Throughput** 不高（24.35%），说明不是计算瓶颈
- `add_kernel` 是典型的 **memory-bound 算子**

### 各版本对比

| 版本 | 特点 | 适用场景 |
|------|------|----------|
| v1 Naive | 每个线程处理 1 个元素，实现最简单 | N 不超过线程总数时 |
| v2 Grid-Stride Loop | 固定 block 数处理任意 N，访存连续 | 通用场景，N 可任意大 |
| v3 float4 向量化 | 一次处理 4 个元素，减少访存指令数 | N 较大且对齐良好时 |

### 关键认知

对于简单的 element-wise add，naive 版本已经访存连续、并行度充足、DRAM 吞吐接近上限，所以 **v2/v3 不一定超过 v1**。向量化和 Grid-Stride Loop 的收益在更复杂的逐元素操作（如多个张量混合运算、带系数的线性组合）中会更明显——那时减少访存指令数才能真正转化为带宽节省。
