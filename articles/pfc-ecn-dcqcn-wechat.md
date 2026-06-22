# PFC/ECN/DCQCN 深度解析：RoCEv2 拥塞控制三件套

> 从协议细节到内核源码，系统拆解 RoCEv2 无损网络的三大基石

---

## 1. 引言：RoCEv2 为什么需要拥塞控制

### 1.1 RoCEv2 的"信任"问题

RDMA 的核心语义建立在**底层网络绝对可信**的假设之上。IB 传输层的可靠连接（RC, Reliable Connection）服务要求发送端发出的每个数据包都能被接收端确认——如果发生丢包，RC QP 需要执行重传，而重传路径走的是软件栈（驱动层重传机制），延迟从微秒级飙升到毫秒级，吞吐量直接坍塌。

InfiniBand 原生网络从设计上保证了这种可信性：IB 链路层的流控机制（基于信用值的链路级流控）确保缓冲区永远不会溢出，理论上零丢包。而 RoCEv2 将 IB 报文封装到 UDP/IP 之上，底层的 Ethernet 本质是 **best-effort** 网络——交换机在拥塞时无条件丢包。这就产生了一个根本性矛盾：

> RDMA 需要无损传输，而 Ethernet 天然有损。

解决这个矛盾就是 RoCEv2 拥塞控制体系的核心使命。它需要在 Ethernet 之上构建一个"准无损"的传输环境，同时尽可能保留 RDMA 的低延迟和高吞吐特性。

### 1.2 拥塞的三种后果

RoCEv2 网络中的拥塞不是一个单一问题，而是三个层次的问题：

**后果一：缓冲区溢出 → 丢包 → RC 重传**

这是最直接的后果。当多个发送端同时向同一个接收端发送数据时，交换机或接收端网卡的缓冲区被填满，后续数据包被丢弃。对于 RC QP，这意味着：

- 发送端的报文没有被确认（ACK 超时）
- 驱动层检测到丢包后触发重传
- 重传期间新发送暂停，吞吐量骤降
- 更严重的是，RC 重传是**串行化**的——一个丢包会阻塞后续所有报文

实测数据表明，一个 RoCEv2 流在 0.1% 丢包率下吞吐量可能下降 50% 以上。

**后果二：PFC 触发 → HoL blocking → 无辜流被牵连**

为了防止丢包，PFC 会在缓冲区将满时触发逐跳背压。但 PFC 是以优先级为粒度的——一个优先级上的拥塞会暂停该优先级的所有流量，包括那些不经过拥塞端口的流。这就是 head-of-line（HoL）blocking：一条慢流阻塞了同优先级的其他所有流。

**后果三：incast 场景 → 吞吐坍塌**

这是分布式存储等场景中的典型问题：N 个工作节点同时向一个聚合节点发送数据（如 N-to-1 通信模式）。当 N 很大时（如 32:1 或 64:1），聚合节点的入向带宽需求远超其处理能力，缓冲区瞬间填满，PFC 和丢包同时发生，吞吐量出现剧烈坍塌。

### 1.3 三件套的分工预览

PFC、ECN、DCQCN 分别解决不同层面的问题，它们之间是**协作而非替代**关系：

| 机制 | 层级 | 作用范围 | 信号类型 | 响应速度 | 主要目标 |
|------|------|----------|----------|----------|----------|
| **PFC** | 链路层（L2） | 逐跳（hop-by-hop） | PAUSE 帧 | 微秒级 | 防止缓冲区溢出导致丢包 |
| **ECN** | 网络层（L3） | 端到端（end-to-end） | CNP 报文 | 毫秒级 | 通知发送端网络拥塞状态 |
| **DCQCN** | 传输层（L4） | 端到端（end-to-end） | 基于 ECN 反馈 | 毫秒级 | 动态调整发送速率 |

![PFC/ECN/DCQCN 三件套架构图](images/pfc-ecn-dcqcn-architecture.png)

形象的类比：

- **PFC** 是止血带——局部施压防止流血过多，但可能造成组织坏死（HoL blocking）
- **ECN** 是早期预警系统——在情况恶化之前发出警报
- **DCQCN** 是精准给药系统——根据警报智能调节流量

### 1.4 一条 Rocket 流的旅程

> 本文用 **Rocket 流**（Rocket Flow）代指一条处于稳定传输状态的 RDMA RC QP 流。取名 Rocket 是因为它全速喷射数据——直到拥塞控制机制告诉它减速。

假设一个 RoCEv2 RC QP 正以 100Gbps 的速度向对端发送 RDMA WRITE 请求。这条 Rocket 流从发送端到接收端要经历以下拥塞控制环节：

```
发送端 ──→ 交换机1 ──→ 交换机2 ──→ 接收端
   |            |            |           |
   | ④ DCQCN   │  ② ECN    │  ② ECN   │ ③ 生成
   │ 降速      │  标记      │  标记     │ CNP
   └────────────┴────────────┴───────────┘
```

**时间线：**

1. **t=0**：发送端全速发出 RDMA WRITE 请求，数据包经过交换机 1 和 2 到达接收端
2. **t=0 + RTT/2**：交换机 2 的出向队列长度超过 ECN 标记阈值（Kmin），在数据包 IP 头中设置 CE（Congestion Experienced）码点
3. **t=0 + RTT**：接收端收到带有 CE 标记的数据包，生成 CNP（Congestion Notification Packet）返回给发送端
4. **t=0 + RTT + ε**：交换机 2 的出向队列继续增长，超过 PFC 的 X-off 阈值，向交换机 1 发送 PAUSE 帧
5. **t=0 + 2×RTT**：发送端收到 CNP，DCQCN 更新 α 拥塞因子，执行乘性减降速
6. **t=0 + 2×RTT + T**：如果不再收到 CNP，DCQCN 开始加性增，逐步恢复速率

这个简化的旅程揭示了三个机制之间的时序关系：**ECN 应该比 PFC 先触发**——如果 PFC 在 ECN 标记之前就触发了，说明 ECN 阈值设置不当，DCQCN 将无法正常发挥作用。

---

## 2. PFC —— 逐跳流量控制

### 2.1 802.1Qbb 协议细节

#### 背景与动机

PFC（Priority Flow Control）由 IEEE 802.1Qbb 标准定义，是 DCB（Data Center Bridging）套件的核心组件之一。它的设计目标是在 Ethernet 上实现"无损"传输——不是消除所有丢包，而是在拥塞发生时通过逐跳背压防止缓冲区溢出导致的丢包。

PFC 是对传统 IEEE 802.3x PAUSE 帧的改进。802.3x PAUSE 是端口级别的：一个 PAUSE 帧暂停整个端口的所有流量。这在多业务混合部署的场景下过于粗粒度——存储流量暂停不应该影响 Web 流量。PFC 的改进在于引入了**优先级粒度**。

#### 8 优先级独立暂停

PFC 将 Ethernet 链路划分为 8 个虚拟通道（对应 802.1Q VLAN PRI 字段的 0-7），每个优先级独立进行流量控制：

