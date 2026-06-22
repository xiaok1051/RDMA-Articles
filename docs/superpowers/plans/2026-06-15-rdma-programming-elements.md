# RDMA 编程元素与流程详解 Implementation Plan

> For agentic workers: This is a content creation plan, executed directly in this session.

**Goal:** Write a ~8000-10000 word Chinese technical article covering RDMA (libibverbs) programming elements and programming flow, with two drawio diagrams.

**Files to create:**
- `images/rdma-object-relationship.png` — drawio export, full object relationship diagram
- `images/rdma-qp-state-machine.png` — drawio export, QP state machine diagram
- `rdma-programming-elements.md` — the article

**Context:** This article is part of an existing RDMA series. Writer is an RDMA expert. Style follows `rocev2-packet-format.md`.

---

### Task 1: Draw the RDMA Programming Object Relationship Diagram (drawio)

- [ ] **Start drawio session**
- [ ] **Draw the diagram** with these elements and arrows:
  - `ibv_device` at top center
  - Arrow: `ibv_device` → `ibv_pd` (创建)
  - `ibv_pd` in center, with arrows radiating to:
    - `ibv_mr` (注册内存, with lkey/rkey annotation)
    - `ibv_cq` (完成队列)
    - `ibv_qp` (队列对)
      - Inside `ibv_qp`: clearly show SQ (Send Queue) and RQ (Receive Queue) as sub-boxes
      - Arrow: SQ → `ibv_send_wr` + `ibv_sge`
      - Arrow: RQ → `ibv_recv_wr` + `ibv_sge`
      - Label: QPN
    - `ibv_srq` (虚线/可选, 共享接收队列)
    - `ibv_ah` (UD模式, 地址句柄)
  - Arrow: `ibv_cq` → `ibv_wc` (轮询完成)
- [ ] **Export** as PNG to `images/rdma-object-relationship.png`

### Task 2: Draw the QP State Machine Diagram (drawio)

- [ ] **Start drawio session** (or reuse)
- [ ] **Draw QP state machine** with states and transitions:
  - States: RESET → INIT → RTR → RTS → (SQD →) SQ → ERR (可选, 简化版只到 RTS)
  - Each transition arrow labeled with the key `ibv_modify_qp()` attr_mask parameters:
    - RESET→INIT: `IBV_QP_STATE | IBV_QP_PORT | IBV_QP_PKEY_INDEX`
    - INIT→RTR: `IBV_QP_STATE | IBV_QP_AV | IBV_QP_PATH_MTU | IBV_QP_DEST_QPN | IBV_QP_RQ_PSN`
    - RTR→RTS: `IBV_QP_STATE | IBV_QP_SQ_PSN | IBV_QP_TIMEOUT | IBV_QP_RETRY_CNT | IBV_QP_RNR_RETRY | IBV_QP_MAX_QP_DEST_RD_ATOMIC | IBV_QP_MIN_RNR_TIMER`
  - Include error transition to ERR state from any state
- [ ] **Export** as PNG to `images/rdma-qp-state-machine.png`

### Task 3: Write the Article

- [ ] **Write** `rdma-programming-elements.md` with the following structure:

**1. 全景关系图** (~300 words)
  - Brief setup explaining the diagram
  - Insert `![RDMA 编程对象关系](images/rdma-object-relationship.png)`
  - Explain the dependency direction

**2. 编程元素** (~4500 words total, ~300-500 per element)
  - 2.1 Device & Context
  - 2.2 Protection Domain (PD)
  - 2.3 Memory Region (MR) — emphasize lkey/rkey mechanism
  - 2.4 Completion Queue (CQ)
  - 2.5 Queue Pair (QP) — emphasize SQ/RQ split, QPN, RC/UC/UD types
  - 2.6 Shared Receive Queue (SRQ)
  - 2.7 Address Handle (AH)
  - 2.8 Work Request & Scatter-Gather Element (WR/SGE)
  - 2.9 Work Completion (WC)

Each element section covers: definition, key data structure fields, create/destroy API, typical usage.

**3. 编程流程** (~2500 words total)
  - 3.1 设备初始化: `ibv_get_device_list()` → `ibv_open_device()` → `ibv_query_device()`
  - 3.2 创建 PD & 注册 MR: `ibv_alloc_pd()` → `ibv_reg_mr()`
  - 3.3 创建 CQ: `ibv_create_cq()` + comp_channel
  - 3.4 创建 QP: `ibv_create_qp()` with qp_init_attr
  - 3.5 QP 状态转换 (核心, ~1000 words): insert QP state machine diagram, explain each transition with code, RC/UC/UD differences
  - 3.6 提交 WR & 轮询 WC: `ibv_post_recv()` → `ibv_post_send()` → `ibv_poll_cq()`
  - 3.7 资源清理: reverse order

**4. 完整代码示例** (~60-80 lines)
  - RC Send-Recv, covers all 7 steps
  - Key lines + interspersed explanations

**5. 总结** (~200 words)

- [ ] **Verify** the article reads well, no placeholder content, consistent style
