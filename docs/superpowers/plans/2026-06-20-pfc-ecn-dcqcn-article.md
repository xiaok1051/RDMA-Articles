# PFC/ECN/DCQCN 深度文章 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 写一篇关于 PFC/ECN/DCQCN 的深度技术文章，面向资深开发者/研究员

**Architecture:** 分层解构式（从物理层到算法层逐层深入），6 章 + 附录，每章穿插 Linux 内核源码分析和调优实践

**Tech Stack:** Markdown（文章正文），draw.io（配图）

**输出文件:**
- 主文章: `pfc-ecn-dcqcn.md`
- HTML 版本: `html/pfc-ecn-dcqcn.html`
- 配图: `images/pfc-*.png`, `images/ecn-*.png`, `images/dcqcn-*.png`

---

### Task 1: 第1章 — 引言：RoCEv2 为什么需要拥塞控制

**Files:**
- Create: `pfc-ecn-dcqcn.md`（章节 1 部分）

- [ ] **Step 1: 撰写 1.1 节 — RoCEv2 的"信任"问题**
  分析 IB 传输层为何假设无损网络，RoCEv2 架在 UDP/Ethernet 上导致的核心矛盾。丢包对 RDMA 性能的具体影响（RC 重传机制、吞吐坍塌的数据）。

- [ ] **Step 2: 撰写 1.2 节 — 拥塞的三种后果**
  吞包 → RC QP 重传 → 延迟飙升
  PFC 风暴 → HoL blocking → 无辜流被牵连
  吞吐坍塌 → incast 场景

- [ ] **Step 3: 撰写 1.3 节 — 三件套分工预览**
  PFC（逐跳背压）/ ECN（交换机标记）/ DCQCN（速率控制）的职责和协作关系

- [ ] **Step 4: 撰写 1.4 节 — 一条 Rocket 流的旅程**
  以 RDMA WRITE 请求为例，跟踪全程：发出 → 交换机排队 → ECN 标记 → CNP 返回 → DCQCN 降速 → PFC 触发

- [ ] **Step 5: 绘制配图 — 三件套架构图**
  使用 draw.io 绘制 PFC/ECN/DCQCN 关系架构图
  输出：`images/pfc-ecn-dcqcn-architecture.png`

---

### Task 2: 第2章 — PFC：逐跳流量控制

**Files:**
- Modify: `pfc-ecn-dcqcn.md`（章节 2 部分）

- [ ] **Step 1: 撰写 2.1 节 — 802.1Qbb 协议细节**
  8×优先级独立暂停机制、PAUSE 帧格式（DA/SA/opcode/time vector）、X-off/X-on 阈值与缓冲区设计

- [ ] **Step 2: 绘制配图 — PFC PAUSE 帧交互时序**
  输出：`images/pfc-pause-sequence.png`

- [ ] **Step 3: 撰写 2.2 节 — PFC 在 RoCEv2 中的角色**
  RoCE 流量映射到优先级 3、DCB 配置、DCBX 协商

- [ ] **Step 4: 撰写 2.3 节 — PFC 的阴暗面**
  HoL blocking 机制与示意图、PFC 死锁因果链、PFC 风暴检测与 watchdog、公平性问题

- [ ] **Step 5: 绘制配图 — HoL blocking 示意图**
  输出：`images/pfc-hol-blocking.png`

- [ ] **Step 6: 撰写 2.4 节 — PFC 相关内核源码**
  Linux DCB 协议栈（`net/dcb/`）、Mellanox 驱动 PFC 处理、PFC 计数器读取路径

- [ ] **Step 7: 撰写 2.5 节 — 监控与调试**
  `ethtool -S` 解读、`mlnx_perf` PFC 监控、问题判断方法

---

### Task 3: 第3章 — ECN：端到端拥塞信号

**Files:**
- Modify: `pfc-ecn-dcqcn.md`（章节 3 部分）

- [ ] **Step 1: 撰写 3.1 节 — ECN 基础（RFC 3168 + 802.1Qau）**
  ECT(0)/ECT(1)/CE 码点、交换机标记规则、CNP 报文格式（OpCode=0x81、FECN/BECN）

- [ ] **Step 2: 绘制配图 — ECN 标记 + CNP 返回流程**
  输出：`images/ecn-marking-flow.png`