```
┌─────────────────────────────────────────────────┐
│                  Ethernet 链路                       │
├───────┬───────┬───────┬─── ... ───┬───────────────┤
│  P0   │  P1   │  P2   │           │     P7        │
│(尽力) │(背景) │(存储) │           │  (网管/管理)   │
├───────┴───────┴───────┴─── ... ───┴───────────────┤
│ 每个优先级有自己的队列和暂停状态                      │
└─────────────────────────────────────────────────┘
```

每个优先级独立维护：
- **发送状态**：正常 / PAUSE（等待恢复）
- **PAUSE 计时器**：接收到 PAUSE 帧时设置，超时后自动恢复
- **缓冲区占用**：接收端的缓冲区水位

#### PAUSE 帧格式

PFC PAUSE 帧是基于 MAC Control 帧的（EtherType = 0x8808），格式如下：

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─────────────────────────────────────────────────────────────────┤
│                     MAC DA (6 B)                            │
│                                  ┌──────────────────────────────┤
│                                  │    MAC SA (6 B)             │
├─────────────────────────────────────────────────────────────────┤
│    0x8808 (MAC Control)         │  Opcode = 0x0101 (PFC)       │
├─────────────────────────────────────────────────────────────────┤
│  Reserved (1 B) │    Priority Enable Vector (2 B)               │
├─────────────────────────────────────────────────────────────────┤
│  Time[0] (2 B)  │  Time[1] (2 B) │ Time[2] (2 B) │ Time[3]     │
├─────────────────────────────────────────────────────────────────┤
│  Time[4] (2 B)  │  Time[5] (2 B) │ Time[6] (2 B) │ Time[7]     │
├─────────────────────────────────────────────────────────────────┤
│                     FCS (4 B)                                   │
└─────────────────────────────────────────────────────────────────┘
```

![PFC PAUSE 帧交互时序](images/pfc-pause-sequence.png)

关键字段：
- **Priority Enable Vector (PEV)**：2 字节的位图，每一位对应一个优先级，置 1 表示该优先级的时间向量有效
- **Time[0-7]**：每个优先级对应的时间值，单位是 **PFC 时间单位**（由链路速率决定，通常为 512 位时间），值为 0 表示取消 PAUSE（X-on）

对于 100Gbps 链路，一个时间单位 ≈ 5.12ns，最大可暂停时间 ≈ 5.12ns × 65535 ≈ 335μs。

关于 PAUSE 帧的接收端行为：

> When an interface receives a PFC PAUSE frame with Time[i] = 0, it MUST resume transmission on priority i (解除暂停状态). When Time[i] > 0, it MUST stop transmission on priority i for the specified duration. If a second PAUSE frame arrives before the first timer expires, the timer is refreshed with the new value.

#### X-off / X-on 阈值设计

PFC 的核心在于接收端的**阈值决策**——何时发送 PAUSE，何时发送解除 PAUSE。这涉及到两个关键阈值：

```
缓冲区占用
    ↑
    │  ┌──────────────────────────────────────┐
    │  │              Headroom               │  ← 绝对不应超过的区域
    │  ├──────────────────────────────────────┤
    │  │        X-off 阈值 ← PAUSE 触发点         │
    │  ├──────────────────────────────────────┤
    │  │                                      │
    │  │         正常工作区间                    │
    │  │                                      │
    │  ├──────────────────────────────────────┤
    │  │        X-on 阈值 ← PAUSE 解除点          │
    │  ├──────────────────────────────────────┤
    │  │                                      │
    │  └──────────────────────────────────────┘
    └────────────────────────────────────────────→ 时间
```

- **X-off 阈值**：当缓冲区占用超过此值时，向对端发送 PAUSE（Time[i] > 0）
- **X-on 阈值**：当缓冲区占用回落到此值以下时，向对端发送解除 PAUSE（Time[i] = 0）
- **Headroom**：X-off 阈值到缓冲区的顶部的空间，用于吸收 PAUSE 生效期间的到达数据

Headroom 的计算公式：

```
Headroom = RTT_max × LinkBandwidth × 2
```

系数 2 是因为：从发送端收到 PAUSE 到真正停止发送，仍然会有一部分在途数据（in-flight packets）到达。对于 100Gbps 链路，1μs RTT 的 headroom 约为 25KB。

### 2.2 PFC 在 RoCEv2 中的角色

#### RoCE 流量的优先级映射

实践中，RoCE 流量几乎总是映射到**优先级 3**。这个选择并非标准规定，而是来自 Mellanox/NVIDIA 的推荐配置，主要考虑：

- 优先级 0-2 留给传统 LAN 流量（尽力而为、背景、存储）
- 优先级 3 给 RoCE 无损流量
- 优先级 4-7 留给 FCoE 等其他 DCB 流量

#### DCB 配置示例

PFC 配置通过 DCB（Data Center Bridging）协议栈完成。在 Mellanox 网卡上的典型配置：

```bash
# 使用 mlnx_qos 工具配置 PFC
mlnx_qos -i eth0 --pfc 0,0,0,1,0,0,0,0

# 参数说明：8 个优先级逐一设置，1=启用 PFC，0=不启用
# 这里优先级 3 启用了 PFC

# 查看当前 PFC 配置
mlnx_qos -i eth0 --show-pfc
```

使用 `dcbtool`：

```bash
dcbtool sc eth0 pfc e:3
# e:3 意为"enable PFC on priority 3"
```

#### DCBX 协商

PFC 配置通过 DCBX（DCB Exchange Protocol，基于 LLDP）在链路两端自动协商。DCBX 确保链路两端的 PFC 配置一致——如果一端配置了优先级 3 的 PFC 而另一端没有，链路上将出现不可预测的行为，最坏情况下会导致持续丢包。

### 2.3 PFC 的阴暗面

PFC 解决了丢包问题，但引入了新的问题。这些副作用在实际部署中往往是比丢包更难处理的挑战。

#### HoL Blocking（头阻阻塞）

这是 PFC 最被诟病的问题。假设一个场景：

```
    发送端1 ──── 100Gbps ────┐
                              ├─── 交换机 ──── 接收端A（拥塞）
    发送端2 ──── 100Gbps ────┘        │
                                      └─── 接收端B（空闲）
```

当接收端A拥塞时，交换机的出向端口向发送端发送 PAUSE，暂停优先级 3 的所有流量。结果：

- 发送端1 到 接收端A 的流量被暂停（这是预期的）
- **发送端2 到 接收端B 的流量也被暂停**（这是问题）
- 即使接收端B 空闲且能接收更多数据

这就是 HoL blocking：一个拥塞的出向队列阻塞了**整个优先级**，即使其他出向端口完全空闲。在有多条流共享同一优先级的大规模部署中，HoL blocking 是性能隔离失败的主要原因。

#### PFC 死锁

在存在环路的拓扑中（即使是无环拓扑，在故障切换时也可能暂态成环），PFC 死锁可能发生：

```
    ┌─────── A ────────┐
    │                   │
    │                   │
    C                   B
    │                   │
    │                   │
    └─────── D ─────────┘
