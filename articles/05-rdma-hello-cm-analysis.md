# rdma_hello_cm 代码详细分析

> RoCEv2 RDMA Hello World 程序 — 基于 librdmacm (RDMA_PS_IB)

---

## 1. 架构概要

本程序使用 **librdmacm** 提供的连接管理能力，通过 `RDMA_PS_IB` 端口空间实现 RDMA SEND/RECV 双向通信。相比 raw ibverbs 版，不需要手写 TCP 辅助通道交换 QP 信息，也不需要手动管理 QP 状态机。

```
+-----------------------+    RDMA CM Events / IB 寻址    +-----------------------+
|  linux01 (server)     |◄──────────────────────────────►|  linux02 (client)     |
|                       |      (RDMA_PS_IB 端口空间)      |                       |
|  rxe0 (192.168.64.4)  |◄────────── RDMA SEND ────────►|  rxe_0 (192.168.64.5) |
+-----------------------+                                +-----------------------+
```

**与 raw ibverbs 版的对比：**

| 方面 | raw ibverbs 版 | rdma_cm 版 |
|------|----------------|------------|
| 连接管理 | TCP 交换 QP 信息 | rdma_cm 事件驱动 |
| QP 状态机 | 手动 INIT→RTR→RTS | rdma_cm 自动处理 |
| 依赖库 | `-libverbs` | `-lrdmacm -libverbs` |
| 代码量 | ~430 行 | ~360 行 |
| 灵活性 | 更高（完全控制） | 更高层抽象 |

---

## 2. 为什么是 RDMA_PS_IB 而不是 RDMA_PS_TCP？

**关键发现：** SoftRoCE 上 `RDMA_PS_TCP` 的 `rdma_bind_addr` 返回 `EADDRNOTAVAIL`（Cannot assign requested address）。

原因分析：

| 端口空间 | bind 结果 | 原因 |
|----------|-----------|------|
| `RDMA_PS_TCP` | ❌ EADDRNOTAVAIL | 内核 rdma_cm 没找到 IP→RDMA 设备的映射 |
| `RDMA_PS_IB` | ✅ OK | IB 端口空间使用 GID/服务 ID 寻址，绕过了 IP 映射 |

`RDMA_PS_IB` 配合 `sockaddr_in`（IPv4 地址）是一种**混合模式**——应用层传 IP:Port，librdmacm 内部将其映射为 IB 寻址所需的 GID 和服务 ID：

```
sockaddr_in(192.168.64.4:18515)
    │
    ▼
RDMA_PS_IB 处理
    │
    ├── IP→GID: 从 RDMA 设备 GID 表中查找匹配条目
    │     192.168.64.4 → GID index 1 → ::ffff:c0a8:4004
    │
    └── Port→Service ID: 端口号映射为 IB 服务 ID
          18515 → 0x0000000000004853
```

---

## 3. 数据结构

### `struct rdma_res` — RDMA 资源

```c
struct rdma_res {
    struct ibv_pd *pd;      // 保护域
    struct ibv_cq *cq;      // 完成队列 (16 个槽位)
    struct ibv_mr *mr;      // 内存区域 (64 字节缓冲区)
    char buf[MSG_SIZE];     // 收发共用缓冲区
};
```

相比 raw ibverbs 版少了 `ibv_context *ctx` 和 `ibv_qp *qp`：
- `ctx` 不需要保存，因为 `res_alloc` 只在需要时从 `cm_id->verbs` 获取
- `qp` 由 rdma_cm 管理，通过 `cm_id->qp` 访问，不需要单独保存

### 连接参数 — `struct rdma_conn_param`

```c
struct rdma_conn_param cp = {
    .initiator_depth     = 1,   // 最大未完成读/原子操作 (发起端)
    .responder_resources = 1,   // 最大未完成读/原子操作 (响应端)
    .retry_count         = 7,   // 重试次数
};
```

这些参数在 `rdma_connect`（client）和 `rdma_accept`（server）时传递给对方，用于协商 QP 的传输能力。

---

## 4. 核心 API：librdmacm 事件驱动模型

### 事件通道与 CM ID

