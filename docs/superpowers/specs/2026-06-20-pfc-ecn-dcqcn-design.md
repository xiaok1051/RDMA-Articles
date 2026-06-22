# PFC/ECN/DCQCN 深度文章 — 设计文档

> 目标读者：资深开发者/研究员
> 范围：PFC + ECN + DCQCN 三大机制全覆盖
> 风格：原理深度 + 内核源码分析 + 调优实践

## 文章架构：分层解构式

总体结构：从物理到算法，逐层拆解，每层独立深入并穿插源码分析，最后讨论交互与调优。

---

## 第1章：引言 — RoCEv2 为什么需要拥塞控制

### 1.1 从 RoCEv2 的"信任"说起
- IB 传输层假设底层是可信的无损网络
- RoCEv2 架在 UDP/Ethernet 之上，而 Ethernet 本质是尽力而为（best-effort）
- 丢包对 RDMA 的性能打击：RC QP 重传导致延迟飙升、吞吐坍塌
- 核心矛盾：RDMA 需要无损，Ethernet 天然有损

### 1.2 拥塞的三种后果
- 吞包 → RC QP 重传 → 延迟飙升
- PFC 风暴 → HoL blocking → 无辜流被牵连
- 吞吐坍塌 → incast 场景

### 1.3 三件套的分工预览
- **PFC**：局部止血（逐跳背压，防止缓冲区溢出）
- **ECN**：早期预警（交换机标记拥塞，通知端侧）
- **DCQCN**：精准治疗（端到端速率控制，动态调整发送速率）

### 1.4 一条 Rocket 流穿过网络的旅程
- 以一个实际 RDMA WRITE 请求为例，跟踪它经历的每个拥塞控制环节
- 从发出 → 交换机排队 → ECN 标记 → CNP 返回 → DCQCN 降速 → PFC 触发全过程

---

## 第2章：PFC — 逐跳流量控制

### 2.1 802.1Qbb 协议细节
- 8 个优先级各自独立暂停机制
- PAUSE 帧格式：DA/SA/opcode/time vector
- X-off/X-on 阈值与缓冲区设计原理
- PFC 配置：如何将 RoCE 流量映射到指定优先级（通常优先级 3）

### 2.2 PFC 在 RoCEv2 中的角色
- PFC 作为 RoCEv2 实现"无损"的基石
- DCB（Data Center Bridging）配置：DCBX 协议协商
- `dcbtool` / `mlnx_qos` 等工具配置示例

### 2.3 PFC 的阴暗面
- **HoL blocking（头阻阻塞）**：一条慢流导致同优先级队列头部阻塞，影响后续所有流
- **PFC 死锁**：环路中的 A 等 B、B 等 A，互相 pause 形成死锁，因果链分析
- **PFC 风暴**：瞬间大量 PAUSE 帧，控制面冲击
- **PFC watchdog**：检测长时间 pause，超时后重置链路
- **公平性问题**：一条流触发 pause，所有同优先级流被无辜牵连

### 2.4 PFC 相关内核源码
- Linux DCB 协议栈：`net/dcb/`
- Mellanox 驱动中的 PFC 处理：`drivers/net/ethernet/mellanox/mlx5/`
- PFC 计数器读取路径

### 2.5 监控与调试
- `ethtool -S <iface>` 中 PFC 相关计数器的解读
- `mlnx_perf` 的 PFC 监控
- 如何判断 PFC 是否在影响性能（counter 非零 = 出问题了）

---

## 第3章：ECN — 端到端拥塞信号

### 3.1 ECN 基础（RFC 3168 + 802.1Qau）
- IP 头 ECT(0)/ECT(1)/CE 码点含义
- 交换机标记规则：队列长度超过阈值 → 将 ECT 设置为 CE
- ECN 在 RoCEv2 中的特殊实现：**CNP（Congestion Notification Packet）**
- CNP 报文格式：IB OpCode=0x81, 目标 QP, FECN/BECN 标志
- CNP 的生成和消费路径

### 3.2 ECN 标记算法
- RED/ECN 集成原理：队列长度 → 标记概率映射
- 标记阈值（Kmin, Kmax）的设计与对性能的影响
  - Kmin 太小 → 过早标记，吞吐浪费
  - Kmax 太大 → 标记太晚，PFC 已先触发
- 瞬时队列 vs 平均队列标记的对比

### 3.3 交换机侧实现
- Mellanox Spectrum：ECN 配置方式（`ecn_spectrum.py`）
- Broadcom Trident3/4：ECN 标记模式差异
- Cisco Nexus：ECN 配置与限制
- 硬件计数器解读：`ecn_marked_counter` / `ecn_marked_packets`

### 3.4 ECN 的局限
- 标记太晚 → PFC 已先触发 → ECN 信号被跳过
- 标记太早 → 吞吐无谓损失
- CNP 本身丢包 → 接收端误判，DCQCN 收敛更慢

---

## 第4章：DCQCN — 端到端速率控制

### 4.1 设计目标与约束
- 基于 ECN 反馈 + 速率处理器（Rate Processor）
- 核心目标：高吞吐 + 低延迟 + 公平 + 快速收敛
- 设计演变：DCTCP → QCN → DCQCN

### 4.2 DCQCN 状态机
- **CNP 接收事件** → α 更新 → Rate 乘性减
- **无 CNP 超时** → Rate 快速恢复（Fast Recovery）
- **字节计数器超时** → Rate 加性增（Additive Increase）
- 状态转换图（三种事件驱动速率变化）

