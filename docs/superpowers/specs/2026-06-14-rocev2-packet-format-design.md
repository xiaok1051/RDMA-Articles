# IB / RoCEv2 报文格式深度解析 — 文章设计

## 概述

一篇面向 RDMA 开发者和网络协议爱好者的技术文章，系统解析 RoCEv2 报文格式，从协议栈封装到底层每个字段的含义，最后用抓包验证收尾。

## 目标读者

- 已有 RDMA 基础（写过 libibverbs 代码），想深入理解 wire format 的开发者
- 需要排查 RDMA 网络问题、需要读懂抓包文件的工程师

## 文章结构

### 1. RoCEv2 协议背景与定位
写作意图：建立上下文，解释 RoCEv2 在整个 RDMA 协议族中的位置。

- RDMA 三种协议对比（IB / RoCEv1 / RoCEv2）
  - IB：私有链路层，需要专用 HCA 和交换机
  - RoCEv1：EtherType 0x8915 直接封装，不可路由
  - RoCEv2：UDP 封装（端口 4791），可路由，生态最广
- 协议栈分层图（Ethernet → IP → UDP → IB）
- 文章范围说明（仅 RoCEv2，IPv4 场景）

### 2. 逐层报文格式解析
核心章节，逐层拆解，每层包含字段表 + 关键说明。

#### 2.1 外层封装（Ethernet / IP / UDP）
- Ethernet: MAC 地址，EtherType=0x0800(IPv4)
- IP: Protocol=17(UDP)，20 字节无选项
- UDP: DstPort=4791，SrcPort 随机高口

#### 2.2 GRH (Global Routing Header) — 40 字节
- 格式来源：借用了 IPv6 路由头格式
- 字段拆解：IPver=6, TClass, FlowLabel, PayloadLen, NextHdr=0x1B, HopLimit, SGID, DGID
- GID 的 IPv4 映射格式（::ffff:a.b.c.d）
- 为什么 RoCEv2 必须带 GRH（全局可路由、替代 LRH）
- 配图：GRH 字段结构图

#### 2.3 BTH (Base Transport Header) — 12 字节
- 字段拆解：OpCode, SE+M+PadCnt+TVEr, P_Key, Dest QP, PSN
- OpCode 完整表格：RdmaReq/RdmaResp, Send/Write/Read/Atomic, First/Middle/Last/Only
- PadCnt 和 4 字节对齐机制
- P_Key 隔离语义（类比 VLAN）
- PSN 的初始化和递增规则
- 配图：BTH 字段结构图

#### 2.4 扩展传输头 (Extended Transport Headers)
- DETH (4B)：UD 专属，Q_Key 字段
- RETH (16B)：RDMA WRITE/READ，VA+R_Key+DMA Length
- AETH (4B)：ACK/Read Response，Syndrome+MSN，流控语义
- AtomicEth (28B)：FetchAdd/CmpSwap
- OpCode → 扩展头映射表

#### 2.5 Payload 与 Padding
- 4 字节对齐规则
- PadCnt 编码

#### 2.6 ICRC (Invariant CRC)
- CRC-32C 算法
- Invariant 覆盖范围（哪些字段被 CRC 保护，哪些不保护）
- 双层 CRC（ICRC + FCS）的意义

### 3. 各 QP 类型报文对比
用表格对比 RC/UC/UD 在报文结构上的差异：

| 维度 | RC | UC | UD |
|------|-----|-----|-----|
| 扩展头 | RETH/AETH/Atomic | 无 | DETH |
| ACK | 有（AETH） | 无 | 无 |
| 重传 | 硬件自动 | 无 | 无 |
| 报文结构 | 完整 | 最简 | 带 DETH |

BTH 中 SE 位在不同 QP 类型下的行为差异。

### 4. 抓包验证
- SoftRoCE 环境 tcpdump 抓包
- Wireshark 逐字段解读 SEND / WRITE / READ 的 Request + Response/ACK
- 把实际字节流和理论格式对应起来

### 5. 总结

## 配图清单

1. RoCEv2 报文完整封装结构（已画 ✅）
2. GRH 字段结构图（已画 ✅）
3. BTH 字段结构图（已画 ✅）
4. 扩展传输头结构图（已画 ✅）
5. OpCode → 扩展头映射表（已画 ✅）

## 参考来源

- IBTA Specification Vol 1 (Release 1.5)
- Linux kernel rdma_rxe 驱动源码
- Wireshark dissection: infiniband.c / librdmawireshark
- rdma-core 源码

## 风格

- 延续用户已有文章的 markdown 风格
- 中文为主，专业术语保留英文（首次出现标注中译）
- 字段表用 markdown 表格
- 关键点用 💡 标注
- 配 wire 格式 ASCII 图 + drawio 结构图
