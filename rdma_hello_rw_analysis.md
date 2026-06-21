# rdma_hello_rw.c 实现分析

## 概述

`rdma_hello_rw` 是一个基于 **libibverbs** 的 RDMA Read/Write Hello World 程序。它使用 **RC（Reliable Connection）** 模式的 QP，通过 **RDMA Write** 和 **RDMA Read** 直接在两端内存之间传输数据，无需对端 CPU 参与数据面操作。

## 与 Send/Recv 版的核心区别

| 特性 | Send/Recv 版 | Read/Write 版 |
|------|-------------|---------------|
| 数据面操作 | `IBV_WR_SEND` + `ibv_post_recv` | `IBV_WR_RDMA_WRITE` + `IBV_WR_RDMA_READ` |
| 对端 CPU 参与 | 需要 — 对端必须提前 `post_recv` 准备接收 | **不需要** — 直接读写对端内存，对端无感知 |
| RQ 消耗 | 消耗 RQ（Receive Queue） | 不消耗 RQ（操作走 SQ） |
| 完成事件 | 双方 CQ 都有完成事件 | **仅发起端** CQ 有完成事件 |
| MR 权限 | `REMOTE_WRITE` | Server 额外需要 `REMOTE_READ` |
| 同步机制 | CQ 完成事件即表示对方已收到 | 需要额外 TCP 信号同步（因为对端不知道数据已到达） |

> **关键区别**：Send/Recv 类似 TCP 的 send/recv——两端都要参与。RDMA Read/Write 类似 DMA——直接读写对方内存，对方进程完全无感知。

## 数据结构

### qp_info（通过 TCP 交换的连接信息）

```c
struct qp_info {
    uint16_t      lid;       /* RoCE 下填 0（InfiniBand 才用 LID） */
    uint32_t      qpn;       /* QP Number — 对端目标 QP */
    uint32_t      psn;       /* Packet Sequence Number — 起始包序号 */
    union ibv_gid gid;       /* Global ID — RoCEv2 使用 index 1 */
    uint64_t      buf_addr;  /* MR 起始虚拟地址 — 对端 RDMA 访问的地址 */
    uint32_t      rkey;      /* Remote Key — 对端访问本端 MR 的权限密钥 */
};
```

相比 Send/Recv 版新增了 **`buf_addr`** 和 **`rkey`** 两个字段，用于告知对端本端内存的位置和访问密钥。

### rdma_res（本地 RDMA 资源）

```c
struct rdma_res {
    struct ibv_context *ctx;  /* IB 设备上下文 */
    struct ibv_pd      *pd;   /* 保护域 */
    struct ibv_cq      *cq;   /* 完成队列 */
    struct ibv_qp      *qp;   /* 队列对 */
    struct ibv_mr      *mr;   /* 内存区域 */
    char                buf[MSG_SIZE];  /* DMA 缓冲区（唯一的数据缓冲区） */
};
```

**注意**：Server 和 Client 共用同一个 `rdma_res` 结构体，但 MR 注册时的权限不同。

## 控制面（TCP）

### 连接建立

TCP 用于带外交互连接信息，流程与 Send/Recv 版一致：

1. `tcp_listen()` — Server 绑定端口，开始 listen
2. `tcp_accept()` — 接受 Client 连接
3. `tcp_connect()` — Client 连接到 Server
4. `tcp_exchange()` — 交换 `qp_info`（QPN + PSN + GID + buf_addr + rkey）

### 同步信号

RDMA Read/Write 完成后，对端无法从 CQ 感知，因此使用 **TCP 信号** 进行带外同步：

```c
static int tcp_signal(int fd, char c);  /* 发信号 */
static int tcp_wait(int fd, char expected);  /* 等信号 */
```

| 信号 | 方向 | 含义 |
|------|------|------|
| `'W'` | Client → Server | RDMA Write 已完成，Server 可以检查内存 |
| `'R'` | Server → Client | 回复已写入，Client 可以 RDMA Read |
| `'D'` | Client → Server | Client 读完，准备断开 |

## 数据面（RDMA Read/Write）

### RDMA Write：写对端内存

```c
static int post_write(struct rdma_res *r, uint64_t remote_addr, uint32_t rkey)
```

- **将本端 `buf` 的内容直接写入对端内存**（由 `remote_addr` + `rkey` 指定）
- 对端 CPU 不会被中断，数据"静默"地出现在对端内存中
- 使用 `IBV_WR_RDMA_WRITE` opcode
- 对端无需任何准备（无需 `post_recv`）

```c
wr.opcode = IBV_WR_RDMA_WRITE;
wr.wr.rdma.remote_addr = remote_addr;  /* 对端 MR 地址 */
wr.wr.rdma.rkey        = rkey;         /* 对端 MR 的 rkey */
```

### RDMA Read：读对端内存

```c
static int post_read(struct rdma_res *r, uint64_t remote_addr, uint32_t rkey)
```

- **从对端内存读数据到本端 `buf`**（由 `remote_addr` + `rkey` 指定）
- 对端 CPU 无感知
- 本端 `buf` 必须已注册 MR（有 `lkey`）

```c
wr.opcode = IBV_WR_RDMA_READ;
wr.wr.rdma.remote_addr = remote_addr;
wr.wr.rdma.rkey        = rkey;
```

### CQ 轮询

与 Send/Recv 版相同的 `wait_cq()` 函数，轮询 CQ 等待操作完成：

- 每 2ms 轮询一次，最多 2 秒
- RDMA Write 完成 = NIC 已确认写操作完成
- RDMA Read 完成 = 数据已到达本端缓冲区

## QP 状态机

### INIT（RESET → INIT）