```

如果 A 因为拥塞向 C 发 PAUSE，C 又需要转发数据给 B，B 向 D 发 PAUSE，D 需要转发数据给 A——这样就形成了一个 PAUSE 依赖环。每个节点都在等待上游解除 PAUSE，但上游也在等待更上游，形成了**分布式死锁**。

PFC watchdog 是解决死锁的常用机制：当收到 PAUSE 超过一定时间（通常 100-500ms）后，网卡强制解除 PAUSE 并丢弃部分数据包。这本质上是用"可控丢包"打破死锁——比无限期死锁要好，但依然会造成性能抖动。

#### PFC 风暴

PFC 风暴是指 PAUSE 帧以极高频次触发的异常状态（每秒数十万次 PFC 帧）。原因通常是：

- ECN 阈值设置过高，PFC 在 DCQCN 有机会响应之前就已触发
- 多条流同时拥塞同一条链路，PFC 频繁交替触发/解除
- 缓冲区配置不当，headroom 过小导致频繁阈值越界

PFC 风暴的典型症状是 `ethtool -S` 中 PFC 帧计数器持续快速增长，同时吞吐出现锯齿形波动。

#### 公平性问题

PFC 不感知流（flow），只感知优先级。这意味着：

- 一个优先级内的所有流共享同一个 PAUSE 信号
- 一条 misbehaving 流（如不响应 ECN 的老化流）可以拖累同一优先级的其他所有流
- 没有机制区分"这个优先级拥塞是由哪条流引起的"

### 2.4 PFC 相关内核源码

#### Linux DCB 协议栈

PFC 配置和管理在 Linux 内核中通过 DCB 子系统实现。核心文件在 `net/dcb/`：

- **`dcb.c`**：DCB 核心框架，提供 netlink 接口供用户态工具（如 lldpad、dcbtool）配置
- **`dcbnl.c`**：DCBNL（DCB Netlink）协议实现，处理 PFC 配置的 get/set 操作

用户态配置通过 RTM_GETDCB/RTM_SETDCB netlink 消息到达内核：

```c
// net/dcb/dcbnl.c 中的关键数据结构
struct dcbnl_pfc {
    __u8    pfc_cap;      // 支持的 PFC 优先级数量
    __u8    pfc_en;       // 启用了 PFC 的优先级位图
    __u16   pfc_on;       // PFC 全局开关
    __u16   mbc;          // 多缓冲能力
    __u16   delay;        // PFC 响应延迟
    __u8    requests[8];  // 每个优先级上接收到的 PAUSE 帧计数
    __u8    indications[8]; // 每个优先级上发出的 PAUSE 帧计数
};
```

#### Mellanox 驱动中的 PFC

Mellanox mlx5 驱动中 PFC 相关代码主要在 `drivers/net/ethernet/mellanox/mlx5/` 目录：

```c
// drivers/net/ethernet/mellanox/mlx5/en/port_buffer.c
// PFC 阈值配置的关键函数

int mlx5e_port_set_priority_buffer(struct mlx5e_priv *priv, u32 *buffer_size)
{
    // 设置 X-off/X-on 阈值
    // 参数通过 MLX5_SET 宏写入硬件寄存器
    MLX5_SET(pptb_reg, pptb, prio_buffer, prio_bitmap);
    MLX5_SET(pptb_reg, pptb, buffer_size, size);
    // ...
}

// PFC 计数器读取
// drivers/net/ethernet/mellanox/mlx5/en/port.c
// ethtool -S 中的 PFC 统计来自这里的硬件计数器
```

**关键观察**：PFC 的阈值配置是通过硬件寄存器直接设置的，驱动层只是将这些参数翻译为硬件可理解的格式。实际的 PAUSE 决策由硬件完成——这是 MAC 控制层的能力，不经过软件处理。

### 2.5 监控与调试

#### ethtool -S 解读

```bash
# 查看 PFC 相关计数器
ethtool -S eth0 | grep -i pfc

# 典型输出：
#     pfc_prio_0_xon: 0
#     pfc_prio_0_xoff: 0
#     pfc_prio_1_xon: 0
#     pfc_prio_1_xoff: 0
#     pfc_prio_2_xon: 0
#     pfc_prio_2_xoff: 0
#     pfc_prio_3_xon: 2341
#     pfc_prio_3_xoff: 2318
#     ...
```

**关键指标**：
- `pfc_prio_N_xoff`：触发了多少次 PAUSE（接收端视角）
- `pfc_prio_N_xon`：解除了多少次 PAUSE
- 如果 xoff 持续增长 → 该优先级正在经历拥塞
- 如果 xoff 每秒钟增长数百次 → 严重 PFC 风暴，需要立即排查

#### mlnx_perf 监控

```bash
# 实时监控 PFC 事件
mlnx_perf --pfc

# 输出格式：每个优先级的 xoff/xon 事件频率
# TIME    PRIO  XOFF_RATE  XON_RATE  DURATION
# 10:00:01 3    120/s      118/s     2ms
# 10:00:02 3    135/s      130/s     3ms
```

#### 判断 PFC 影响的方法

1. **PFC counter 非零**：这意味着某处存在拥塞。在理想的无损网络中，PFC 帧应该很少出现
2. **PFC counter 持续增长**：说明拥塞是持续性的而非突发性的，需要检查 ECN 和 DCQCN 配置
3. **同时段吞吐异常**：如果 PFC 高发时吞吐剧烈波动，说明 PFC 正在触发/解除的循环中，是典型的参数配置不当

一个有用的经验法则：PFC 帧应当只出现在毫秒级的拥塞突发中。如果 PFC 连续出现超过 10ms，说明拥塞控制体系的上层（ECN + DCQCN）没有在 PFC 之前有效响应。

---

## 3. ECN —— 端到端拥塞信号

### 3.1 ECN 基础

#### IP ECN 码点

ECN（Explicit Congestion Notification，RFC 3168）使用 IP 头中的 2 位 ECN 字段：

| 码点 | 名称 | 含义 |
|------|------|------|
| 00 | Not-ECT | 不支持 ECN |
| 01 | ECT(1) | ECN Capable Transport，端点支持 ECN |
| 10 | ECT(0) | ECN Capable Transport（与 ECT(1) 等价） |
| 11 | CE | Congestion Experienced，交换机遇到拥塞 |

在 RoCEv2 中，发送端发出的数据包将 ECN 字段设置为 ECT(0)（10 二进制）。当数据包经过的交换机检测到拥塞（队列长度超过阈值）时，将 ECT(0) 改写为 CE（11 二进制）。接收端收到 CE 标记的数据包后，生成 CNP（Congestion Notification Packet）反馈给发送端。

#### CNP 报文

CNP 是 RoCEv2 中 ECN 反馈的载体。它不是 TCP 的 ECN-Echo（ECE），而是一个独立的 IB 传输层控制报文。

CNP 报文特征：
- **OpCode**：0x81（IB OpCode 中的 CNP）
- **目标**：触发 ECN 的原数据包的发送端 QP
- **FECN/BECN 标志**：FECN（Forward ECN）在数据包中标记，BECN（Backward ECN）在 CNP 中回传
- **长度**：最小 IB 报文长度（无 payload）
- **优先级**：通常与所属的 RoCE 流处于同一优先级

CNP 的生成路径（接收端侧大致处理流程）：

```
接收端收到数据包
    │
    ├── IP 头 ECN == CE ?
    │       │
    │       是 → 检查 FECN 标志
    │              │
    │              是 → 生成 CNP，OpCode=0x81，目标 QP=原数据包的源 QP
    │              │
    │              返回 CNP 给发送端
    │
    └── 否 → 正常处理
