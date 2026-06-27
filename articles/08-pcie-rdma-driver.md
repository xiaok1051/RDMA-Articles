# RDMA 驱动开发需要的 PCIe 知识

> 如果你是 RDMA 驱动开发者，这篇文章告诉你 PCIe 方面**需要知道什么、可以忽略什么**。
>
> 你不需要成为 PCIe 硬件专家——只需要理解 BAR、MMIO、MSI-X、DMA 这几个概念如何在驱动代码中落地。每个概念都配有代码片段和原理说明，按 probe 执行顺序组织，读完就能跟 erdma/mlx5/ionic 等驱动的 probe 流程对上号。

## 1. PCIe 物理拓扑

```
CPU
 │
 └── Root Complex (RC)        ← CPU 连接 PCIe 世界的"桥头堡"
       │
       ├── PCIe Switch        ← 像交换机，扩展出多个端口
       │    ├── Endpoint      ← 你的 RDMA 网卡
       │    └── Endpoint
       │
       └── Endpoint (直连)
```

驱动开发者只关心 **Endpoint（网卡本身）**，Root Complex 和 Switch 是硬件/firmware 管的。

---

## 2. 配置空间（Configuration Space）

每个 PCIe 设备上电后都有一块配置空间，内核 PCI 子系统枚举总线时读这块空间来发现设备。

前 64 字节布局（RDMA 驱动关心的）：

```
偏移   大小   字段                     说明
────   ───   ────────────────────   ──────────────────────────
0x00    2    Vendor ID              厂商 ID: Alibaba = 0x1ded
0x02    2    Device ID              产品 ID: erdma = 0x107f
                                      ↑ 这两项就是 pci_device_id 的匹配依据
0x04    2    Command Register
0x06    2    Status Register
...
0x10    4    BAR0
0x14    4    BAR1
0x18    4    BAR2
0x1C    4    BAR3
0x20    4    BAR4
0x24    4    BAR5                    ← 最多 6 个 BAR
...
0x34    4    Capabilities Pointer    ← 指向 MSI-X 等能力结构
```

内核自动完成枚举和匹配：

```c
pci_read_config_word(pdev, PCI_VENDOR_ID, &vendor);  // 读 0x00
pci_read_config_word(pdev, PCI_DEVICE_ID, &device);  // 读 0x02
// 匹配 pci_device_id 表 → 调 probe()
```

---

## 3. BAR（Base Address Register）

### 是什么

每个 PCIe 设备有 6 个 BAR 寄存器（BAR0~BAR5），描述设备暴露给软件的地址空间大小和类型。

### 工作机制

```
PCI 枚举时（BIOS/内核）：
  1. 读 BAR → 硬件告诉你"我需要多大地址空间"（写全 1 再读回来）
  2. 分配物理地址 → 系统在物理地址空间里给你划一块地
  3. 写 BAR → 把分配好的起始地址写回去

驱动 probe 时：
  pci_resource_start(pdev, bar_index)  → 拿到那块的物理起始地址
  devm_ioremap()                       → 映射到内核虚拟地址空间
  之后通过 MMIO 读写硬件寄存器
```

### 类比

BAR 就像**设备留给 CPU 的一扇窗**。硬件内部的寄存器、内存、doorbell 都在这个窗口后面。

### 3.5. Doorbell（门铃寄存器）

Doorbell 是 RDMA 驱动中最频繁的 MMIO 操作——**驱动通知硬件"有活干了"**：

- **SQ Doorbell**: 驱动把 WQE 写入 DMA 内存后，写 SQ doorbell 告诉硬件去取
- **CQ Doorbell**: 驱动处理完 CQE 后，写 CQ doorbell 告诉硬件可以重用 CQ 槽位

门铃寄存器在 BAR 映射的 MMIO 空间中。每次 `writel(sq_db_val, dev->bar + SQ_DB_OFFSET)` 就是一次 doorbell。

因为 doorbell 在**数据路径**上——每个发送包都要写一次——性能至关重要。这就是 RDMA 驱动用 `ioremap_wc()` 映射 BAR 的原因（详见下文）。

---

## 4. MMIO（Memory-Mapped Input/Output）

### 核心思想

**把硬件寄存器映射成内存地址，用读写内存的指令来操作硬件。**

```
传统端口 I/O（PIO）: 用 in/out 专用指令读写设备
MMIO: 用普通的 load/store 指令读写映射后的地址
```

MMIO 胜出是因为：**不需要特殊指令，一切就是 load/store**。

### 驱动中的 MMIO 流程

```c
// 1. probe 里建立映射
dev->func_bar = devm_ioremap(&pdev->dev, 物理地址, 长度);

// 2. 之后所有硬件访问都是 MMIO
u32 version = readl(dev->func_bar + ERDMA_REGS_VERSION_REG);   // MMIO 读
writel(val, dev->func_bar + db_reg_offset);                     // MMIO 写
```

`readl()` / `writel()` 保证：
- 32 位原子操作
- 带内存屏障，防止编译器/CPU 重排
- 翻译成 PCIe TLP 事务