### 4.3 α 滤波器深度分析
- α 是滑动拥塞因子：`α = (1-g) × α + g × Fb`
  - Fb = 1 （收到 CNP）/ Fb = 0 （未收到 CNP）
  - g 是滤波器增益：高 g → 快速响应 / 低 g → 稳定不易振荡
- 速率调整公式：
  - 乘性减：`rate = rate × (1 - α/2)`
  - 加性增：`rate = rate + γ × MTU`
- 快速恢复：`rate = rate + β × rate_limiter`（β 是恢复因子）

### 4.4 速率收敛性分析
- AIMD 竞争平衡原理
- 多流公平收敛的分析
- 与标准 TCP AIMD 的对比
- 收敛速度与稳定性的权衡

### 4.5 Linux 内核源码分析
- `drivers/infiniband/hw/mlx5/qp.c`：`dcqcn_update_rate()` 函数
- 关键数据结构：`mlx5_rate_limit` / `mlx5_qp` 中的速率字段
- CNP 接收路径 → 速率调整的完整调用链
- 参数寄存器配置：`MLX5_SET(dcqcn_params, ...)`

### 4.6 参数调优指南
- g（滤波器增益）：推荐初始值 1/16 ~ 1/128
- β（快速恢复因子）：推荐初始值 50%
- γ（加性增因子）：推荐初始值 1 ~ 2 MTU
- 常见配置错误与症状（如 g 太大导致速率剧烈振荡）

---

## 第5章：三者的交互与冲突 — 这才是核心难点

### 5.1 PFC vs ECN 的响应时间竞赛
- 谁的阈值先到谁先触发
- 交换机缓冲区分配：为 PFC 预留 vs 为 ECN 标记预留
- 缓冲区大小、ECN 阈值、PFC 阈值三者的数值关系

### 5.2 典型问题分析
- **PFC 风暴掩盖 ECN**：交换机缓冲区还没到 ECN 阈值就已发 PAUSE
- **ECN 标记滞后**：ECN 阈值设得太高 → PFC 早于 ECN → DCQCN 失去作用
- **参数配置不当导致的吞吐振荡**：DCQCN 降速后 PFC 解除 → DCQCN 升速 → PFC 再次触发

### 5.3 调优策略
- 缓冲区分层设计：headroom 用于 PFC，shared pool 用于 ECN 标记
- 推荐的阈值关系：ECN_Kmin < ECN_Kmax < PFC_Xoff
- 实际案例：incast 32:1 场景下的行为分析

---

## 第6章：调优实践与前沿方向

### 6.1 参数配置清单
| 组件 | 参数 | 推荐范围 | 说明 |
|------|------|----------|------|
| PFC | 优先级映射 | RoCE→3 | DCB 配置 |
| PFC | headroom 大小 | 取决于 RTT | 防止丢包 |
| PFC | watchdog 超时 | 参考厂商建议 | 防止死锁 |
| ECN | Kmin | 缓冲区总大小 × 20%~40% | 标记起始阈值 |
| ECN | Kmax | 缓冲区总大小 × 60%~80% | 标记概率=1 的阈值 |
| DCQCN | g | 1/16 ~ 1/128 | 滤波器增益 |
| DCQCN | β | 50% | 快速恢复因子 |
| DCQCN | γ | 1~2 MTU | 加性增步长 |

### 6.2 监控与问题排查
- `ethtool -S`：PFC 帧计数、ECN 标记计数
- `mlnx_perf --pfc` / `mlnx_perf --ecn`
- `collectl --socket`：网络延迟和流量监控
- 典型场景的诊断流程：
  1. PFC counter 非零？→ 检查 PFC 触发原因
  2. ECN 标记过多？→ 检查标记阈值
  3. 吞吐不稳定？→ 检查 DCQCN 参数

### 6.3 前沿方向
- **HPCC（High Precision Congestion Control）**：利用 INT（In-band Network Telemetry）精确获取队列信息
- **INT 辅助拥塞控制**：比 ECN 的单比特信号更精确
- **DCQCN vs TIMELY vs HPCC 对比**：
  - DCQCN：基于 ECN，信号粗粒度
  - TIMELY：基于 RTT，无需交换机支持
  - HPCC：基于 INT，精度最高但需硬件支持
- **无损网络替代方案：RoCEv2 over lossy + 重传优化**
  - Google 的 Swift 拥塞控制
  - 用轻量级重传替代 PFC，消除 HoL blocking 问题

---

## 附录

### A1. 术语表
- CNP: Congestion Notification Packet
- DCQCN: Data Center Quantized Congestion Notification
- ECN: Explicit Congestion Notification
- FECN/BECN: Forward/Backward ECN
- HoL: Head-of-Line blocking
- PFC: Priority Flow Control (802.1Qbb)
- RP: Rate Processor

### A2. 参考文献
- IEEE 802.1Qbb (PFC)
- IEEE 802.1Qau (QCN)
- RFC 3168 (ECN in IP)
- DCQCN: "Congestion Control for Large-Scale RDMA Deployments" (SIGCOMM 2015)
- HPCC: "High Precision Congestion Control" (SIGCOMM 2019)
- TIMELY: "RTT-based Congestion Control for the Datacenter" (SIGCOMM 2015)
- Linux kernel: drivers/infiniband/hw/mlx5/qp.c

### A3. 配图清单
1. RoCEv2 拥塞控制三件套架构图（PFC/ECN/DCQCN 关系）
2. PFC PAUSE 帧交互时序图
3. ECN 标记流程示意图
4. DCQCN 状态机转换图
5. 三条 Rocket 流的拥塞控制旅程图
6. 缓冲区阈值分层示意图（ECN/PFC 阈值关系）