```

一个关键细节：**接收端不会为每一个 CE 标记的数据包都生成 CNP**。因为 CNP 本身也是网络流量，如果每条带有 CE 的数据包都触发一个 CNP，在拥塞发生时只会让拥塞更严重。实际的 CNP 生成有速率限制（rate-limited），通常每 N 个 CE 包生成一个 CNP，或者在窗口中至少发一个 CNP。

#### CNP 报文格式

```
┌─────────────────────────────────────────────┐
│              Ethernet Header (14 B)          │
├─────────────────────────────────────────────┤
│              IPv4 Header (20 B)              │
├─────────────────────────────────────────────┤
│              UDP Header (8 B)                │
├─────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────┐│
│  │   BTH — Base Transport Header (12 B)    ││
│  │   OpCode = 0x81 (CNP)                   ││
│  │   Dest QP = 原数据包的源 QP              ││
│  │   FECN 标志位处理                         ││
│  ├─────────────────────────────────────────┤│
│  │   I/CRC (4 B)                           ││
│  └─────────────────────────────────────────┘│
├─────────────────────────────────────────────┤
│              FCS (4 B)                       │
└─────────────────────────────────────────────┘
```

![ECN 标记与 CNP 返回流程](images/ecn-marking-flow.png)

### 3.2 ECN 标记算法

#### RED/ECN 集成

ECN 标记不是简单的"队列超过阈值就标记"，而是与 RED（Random Early Detection）算法集成，实现渐进的标记概率。

基本逻辑：

```
标记概率 P
    ↑
 1.0 │                          ┌─────── P = 1.0
    │                     ┌─────┤
    │                ┌────┤     max_probability
    │           ┌────┤    │
    │      ┌────┤   probability
    │ ┌────┤    │   增长曲线
    ││    │    │
   0│─────┘    │    │    │
    └────────────────────────────────────────→ 队列长度
       Kmin    Kmax   limit
```

三个关键参数：
- **Kmin**：开始标记的阈值，队列长度低于此值时完全不标记
- **Kmax**：保证标记的阈值，队列长度达到此值时标记概率为 1.0
- **Max Probability**：在 Kmin 到 Kmax 之间的最大标记概率（通常设置为 0.1~0.2）

标记概率公式（线性增长）：

```
P = (qlen - Kmin) / (Kmax - Kmin) × max_probability
```

当 `qlen < Kmin`：标记概率 = 0
当 `Kmin ≤ qlen < Kmax`：标记概率 = P
当 `qlen ≥ Kmax`：标记概率 = 1.0

#### 标记阈值的影响

Kmin 和 Kmax 的设置对性能有直接影响。下表总结了不同配置的影响：

| 配置 | 效果 | 风险 |
|------|------|------|
| Kmin 太小 | ECN 提前标记，DCQCN 更早降速 | 吞吐无谓损失，链路利用率下降 |
| Kmin 太大 | ECN 标记滞后 | PFC 在 ECN 之前触发，DCQCN 失效 |
| Kmax - Kmin 太小 | 标记概率陡增，速率剧烈波动 | DCQCN 不停触发→恢复，吞吐抖动 |
| Kmax - Kmin 太大 | 标记概率增长缓慢，响应不足 | 队列持续增长，最终触发 PFC |

在实际部署中，Kmin 的调优是最关键的。推荐的初始值为交换机的每个优先级缓冲区大小的 20-40%。

#### 瞬时队列 vs 平均队列

ECN 标记可以用两种方式衡量队列长度：

**瞬时队列（Instant Queue Length）**：
```
if (qlen > Kmin) mark_packet_with_ECN();
```
- 响应最快
- 但对微突发（micro-burst）敏感，容易产生误标记

**平均队列（Averaged Queue Length）**：
```
avg_qlen = (1 - w) × avg_qlen + w × qlen
if (avg_qlen > Kmin) mark_packet_with_ECN();
```
- 更平滑，过滤掉微突发噪声
- 但延迟了响应，w 的选择需要在平滑度和响应速度之间权衡

RoCEv2 交换机通常使用瞬时队列方式，因为 RDMA 流量对延迟极其敏感——任何额外的延迟都会直接转化为性能损失。平均队列的平滑效果虽然是优点，但在微秒级敏感的 RDMA 场景中，瞬时的拥塞也需要立即响应。

### 3.3 交换机侧实现

#### Mellanox Spectrum

Mellanox Spectrum 系列交换机的 ECN 配置通过 SDK 的 Python 封装实现：

```python
# ecn_spectrum.py (简化示例)
from sdk import switch

# 创建 ECN 配置对象
ecn_config = switch.ECNConfig()
ecn_config.set_ecn_enable(True)
ecn_config.set_red_enable(True)

# 配置端口组
port_group = switch.PortGroup([1, 2, 3, 4])
ecn_config.bind_port_group(port_group)

# 设置 ECN 标记参数
ecn_config.set_ecn_config(
    min_threshold=200,     # Kmin: 200 KB
    max_threshold=600,     # Kmax: 600 KB  
    max_probability=0.2,   # 最大标记概率 20%
    ecn_only=True,         # 只做 ECN 标记，不丢弃
)
```

实际部署中，Mellanox 推荐使用 `mlnx_qos -i <interface> --ecn` 命令行工具进行配置。

#### Broadcom Trident

Broadcom Trident 系列的 ECN 配置通过 MMU（Memory Management Unit）寄存器实现。关键配置参数：

```
MMU_ECN_CONFIG:
  - ECN_THRESHOLD_MIN: Kmin 值
  - ECN_THRESHOLD_MAX: Kmax 值
  - ECN_PROBABILITY: 标记概率曲线
  - ECN_DISABLE: 禁用 ECN
```

Trident 的 ECN 实现的主要限制是标记粒度和配置复杂度——BD 的 SDK 不如 Mellanox 的直观，有时需要直接操作寄存器。

#### 硬件计数器

交换机侧的关键 ECN 计数器：

```bash
# Mellanox 交换机查看 ECN 计数器
switch # show interface ethernet 1/1 priority-group ecn

# 输出：
# PG0: ECN-marked packets: 0
# PG1: ECN-marked packets: 0
# PG2: ECN-marked packets: 0
# PG3: ECN-marked packets: 129847  <-- RoCE 优先级上有大量 ECN 标记
# PG4: ECN-marked packets: 0
```

结合 PFC 计数器和 ECN 计数器可以判断拥塞控制体系是否正常工作：

| 计数器组合 | 含义 |
|-----------|------|
| ECN 标记多 + PFC 事件少 | 健康：ECN 在 PFC 之前有效响应 |
| ECN 标记少 + PFC 事件多 | 问题：PFC 先于 ECN 触发，需要调低 ECN 阈值 |
| 两者都多 | 拥塞严重，需要检查 DCQCN 是否正常工作 |
| 两者都少但性能差 | 问题不在拥塞控制，需要排查其他原因（如链路误码） |

### 3.4 ECN 的局限

#### 标记时机困境

ECN 面临的核心困境是标记阈值的选择：

```
缓冲区占用
    ↑
    │                                ┌ PFC X-off 触发
    │                          ┌─────┤
    │                    ┌─────┤     │
    │              ┌─────┤ ECN 标记区间 │   ← 理想标记区间
    │        ┌─────┤     │     │
    │  ┌─────┤     │     │     │
    │  │     │     │     │     │
    │  │     │     │     │     │
    └──┴─────┴─────┴─────┴─────┴────────→ 时间
      Kmin   Kmax  X-off