### MMIO vs DMA

```
MMIO: CPU 主动读写设备寄存器
      CPU ——load/store——→ PCIe 设备（通过 BAR）

DMA:  设备主动读写内存
      PCIe 设备 ——read/write——→ 内存（不需要 CPU 参与）
```

MMIO = CPU **控制**设备；DMA = 设备**搬运数据**。两者配合：CPU 通过 MMIO 设置好 DMA 描述符，写 doorbell，硬件自动做 DMA。

### 4.1. ioremap_wc：写合并映射

默认 `ioremap()` 使用 UC（Uncacheable）映射，每次 `writel()` 直接生成一个 PCIe Memory Write TLP。这对控制寄存器是正确的（写入必须立即到达），但对 doorbell 路径是性能灾难——每次发送都走一次完整 PCIe 事务。

RDMA 驱动中的典型做法：

```c
dev->bar = devm_ioremap_wc(&pdev->dev,
                           pci_resource_start(pdev, BAR_INDEX),
                           pci_resource_len(pdev, BAR_INDEX));
```

WC（Write-Combining）映射允许 CPU 将多次写入合并在 WCB 中，一次刷出到 PCIe 总线：

```
UC 映射：4 次单独 writel → 4 个独立的 Memory Write TLP（各 4 字节）
WC 映射：4 次连续写入 → CPU 合并 → 1 个 Memory Write TLP（16 字节）
```

对于 64B 的 WQE doorbell，配合 `memcpy_toio()` + WC 映射，可以做到单次 cacheline 写入。

**WC 的代价**：WC 区域不可读——读操作会触发 WCB 强制刷出，然后走 non-posted read TLP，性能极差。控制类寄存器仍用普通 `ioremap()`，只有数据路径 doorbell 用 `ioremap_wc()`。

### 4.2. Posted Write 与 Readback Flush

`writel()` 是 PCIe **posted 事务**——数据发出后不需要 Completion TLP 回应。CPU 不等它到达硬件就返回了：

```c
writel(doorbell_val, dev->bar + DB_OFFSET);
// 执行到这里时 writel 可能还在路上
```

如果需要保证 writel 已经到达硬件，可以用 **readback flush**：

```c
writel(cmd, dev->bar + CMD_REG);
readl(dev->bar + CMD_REG);   // non-posted 读，强制等前面的 posted 写完成
```

`readl()` 是 non-posted 事务——发出 Memory Read TLP，必须等设备返回 Completion TLP。PCIe ordering 规则保证：non-posted 事务不会越过前面的 posted 事务，所以 readl 返回时，前面的 writel 一定已经到达设备。

---

## 5. MSI-X（Message Signaled Interrupts eXtended）

### 传统 PCI 中断的问题

多个设备共享 INTA#/INTB# 引脚 IRQ，驱动在 handler 里要挨个查是不是自己的设备，效率低。

### MSI-X 的原理

设备通过**写特定地址**来触发中断，不再走引脚。每个 MSI-X vector 包含：

```
[目标地址 (64bit)] + [data (16bit)]
```

设备写这个地址 → 中断控制器收到 → 翻译成 CPU IRQ。

### 补充：Capabilities 链表

MSI-X 不是凭空出现的特性，它挂在 PCIe **Capabilities 链表**上：

```
配置空间 0x34 → Cap Pointer → Cap A → Cap B → ... → MSI-X Cap → NULL
```

每个 Capability 包含 ID、下一项偏移、以及专有字段。MSI-X Capability 里定义了 Table 和 PBA 在 BAR 空间中的偏移。

内核通过 `pci_find_capability(pdev, PCI_CAP_ID_MSIX)` 遍历链表找到 MSI-X。驱动开发者通常不需要手动遍历，但理解这个机制有助于明白为什么 MSI-X 需要单独分配（非 PCI 规范强制）、以及为什么 MSI-X Table 和 PBA 会占用 BAR 空间的一部分。

### 关键特点

- **每个 vector 独立** — 不共享，handler 不需要猜
- **最多 2048 个 vector**（MSI 只支持 32 个）
- **可绑 CPU** — `irq_set_affinity_hint()` 实现中断亲缘性

### RDMA 驱动中的使用

```c
// 第一步：分配 MSI-X vector
// 这一句封装了：找 MSI-X Capability → 配置 Table → 使能
pci_alloc_irq_vectors(dev->pdev, 1, num_cpus + 1, PCI_IRQ_MSIX);

// 第二步：拿 Linux IRQ 号
int irq = pci_irq_vector(pdev, vector_index);

// 第三步：注册 handler
request_irq(irq, my_handler, 0, "my-device", dev);

// 第四步：绑 CPU
irq_set_affinity_hint(irq, &cpumask);
```

典型分配策略：

```
Vector 0    → 管理中断（命令队列完成、硬件事件）
Vector 1~N  → 数据中断（CQ 完成事件），每个绑一个 CPU
```

---

## 6. TLP（Transaction Layer Packet）— 理解原理即可