```c
struct rdma_event_channel *ec;   // 事件通道 — 接收 CM 事件的管道
struct rdma_cm_id *cm_id;        // CM ID — 一个 RDMA 连接的句柄
```

API 调用模式：
```
rdma_create_event_channel() → 创建事件通道
    └── rdma_create_id() → 创建 CM ID（绑定到一个事件通道）
```

CM ID 在 server 端有双重角色：
- **listen_id**: 监听用的 CM ID，负责接收连接请求
- **child**: CONNECT_REQUEST 事件中自动创建的 CM ID，代表一个具体连接

### 事件类型

| 事件 | 值 | 说明 |
|------|-----|------|
| `RDMA_CM_EVENT_CONNECT_REQUEST` | 4 | Server 收到连接请求 |
| `RDMA_CM_EVENT_ESTABLISHED` | 9 | 连接建立完成 |
| `RDMA_CM_EVENT_DISCONNECTED` | 10 | 对端断开连接 |

### 事件处理流程

```
rdma_get_cm_event(ec, &ev)   → 阻塞等待事件
    ev->event                → 事件类型
    ev->id                   → 关联的 CM ID
    ev->listen_id            → 对应的 listen CM ID（仅 CONNECT_REQUEST）
处理事件...
rdma_ack_cm_event(ev)        → 确认事件（必须，否则会阻塞后续事件）
```

**重要:** 每个 `rdma_get_cm_event` 获取的事件**必须**用 `rdma_ack_cm_event` 确认，否则事件队列会阻塞。

---

## 5. Server 流程详解

### 步骤分解

```
                    Event Channel
                         │
    rdma_create_event_channel()
                         │
    rdma_create_id(ec, &listen_id, NULL, RDMA_PS_IB)
                         │
    rdma_bind_addr(listen_id, (sockaddr*)&addr)
                         │
    rdma_listen(listen_id, 1)
                         │
    ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┼─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
                         │  (blocking wait)
    rdma_get_cm_event(ec, &ev) ◄─── CONNECT_REQUEST
     │                          (ev->id = child)
     │
     ├── res_alloc(child->verbs)        ── PD, CQ, MR
     ├── rdma_create_qp(child, pd, &qa) ── QP (RESET 状态)
     ├── post_recv(child->qp, res)      ── 预备接收缓冲区
     ├── rdma_accept(child, &cp)        ── QP → RTR → RTS
     └── rdma_ack_cm_event(ev)
                         │
    rdma_get_cm_event(ec, &ev) ◄─── ESTABLISHED
     │
     ├── rdma_ack_cm_event(ev)
     ├── wait_cq(res)              ◄─── RECV 完成
     │    └── printf("received: %s")
     ├── post_send(child->qp, res, reply)
     └── wait_cq(res)              ◄─── SEND 完成
                         │
    rdma_disconnect(child)
    rdma_get_cm_event(ec, &ev) ◄─── DISCONNECTED
    rdma_ack_cm_event(ev)
```

### `setup_cm_id()` — 统一的 ID 创建与绑定

```c
static int setup_cm_id(struct rdma_event_channel *ec,
                       struct rdma_cm_id **id,
                       const char *bind_ip, int port)
{
    rdma_create_id(ec, id, NULL, RDMA_PS_IB);

    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_port   = htons(port),
    };
    if (bind_ip)
        inet_pton(AF_INET, bind_ip, &addr.sin_addr);
    else
        addr.sin_addr.s_addr = htonl(INADDR_ANY);

    rdma_bind_addr(*id, (struct sockaddr *)&addr);
}
```

- 绑定到 `192.168.64.4` → 指定 RXE 设备 `rxe0`
- `INADDR_ANY` → 自动选择（SoftRoCE 上可能失败，此时需要显式指定 IP）
- 绑定不指定具体端口，由 `rdma_listen` 确定监听 socket

### CONNECT_REQUEST 事件处理

当 server 收到客户端连接请求时：

1. **`ev->id` 指向新的 CM ID** (`child`)，它继承 listen_id 的 event channel
2. **`ev->listen_id` 指向 listen_id**，用于区分监听器和连接
3. **`child->verbs` 已自动确定**，指向 RXE 设备的 ibv_context

