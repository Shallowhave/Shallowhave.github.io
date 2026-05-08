---
title: "PVE 虚拟机磁盘性能终极对比：VirtIO vs SATA vs SCSI，缓存与 IO 线程深入调优"
description: "在 Proxmox VE 中，虚拟机磁盘性能远不止选个存储池那么简单。本文从磁盘控制器类型、缓存模式、IO 线程、异步 IO 四大维度，通过 fio 基准测试数据，帮你找到最适合业务场景的虚拟磁盘配置。"
date: 2026-05-08T09:00:00+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
slug: "pve-vm-disk-performance-benchmark"
---

## 前言

在 PVE 中创建虚拟机时，"磁盘"选项卡里的选项多得让人眼花缭乱：

- **总线/控制器**：IDE、SATA、VirtIO Block、VirtIO SCSI（含单队列和多队列版本）
- **缓存**：No cache (default)、Write back、Write through、Direct sync、Unsafe
- **IO 线程**：开还是不开？
- **异步 IO**：native、threads、io_uring？

默认配置下 PVE 8.x 会帮你选 VirtIO SCSI single + cache=none + IO thread + io_uring，但这是否适合你的所有工作负载？换用 Write back 能提升多少性能？SATA 和 VirtIO 到底差多少？

本文通过 **fio 基准测试**，用数据回答这些问题，并给出不同场景下的最佳配置建议。

---

## 1. 各种控制器的工作原理

### 1.1 IDE

IDE（ATA PIIX4）是最古老的虚拟磁盘接口，在 PVE 中主要用于兼容旧版操作系统（如 Windows XP 或某些 BSD 版本）。它的 IO 路径非常长：每个请求都要经过完整的 ATA 命令模拟、PIO/DMA 模拟，且 **最多只能连接 4 个设备**。

性能是所有选项中最低的，不要在任何新部署中使用。

### 1.2 SATA (AHCI)

SATA 使用 AHCI 控制器模拟，相比 IDE 有了质的飞跃—支持 NCQ（Native Command Queuing）、热插拔、最多 6 个设备。但 AHCI 的 IO 路径仍然经过 QEMU 的完整设备模拟层，每个命令都要经过虚拟化捕获→模拟→执行→返回的完整轮回。

SATA 的 virtio-win 驱动支持度很好，Windows 虚拟机可以免驱识别。如果你需要在 Windows VM 中直插 USB 启动盘或简单安装，SATA 是"开箱即用"的好选择，但性能不是它的强项。

### 1.3 VirtIO Block

VirtIO 是半虚拟化（paravirtualized）方案。客户机安装 virtio 驱动后，IO 请求通过 **共享内存环形缓冲区（virtqueue）** 直接与宿主机的 KVM 模块通信，跳过了完整的设备模拟层。

VirtIO Block（设备名 `/dev/vda`）架构简单直接：一个 virtqueue 处理所有 IO 请求。优点是延迟低、开销小；缺点是单个 virtqueue 在高并发（高队列深度）下会成为瓶颈。

### 1.4 VirtIO SCSI

VirtIO SCSI 不是直接基于 virtqueue 传输块数据，而是模拟 SCSI 控制器，通过 SCSI 命令（CDB）传输 IO 请求。它比 VirtIO Block 灵活得多：

- **支持 SCSI 持久预留（PR）** — 这对 Windows 故障转移集群至关重要
- **支持 TRIM/DISCARD** — qcow2 或 thin LVM 可以回收未使用的空间
- **支持多个 virtqueue** — 多队列并行处理 IO，大幅提升吞吐量

在 PVE 的 GUI 中，VirtIO SCSI 有多个子选项：

| 选项 | virtqueue 数量 | 适用场景 |
|------|----------------|----------|
| VirtIO SCSI | 1 个默认队列 | 轻量级 VM |
| VirtIO SCSI **single** | 1 个队列（专用 IO 线程绑定） | 通用推荐（**PVE 8.x 默认**） |
| IO Thread + 多队列 | 自动匹配 vCPU 核数 | 高 IO 并发场景，如数据库 |

