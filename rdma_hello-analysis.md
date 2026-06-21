# rdma_hello 代码详细分析

> RoCEv2 RDMA Hello World 程序 — 基于 raw libibverbs API

---

## 1. 架构概要

整个程序围绕 **RDMA 通信的核心流程** 展开，设计为单二进制同时支持 server/client 模式：

```
+-----------------------+          TCP (带外)          +-----------------------+
|  linux01 (server)     |◄────── 交换 QP 信息 ──────►|  linux02 (client)     |
|                       |                              |                       |
|  rxe0 (192.168.64.4)  |◄────── RDMA SEND ─────────►|  rxe_0 (192.168.64.5) |
+-----------------------+                              +-----------------------+
```

**为什么不用 rdma_cm？** SoftRoCE 上 `rdma_bind_addr` 无法绑定到指定 IP（EADDRNOTAVAIL），因为 RXE 设备没有在 rdma_cm 内核模块中注册地址映射。改用 raw ibverbs 配合 TCP 交换连接信息是更通用的做法。

---

## 2. 数据结构

### `struct qp_info` — 通过 TCP 交换的 QP 连接信息

```c
struct qp_info {
    uint16_t      lid;      // LID (Local ID) — RoCEv2 下固定为 0
    uint32_t      qpn;      // QP Number — 远端 QP 的唯一标识
    uint32_t      psn;      // Packet Sequence Number — 初始包序号
    union ibv_gid gid;      // GID (Global ID) — 128 位，RoCEv2 下为 IPv4 映射格式
};
```

`union ibv_gid` 是 128 位的全局标识符。RoCEv2 的 GID index 1 填充为 IPv4 映射格式 `::ffff:192.168.x.x`。

### `struct rdma_res` — 本地 RDMA 资源

```c
struct rdma_res {
    struct ibv_context *ctx;   // 设备上下文（对应一个 RNIC 或 RXE）
    struct ibv_pd      *pd;    // Protection Domain — 保护域，隔离不同应用的资源
    struct ibv_cq      *cq;    // Completion Queue — 完成事件队列
    struct ibv_qp      *qp;    // Queue Pair — 包含 SQ(发送) + RQ(接收)
    struct ibv_mr      *mr;    // Memory Region — 注册给 RDMA 使用的内存区
    char                buf[MSG_SIZE]; // 收发共用缓冲区
};
```

这是 RDMA 编程中最核心的 5 个对象，关系如下：

```
ibv_context (设备)
    └── ibv_pd (保护域)
            ├── ibv_mr (内存注册)
            ├── ibv_qp (队列对)
            │       ├── SQ (Send Queue)  ── 投递 ibv_send_wr
            │       └── RQ (Recv Queue)  ── 投递 ibv_recv_wr
            └── ibv_cq (完成队列) ◄── SQ/RQ 的完成事件汇聚于此
```

---

## 3. 资源管理

### 打开 RXE 设备 — `open_rxe_device()`

```c
static int open_rxe_device(struct ibv_context **ctx)
```

- 调用 `ibv_get_device_list()` 枚举系统上所有 RDMA 设备
- 依次 `ibv_open_device()` 打开，成功即返回第一个可用的设备
- 在 SoftRoCE 环境下，列表包含 `rxe0` 或 `rxe_0`
- 用 `ibv_free_device_list()` 释放设备列表

### 分配资源 — `res_alloc()`

```c
static struct rdma_res *res_alloc(void)
```

按顺序创建 4 个对象，任何一个失败则反向清理：

```
ibv_alloc_pd()    → Protection Domain
ibv_create_cq()   → Completion Queue（16 个槽位）
ibv_reg_mr()      → 注册 buf 到 RDMA 硬件（支持本地写 + 远端写）
```

- `IBV_ACCESS_LOCAL_WRITE` — 允许本端 CPU 写
- `IBV_ACCESS_REMOTE_WRITE` — 允许远端 RDMA 写（SEND/RECV 需要）

### 释放资源 — `res_free()`

逆序销毁，每个指针都做 NULL 检查：

```
ibv_destroy_qp() → ibv_dereg_mr() → ibv_destroy_cq() → ibv_dealloc_pd() → ibv_close_device()
```

### 创建 QP — `create_qp()`

```c
struct ibv_qp_init_attr attr = {
    .send_cq  = r->cq,
    .recv_cq  = r->cq,           // 收发共用同一个 CQ
    .cap = {
        .max_send_wr  = 8,       // 最多 8 个待处理发送 WR
        .max_recv_wr  = 8,       // 最多 8 个待处理接收 WR
        .max_send_sge = 1,       // 每个 WR 最多 1 个 SGE
        .max_recv_sge = 1,
    },
    .qp_type = IBV_QPT_RC,       // Reliable Connection — 可靠连接
};
```

