# RDMA 编程元素与流程详解 — 设计文档

## 概述

一篇面向 RDMA 研发人员的深度文章，系统梳理 RDMA (libibverbs) 编程的核心元素和完整编程流程。作为 RDMA 系列中承上启下的一篇，从概念到代码的桥梁。

## 目标读者

- RDMA 相关研发人员
- 需要深度内容，不设基础入门篇幅

## 文章结构

```
1. 全景关系图（drawio）
2. 编程元素（9 个元素，卡片式介绍）
3. 编程流程（7 个步骤）
4. 精简代码示例（RC Send-Recv, ~60-80 行）
5. 总结
```

### 1. 全景关系图

用 drawio 绘制一张 RDMA 编程对象关系图，展示所有元素的依赖关系。

关键映射：
- `ibv_device` → `ibv_pd`（设备创建保护域）
- `ibv_pd` → `ibv_mr`（保护域内注册内存）
- `ibv_pd` → `ibv_cq`（保护域关联完成队列）
- `ibv_pd` → `ibv_qp`（保护域创建队列对）
  - `ibv_qp` → SQ (Send Queue) → `ibv_send_wr` + `ibv_sge`
  - `ibv_qp` → RQ (Receive Queue) → `ibv_recv_wr` + `ibv_sge`
  - `ibv_qp` → QPN (Queue Pair Number)
- `ibv_pd` → `ibv_srq`（可选，多个 QP 共享接收队列）
- `ibv_pd` → `ibv_ah`（UD 模式下的地址句柄）
- `ibv_cq` → `ibv_wc`（从完成队列轮询完成事件）

### 2. 编程元素篇（9 个元素）

每个元素控制在 4-6 个要点：定义、关键字段、创建/销毁 API、典型用法。

| # | 元素 | 数据结构 | 要点 |
|---|------|---------|------|
| 2.1 | Device & Context | `ibv_device`, `ibv_context` | 物理设备的软件表示，`ibv_get_device_list()` / `ibv_open_device()` / `ibv_query_device()` |
| 2.2 | PD | `ibv_pd` | 资源隔离边界，`ibv_alloc_pd()` / `ibv_dealloc_pd()` |
| 2.3 | MR | `ibv_mr` | 内存注册（pin pages），`ibv_reg_mr()` / `ibv_dereg_mr()`，lkey/rkey 机制 |
| 2.4 | CQ | `ibv_cq` | 完成通知通道，`ibv_create_cq()` / `ibv_destroy_cq()`，`ibv_poll_cq()` |
| 2.5 | QP | `ibv_qp` | **核心元素**。SQ + RQ 组成队列对，QPN 唯一标识。`ibv_create_qp()` / `ibv_destroy_qp()`。RC/UC/UD 类型。重点说明 SQ 和 RQ 的职责分离 |
| 2.6 | SRQ | `ibv_srq` | 多个 QP 共享 RQ，减少预先 posted receive 数量，`ibv_create_srq()` |
| 2.7 | AH | `ibv_ah` | UD 模式下描述目标地址（GID + QPN），`ibv_create_ah()` |
| 2.8 | WR & SGE | `ibv_send_wr`, `ibv_recv_wr`, `ibv_sge` | WR：描述一次操作请求；SGE：描述数据缓冲区位置 |
| 2.9 | WC | `ibv_wc` | 操作完成状态，`ibv_poll_cq()` 获取，包含 status / opcode / QPN 等信息 |

### 3. 编程流程篇（7 步）

```
Step 1: 设备初始化
  ibv_get_device_list() → ibv_open_device() → ibv_query_device()

Step 2: 创建 PD & 注册 MR
  ibv_alloc_pd() → ibv_reg_mr()

Step 3: 创建完成通道 & CQ
  ibv_create_comp_channel() → ibv_create_cq()

Step 4: 创建 QP
  ibv_create_qp() → 指定 qp_type (RC/UC/UD)

Step 5: QP 状态转换 ← 核心难点
  RESET → INIT → RTR → RTS
  每种转换的 ibv_modify_qp() 参数，RC/UC/UD 差异
  使用 drawio 绘制 QP 状态机图

Step 6: 提交 WR & 轮询 WC
  ibv_post_recv() → ibv_post_send() → ibv_poll_cq()

Step 7: 资源清理（逆序）
```

重点篇幅分配：
- Step 5 (QP 状态转换) 最多，包含状态机图
- Step 6 (WR & WC) 配合代码展示
- 其余步骤简洁说明关键 API

#### QP 状态转换（全文核心）

| 转换 | modify_qp attr_mask | 关键属性 | 说明 |
|------|--------------------|---------|------|
| RESET → INIT | IBV_QP_STATE + IBV_QP_PORT + IBV_QP_PKEY_INDEX | port_num, pkey_index | 指定绑定的端口 |
| INIT → RTR | IBV_QP_STATE + IBV_QP_AV + IBV_QP_PATH_MTU + IBV_QP_DEST_QPN + IBV_QP_RQ_PSN | ah_attr, dest_qpn, rq_psn, path_mtu | 配置对端信息。RC/UC 需要完整参数，UD 只需 dest_qpn |
| RTR → RTS | IBV_QP_STATE + IBV_QP_SQ_PSN + IBV_QP_TIMEOUT + IBV_QP_RETRY_CNT + IBV_QP_RNR_RETRY + IBV_QP_MAX_QP_DEST_RD_ATOMIC + IBV_QP_MIN_RNR_TIMER | sq_psn, timeout, retry_cnt | 激活发送队列。RC 需要所有参数，UD 只需 sq_psn |

### 4. 精简代码示例

RC 可靠连接 Send-Recv 通信示例，覆盖全部 7 步。控制篇幅在 60-80 行。

格式：关键行 + 说明文字穿插，不留大段无注释代码。

包含：两端初始化 → 交换 QPN → 连接 → 收发 → 清理。

### 5. 总结（3 个要点）

1. RDMA 编程的核心是绕开内核——Userspace 直通硬件，所有元素设计都围绕这个目标
2. QP 状态机是最大的心智负担——RTR/RTS 参数填写是新手最容易出错的地方
3. 资源有严格的依赖顺序——创建正向，销毁反向，不能乱

## 视觉元素

1. **RDMA 编程对象关系图** — drawio 绘制，放在开篇第 1 节
2. **QP 状态机图** — drawio 绘制，放在 Step 5

## 篇幅控制

- 元素篇：每个元素 300-500 字，总计不超 4500 字
- 流程篇：Step 5 可 1000 字，其余步骤 200-400 字，总计不超 2500 字
- 代码示例：60-80 行
- 全文预计：8000-10000 字（含代码）

## 写作风格

- 延续 `rocev2-packet-format.md` 的风格：专业、简洁、有深度
- 中文为主，API 名称和数据字段使用英文原名
- 不使用 emoji
- 表格/代码块/小标题作为主要排版手段
