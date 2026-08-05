+++
date = '2026-08-05T21:26:13+08:00'
draft = false
title = 'RMSNorm CUDA Kernel 实现与优化'
tags = ['CUDA', 'RMSNorm', 'LLM', '算子优化']
+++

## 1. 背景

RMSNorm（Root Mean Square Layer Normalization）是 LLM 推理中最常用的归一化算子之一，被 LLaMA、Mistral 等主流模型广泛采用。相比 LayerNorm，它去掉了均值计算，只保留缩放，计算量更小，推理更快。

---

## 2. RMSNorm 数学公式

对于输入矩阵 `input[rows, hidden_dim]`，每一行独立计算：

$$
mean\_square = \frac{\sum_{j} input[i, j]^2}{hidden\_dim}
$$

$$
output[i, j] = input[i, j] \times \frac{1}{\sqrt{mean\_square + \epsilon}} \times weight[j]
$$

核心步骤：

1. **计算平方和**：对每行的所有元素求 `x²` 并累加
2. **归一化因子**：`1 / sqrt(mean_square + ε)`
3. **缩放输出**：归一化后乘以可学习的 `weight`

---

## 3. Baseline 实现：一个 Block 负责一行

### 3.1 线程组织

采用 **one block per row** 的策略：

```text
grid  = (rows, 1, 1)        # 每行一个 block
block = (256, 1, 1)          # 每个 block 256 个线程
```

### 3.2 执行流程

```text
Thread 0~255 读取 input 的部分元素
       │
       ▼
计算 local sum(x²)
       │
       ▼
Shared Memory Reduction（块内归约）
       │
       ▼
计算 rsqrt(mean_square + ε)
       │
       ▼
再次读取 input，乘以归一化因子和 weight
       │
       ▼
写回 output
```

### 3.3 为什么需要 shared memory reduction

每个线程只处理部分元素，持有各自的局部 `sum(x²)`。要得到整行的 `mean_square`，需要把所有线程的局部和加起来——这就是 reduction。

```cpp
__shared__ float shared_sum[256];

// 每个线程写入自己的局部和
shared_sum[threadIdx.x] = local_sum;
__syncthreads();

// 块内归约：折半求和
for (int s = blockDim.x / 2; s > 0; s >>= 1) {
    if (threadIdx.x < s) {
        shared_sum[threadIdx.x] += shared_sum[threadIdx.x + s];
    }
    __syncthreads();
}

// 此时 shared_sum[0] 即为整行的 sum(x²)
```

### 3.4 关键代码结构

```cpp
template <typename T>
__global__ void rmsNorm_kernel(
    const T* input, const T* weight, T* output,
    int rows, int hidden_dim, float eps) {

    int row = blockIdx.x;  // 当前处理的行
    int tid = threadIdx.x;

    // 偏移到当前行
    const T* input_row  = input  + row * hidden_dim;
    const T* weight_row = weight;          // weight 在行间共享
    T*       output_row = output + row * hidden_dim;

    __shared__ float shared_sum[256];
    float local_sum = 0.0f;

    // 第一遍：计算 sum(x²)
    for (int i = tid; i < hidden_dim; i += blockDim.x) {
        float val = static_cast<float>(input_row[i]);
        local_sum += val * val;
    }

    shared_sum[tid] = local_sum;
    __syncthreads();

    // Reduction
    for (int s = blockDim.x / 2; s > 0; s >>= 1) {
        if (tid < s) {
            shared_sum[tid] += shared_sum[tid + s];
        }
        __syncthreads();
    }

    // 计算归一化因子
    float mean_square = shared_sum[0] / hidden_dim;
    float rms = rsqrtf(mean_square + eps);

    // 第二遍：写回输出
    for (int i = tid; i < hidden_dim; i += blockDim.x) {
        float val = static_cast<float>(input_row[i]);
        float w   = static_cast<float>(weight_row[i]);
        output_row[i] = static_cast<T>(val * rms * w);
    }
}
```

### 3.5 特点

- 支持 **float** 和 **half**（FP16）
- 使用 shared memory 做块内归约
- 输入被读取**两次**：第一次算平方和，第二次算输出

---

## 4. 测试结果

编译：

```bash
SKIP_ATTENTION=1 make
```

输出：

```text
=== rmsNorm Tests ===

float:
Test #1~13 Passed

half:
Test #1~13 Passed
```

正确性通过，float / half 均正常。

---

## 5. 后续优化方向

当前版本属于 baseline，有几个明确的优化方向。

### 5.1 Warp Shuffle Reduction

当前用 `__shared__` + `__syncthreads()` 做 reduction，256 线程需要多轮同步。

优化：使用 `__shfl_down_sync()` 利用 warp 内线程通信。

```text
thread local sum
       │
       ▼
warpReduce（__shfl_down_sync，寄存器级通信）
       │
       ▼
warp results → 最后的 final sum
```

**优势：**

- 减少 shared memory 使用
- 减少 `__syncthreads()` 同步次数
- warp 内 shuffle 比 shared memory 延迟更低

### 5.2 float4 向量化

当前一次读取一个 `float`：

```cpp
float value = input[index];
```

优化：使用 `float4` 一次加载 4 个 float（128-bit 访存）。

```cpp
const float4* input4 = reinterpret_cast<const float4*>(input);
float4 val = input4[idx];  // 一次加载 4 个元素
```

**优势：**

- 减少 memory instruction 数量
- 提升 global memory throughput
- 与逐元素操作算子的优化思路一致 [[逐元素操作算子]]

### 5.3 half2 优化

当前处理 half 时，一个线程只处理一个 FP16。

优化：使用 `half2` 一次加载两个 FP16（也是 32-bit，刚好一个 float 的宽度）。

```cpp
half2 val = input_half2[idx];
float2 fval = __half22float2(val);  // 转为 float2 计算
```

**适用场景：** LLM FP16 推理、Transformer kernel。

### 5.4 减少 Global Memory 访问

当前 input 被读取了**两次**（第一次算平方和，第二次算输出）。优化方向：

- **Shared memory cache**：第一次读取时把 input 缓存到 shared memory，第二次从 shared memory 读
- **Register cache**：如果 `hidden_dim` 不大，可以一次性读入寄存器

**权衡：** shared memory 容量有限（每 SM 约 100KB），`hidden_dim` 大时需要用 grid-stride loop 分块处理。

---

## 6. 总结

| 项目 | 说明 |
|------|------|
| 数学原理 | `x → x / rms(x) * weight`，去掉均值，只保留缩放 |
| 线程策略 | one block per row，256 threads |
| 归约方式 | shared memory reduction |
| 当前状态 | float/half 正确性通过 |
| 后续优化 | warp shuffle → float4 → half2 → 减少重复访存 |