```c
rdma_get_cm_event(ec, &ev);
// ev->event = RDMA_CM_EVENT_CONNECT_REQUEST
// ev->id = child (新连接)
// ev->listen_id = listen_id (监听器)

child = ev->id;

// 此时 child->verbs 已绑定到 RDMA 设备
res = res_alloc(child->verbs);  // 基于正确的设备创建资源
```

### 与 raw ibverbs 版 Server 的对比

| raw ibverbs 版 | rdma_cm 版 | 优势 |
|----------------|------------|------|
| 需先 `tcp_listen` + `tcp_accept` | rdma_cm 自动处理连接 | 省去辅助通道 |
| 手动 `qp_transition_init/rtr/rts` | `rdma_accept` 自动完成 | 减少出错可能 |
| 需 `struct qp_info` + `tcp_exchange` | 隐式通过 CM 消息交换 | 隐藏底层细节 |

---

## 6. Client 流程详解

### 步骤分解

```
    rdma_create_event_channel()
    rdma_create_id(ec, &cm_id, NULL, RDMA_PS_IB)
                         │
    rdma_resolve_addr(cm_id, NULL, &dst, 2000)
    rdma_get_cm_event(ec, &ev) ◄─── ADDR_RESOLVED
    rdma_ack_cm_event(ev)
                         │
    rdma_resolve_route(cm_id, 2000)
    rdma_get_cm_event(ec, &ev) ◄─── ROUTE_RESOLVED
    rdma_ack_cm_event(ev)
                         │
    ├── res_alloc(cm_id->verbs)
    ├── rdma_create_qp(cm_id, pd, &qa)
    ├── post_recv(cm_id->qp, res)    ── 先预备接收缓冲区
    └── rdma_connect(cm_id, &cp)     ── QP → RTR → RTS + 发送 CM 请求
                         │
    rdma_get_cm_event(ec, &ev) ◄─── ESTABLISHED
                         │
    ├── post_send(cm_id->qp, res, msg)
    ├── wait_cq(res)                 ── SEND 完成
    ├── wait_cq(res)                 ── RECV 完成(服务器回复)
    ├── printf("received: %s")
    │
    rdma_disconnect(cm_id)
    rdma_get_cm_event(ec, &ev) ◄─── DISCONNECTED
```

### 地址解析与路由解析

**`rdma_resolve_addr()`:** 将目标 IP:Port 解析到具体的 RDMA 设备

```c
struct sockaddr_in dst = {
    .sin_family = AF_INET,
    .sin_port   = htons(18515),
};
inet_pton(AF_INET, "192.168.64.4", &dst.sin_addr);

rdma_resolve_addr(cm_id, NULL, (struct sockaddr *)&dst, 2000);
// 2000 = timeout (ms)
```

- 成功后在 `cm_id->verbs` 中确定 RDMA 设备上下文
- 确定了本端将使用的 GID index（RDMA_PS_IB 下自动选择 RoCEv2 GID）

**`rdma_resolve_route()`:** 计算到目标的路径信息

```c
rdma_resolve_route(cm_id, 2000);
```

- 成功后路径信息（GRH、DGID 等）已缓存在 CM ID 中
- 这些信息在 `rdma_connect` 时自动使用

### 先 post_recv 再 connect

```c
post_recv(cm_id->qp, res);    // 先投递接收
rdma_connect(cm_id, &cp);     // 再发起连接
```

这保证 server 回复到达时，接收缓冲区已经就位。RDMA 的一个重要设计原则：**接收端必须提前 post RECV**，因为 RDMA 控制器是直接 DMA 到用户缓冲区的，没有内核 buffer 兜底。

### 为什么 client 需要 2 次 wait_cq？

```
| 顺序 | wr_id | 操作 | 说明                      |
|------|-------|------|---------------------------|
| 1    | 2     | SEND | client 发送完成           |
| 2    | 1     | RECV | server 的回复到达         |
```

SEND 和 RECV 各产生一个 CQ 完成事件，各需一次 `wait_cq` 来消费。

---