- [ ] **Step 3: 撰写 3.2 节 — ECN 标记算法**
  RED/ECN 集成、Kmin/Kmax 阈值设计、瞬时 vs 平均队列对比

- [ ] **Step 4: 撰写 3.3 节 — 交换机侧实现**
  Mellanox Spectrum（`ecn_spectrum.py`）、Broadcom Trident、Cisco Nexus、硬件计数器

- [ ] **Step 5: 撰写 3.4 节 — ECN 局限**
  标记时机困境、CNP 丢包误判

---

### Task 4: 第4章 — DCQCN：端到端速率控制

**Files:**
- Modify: `pfc-ecn-dcqcn.md`（章节 4 部分）

- [ ] **Step 1: 撰写 4.1 节 — 设计目标与约束**
  基于 ECN + RP、DCTCP→QCN→DCQCN 演变

- [ ] **Step 2: 撰写 4.2 节 — DCQCN 状态机**
  CNP 接收 → α 更新 → Rate 乘性减
  无 CNP 超时 → 快速恢复
  字节计数器 → 加性增

- [ ] **Step 3: 绘制配图 — DCQCN 状态机**
  输出：`images/dcqcn-state-machine.png`

- [ ] **Step 4: 撰写 4.3 节 — α 滤波器深度分析**
  α = (1-g)×α + g×Fb、g 的权衡、乘性减 / 加性增 / 快速恢复公式

- [ ] **Step 5: 撰写 4.4 节 — 速率收敛性分析**
  AIMD 竞争平衡、多流公平收敛、与 TCP AIMD 对比

- [ ] **Step 6: 撰写 4.5 节 — Linux 内核源码分析**
  `drivers/infiniband/hw/mlx5/qp.c` 中 `dcqcn_update_rate()`、关键数据结构、调用链

- [ ] **Step 7: 撰写 4.6 节 — 参数调优指南**
  g/β/γ 推荐值、常见配置错误

---

### Task 5: 第5章 — 三者的交互与冲突

**Files:**
- Modify: `pfc-ecn-dcqcn.md`（章节 5 部分）

- [ ] **Step 1: 撰写 5.1 节 — PFC vs ECN 响应时间竞赛**
  阈值先后决定机制、缓冲区预留分析

- [ ] **Step 2: 绘制配图 — 缓冲区阈值分层示意图**
  输出：`images/buffer-threshold-hierarchy.png`

- [ ] **Step 3: 撰写 5.2 节 — 典型问题分析**
  PFC 掩盖 ECN、ECN 滞后、吞吐振荡

- [ ] **Step 4: 撰写 5.3 节 — 调优策略**
  headroom/shared pool 设计、ECN_Kmin < ECN_Kmax < PFC_Xoff、incast 案例

---

### Task 6: 第6章 — 调优实践与前沿方向

**Files:**
- Modify: `pfc-ecn-dcqcn.md`（章节 6 部分 + 附录）

- [ ] **Step 1: 撰写 6.1 节 — 参数配置清单**
  PFC/ECN/DCQCN 参数表和推荐范围

- [ ] **Step 2: 撰写 6.2 节 — 监控与问题排查**
  `ethtool -S`、`mlnx_perf`、`collectl`、典型诊断流程

- [ ] **Step 3: 撰写 6.3 节 — 前沿方向**
  HPCC、INT、DCQCN vs TIMELY vs HPCC 对比、Swift / lossy RoCE

- [ ] **Step 4: 撰写附录**
  术语表、参考文献、配图清单

---

### Task 7: 全文审校与生成 HTML

**Files:**
- Modify: `pfc-ecn-dcqcn.md`（全文）
- Create: `html/pfc-ecn-dcqcn.html`

- [ ] **Step 1: 通读全文，检查一致性**
  术语统一（ECN/CNP/PFC 等）、章节交叉引用正确、数据一致

- [ ] **Step 2: 检查配图是否全部就位**
  对照附录 A3 配图清单逐一确认

- [ ] **Step 3: 生成 HTML 版本**
  使用 pandoc 或手动转换为 HTML，存入 `html/pfc-ecn-dcqcn.html`

- [ ] **Step 4: 最终审阅**
  全文质量检查