```

- 如果 Kmin 设得太靠近 X-off → ECN 标记和 PFC 几乎同时触发 → DCQCN 来不及降速 → PFC 风暴
- 如果 Kmin 设得太低 → ECN 过早标记 → 发送端过早降速 → 吞吐浪费
- 如果 Kmax - Kmin 区间太小 → 标记概率急剧上升 → 速率剧烈振荡

这个困境的根源在于 ECN 是一个**单比特信号**——它只告诉发送端"拥塞了"，但不说"多拥塞"。DCQCN 试图通过 α 滤波器从单比特信号中"量化"拥塞程度，但这个过程取决于 CNP 的频率和分布，不精确。

#### CNP 丢包

CNP 本身也是数据包，也会在拥塞时被交换机丢弃。如果 CNP 丢失：

- 发送端不知道有 CE 标记，不降速
- DCQCN 的 α 滤波器无法准确更新
- 拥塞持续恶化，最终触发 PFC

在某些场景下，CNP 丢包率可能超过 5%，导致 DCQCN 的有效反馈信号严重失真。这是 RoCEv2 拥塞控制的已知弱点之一。

---

## 4. DCQCN —— 端到端速率控制

### 4.1 设计目标与约束

DCQCN（Data Center Quantized Congestion Notification）由 Microsoft 和 Mellanox 联合提出，论文发表于 SIGCOMM 2015。它是 RoCEv2 拥塞控制的核心算法，也是目前业界部署最广泛的方案。

#### 设计目标

DCQCN 的设计目标是在 RoCEv2 场景下实现以下特性：

1. **高吞吐**：充分利用链路带宽
2. **低队列延迟**：保持交换机队列短，减少排队延迟
3. **公平性**：多条流竞争时公平分配带宽
4. **快速收敛**：流数量或负载变化时快速达到新的平衡

#### 设计演变

DCQCN 融合了两种经典拥塞控制算法的思路：

```
DCTCP (Data Center TCP)
  │ SIGCOMM 2010
  │ ECN 标记 → α 滤波器 → 窗口调整
  │
  └──→ QCN (Quantized Congestion Notification)
       │ IEEE 802.1Qau
       │ 基于反馈的速率控制
       │
       └──→ DCQCN
            │ SIGCOMM 2015
            │ ECN + 速率控制 + α 滤波器
            │ 专为 RoCEv2 设计
```

- **DCTCP** 贡献了 α 滤波器的思想——用"拥塞程度"（收到 CE 标记的比例）而不是简单的二进制信号来调整窗口
- **QCN** 贡献了速率控制器的框架——发送端维护一个 rate limit，根据反馈信号调整
- **DCQCN** 将两者结合并适配到 RoCEv2 环境：用 ECN 替代 QCN 的专有反馈，用速率控制替代 TCP 的窗口控制

### 4.2 DCQCN 状态机

![DCQCN 状态机](images/dcqcn-state-machine.png)

DCQCN 的状态机由三种事件驱动：**收到 CNP**、**无 CNP 超时**、**字节计数器超时**。

```
                    ┌─────────────────────────────┐
                    │       正常发送状态            │
                    │      Rate = Target Rate       │
                    └──────────┬──────────────────┘
                               │
            ┌──────────────────┼──────────────────┐
            │                  │                  │
            ▼                  ▼                  ▼
    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
    │  收到 CNP    │  │ 无 CNP 超时  │  │ 字节计数器    │
    │              │  │              │  │ 超时          │
    └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
           │                 │                 │
           ▼                 ▼                 ▼
    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
    │ 更新 α       │  │ 快速恢复     │  │ 加性增       │
    │ Rate ↓ 乘性减│  │ Rate ↑  β因子│  │ Rate ↑  γ步长│
    └──────────────┘  └──────────────┘  └──────────────┘
```

#### 事件 1：收到 CNP → 乘性减（Multiplicative Decrease）

```
收到 CNP 时：
    // 1. 更新 α（拥塞因子）
    Fb = 1  (上次更新间隔内收到了 CNP)
    α = (1 - g) × α + g × Fb

    // 2. 乘性减速率
    target_rate = current_rate × (1 - α / 2)
    rate = max(target_rate, min_rate)
```

`α` 是 DCQCN 的核心参数，代表最近测量的拥塞程度。α 越大，乘性减的幅度越大。`g` 是滤波器增益，控制 α 更新的平滑程度。

#### 事件 2：无 CNP 超时 → 快速恢复（Fast Recovery）

```
无 CNP 超时时（通常每个 RTT 检查一次）：
    // 快速恢复：尝试恢复到乘性减之前的速率
    recovery_rate = rate + β × (rate_limiter - rate)
    
    // rate_limiter 是乘性减前的速率记录
    // β 是恢复因子，通常为 50%
```

快速恢复的直觉：如果链路在一段时间内没有持续拥塞（没有 CNP），那么速率可以逐步恢复到 MDF 之前的水平。β 控制恢复速度——β 太大恢复太快可能再次引发拥塞，太小则恢复太慢。

#### 事件 3：字节计数器超时 → 加性增（Additive Increase）

```
每发送 N 字节后：
    // 加性增：缓慢探测可用带宽
    rate = rate + γ × MTU_per_RTT_size
```

字节计数器比定时器更适合 RDMA 场景——在高速网络（100Gbps+）中，基于定时的 AI 可能因为短 RTT 而增长过快。使用字节计数器确保在相同的发送量后执行 AI，使速率增长与链路速率解耦。

#### 速率计算的完整伪代码

```
// DCQCN 核心速率调整算法
// 每次收到 CNP 或超时触发

rate_limiter = max(rate, rate_limiter)  // 记录历史最大速率

if (收到 CNP):
    Fb = 1
    α = (1 - g) × α + g × Fb
    rate = max(rate × (1 - α / 2), min_rate)
    rate_limiter = rate  // 更新限速

if (无 CNP 超时):
    rate = rate + β × (rate_limiter - rate)

if (字节计数器超时):
    rate = rate + γ × MTU_SIZE
    rate_limiter = max(rate_limiter, rate)