## 7. CM 事件驱动 vs raw ibverbs 手动管理

### QP 状态机对比

```
            raw ibverbs                        rdma_cm
            ───────────                        ───────
            
            [RESET]                            [RESET]
               │                                  │
               │ qp_transition_init()              │ rdma_create_qp()
               ▼                                  ▼
            [INIT]                              [INIT]
               │                                  │
               │ qp_transition_rtr()               │ (rdma_connect/accept 内部)
               ▼                                  ▼
            [RTR]                               [RTR]
               │                                  │
               │ qp_transition_rts()               │ (rdma_connect/accept 内部)
               ▼                                  ▼
            [RTS]                               [RTS]
               │                                  │
               │ post_send / wait_cq               │ post_send / wait_cq
               ▼                                  ▼
            数据传输                             数据传输
```

rdma_cm 版省去了 3 个手动状态转换函数（约 50 行代码），这些转换在 `rdma_connect` 和 `rdma_accept` 内部自动完成。

### 错误处理对比

**raw ibverbs 版：** 每步都要检查返回值，状态机出错需要手动排查是哪个转换失败了。

**rdma_cm 版：** CM 事件本身包含了连接状态信息。如果 `rdma_get_cm_event` 返回非 `ESTABLISHED` 事件（如 `REJECTED`、`CONNECT_ERROR`），可以直接判断连接失败。

---

## 8. 通信协议对比

### 整个交互过程的时序

```
Client                               Server
  │                                     │
  ├── rdma_resolve_addr()              │
  │   (IP→RDMA 设备映射)                │
  │◄── ADDR_RESOLVED ──────────────────│
  │                                     │
  ├── rdma_resolve_route()             │
  │   (计算路径、GID)                    │
  │◄── ROUTE_RESOLVED ──────────────── │
  │                                     │
  ├── rdma_create_qp()                 │
  ├── post_recv()                      │
  ├── rdma_connect()                   │
  │   ─── CM ConnectReq ──────────────►│  rdma_get_cm_event()
  │                                    │    → CONNECT_REQUEST
  │                                    ├── res_alloc()
  │                                    ├── rdma_create_qp()
  │                                    ├── post_recv()
  │                                    ├── rdma_accept()
  │   ◄── CM ConnectResp ───────────── │  QP: RESET→INIT→RTR→RTS
  │   QP: RESET→INIT→RTR→RTS           │
  │◄── ESTABLISHED ───────────────────►│◄── ESTABLISHED
  │                                     │
  ├── post_send("Hello from client!")  │
  │   ───── RDMA SEND ────────────────►│  wait_cq() → RECV 完成
  │   wait_cq() → SEND 完成            │  printf("received: %s")
  │                                     ├── post_send("Hello from server!")
  │                                    │  wait_cq() → SEND 完成
  │   ◄──── RDMA SEND ────────────────│
  │   wait_cq() → RECV 完成            │
  │   printf("received: %s")           │
  │                                     │
  ├── rdma_disconnect()                │
  │   ─── CM Disconnect ──────────────►│  rdma_get_cm_event()
  │◄── DISCONNECTED ──────────────────►│◄── DISCONNECTED
```

### raw ibverbs 版 vs rdma_cm 版的带宽对比

两者在数据传输阶段完全相同（都是 `IBV_WR_SEND` + `ibv_poll_cq`），所以性能没有差异。区别仅在连接建立方式。

---

## 9. 代码结构总览

```
rdma_hello_cm.c (~360 行)
├── 头文件 / 常量 / 数据结构         (L1-L26)
├── RDMA 资源管理                    (L28-L60)
│   ├── res_alloc()
│   └── res_free()
├── RDMA 操作                        (L62-L111)
│   ├── post_recv()
│   ├── post_send()
│   └── wait_cq()
├── 连接管理                         (L113-L141)
│   └── setup_cm_id()
├── Server                           (L143-L230)
│   └── run_server()
├── Client                           (L232-L326)
│   └── run_client()
└── Main / CLI                       (L328-L361)
    └── main()
```

---

## 10. 编译和运行

### 编译

```bash
gcc -Wall -Wextra -o rdma_hello_cm rdma_hello_cm.c -lrdmacm -libverbs
```