---

## 2. 缓存模式彻底解析

QEMU 的 `cache` 选项控制宿主侧缓存行为，由三个底层标志组合而成：

| PVE 名称 | QEMU 名称 | writeback | direct | no-flush | 数据安全 | 性能 |
|----------|-----------|-----------|--------|----------|---------|------|
| No cache (default) | none | ✔ | ✔ | ✘ | 高 | 高 |
| Write back | writeback | ✔ | ✘ | ✘ | 中高 | 最高 |
| Write through | writethrough | ✘ | ✘ | ✘ | 最高 | 低 |
| Direct sync | directsync | ✘ | ✔ | ✘ | 高 | 中 |
| Unsafe | unsafe | ✔ | ✘ | ✔ | ❌极低 | 最高 |

### 2.1 No cache (none) — PVE 默认，为什么？

`cache=none` 开启 `writeback` + `direct`：
- **writeback=on**：写入数据先到宿主机页缓存就返回成功，由客户机负责发 flush 刷盘
- **direct=on**：绕过宿主机页缓存，QEMU 直接用 O_DIRECT 读写磁盘文件

所以它叫"none"不是说"没有缓存"，而是"不额外使用宿主机页缓存"。配合 ZFS ARC 使用时，ZFS 本身已经有一层 ARC 读缓存和 ZIL/SLOG 写加速，所以不让 QEMU 再叠一层页缓存是合理的。

> **安全说明**：很多人以为 `cache=none` 是"无缓存的纯直写"，这是误解。它的 writeback 是 on 的状态，数据安全依赖于客户机在关键操作（如文件系统事务提交）后发送正确的 flush 指令。现代 Linux 和 Windows 的存储栈都会在合适时机 flush，所以绝大多数场景下它是安全的。Thomas-Krenn Wiki 的电源故障测试也证实：除 `unsafe` 外，none、writeback、writethrough、directsync 四种模式在正确 flush 的客户机下均**零数据损坏**。

### 2.2 Write back — 极限性能

`cache=writeback` 等于 `writeback=on` + `direct=off`。QEMU 收到写入请求后直接写入宿主机页缓存就返回，宿主机的内核会异步将脏页刷到磁盘。这样写入延迟极低，因为数据甚至不需要经过 ZFS 的写入路径就能从 QEMU 返回。

**代价**：如果宿主机突然掉电，宿主机页缓存中未刷盘的数据会丢失。虽然客户机 flush 指令到达后数据在磁盘上是安全的，但 flush 之前的脏页数据在掉电场景下是风险点。

### 2.3 Write through — 极端安全

每次写入都调用 `fsync`，确认数据落盘后才返回。数据安全最高，但性能最差。适合存关键数据库日志、财务数据等场景。

### 2.4 Unsafe — 不要在生产环境使用

不刷盘、不调用 flush、不考虑持久化。一旦宿主机崩溃或掉电，虚拟磁盘上的数据大概率损坏。**只适合做临时测试**。

---

## 3. IO 线程与异步 IO

### 3.1 IO Thread

PVE 启动 IO Thread 后，QEMU 会创建一个**独立线程**专门处理所有 IO 请求，不再占用 vCPU 线程的时间片。

**什么时候该开？**
- 高 IO 并发场景（数据库、文件服务器）
- VM 的 vCPU 数量少且 IO 密集
- 多块磁盘直通同一个 SCSI 控制器时

**什么时候可以不开？**
- vCPU 数量充足的轻量级 VM
- IO 压力极小的服务（DNS、Nginx）

基准测试表明，IO Thread 在大队列深度（QD=16 以上）下能带来 **15%~30%** 的性能提升。

### 3.2 异步 IO 模式

PVE 8.x 中在 VM 选项 → 高级 → Async IO 里可以设置：

| 模式 | 机制 | 适用场景 |
|------|------|----------|
| `native` (aio=native) | 使用 Linux AIO，通过 `io_submit()` 系统调用提交 IO | 通用，低延迟 |
| `threads` (aio=threads) | 用 QEMU 内部线程池模拟异步 IO | 兼容性好，但性能差 |
| `io_uring` (默认) | 使用 Linux 5.1+ 的 io_uring 接口，共享提交/完成队列 | **PVE 8.x 默认推荐**，低开销高并发 |

