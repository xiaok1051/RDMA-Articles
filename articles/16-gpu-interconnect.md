# GPU 互联深度解析：NVLink、NVSwitch、PCIe 与 CUDA

## 1. 问题：算力在涨，互联在拖后腿

先看一组数字：

- NVIDIA H100 FP16 Tensor Core 算力：~2000 TFLOPS
- HBM3 显存带宽：3.35 TB/s
- PCIe 5.0 x16 带宽：64 GB/s

**算力与 PCIe 带宽的比值约为 30000:1。** GPU 一秒钟能算 2000 万亿次，但要把算完的数据送出去，PCIe 这条管子只有 64 GB/s。就算走 NVLink 4.0（900 GB/s 双向），跨 GPU 访问仍然比本地 HBM 慢将近 4 倍。

这不是理论推演。在 GPT-3 级别的分布式训练中，梯度同步（AllReduce）占用总训练时间的 20-40%，这个比例随着 GPU 数量增加而上升。用 Amdahl's Law 简单推导：如果通信占 30%，即使计算部分无限加速，整体最多加速 3.3 倍。

PCIe 带宽的演进速度也追不上算力：

| PCIe 代际 | 单 lane 带宽 | x16 带宽 | 代表年份 |
|-----------|-------------|---------|---------|
| Gen3 | 8 GT/s | 16 GB/s | 2010 |
| Gen4 | 16 GT/s | 32 GB/s | 2017 |
| Gen5 | 32 GT/s | 64 GB/s | 2019 |
| Gen6 | 64 GT/s | 128 GB/s | 2022 |

每代翻倍，但 GPU 算力每代涨 3-5 倍（V100 120 TFLOPS → A100 624 TFLOPS → H100 2000 TFLOPS）。PCIe 永远在追，永远追不上。

NVIDIA 不是没看到这个瓶颈。他们的对策就是 **NVLink** 和 **NVSwitch**——一套完全独立于 PCIe 的私有互联体系。这篇文章从 GPU 内部的存储层级出发，逐层讲清楚 PCIe、NVLink、NVSwitch 的硬件原理，以及 CUDA 如何用 P2P、Unified Memory、Stream 等 API 驾驭这些硬件。

## 2. GPU 内部架构：数据搬运为什么是瓶颈

先看 GPU 内部的数据是怎么流动的。

![GPU 内部架构](../images/16-gpu-internal-arch.png)

H100 基于 GH100 GPU，包含 132 个 SM（Streaming Multiprocessor），每个 SM 内含 128 个 FP32 CUDA Core 和 4 个第四代 Tensor Core。这些计算单元需要源源不断的数据供给，而数据来自一个层级化的存储体系：

| 存储层级 | 容量（per GPU） | 带宽（per GPU） | 延迟 |
|---------|---------------|---------------|------|
| Register File | 256KB / SM | ~100 TB/s (aggregate) | ~0 cycles |
| L1 / Shared Memory | 256KB / SM | ~33 TB/s (aggregate) | ~30 cycles |
| L2 Cache | 50 MB | ~12 TB/s | ~200 cycles |
| HBM3 | 80 GB | **3.35 TB/s** | ~300-500 cycles |
| NVLink 4.0 | — | **900 GB/s** (双向) | ~1-3 μs |
| PCIe 5.0 x16 | — | **64 GB/s** | ~10 μs |

关键数字：**HBM 带宽是 PCIe 的 52 倍**（3.35 TB/s vs 64 GB/s）。这意味着 GPU 访问本地显存的速度，比通过 PCIe 访问另一块 GPU 的显存快 50 倍以上。就算走 NVLink，差距也是 3.7 倍。

这就是互联带宽成为瓶颈的根本原因：**计算单元离数据越远，带宽断崖式下跌。**

HBM3 的物理结构值得一提——它不是普通的 DRAM 芯片，而是 8-12 层 DRAM die 垂直堆叠，通过 TSV（Through-Silicon Via）穿孔连接，再通过硅中介层（Silicon Interposer）与 GPU die 互联。5 个 HBM3 Stack 沿 GPU die 的上下边缘排列，每个 Stack 提供 1024-bit 的内存接口。NVLink PHY 同样沿 die 边缘排列——18 个 port，每个 50 GB/s，共 900 GB/s 双向。

所以 GPU die 的边缘同时排列着 HBM PHY 和 NVLink PHY，两者共享 die perimeter。die 内部则是 132 个 SM 和 50MB L2 Cache。SM 通过 Crossbar 网络访问 L2，L2 再路由到 HBM 或 NVLink/PCIe。

这个物理约束决定了一件重要的事：**NVLink port 数量受 die 边缘面积限制。** H100 有 18 个 NVLink port，这不是随便选的——die 的四条边要同时容纳 HBM PHY（占两条边的大部分）、NVLink PHY（占两条边的剩余部分）和 PCIe Controller（占一小段）。NVLink port 越多，留给 HBM 的 die edge 就越少，这是一个零和博弈。

理解了 GPU 内部的数据瓶颈，接下来我们看数据出了 GPU die 之后，经过哪些路径到达其他 GPU 或网络。