PCIe 上用数据包通信。`readl`/`writel` 在总线层面的真实样子：

```
写 TLP（Posted，不需要回复）：
  writel(val, addr)
  → 内存控制器发现地址是 MMIO
  → 生成 Memory Write TLP → PCIe 总线 → 设备解码地址写入寄存器

读 TLP（Non-Posted，需要回复）：
  val = readl(addr)
  → 生成 Memory Read TLP → PCIe 总线 → 设备
  → 设备返回 Completion TLP（含数据）
  → CPU 拿到值
```

**驱动开发者不需要写 TLP 相关代码**，全被 CPU/硬件封装了。但理解这个机制有助于 debug。

---

## 7. DMA（Direct Memory Access）

RDMA 的核心：**网卡直接读写主机内存，不经过 CPU**。

从 PCIe 角度看：

```
DMA 写（网卡 → 内存）：
  网卡发起 Memory Write TLP
  → 地址 = 驱动告诉网卡的物理地址
  → TLP 经过 Root Complex → 写入 DRAM

DMA 读（内存 → 网卡）：
  网卡发起 Memory Read TLP
  → Root Complex 从 DRAM 读取数据
  → 返回 Completion TLP 给网卡
```

这就是 RDMA 能零拷贝的原因——数据直接从网卡到应用内存。

驱动中通过 DMA API 管理：

```c
// 设置 DMA 地址宽度（设备支持 64 位还是 32 位）
dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(64));

// 分配 DMA 内存（硬件可直接访问）
dma_pool_create("erdma_resp_pool", &pdev->dev, size, align, 0);

// 映射内存给 DMA（注册 MR 时用到）
dma_addr_t dma_addr = dma_map_single(&pdev->dev, cpu_addr, size, dir);
```

### 关键一步：开启 Bus Master

驱动 probe 中必须调用 `pci_set_master(pdev)`，它设置 PCI Command Register 中的 Bus Master 位。**没有这一位，设备不能发起任何 DMA**——PCIe 总线会阻止设备发出的 Memory Read/Write TLP。

类似地，`pci_enable_device(pdev)` 设置 Command Register 的 Memory Space 位，告诉总线允许设备响应 MMIO 访问（即 CPU 读写 BAR 时设备能回应 Completion）。

事实上，`pci_enable_device` + `pci_set_master` + `pci_request_regions` 是所有 PCIe 驱动 probe 的**标准三板斧**：

```c
static int my_pci_probe(struct pci_dev *pdev, const struct pci_device_id *id)
{
    pci_enable_device(pdev);     // 第 1 板斧：开启 MMIO 响应
    pci_set_master(pdev);        // 第 2 板斧：允许 DMA
    pci_request_regions(pdev);   // 第 3 板斧：声明 BAR 资源所有权
    // ... 接下来分配 MSI-X、ioremap BAR、初始化硬件
}
```

---

## 8. RDMA 驱动的 PCIe 知识边界

```
需要理解：                                   可以忽略：
────────────────────────────                ────────────────────────────
PCIe 设备模型（vendor/device ID）             PCIe 链路层训练/协商
BAR 是什么、怎么 ioremap                      Lane 数/速率/Gen1~5
MSI-X 是什么、怎么分配                        Flow Control / Credits
MMIO readl/writel 的含义                      AER/DPC 错误处理细节
DMA 的基本概念（设备读写内存）                 ATS/SR-IOV 等高级特性
TLP 存在（知道底层是包传输）                   I/O 虚拟化（IOMMU/SMMU）
                                              PCIe 电源管理（ASPM）
```

---

## 9. RDMA 驱动 PCIe API 速查

```c
// probe 中的标准 PCIe 操作序列
pci_enable_device(pdev);                    // 开启 PCI 命令寄存器
pci_set_master(pdev);                       // 开启 Bus Master，允许 DMA

pci_select_bars(pdev, IORESOURCE_MEM);      // 扫描内存 BAR
pci_request_selected_regions(pdev, bars, name); // 申请 BAR 资源所有权

pci_resource_start(pdev, bar);              // BAR 分配的物理地址
pci_resource_len(pdev, bar);                // BAR 大小
devm_ioremap(dev, addr, len);               // 物理地址 → 虚拟地址（MMIO）

pci_alloc_irq_vectors(pdev, min, max, PCI_IRQ_MSIX);  // 分配 MSI-X
pci_irq_vector(pdev, idx);                  // MSI-X index → Linux IRQ 号
request_irq(irq, handler, 0, name, dev);    // 注册中断处理函数
irq_set_affinity_hint(irq, &cpumask);       // 中断绑核

dma_set_mask_and_coherent(pdev, DMA_BIT_MASK(64)); // DMA 地址宽度
dma_pool_create(name, &pdev->dev, size, align, 0); // DMA 内存池
dma_map_single(&pdev->dev, buf, size, dir); // CPU 地址 → DMA 地址
```

这些函数在每个 PCIe 驱动的 probe 里**几乎一模一样**，照着 erdma_main.c 写就行。