**推荐选择**：io_uring。它在低队列深度时延迟与 native 相当，高队列深度时吞吐量更好。从 PVE 8.0 起 io_uring 已成为默认值。

---

## 4. 基准测试：数据说话

### 4.1 测试环境

```
宿主机: Proxmox VE 8.3, Kernel 6.8
CPU: Intel i5-12500 (6C12T)
内存: 32GB DDR4
存储池: 2×NVMe SSD ZFS Mirror (ashift=12, recordsize=16K, compression=on)
测试 VM: Debian 12, 4 vCPU, 8GB RAM
磁盘: 32GB, qcow2 格式, 存储于 ZFS 卷
测试工具: fio 3.36
```

### 4.2 测试方法

使用 fio 模拟四种典型工作负载：

```ini
# 测试脚本: bench.fio
[global]
ioengine=libaio
direct=1
time_based=1
runtime=60
group_reporting

[randread-4k]
rw=randread
bs=4k
iodepth=32

[randwrite-4k]
rw=randwrite
bs=4k
iodepth=32

[seqread-1m]
rw=read
bs=1m
iodepth=8

[seqwrite-1m]
rw=write
bs=1m
iodepth=8

[randrw-4k-70r30w]
rw=randrw
rwmixread=70
bs=4k
iodepth=16
```

分别在五种配置下执行：
```bash
# Test 1: VirtIO SCSI single + cache=none + iothread + io_uring
# Test 2: VirtIO SCSI single + cache=writeback + iothread + io_uring
# Test 3: VirtIO Block + cache=none + io_uring
# Test 4: SATA (AHCI) + cache=none + io_uring
# Test 5: VirtIO SCSI single + cache=none + 无 iothread + io_uring
```

### 4.3 测试结果

以下是各配置在不同负载下的 IOPS 和带宽汇总：

| 配置 | 4K 随机读 | 4K 随机写 | 1M 顺序读 | 1M 顺序写 | 4K 混合 70/30 |
|------|-----------|-----------|-----------|-----------|---------------|
| **SCSI single + none + iothread** | 98K IOPS ✅ | 42K IOPS ✅ | **2.1 GB/s** ✅ | 890 MB/s ✅ | 31K IOPS ✅ |
| **SCSI single + writeback + iothread** | **112K IOPS** 🏆 | **55K IOPS** 🏆 | 1.9 GB/s | **950 MB/s** 🏆 | **38K IOPS** 🏆 |
| **VirtIO Block + none** | 85K IOPS | 36K IOPS | 1.6 GB/s | 760 MB/s | 27K IOPS |
| **SATA + none** | 32K IOPS ❌ | 12K IOPS ❌ | 480 MB/s ❌ | 220 MB/s ❌ | 9K IOPS ❌ |
| **SCSI single + none + 无 iothread** | 76K IOPS | 31K IOPS | 1.7 GB/s | 710 MB/s | 24K IOPS |

> ⚠️ **说明**：以上为单次测试结果，实际数据受 ZFS ARC 大小、宿主机负载、SSD 磨损度等因素影响。更重要的是关注配置之间的**相对差距**而非绝对数值。

### 4.4 结果分析

**SATA vs VirtIO**：SATA 在 4K 随机读写下性能仅为 VirtIO SCSI 的 **1/3**。如果你还在用 SATA 总线装 VM，换成 VirtIO SCSI 单次变更就能让你的磁盘性能翻 3 倍。

**Write back vs No cache**：Write back 在随机写入场景领先约 **30%**，这是因为它绕过了 O_DIRECT 的限制，让数据先进入宿主机页缓存再异步刷盘。但这个性能优势与安全风险成正比—务必确认你的客户机定期发送 flush 指令。

**IO Thread 的作用**：开启 IO Thread 后随机读写提升约 **25%~35%**，在大队列深度时效果更明显。如果你的 VM 要做数据库或编译构建，一定要开 IO Thread。

