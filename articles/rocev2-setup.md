# RoCEv2 环境搭建与测试

> 在 Apple Silicon (M5) macOS 上通过 UTM 虚拟机 + Linux SoftRoCE 模拟 RoCEv2  RDMA 网络环境。

---

## 1. 环境信息

| 节点 | IP 地址 | OS | 角色 |
|------|---------|----|------|
| linux01 | 192.168.64.4 | Ubuntu 22.04 | RDMA 服务端 |
| linux02 | 192.168.64.5 | Ubuntu 22.04 | RDMA 客户端 |

网络接口：`enp0s1`（virtio，mtu 1500）

---

## 2. SoftRoCE 原理

SoftRoCE (RXE) 是 Linux 内核的软件 RDMA 实现，它在**以太网设备**之上模拟 InfiniBand HCA（Host Channel Adapter），将 RDMA 语义（Send/Write/Read）封装到标准 UDP 数据包中传输。

RoCEv2 封包格式：
```
[IB Payload] -> [BTH] -> [UDP] -> [IP] -> [Ethernet]
```

对比 RoCEv1（在 Ethernet 层直接封装 EtherType 0x8915），RoCEv2 使用 UDP 端口 4791，**可路由**。

---

## 3. 安装步骤

### 3.1 安装 RDMA 软件包

在两台 VM 上分别执行：

```bash
sudo apt update
sudo apt install -y rdma-core infiniband-diags perftest ibverbs-utils iproute2
```

### 3.2 加载内核模块

```bash
sudo modprobe rdma_rxe
```

### 3.3 创建 RXE 设备

将 RXE 绑定到以太网接口：

**linux01：**
```bash
sudo rdma link add rxe0 type rxe netdev enp0s1
```

**linux02：**
```bash
sudo rdma link add rxe_0 type rxe netdev enp0s1
```

> 注：设备名 `rxe0` / `rxe_0` 可自定义，任意合法名称均可。

### 3.4 验证安装

```bash
# 查看 RDMA 链路状态
rdma link show

# 查看 HCA 详细信息
ibv_devinfo

# 查看 GID 表
show_gids
```

预期输出示例（linux01）：
```
link rxe0/1 state ACTIVE physical_state LINK_UP netdev enp0s1

hca_id: rxe0
    transport:          InfiniBand (0)
    state:              PORT_ACTIVE (4)
    active_mtu:         1024 (3)
    link_layer:         Ethernet
```

GID 表中 IPv4 格式（`00:00:...:ff:ff:192.168.64.x`）标识使用的是 **RoCEv2**。

---

## 4. 持久化配置

配置开机自动加载模块和创建 RXE 设备。

### 4.1 内核模块自动加载

```bash
echo rdma_rxe | sudo tee /etc/modules-load.d/rdma_rxe.conf
```

### 4.2 RXE 设备自动创建

**linux01：**
```bash
echo 'add rxe0 type rxe netdev enp0s1' | sudo tee /etc/rdma/rxe.conf
```

**linux02：**
```bash
echo 'add rxe_0 type rxe netdev enp0s1' | sudo tee /etc/rdma/rxe.conf
```

---

## 5. 性能测试

### 5.1 RDMA Write 带宽测试

**服务端（linux01）：**
```bash
ib_write_bw -d rxe0 --report_gbits
```

**客户端（linux02）：**
```bash
ib_write_bw -d rxe_0 --report_gbits 192.168.64.4
```

结果：
```
#bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]
65536      5000            1.85               1.85
```

### 5.2 RDMA Read 带宽测试

**服务端（linux01）：**
```bash
ib_read_bw -d rxe0 --report_gbits
```

**客户端（linux02）：**
```bash
ib_read_bw -d rxe_0 --report_gbits 192.168.64.4
```

结果：
```
#bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]
65536      1000            2.54               2.54
```

> SoftRoCE 完全依赖 CPU 做软件封装，性能受限于 CPU 和 virtio 网络效率，通常 2~10 Gb/s。生产环境需要真实 RDMA 网卡。

---

## 6. 常用命令参考

| 用途 | 命令 |
|------|------|
| 查看 RDMA 链路 | `rdma link show` |
| 查看 HCA 信息 | `ibv_devinfo` |
| 查看 GID 表 | `show_gids` |
| 查看统计计数 | `rdma stat show` |
| Write 带宽测试 | `ib_write_bw -d <dev> --report_gbits` |
| Read 带宽测试 | `ib_read_bw -d <dev> --report_gbits` |
| Send 带宽测试 | `ib_send_bw -d <dev> --report_gbits` |
| RC pingpong 延迟 | `ibv_rc_pingpong -d <dev> -g 1` |
| UC pingpong 延迟 | `ibv_uc_pingpong -d <dev> -g 1` |
| UDP pingpong 延迟 | `ibv_ud_pingpong -d <dev> -g 1` |

> `-g 1` 指定使用 GID index 1（RoCEv2 IPv4 GID），而非 index 0（RoCEv1 或缺省值）。

---

## 7. 注意事项

1. **MTU 一致性**：两端 MTU 应匹配，`ibv_devinfo` 中 active_mtu 显示协商后的值（本例为 1024）。
2. **防火墙**：RoCEv2 使用 UDP 端口 4791，需确保未被阻塞。
3. **性能**：SoftRoCE 仅适用于功能验证和开发测试，不适合性能基准测试。
4. **Apple Silicon 虚拟机**：UTM 使用 virtio 网络，CPU 模拟开销会影响带宽；如需更高性能可尝试调整 vCPU 和 virtio 队列数。

---

## 8. 参考

- [RDMA Core Project](https://github.com/linux-rdma/rdma-core)
- [SoftRoCE — Linux Kernel Documentation](https://www.kernel.org/doc/html/latest/infiniband/opa.html)
- [libibverbs — Programming Guide](https://github.com/linux-rdma/rdma-core/tree/master/libibverbs)