**IBV_QPT_RC** 是最常用的 QP 类型，保证有序、可靠、不重复交付，类似 TCP。

---

## 4. TCP 辅助通道

### `tcp_listen()` / `tcp_accept()` / `tcp_connect()`

标准的 TCP socket 封装，用于在 RDMA 数据传输之前交换 QP 元数据。因为 RDMA 连接建立需要双方互知 QPN、GID、PSN 等参数，而这些信息无法通过 RDMA 链路本身传递（链路还没建好），所以需要一个**带外通道（out-of-band）**。

### `tcp_exchange()`

```c
write(fd, local, sizeof(*local));  // 发送本地 QP 信息
read(fd, peer, sizeof(*peer));     // 接收远端 QP 信息
```

信息交换顺序决定了 connection 的对称性——这里 client 和 server 在 TCP 层面角色不同，但交换的内容是完全对称的。

---

## 5. RDMA 数据收发

### Work Request 机制

RDMA 的核心操作模型：**Work Request → Work Queue → Work Completion**

```
App 调用 ibv_post_send/ibv_post_recv
    ↓ 投递 WR
Send Queue / Recv Queue
    ↓ 硬件执行
ibv_poll_cq 轮询 CQ 获取 Work Completion (WC)
```

### `post_recv()`

```c
struct ibv_sge sge = {
    .addr   = (uintptr_t)r->buf,  // 缓冲区地址
    .length = MSG_SIZE,           // 长度
    .lkey   = r->mr->lkey,        // 内存区域本地 key（注册 MR 时分配）
};
struct ibv_recv_wr wr = {
    .wr_id   = 1,                 // 用户自定义 ID，完成时通过 WC 返回
    .sg_list = &sge,
    .num_sge = 1,
};
ibv_post_recv(r->qp, &wr, &bad);
```

- `lkey` 是 MR 注册后得到的本地标识符，硬件用它来验证内存访问权限
- `wr_id` 在 poll CQ 时原样返回，用来区分是哪个 WR 完成了（本例 send 用 2，recv 用 1）
- SGE (Scatter/Gather Element) 描述了一段内存，RDMA 硬件直接从这段内存读写

### `post_send()`

```c
struct ibv_send_wr wr = {
    .wr_id      = 2,
    .opcode     = IBV_WR_SEND,     // SEND 操作（无 remote key 要求）
    .send_flags = IBV_SEND_SIGNALED, // 完成时生成 CQ 事件
    .sg_list    = &sge,
    .num_sge    = 1,
};
ibv_post_send(r->qp, &wr, &bad);
```

- **IBV_WR_SEND** — 最基础的 RDMA 操作，类似 TCP 的 send。数据从本端 SGE 送到对端预先 post 的 RECV 缓冲区。
- **IBV_SEND_SIGNALED** — 必须设置，否则操作完成后 CQ 上不会生成完成事件，`wait_cq()` 将永远等不到结果。

### `wait_cq()`

```c
for (int i = 0; i < 1000; i++) {
    int n = ibv_poll_cq(r->cq, 1, &wc);
    if (n > 0) {
        if (wc.status == IBV_WC_SUCCESS) return 0;
        // 处理错误...
    }
    usleep(2000);  // 2ms 间隔，总共等 2 秒
}
```

- `ibv_poll_cq` 是**非阻塞**的，返回 0 表示当前没有完成事件
- 轮询间隔 2ms，超时 2 秒（1000 次）
- WC 的 `status` 字段必须为 `IBV_WC_SUCCESS`，否则表示传输失败

---

## 6. QP 状态机

这是整个程序最核心也最难理解的部分。一个 RC QP 必须严格经过以下状态转换：

```
RESET ──► INIT ──► RTR (Ready To Receive) ──► RTS (Ready To Send)
```

### RESET → INIT

```c
attr.qp_state        = IBV_QPS_INIT;
attr.pkey_index      = 0;        // 分区键索引，Ethernet/RoCE 下为 0
attr.port_num        = 1;        // IB 端口号，SoftRoCE 固定为 1
attr.qp_access_flags = IBV_ACCESS_LOCAL_WRITE | IBV_ACCESS_REMOTE_WRITE;
```

INIT 状态允许 QP 接收 post_recv 投递的 WR，但还不能发送或接收网络数据。

### INIT → RTR