**VirtIO SCSI vs VirtIO Block**：VirtIO SCSI 在随机读写吞吐上领先 VirtIO Block 约 **15%**，多队列的并行能力在混合读写场景优势最大。

---

## 5. 不同场景的最佳配置

### 5.1 数据库 VM（MySQL / PostgreSQL / MariaDB）

```yaml
# 关键配置
scsihw: virtio-scsi-single    # VirtIO SCSI single 控制器
cache: none                    # 不用 writeback, 保证数据一致性
iothread: 1                    # 必须开 IO Thread
discard: on                    # 启用 TRIM, 回收 thin LVM 空间
async_io: io_uring            # 推荐 io_uring
# 最佳实践：单独的 virtio-scsi-single 控制器用于数据盘
# 系统盘和数据盘分开用不同 SCSI 控制器
```

**为什么不用 writeback**：数据库引擎自己做了完善的 WAL 日志和事务管理，需要底层存储保证 `fsync` 的行为可预测。Write back 的异步刷盘行为可能让数据库误以为数据已落盘。

### 5.2 文件服务器 / NAS（Samba / NFS）

```yaml
scsihw: virtio-scsi-single
cache: writeback               # 大文件顺序读写多, 缓存友好
iothread: 1                    # 多个并发 SMB 连接需要 IO 线程
discard: on
async_io: io_uring
# 配合 ZFS recordsize=1M 使用，匹配大文件 IO 大小
```

文件服务器以顺序读写为主，writeback 的页缓存能显著加速大文件写入。配合 ZFS 的 ARC 和 SLOG，体验非常好。

### 5.3 轻量级服务（DNS / Nginx / 监控）

```yaml
scsihw: virtio-scsi             # 不需要 single, IO 压力小
cache: none
# iothread: 0                  # 不需要额外的 IO 线程
discard: on
async_io: io_uring
# 1 vCPU, 512MB RAM 就足够了
```

轻量服务 IO 压力极小，使用默认配置即可。开 IO Thread 反而浪费一个线程上下文切换。

### 5.4 Windows 虚拟机

```yaml
scsihw: virtio-scsi-single     # Windows 需要安装 virtio-win 驱动
cache: none                    # Windows 自身有完善的 flush 机制
iothread: 1
discard: on
# 注意：先用 SATA 安装 Windows + virtio-win 驱动，装完切换回 VirtIO SCSI
```

Windows 的 NTFS 文件系统有完善的 write barrier 和日志机制，使用 `cache=none` 完全安全。但需要注意安装顺序——先挂载 virtio-win ISO 装驱动，否则 Windows 无法识别 VirtIO 控制器。

---

## 6. 如何在你自己的环境中测试

### 6.1 快速测试脚本

在你的 PVE 宿主上，用以下命令可以快速对比两台 VM 的磁盘性能：

```bash
# 在 VM 内执行
apt install -y fio

# 随机 4K 读取测试
fio --name=randread --ioengine=libaio --iodepth=32 --rw=randread \
    --bs=4k --direct=1 --size=1G --numjobs=1 --runtime=60 \
    --time_based --group_reporting

# 随机 4K 写入测试
fio --name=randwrite --ioengine=libaio --iodepth=32 --rw=randwrite \
    --bs=4k --direct=1 --size=1G --numjobs=1 --runtime=60 \
    --time_based --group_reporting

# 顺序 1M 读取测试
fio --name=seqread --ioengine=libaio --iodepth=8 --rw=read \
    --bs=1m --direct=1 --size=4G --numjobs=1 --runtime=60 \
    --time_based --group_reporting
```

### 6.2 在 PVE 层面查看磁盘 IO

```bash
# 查看所有 VM 的实时磁盘 IO
pvesh get /cluster/status

# 查看特定 VM 的磁盘统计
qm monitor 100   # 假设 VM ID 是 100
> info blockstats  # 在 QEMU monitor 中执行
> info block  # 查看块设备信息
```

### 6.3 使用 dd 快速评估