// 硬件速率限制
rate = clamp(rate, min_rate, max_rate)
```

### 4.3 α 滤波器深度分析

α 滤波器是 DCQCN 中最精巧的设计之一。它的作用是从 ECN 的**单比特信号**中提取出**量化拥塞程度**。

#### α 的定义

```
α = (1 - g) × α + g × Fb
```

- **α**：拥塞因子，范围 [0, 1]。α → 1 表示严重拥塞，α → 0 表示链路空闲
- **g**：滤波器增益，范围 (0, 1]。g → 1 表示完全相信最新信号，g → 0 表示历史平滑
- **Fb**：反馈比特，在当前更新周期内是否收到 CNP（Fb ∈ {0, 1}）

#### g 的权衡

g 的选择直接影响 DCQCN 的行为：

| g 值 | 效果 | 适用场景 |
|------|------|----------|
| 大（1/4） | α 快速跟随 Fb 变化，拥塞响应迅速 | 突发性强的负载，但可能过冲 |
| 中（1/16） | 平衡响应速度和稳定性 | **推荐初始值**，大多数场景适用 |
| 小（1/128） | α 变化缓慢，速率稳定 | 负载平稳、对抖动敏感的场景 |

#### α 的物理含义

如果在一个周期内收到了 k 个 CNP，理论上：

```
α = g × 1 + (1-g) × g × 1 + (1-g)² × g × 1 + ...
  = 1 - (1-g)^k
```

当 k → ∞ 时（持续收到 CNP），α → 1。
当 k → 0 时（没有 CNP），α → 0（由于历史 α 的指数衰减）。

α 在 0 到 1 之间的值代表"拥塞程度"——0.5 意味着约 50% 的时机处于拥塞状态。

### 4.4 速率收敛性分析

#### 单流收敛

在单一流场景下，DCQCN 的 AIMD 收敛到平衡点：

```
平衡点发生在：
    加性增 = 乘性减
    
    每字节计数器周期：rate += γ × MTU
    每 CNP：rate -= rate × α / 2
    
    在平衡点：
    γ × MTU × N_AI = rate × α / 2 × N_MD
```

其中 N_AI 是加性增的次数，N_MD 是乘性减的次数。

#### 多流公平收敛

对于 N 条共享同一瓶颈链路的 DCQCN 流，公平收敛由 AIMD 的"收敛到公平"特性保证：

```
假设有两条流 f1 和 f2，速率分别为 r1 和 r2：

1. 如果 r1 > r2 → f1 的加性增量更小（相对自身速率），乘性减量更大
                 → r1 下降速度更快，r2 下降更慢
                 → 两者速率差距缩小

2. 如果 r1 = r2 → 两者经历相同的加性增和乘性减
                 → 维持在公平平衡点
```

与 TCP 的 AIMD 区别在于速度：DCQCN 在高速链路（100Gbps+）上收敛更快，因为它的加性增基于字节计数而非 RTT 定时器。

#### 与 TCP AIMD 的对比

| 特性 | TCP AIMD | DCQCN |
|------|----------|-------|
| 信号 | 丢包（二值） | ECN（+CNP 二值） |
| 增加方式 | 每 RTT +1 MSS | 每 N 字节 +γ×MTU |
| 减小方式 | 丢包时 ×1/2 | 乘性减 ×(1-α/2) |
| 拥塞程度感知 | 无（只有丢包与否） | 有（α 量化拥塞程度） |
| 高速收敛 | 慢（RTT 依赖） | 快（字节计数与 RTT 无关） |

### 4.5 Linux 内核源码分析

DCQCN 在 Linux 中的实现主要在 Mellanox mlx5 驱动中。核心代码在 `drivers/infiniband/hw/mlx5/` 目录下。

#### 关键数据结构和函数

```c
// drivers/infiniband/hw/mlx5/qp.c

// DCQCN 参数配置
struct mlx5_dcqcn_params {
    u32 min_rate;       // 最小速率
    u32 max_rate;       // 最大速率
    u16 rate_limiter;   // 速率限幅器
    u8  g;              // α 滤波器增益
    u8  beta;           // 快速恢复因子
    u8  gamma;          // 加性增步长
};

// DCQCN 速率更新函数
// 每次收到 CNP 时由驱动调用
static void dcqcn_update_rate(struct mlx5_qp *qp, bool cnp_received)
{
    struct mlx5_dcqcn *dcqcn = &qp->dcqcn;
    
    // 更新 α
    if (cnp_received) {
        dcqcn->fb = 1;
    }
    dcqcn->alpha = (256 - dcqcn->g) * dcqcn->alpha / 256 
                    + dcqcn->g * dcqcn->fb;
    
    if (cnp_received) {
        // 乘性减
        u32 decrease = dcqcn->rate * dcqcn->alpha / 512;
        dcqcn->rate = max(dcqcn->rate - decrease, dcqcn->min_rate);
        dcqcn->rate_limiter = dcqcn->rate;
        dcqcn->fb = 0;
    } else {
        // 快速恢复
        u32 diff = dcqcn->rate_limiter - dcqcn->rate;
        dcqcn->rate += diff * dcqcn->beta / 256;
        
        // 加性增
        dcqcn->byte_count += mtu;
        if (dcqcn->byte_count >= dcqcn->byte_count_thresh) {
            dcqcn->rate += dcqcn->gamma * mtu;
            dcqcn->byte_count = 0;
        }
    }
    
    // 硬件速率配置
    mlx5_set_rate_limit(qp, clamp(dcqcn->rate, 
                                  dcqcn->min_rate, 
                                  dcqcn->max_rate));
}
```

#### CNP 接收路径

CNP 从网卡硬件到 DCQCN 模块的完整调用链：

```
1. 网卡硬件收到 UDP 报文
2. 硬件解析 BTH OpCode == 0x81（CNP）
3. 硬件产生 CQE（Completion Queue Entry），标记为 CNP 类型
4. 驱动轮询 CQ 时发现 CNP 类型 CQE
5. 调用 dcqcn_update_rate(qp, cnp_received=true)
6. 驱动将新的速率限制写入硬件寄存器
```

驱动源码中关键路径：

```c
// drivers/infiniband/hw/mlx5/cq.c
// CQ 轮询中处理 CNP