```c
attr.qp_state           = IBV_QPS_RTR;
attr.path_mtu           = IBV_MTU_1024;   // 与 ibv_devinfo 的 active_mtu 匹配
attr.dest_qp_num        = peer.qpn;       // 远端的 QP 号
attr.rq_psn             = peer.psn;       // 远端期望的接收 PSN
attr.max_dest_rd_atomic = 1;              // 最大未完成的读/原子操作数
attr.ah_attr.is_global  = 1;              // 需要 GRH (Global Routing Header) — RoCEv2 必须
attr.ah_attr.grh.dgid   = peer.gid;       // 远端 GID
attr.ah_attr.grh.sgid_index = 1;          // 本地 GID index 1 = RoCEv2 IPv4
attr.ah_attr.grh.hop_limit  = 1;
attr.ah_attr.port_num   = 1;
```

**这是最关键的一步。** RTR 状态配置了到对端 QP 的路由信息（GID、QPN）和传输参数（MTU、PSN）。

**GID index 1 为什么是 RoCEv2？**

| Index | GID 格式 | 协议 |
|-------|----------|------|
| 0 | `fe80::34f0:e0ff:fe65:26dd` | RoCEv1 (IPv6 link-local) |
| 1 | `::ffff:192.168.64.4` | **RoCEv2** (IPv4 可路由) |

RoCEv1 用 EtherType 0x8915 封装，不可路由。RoCEv2 用 UDP 端口 4791，可路由。GID index 1 的 IPv4 映射格式明确标识了 RoCEv2。

### RTR → RTS

```c
attr.qp_state   = IBV_QPS_RTS;
attr.timeout    = 14;       // 本地 ACK 超时（4.096μs * 2^timeout ≈ 134ms）
attr.retry_cnt  = 7;        // 重试次数
attr.rnr_retry  = 7;        // RNR (Receiver Not Ready) 重试次数
attr.sq_psn     = local.psn; // 本端 SQ 的初始 PSN
```

RTS 状态表示 QP 已经完全就绪，可以发送和接收 RDMA 数据。

### 状态机总图

```
 Client QP                           Server QP
 ─────────                           ─────────
   RESET                               RESET
     │                                  │
     │ create_qp()                      │ create_qp() (on CONNECT_REQUEST)
     ▼                                  ▼
   INIT ───────────────────────────── INIT
     │                                  │
     │ qp_transition_rtr()              │ qp_transition_rtr()
     ▼                                  ▼
   RTR ────────────────────────────── RTR
     │                                  │
     │ qp_transition_rts()              │ qp_transition_rts()
     ▼                                  ▼
   RTS ───────── RDMA ─────────────── RTS
     │            SEND                  │
     │══════════════════════════════════►│
     │            SEND                  │
     │◄══════════════════════════════════│
```

---

## 7. Server 完整流程

### 步骤分解

```
1. res_alloc()        ── 打开 RXE 设备, 创建 PD/CQ/MR
2. create_qp()        ── 创建 RC QP (处于 RESET 状态)
3. ibv_query_gid()    ── 读取 GID index 1 (RoCEv2)
4. tcp_listen()       ── 监听 TCP 端口 18515
5. tcp_accept()       ── 等待客户端 TCP 连接
6. tcp_exchange()     ── 交换 QP info (QPN, PSN, GID)
7. qp_transition_init() ── RESET → INIT
8. post_recv()        ── 投递 RECV WR (等待客户端消息)
9. qp_transition_rtr()  ── INIT → RTR (配置对端路由)
10. qp_transition_rts() ── RTR → RTS (QP 就绪)
11. wait_cq()         ── 等待 RECV 完成
12. printf("received: %s")  ── 打印客户端消息
13. post_send()       ── 回复 "Hello from server!"
14. wait_cq()         ── 等待 SEND 完成
15. 清理               ── 关闭 TCP/RDMA 资源
```

### 为什么 post_recv 在 RTR/RTS 之前？

post_recv 可以在 INIT 或之后任意状态投递。WR 会挂在 RQ 上，等 QP 进入 RTR 状态后自动生效。**先 post_recv 再转 RTR** 确保当远端数据到达时，接收缓冲区已经就位。

---

## 8. Client 完整流程

### 步骤分解

```
1. res_alloc()        ── 打开 RXE 设备, 创建 PD/CQ/MR
2. create_qp()        ── 创建 RC QP
3. ibv_query_gid()    ── 读取 GID index 1
4. tcp_connect()      ── 连接服务器 TCP 端口
5. tcp_exchange()     ── 交换 QP info
6. qp_transition_init() ── RESET → INIT
7. qp_transition_rtr()  ── INIT → RTR
8. qp_transition_rts() ── RTR → RTS
9. post_recv()        ── 投递 RECV (准备接收服务器的回复)
10. post_send()       ── 发送 "Hello from client!"
11. wait_cq()         ── 等待 SEND 完成 (WC status == SUCCESS)
12. wait_cq()         ── 等待 RECV 完成 (服务器回复)
13. printf("received: %s")  ── 打印服务器回复
14. 清理
```