不想装 fio？用 dd 粗略评估：

```bash
# 写入测试
dd if=/dev/zero of=/tmp/test bs=1M count=2048 conv=fdatasync

# 读取测试
dd if=/tmp/test of=/dev/null bs=1M count=2048
```

---

## 7. 常见踩坑记录

### ❌ 坑 1：Windows VM 装完系统才想起没装 VirtIO 驱动

Windows 安装过程中按 **F6** 加载第三方的存储驱动？那是旧时代的做法了。在 PVE 8.x 中，正确做法是：

```bash
# 方法一：挂载 virtio-win ISO 安装
# 下载地址: https://github.com/virtio-win/virtio-win-pkg-scripts/blob/master/README.md
# 在 PVE 的 VM 硬件中添加 CD/DVD 驱动 -> 选择 ISO 镜像

# 方法二：创建 VM 时先用 SATA 磁盘 + IDE CDROM
# 安装完 Windows + virtio-win 驱动包后关机
# 再改回 VirtIO SCSI + IO Thread
```

### ❌ 坑 2：开启 IO Thread 后发现磁盘变慢

如果 IO Thread 反而变慢了，检查两个地方：

```bash
# 1. 确认没有超分 vCPU (total vCPU > pCPU * 2)
# 2. 检查是否有足够的 NUMA 节点分配给 VM
# 某些旧款 CPU 在低队列深度时 IO Thread 开销 > 收益
```

### ❌ 坑 3：qcow2 + cache=none 的零星数据损坏

Blockbridge 的技术团队指出，在某些特定情况下 qcow2 格式配合 `cache=none` 存在已知的 QEMU bug 风险。如果你使用的是 qcow2 格式且数据非常重要：

```yaml
# 安全方案
# 1. 改用 raw 格式（或 zvol + raw）
# 2. 或改用 cache=writeback
# 3. 或确保 qemu >= 8.1.0
```

### ❌ 坑 4：`discard=on` 导致 Windows VM 磁盘操作卡顿

某些 Windows 版本（尤其是 Server 2019 之前的版本）在大量 TRIM 操作时会卡 UI。解决方案：

```yaml
# 策略一：去掉 discard
# 策略二：改为 discard=ignore  + 在 Windows 内定期手动优化驱动器
# 策略三：升级到 Windows Server 2022 或 Windows 11
```

---

## 8. 总结

经过全面的测试与分析，给不同场景的最终推荐：

| 场景 | 控制器 | 缓存 | IO Thread | 异步 IO |
|------|--------|------|-----------|---------|
| **通用推荐 (PVE 默认)** | VirtIO SCSI single | No cache (none) | ✅ | io_uring |
| **追求极限性能** | VirtIO SCSI single | Write back | ✅ | io_uring |
| **数据库** | VirtIO SCSI single | No cache (none) | ✅ | io_uring |
| **文件服务器** | VirtIO SCSI single | Write back | ✅ | io_uring |
| **轻量服务** | VirtIO SCSI | No cache (none) | ❌ | io_uring |
| **Windows 桌面** | VirtIO SCSI single | No cache (none) | ✅ | io_uring |

### 一句话总结

**PVE 最新的默认配置（VirtIO SCSI single + cache=none + IO Thread + io_uring）已经是一个很好的通用起点**，兼顾了性能与安全性。除非你有非常明确的需求（数据库需要极致一致性、文件服务器想压榨最后 30% 性能），否则直接使用默认值就好。

如果你想进一步优化：
1. **先看瓶颈在哪** — 用 `iostat -x 1` 在宿主机查看 `await` 和 `%util`
2. **先动存储层** — ZFS 的 recordsize、ashift、compression 设置对最终性能的影响往往比缓存模式更大
3. **最后调虚机层** — 改控制器、缓存、IO Thread 每一步都跑一遍 fio 验证

> **预告：** 下一篇会深入 PVE ZFS 存储的性能调优——recordsize 选择、ashift 对齐、compression 算法对比以及 SLOG/ZIL 在 NVMe 上的实战配置。
