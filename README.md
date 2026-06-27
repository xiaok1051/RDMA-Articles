# RDMA 技术文章系列

从零开始学习 RDMA 技术，涵盖 RoCEv2 环境搭建、编程入门、协议解析、内核调用栈分析与分布式系统。

## 文章列表

### Phase 1 — 环境搭建

| 编号 | 标题 | 简介 |
|------|------|------|
| 01 | [RoCEv2 环境搭建与测试](./articles/01-rocev2-setup.md) | Apple Silicon macOS 上通过 UTM + Linux SoftRoCE 模拟 RoCEv2 网络 |
| 02 | [RDMA 编程元素与流程详解](./articles/02-rdma-programming-elements.md) | libibverbs 编程元素、对象关系与完整编程流程 |

### Phase 2 — 编程入门

| 编号 | 标题 | 简介 |
|------|------|------|
| 03 | [rdma_hello 代码详细分析](./articles/03-rdma-hello-analysis.md) | 基于 raw libibverbs API 的 Hello World 程序 |
| 04 | [rdma_hello_rw.c 实现分析](./articles/04-rdma-hello-rw-analysis.md) | 基于 libibverbs 的 RDMA Read/Write 程序 |
| 05 | [rdma_hello_cm 代码详细分析](./articles/05-rdma-hello-cm-analysis.md) | 基于 librdmacm (RDMA_PS_IB) 的 Hello World 程序 |

### Phase 3 — 协议与网络

| 编号 | 标题 | 简介 |
|------|------|------|
| 06 | [RoCEv2 报文格式深度解析](./articles/06-rocev2-packet-format.md) | 从 Ethernet 到 ICRC，逐层拆解 RoCEv2 的每一个字节 |
| 07 | [PFC/ECN/DCQCN 深度解析](./articles/07-pfc-ecn-dcqcn.md) | RoCEv2 拥塞控制三件套 — 协议细节到内核源码 |

### Phase 4 — 硬件接口

| 编号 | 标题 | 简介 |
|------|------|------|
| 08 | [RDMA 驱动开发需要的 PCIe 知识](./articles/08-pcie-rdma-driver.md) | RDMA 驱动开发者需要知道的 PCIe 核心知识 |

### Phase 5 — 内核调用栈

| 编号 | 标题 | 简介 |
|------|------|------|
| 09 | [ibv_get_device_list / ibv_open_device](./articles/09-ibv-device-list-open-device.md) | 设备发现与打开流程（以 erdma 网卡为例） |
| 10 | [ibv_alloc_pd](./articles/10-ibv-alloc-pd.md) | Protection Domain 分配流程 |
| 11 | [ibv_reg_mr](./articles/11-ibv-reg-mr.md) | Memory Region 注册流程 |
| 12 | [ibv_create_cq](./articles/12-ibv-create-cq.md) | Completion Queue 创建流程 |
| 13 | [ibv_create_qp](./articles/13-ibv-create-qp.md) | Queue Pair 创建流程 |
| 14 | [ibv_modify_qp](./articles/14-ibv-modify-qp.md) | Queue Pair 状态机迁移流程 |

### Phase 6 — 分布式系统

| 编号 | 标题 | 简介 |
|------|------|------|
| 15 | [NCCL 深度解析：一次 AllReduce 的旅程](./articles/15-nccl-architecture.md) | NCCL 架构、通信原语、Ring AllReduce 与集合操作 |

## 目录结构

```
.
├── articles/          # Markdown 文章
├── images/            # 配图（PNG / SVG / drawio）
├── html/              # 微信公众号排版 HTML
├── docs/superpowers/  # 设计文档与实施计划
└── CLAUDE.md          # 项目协作指南
```

## 阅读建议

1. **新手** — 按 Phase 1 → 2 的顺序，先搭环境再学编程
2. **有 RDMA 基础** — 直接跳至 Phase 3（协议）或 Phase 5（内核栈）
3. **分布式系统开发者** — Phase 2 热身 → Phase 6（NCCL）
4. **RDMA 驱动开发者** — Phase 4 必读，Phase 5 是核心参考

## 关于作者

系列文章由 Kun Wang 撰写，持续更新中。