```c
.qp_access_flags = IBV_ACCESS_LOCAL_WRITE
                 | IBV_ACCESS_REMOTE_WRITE
                 | IBV_ACCESS_REMOTE_READ,
```

相比 Send/Recv 版多了 `IBV_ACCESS_REMOTE_READ`，因为 Client 需要通过 RDMA Read 读取 Server 的内存。

### RTR（Ready to Receive）

配置对端的 QPN、PSN 和 GID，与 Send/Recv 版完全一致。

### RTS（Ready to Send）

配置重传参数和本端发端 PSN，与 Send/Recv 版完全一致。

## MR 注册权限

```c
// Server（res_alloc(1)）
int mr_flags = IBV_ACCESS_LOCAL_WRITE
             | IBV_ACCESS_REMOTE_WRITE   /* Client 可以写我 */
             | IBV_ACCESS_REMOTE_READ;   /* Client 可以读我 */

// Client（res_alloc(0)）
int mr_flags = IBV_ACCESS_LOCAL_WRITE;   /* 不需要暴露给对端 */
```

**注意**：Client 的 MR 只有 `LOCAL_WRITE`，因为 Client 只作为 RDMA Read 的**发起端**（数据读取到本端 buf 需要本地写权限）。Server 的内存需要暴露给 Client，所以需要 `REMOTE_WRITE`（Client 写过来）和 `REMOTE_READ`（Client 读过去）。

## 完整执行流程

```
Server (linux01)                      Client (linux02)
    |                                      |
    |  1. 打开 RDMA 设备                    |  1. 打开 RDMA 设备
    |  2. 注册 MR (REMOTE_WRITE+READ)       |  2. 注册 MR (仅 LOCAL_WRITE)
    |  3. 创建 QP                          |  3. 创建 QP
    |  4. TCP listen                       |  4. TCP connect 192.168.64.4
    |<========== TCP exchange qp_info ======>|
    |     (传 buf_addr + rkey)              |     (获取对端 buf_addr + rkey)
    |  5. QP: INIT → RTR → RTS              |  5. QP: INIT → RTR → RTS
    |                                      |
    |                                      |  6. 写 "Hello from client!" 到本端 buf
    |                                      |  7. RDMA Write → Server 的 buf
    |                                      |  8. wait_cq() ← WRITE 完成
    |                                      |  9. TCP signal 'W'
    |  6. tcp_wait('W')                    |
    |  7. 打印 buf: "Hello from client!"    |
    |  8. 写 "Hello from server!" 到 buf    |
    |  9. TCP signal 'R'                   |
    |                                      | 10. tcp_wait('R')
    |                                      | 11. RDMA Read ← Server 的 buf
    |                                      | 12. wait_cq() ← READ 完成
    |                                      | 13. 打印 buf: "Hello from server!"
    |                                      | 14. TCP signal 'D'
    | 10. tcp_wait('D')                    |
```

## 运行示例

```bash
# Server（linux01）
wangkun@linux01:~$ sudo ./rdma_hello_rw -s
server: opening RDMA device...
  device: rxe0
server: buf_addr=0xaaaadb5c7868 rkey=0x11e1
server: waiting for TCP connection on port 18515...
  tcp: client from 192.168.64.5:44692
server: exchanging QP/MR info...
  local QPN: 33, peer QPN: 17
server: QP connected
server: waiting for RDMA write...
server: received via RDMA write: Hello from client!
server: done

# Client（linux02）
wangkun@linux02:~$ sudo ./rdma_hello_rw -c 192.168.64.4
client: opening RDMA device...
  device: rxe_0
client: buf_addr=0xaaaae1dc6868 rkey=0x0
client: connecting to 192.168.64.4:18515...
client: exchanging QP/MR info...
  local QPN: 17, peer QPN: 33
  peer buf_addr=0xaaaadb5c7868 rkey=0x11e1
client: QP connected
client: RDMA writing to server...
client: RDMA write completed
client: waiting for RDMA read signal...
client: RDMA reading from server...
client: RDMA read completed
client: received via RDMA read: Hello from server!
```

## 设计要点

### 1. 为什么 TCP 信号是必需的？

RDMA Write 完成后，**只有发起端（Client）知道操作已完成**。Server 不知道数据何时到达自己的内存。如果 Server 在数据到达前读取 `buf`，读到的是旧数据。因此需要 TCP 信号作为带外同步机制。

Send/Recv 不需要这个同步：`post_send` 完成后，对端的 `wait_cq`（`post_recv` 的完成事件）自然就知道数据到了。

### 2. 为什么 Client 的 rkey = 0x0？

Client 的 MR 只注册了 `IBV_ACCESS_LOCAL_WRITE`，没有暴露给对端，所以 rkey 无意义。Client 作为 RDMA Read/Write 的**发起端**，只需要知道 Server 的 `buf_addr` 和 `rkey` 即可。Server 不需要访问 Client 的内存。

### 3. QP Access Flags 的一致性

QP 在 INIT 阶段的 `qp_access_flags` 和 MR 注册时的权限需要匹配：

- **QP INIT flags**：声明本 QP 允许对端通过此 QP 做什么操作
- **MR flags**：声明本 MR 允许什么访问

如果 QP 允许 `REMOTE_WRITE` 但 MR 不允许，实际操作为会失败（由 NIC 硬件校验）。

### 4. 零拷贝特性

RDMA Read/Write 的真正价值在于：

- **无需对端 CPU 参与**：对端可以忙于计算，数据悄悄到达或被取走
- **无需 RQ 资源**：不消耗接收队列，适合大量数据传输
- **不过对端内核**：数据直接 DMA 到用户态内存，不经过对端内核协议栈