static void mlx5_handle_cnp(struct mlx5_ib_qp *qp, struct mlx5_cqe64 *cqe)
{
    // 检查 CQE 是否标记为 CNP
    if (cqe->cnp) {
        // 触发 DCQCN 速率更新
        dcqcn_update_rate(qp, true);
    }
}
```

### 4.6 参数调优指南

DCQCN 有 3 个核心参数（g, β, γ），它们的初始值和调优方向如下：

#### g（α 滤波器增益）

- **推荐初始值**：1/16（代码中表示为 16/256）
- **调优方向**：
  - 吞吐不稳定、剧烈波动 → 减小 g（如 1/32, 1/64）
  - 对拥塞响应太慢、PFC 先触发 → 增大 g（如 1/8, 1/4）
- **效果**：g 太大会让 α 对每次 CNP 都强烈反应，速率剧烈振荡；g 太小则 α 更新滞后，拥塞响应慢

#### β（快速恢复因子）

- **推荐初始值**：50%（代码中表示为 128/256）
- **调优方向**：
  - 乘性减后恢复太慢、吞吐偏低 → 增大 β（如 75%）
  - 恢复时再次引发拥塞 → 减小 β（如 25%）
- **效果**：β 控制"从乘性减中多快恢复"。β = 100% 意味着立即恢复到 decrement 前的速率，β = 0% 意味着不恢复（仅靠加性增）

#### γ（加性增步长）

- **推荐初始值**：1 MTU per rate_update_cycle
- **调优方向**：
  - 吞吐恢复太慢 → 增大 γ（如 2-4 MTU）
  - 速率过冲导致频繁 MD → 减小 γ（如 0.5 MTU）
- **效果**：γ 控制加性增的步长，决定了流在无拥塞时扫描可用带宽的速度

#### 常见配置错误

| 症状 | 可能原因 | 排查方向 |
|------|----------|----------|
| PFC 频繁触发但 ECN 标记少 | ECN 阈值太高或 DCQCN 参数太保守 | 降低 Kmin，增大 g |
| 吞吐锯齿形波动 | α 振荡（g 太大）或加性增太快（γ 太大） | 减小 g 和 γ |
| 多流不收敛（一头独大） | 快速恢复因子 β 不对称 | 检查 β 配置是否一致 |
| 空闲链路吞吐不足 | 加性增太慢（γ 太小） | 增大 γ |

---

## 5. 三者的交互与冲突

前四章分别介绍了 PFC、ECN、DCQCN 各自的工作原理。但实际部署中，这三者不是独立起作用的——它们同时作用在 Rocket 流上，相互之间既有协作也有冲突。理解三者之间的交互，才是拥塞控制调优的核心。

### 5.1 PFC vs ECN 的响应时间竞赛

#### 谁先触发？

当拥塞发生时，PFC 和 ECN 在同一个交换机出向队列上以不同的阈值独立判断：

```
缓冲区占用
    ↑
    │   ← 这里丢包             ← 尾丢弃阈值
    │   ← PFC 触发 PAUSE      ← X-off 阈值 (约 80% 缓冲区)
    │   ← ECN 标记概率 = 1.0  ← Kmax
    │   ← ECN 开始标记        ← Kmin (约 20-40% 缓冲区)
    │   ← 正常转发（不标记）
    └──────────────────────────────────────────→ 时间
```

理想的触发顺序是：

**ECN 先触发（Kmin） → DCQCN 响应降速 → 队列回落 → PFC 不需要触发**

但在以下情况中，PFC 可能在 ECN 之前触发：

1. **Kmin 设置太高**：几乎和 X-off 阈值重叠，ECN 刚标记 PFC 就触发了
2. **DCQCN 响应太慢**：即使 ECN 标记了，DCQCN 没有及时降速，队列继续增长
3. **拥塞太剧烈**：incast 场景中大量数据瞬间涌入，队列在几微秒内从 0 冲到 X-off

#### 三者的数值关系

假设一个交换机的出向端口有 512KB 的优先级缓冲区，典型的阈值配置：

```
参数              取值        占缓冲区的比例
─────────────────────────────────────────────────
Kmin (ECN 开始)   100 KB      ~20%
Kmax (ECN 全标)   300 KB      ~60%
X-off (PFC 触发)  400 KB      ~78%
Headroom          112 KB      ~22%
─────────────────────────────────────────────────
总缓冲区          512 KB      100%
```

这里的关键是：**ECN_Kmin < ECN_Kmax < PFC_Xoff** 的层级关系必须保持。从 Kmax 到 X-off 之间的区域（300 KB → 400 KB）是 DCQCN 的"响应窗口"——ECN 已经要求降速，但 PFC 还没有触发。如果 DCQCN 在这个窗口内成功降速，队列不会触及 X-off。

### 5.2 典型问题分析

#### 问题 1：PFC 风暴掩盖 ECN

这是最常见的配置问题。现象：

- `ethtool -S` 显示 PFC xoff 计数器每秒增长数百次
- ECN 标记计数器几乎没有增长（或增长远慢于 PFC）
- 吞吐量剧烈波动

根源：ECN 的 Kmin 设置太高，导致在 ECN 开始标记之前，队列长度就已经超过了 X-off 阈值。交换机根本没有机会标记 ECN 就发出了 PAUSE。

**后果**：DCQCN 没有得到 ECN 信号，不会降速，而 PFC 在反复触发和解除。整个拥塞控制体系退化为纯 PFC 控制——这恰恰是要避免的。

#### 问题 2：ECN 标记滞后

现象：
- PFC 和 ECN 几乎同时触发（PFC 计数器先增长，几微秒后 ECN 计数器也开始增长）
- DCQCN 降速时 PFC 已经触发了多轮

根源：Kmin 设置合理，但 Kmax 到 X-off 之间的窗口太小。标记概率在 Kmin 到 Kmax 之间是这样的：

```
P = (qlen - Kmin) / (Kmax - Kmin) × max_probability
```

如果 Kmax = 350 KB, X-off = 400 KB，那么 ECN 标记概率在队列长度达到 350 KB 时才达到 100%，而从 350 KB 到 400 KB 只有 50 KB 的空间。在 100Gbps 链路上，50 KB 只需要约 4μs 填满——DCQCN 在 4μs 内不可能完成降速。

**结果**：PFC 比 DCQCN 快，端到端拥塞控制失败。

#### 问题 3：吞吐振荡

这是一个正反馈循环：

```
1. PFC 触发 → 发送端暂停
2. 队列因 PAUSE 而清空 → PFC 解除
3. 发送端恢复发送 → 同时 DCQCN 也开始加性增
4. 队列迅速重新填满 → 再次触发 PFC
5. 回到步骤 1
```

这种振荡的频率取决于 RTT、缓冲区大小和 DCQCN 参数的组合。在某些配置下，振荡频率在 1-10Hz 之间，对上层应用的延迟造成显著的周期性影响。

### 5.3 调优策略

![缓冲区阈值分层示意图](images/buffer-threshold-hierarchy.png)

#### 缓冲区分层设计

现代数据中心交换机支持细粒度的缓冲区管理。推荐的分层策略：

```
总缓冲区
├── Reserved (每个端口+优先级固定预留)
│   └── 用于 PFC headroom（吸收 PAUSE 在途数据）
├── Shared Pool
│   ├── ECN 标记区间（Kmin ~ Kmax）
│   └── 超过 Kmax 后的硬限
└── 静态分配
```

- **Reserved**：为每个端口和优先级预留固定大小的 headroom，用于 PFC 的 X-off 到 buffer limit 之间的空间
- **Shared Pool**：所有动态分配共享，ECN 标记在这个池中运作

#### 推荐的阈值关系

```
Headroom > 2 × BDP（带宽延迟积）
Kmax     < X-off - Headroom
Kmin     = Kmax × 0.3~0.5
```

在数值上（以 100Gbps, 512KB 缓冲区为例）：

```
X-off    = 400 KB (78%)
Kmax     = 300 KB (59%)
Kmin     = 120 KB (23%)   ← Kmin 应在 Kmax 的 30-50%
```

#### Incast 场景案例

假设 32 个工作节点的存储读取请求同时到达 1 个聚合节点（32:1 incast）：

```
总入向带宽: 32 × 25Gbps = 800Gbps（假设每个节点 25Gbps）
出向带宽:   100Gbps（聚合端口）
过载比:     8:1
```

在 incast 开始的瞬间，聚合端口的缓冲区在几微秒内被填满。ECN 在 Kmin 处开始标记，但队列增长极快。这时唯一能阻止丢包的是 PFC 的 headroom。

解决策略：
1. **Kmin 适当降低**：让 ECN 在 incast 早期就开始标记，尽快触发 DCQCN 降速
2. **Headroom 足够大**：吸收 DCQCN 降速前到达的所有数据
3. **DCQCN 的 g 适当增大**：让 α 更快响应 CNP，加快乘性减

---

## 6. 调优实践与前沿方向

### 6.1 参数配置清单

以下是一个经过实践的初始参数配置（100Gbps RoCEv2 环境）：

#### PFC 参数

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 优先级映射 | RoCE → P3, 其他 → P0-2 | 标准 DCB 映射 |
| PFC 使能 | 仅 RoCE 优先级 | 其他优先级不启用 PFC |
| Headroom | ≥ 2 × 链路 BDP | 吸收 PAUSE 在途数据 |
| Watchdog 超时 | 100-500 ms | 防止 PFC 死锁 |

#### ECN 参数

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| Kmin | 缓冲区大小 × 20-25% | 标记起始阈值 |
| Kmax | 缓冲区大小 × 55-65% | 保证标记阈值 |
| Max Probability | 0.1-0.2 | 标记概率上限 |
| ECN Mode | ECN-only | 只标记不丢包 |

#### DCQCN 参数

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| g (α 增益) | 1/16 | 平衡响应速度和稳定 |
| β (恢复因子) | 50% | 快速恢复速率 |
| γ (加性增) | 1 MTU | 加性增步长 |
| Min Rate | 1 Gbps | 最低发送速率 |
| Rate Update Interval | 每 RTT | CNP 检查周期 |

### 6.2 监控与问题排查

#### 工具链

```bash
# 1. 检查 PFC 状态
ethtool -S eth0 | grep -E "pfc|pause"
# 观察：pfc_prio_3_xoff 是否持续增长