### 为什么 client 需要 2 次 wait_cq()？

client 先后 post 了 1 个 RECV 和 1 个 SEND（带 SIGNALED 标志），所以 CQ 上会产生 **2 个完成事件**：

| 顺序 | wr_id | opcode | 说明 |
|------|-------|--------|------|
| 1 | 2 (SEND) | SEND | 本端发送完成 |
| 2 | 1 (RECV) | RECV | 对端回复到达 |

先 call wait_cq 拿到 SEND 完成，再 call 一次拿到 RECV 完成。

---

## 9. 与 TCP 的对比

| 概念 | TCP | RDMA (ibverbs) |
|------|-----|-----------------|
| 端点标识 | IP:Port | GID + QPN |
| 连接建立 | connect/accept 握手 | TCP 交换 QP info + 手动 QP 状态机 |
| 发送 | write/send | ibv_post_send (WR 入队) |
| 接收 | read/recv | ibv_post_recv (预先注册缓冲区) |
| 完成通知 | select/epoll 事件 | ibv_poll_cq (轮询 CQ) |
| 数据拷贝 | CPU 参与，内核→用户 | DMA 直接到用户缓冲区 (zero-copy)|
| 可靠性 | TCP 协议层保证 | QP 硬件层保证 (RC 模式) |

---

## 10. MTU 和超时参数

### MTU (Path MTU)

```c
attr.path_mtu = IBV_MTU_1024;
```

`ibv_devinfo` 中 `active_mtu: 1024` 表示两端协商后的 MTU。可选的 MTU 值：256 / 512 / 1024 / 2048 / 4096。RXE 当前协商为 1024。

### Timeout

```c
attr.timeout = 14;
```

本地 ACK 超时 = `4.096μs × 2^timeout`。timeout=14 时约 67ms。如果对端在超时时间内没有回复 ACK，QP 会触发重试或进入 error 状态。

### Retry Count

```c
attr.retry_cnt = 7;
attr.rnr_retry = 7;
```

- `retry_cnt` — 发送端重试次数（7 次）
- `rnr_retry` — 对端 RNR (Receiver Not Ready) 时的重试次数（7 = 无限）

---

## 11. 错误处理模式

程序使用 **goto + out 标签** 的统一清理模式：

```c
int ret = 1;
// ... 初始化资源 ...
if (some_error) {
    perror("...");
    goto out;  // 跳转到统一的清理代码
}
// ... 正常处理 ...
ret = 0;
out:
    // 销毁所有已分配的资源
    return ret;
```

优点：
- 避免每处错误都写一遍 cleanup
- 资源按分配顺序逆序释放
- `ret = 1`（失败）/ `0`（成功）作为返回值

---

## 12. 编译和运行

### 编译

```bash
gcc -Wall -Wextra -o rdma_hello rdma_hello.c -libverbs
```

- `-libverbs` — 链接 InfiniBand verbs 库（包含 ibv_* API）
- 不需要 `-lrdmacm`（本程序不使用 rdma_cm）

### 运行

```bash
# 服务端（linux01）
./rdma_hello -s -p 18515

# 客户端（linux02）
./rdma_hello -c 192.168.64.4 -p 18515
```

### 预期输出

```
=== server (linux01) ===                              === client (linux02) ===
server: opening RDMA device...                        client: opening RDMA device...
  device: rxe0                                          device: rxe_0
server: waiting for TCP connection on port 18515...   client: connecting to 192.168.64.4:18515...
  tcp: client from 192.168.64.5:54198                 client: exchanging QP info...
server: exchanging QP info...                           local QPN: 17, peer QPN: 33
  local QPN: 33, peer QPN: 17                         client: QP connected
server: QP connected                                  client: send completed
server: received: Hello from client!                  client: received: Hello from server!
server: reply sent
```

---

## 13. 扩展阅读

- [RDMA Core 官方文档](https://github.com/linux-rdma/rdma-core)
- [libibverbs API 参考](https://www.rdmamojo.com/2013/06/08/libibverbs-api/)
- [InfiniBand Architecture Specification](https://www.infinibandta.org/ibta-specification/)
- [SoftRoCE — Linux Kernel](https://github.com/linux-rdma/rdma-core/tree/master/providers/rxe)