- `-lrdmacm` — 链接 librdmacm（连接管理、CM 事件）
- `-libverbs` — 链接 libibverbs（PD/CQ/QP/MR/SEND/RECV）

### 运行

```bash
# 服务端 (linux01)
./rdma_hello_cm -s -b 192.168.64.4

# 客户端 (linux02)
./rdma_hello_cm -c 192.168.64.4
```

### 预期输出

```
=== server (linux01) ===                              === client (linux02) ===
server: listening on 192.168.64.4:18515 (RDMA_PS_IB)  client: address resolved
server: connection request                             client: route resolved
server: accepted                                       client: connecting...
server: connection established                         client: connected to 192.168.64.4:18515
server: received: Hello from client!                   client: send completed
server: reply sent                                     client: received: Hello from server!
server: disconnected                                   client: disconnected
```

---

## 11. 关键参数说明

### `rdma_resolve_addr` 超时

```c
rdma_resolve_addr(cm_id, NULL, (struct sockaddr *)&dst, 2000);
```

第四个参数 `timeout = 2000ms`。如果 2 秒内无法完成地址解析，返回超时错误。

### `rdma_conn_param.retry_count`

```c
.retry_count = 7,
```

通信失败时的重试次数。7 次约等于 "几乎无限重试"。如果设为 0，首次失败即断开。

### CQ 轮询超时

```c
for (int i = 0; i < 1000; i++) {
    ibv_poll_cq(r->cq, 1, &wc);
    usleep(2000);  // 2ms
}
```

总超时 = 1000 × 2ms = **2 秒**。对于跨 VM 的 RDMA 通信，2 秒足以覆盖正常情况。

---

## 12. Memory 记录

程序通过 CM 事件驱动，不需要手动维护连接状态。但需要理解以下映射关系：

| librdmacm 概念 | 对应 ibverbs 概念 | 说明 |
|----------------|-------------------|------|
| `rdma_cm_id` | — | 封装了 CM 连接状态 + ibv QP |
| `cm_id->verbs` | `ibv_context` | 自动绑定到 RDMA 设备 |
| `cm_id->pd` | `ibv_pd` | rdma_cm 不自动创建 PD，需手动分配 |
| `cm_id->qp` | `ibv_qp` | 通过 `rdma_create_qp` 创建 |
| `rdma_event_channel` | — | 接收 CM 完成事件的通知管道 |

---

## 13. 与 Socket 编程的类比

对于熟悉 TCP socket 编程的人来说，rdma_cm API 有直接的对应关系：

| TCP Socket | librdmacm (RDMA_PS_IB) |
|------------|------------------------|
| `socket()` | `rdma_create_event_channel()` + `rdma_create_id()` |
| `bind()` | `rdma_bind_addr()` |
| `listen()` | `rdma_listen()` |
| `accept()` | `rdma_get_cm_event(CONNECT_REQUEST)` → `rdma_accept()` |
| `connect()` | `rdma_resolve_addr()` → `rdma_resolve_route()` → `rdma_connect()` |
| `send()` | `ibv_post_send()` + `ibv_poll_cq()` |
| `recv()` | `ibv_post_recv()` + `ibv_poll_cq()` |
| `close()` | `rdma_disconnect()` → `rdma_destroy_id()` |

关键区别：
- RDMA 的接收端**必须提前 post buffer**（否则数据无法到达）
- RDMA 用 CQ 轮询而非中断/select 来通知完成
- RDMA 数据传输是 **zero-copy**（硬件 DMA 直接到用户缓冲区）

---

## 14. 扩展阅读

- [librdmacm 官方文档](https://github.com/linux-rdma/rdma-core/tree/master/librdmacm)
- [RDMA CM Events 详解](https://www.rdmamojo.com/2013/02/02/rdma_cm-events/)
- [RDMA_PS_IB vs RDMA_PS_TCP](https://www.rdmamojo.com/2015/06/20/rdma_cm-port-space/)
- [SoftRoCE (RXE) 项目](https://github.com/linux-rdma/rdma-core/tree/master/providers/rxe)