# 2. 检查 ECN 标记（交换机侧）
show interfaces ethernet 1/1 priority-group ecn
# 观察：ECN-marked packets 增长率

# 3. 端到端统计（建议周期性采集）
mlnx_perf --pfc --ecn --interval 1

# 4. 网卡统计
ethtool -S eth0 | grep -E "ecn|cnp"
```

#### 诊断流程图

```
性能问题？
  │
  ├── ethtool -S → PFC counter 增长？
  │       │
  │       是 → ECN counter 也在增长？
  │       │       │
  │       │       是 → DCQCN 配置检查（g, β, γ）
  │       │       否 → ECN 阈值可能太高，降低 Kmin
  │       │
  │       否 → 丢包计数增长？
  │               │
  │               是 → PFC 未覆盖（headroom 不足或配置错误）
  │               否 → 检查链路误码率（CRC Error）
  │
  └── 进一步：
      ├── 检查 DCQCN 参数：g 适中？β 合理？γ 不激进？
      └── 检查交换机缓冲区分配：headroom 够？
```

### 6.3 前沿方向

#### HPCC（High Precision Congestion Control）

HPCC（SIGCOMM 2019）利用 INT（In-band Network Telemetry）获取精确的队列信息，取代 ECN 的单比特信号。

**与 DCQCN 的核心区别：**

```
DCQCN: 得到信号"拥塞了" → α 猜测拥塞程度 → 调整速率
HPCC:  得到"队列长度 = X" → 精确计算目标速率 → 直接设置速率
```

HPCC 的优点是避免了 α 滤波器的延迟和不精确，可以实现更快的收敛和更低的队列延迟。代价是需要交换机支持 INT，这需要新一代硬件。

#### DCQCN vs TIMELY vs HPCC 对比

| 特性 | DCQCN | TIMELY | HPCC |
|------|-------|--------|------|
| 信号来源 | ECN（交换机队列） | RTT（端侧测量） | INT（交换机队列） |
| 精度 | 单比特 | 微秒级 RTT | 精确队列长度 |
| 交换机要求 | ECN 支持（几乎全部） | 无 | INT 支持（新硬件） |
| 响应速度 | RTT 级别 | RTT 级别 | 接近实时 |
| 部署范围 | 最广泛 | 较少 | 新兴 |

#### Google Swift 与 lossy RoCE

Google 在 2023 年提出了 Swift 拥塞控制，核心思想是：**不要用 PFC，让 RoCE 跑在 lossy 网络上**。

Swift 的思路：
- 放弃 PFC 和"无损"假设
- 用更快的重传机制处理丢包
- 拥塞控制基于 RTT 测量（类似 TIMELY 但更激进）

代价是重传带来的额外延迟，但好处是消除了 PFC 的所有副作用（HoL blocking、死锁、风暴）。Google 在内部生产环境中验证了这种方案的可行性。

这对 RoCEv2 拥塞控制的未来提出了一个根本性问题：**"无损"是否值得？** 或者说，PFC 带来的复杂度是否超过了它解决的问题？

---

## 附录

### A1. 术语表

| 缩写 | 全称 | 中文 |
|------|------|------|
| CNP | Congestion Notification Packet | 拥塞通知报文 |
| DCB | Data Center Bridging | 数据中心桥接 |
| DCQCN | Data Center Quantized Congestion Notification | 数据中心量化拥塞通知 |
| ECN | Explicit Congestion Notification | 显式拥塞通知 |
| FECN | Forward ECN | 前向 ECN |
| BECN | Backward ECN | 后向 ECN |
| HoL | Head-of-Line blocking | 头阻阻塞 |
| HPCC | High Precision Congestion Control | 高精度拥塞控制 |
| INT | In-band Network Telemetry | 带内网络遥测 |
| PFC | Priority Flow Control | 优先级流控 |
| QCN | Quantized Congestion Notification | 量化拥塞通知 |
| RDMA | Remote Direct Memory Access | 远程直接内存访问 |
| RoCE | RDMA over Converged Ethernet | 融合以太网上的 RDMA |

### A2. 参考文献

1. IEEE 802.1Qbb - Priority Flow Control standard
2. IEEE 802.1Qau - Congestion Notification standard
3. RFC 3168 - The Addition of Explicit Congestion Notification (ECN) to IP
4. Y. Zhu et al., "Congestion Control for Large-Scale RDMA Deployments", SIGCOMM 2015 (DCQCN 原始论文)
5. M. Alizadeh et al., "Data Center TCP (DCTCP)", SIGCOMM 2010
6. R. Mittal et al., "TIMELY: RTT-based Congestion Control for the Datacenter", SIGCOMM 2015
7. Y. Li et al., "HPCC: High Precision Congestion Control", SIGCOMM 2019
8. Linux kernel source: drivers/infiniband/hw/mlx5/qp.c
9. Linux kernel source: net/dcb/
10. Mellanox/NVIDIA, "RoCE for Enterprise" 部署指南

### A3. 配图清单

- `images/pfc-ecn-dcqcn-architecture.svg` — PFC/ECN/DCQCN 三件套架构图
- `images/pfc-pause-sequence.svg` — PFC PAUSE 帧交互时序图
- `images/ecn-marking-flow.svg` — ECN 标记 + CNP 返回流程图
- `images/dcqcn-state-machine.svg` — DCQCN 状态机转换图
- `images/buffer-threshold-hierarchy.svg` — 缓冲区阈值分层示意图（ECN/PFC 关系）
