---
title: "Proxmox Backup Server 完全指南：增量备份、去重与自动化策略"
description: "从零部署 Proxmox Backup Server，实现高效增量备份与全局去重。深入解析 ZFS 存储调优、备份保留策略、GC 与 Verify 自动化，以及 PVE 多节点集中备份架构。"
date: 2026-05-09T01:02:55+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
slug: "proxmox-backup-server-guide"
---

## 前言

玩 Homelab 最怕什么？不是硬件挂了，是**数据没了**。

Proxmox VE 自带的 `vzdump` 虽然能用，但每次都是全量备份、没有去重、远程同步靠 NFS/CIFS。一旦虚拟机多了，备份窗口拉得越来越长，存储空间也直线上升。更扎心的是——你上一次验证备份能不能恢复，是什么时候？

Proxmox Backup Server（以下简称 PBS）正是为解决这些问题而生的。它基于 Rust 重写的底层引擎，提供**增量备份、客户端去重、加密传输、定时验证**等企业级功能，而且完全免费。

本文从零开始，涵盖 PBS 的安装部署、ZFS 存储调优、备份策略设计、Garbage Collection 与 Verify 自动化，以及多节点 PVE 集中备份架构。所有命令均可直接复制执行。

---

## 1. PBS 核心概念

在动手之前，先理解 PBS 的几个关键设计。

### 1.1 增量备份 ≠ 差量备份

PBS 用的是 **chunk-level 增量备份**。每次备份时，客户端（PVE）将虚拟磁盘按 4MB 块切割，计算每个块的 SHA256 哈希值，发送到服务端。服务端检查该块是否已存在——存在就跳过，不存在就压缩存储。所以：

- **首次备份** = 全量（慢）
- **后续备份** = 只传输变化的块（极快）
- **每个快照都可独立恢复**（不需要依赖前一个备份）

### 1.2 去重是全局的

PBS 的去重不是按 VM 计算的，而是**跨所有 VM 共享同一个 chunk 索引**。假设你有 10 个 Ubuntu 22.04 VM，它们的系统盘里大量相同的文件会产生完全相同的 chunk，PBS 只存储一份。

实际测试中，同类 OS 的 VM 可以达到 **3-5 倍的存储节省**。

### 1.3 核心组件

| 组件 | 作用 |
|------|------|
| **Datastore** | 存储备份数据的位置，一个目录对应一个 datastore |
| **Chunk Store** | 实际存放去重后的数据块（`.chunks/` 目录） |
| **Namespace** | 逻辑隔离（PBS 3.0+），不同业务可用不同 namespace |
| **Garbage Collection** | 清理未引用的 chunk，回收空间 |
| **Verify** | 定期验证备份数据完整性（读取每个 chunk 并校验哈希） |

---

## 2. 安装部署 PBS

### 2.1 安装方式对比

| 方式 | 优点 | 缺点 | 推荐场景 |
|------|------|------|----------|
| **ISO 裸机安装** | 性能最佳，独立管理 | 需要独立物理机 | 生产环境 |
| **PVE VM 安装** | 灵活，快照/迁移方便 | 略损耗性能 | ✅ Homelab 首选 |
| **LXC 安装** | 轻量，共享内核 | 官方不建议 | 测试环境 |

### 2.2 在 PVE 上创建 PBS VM

```bash
# 1. 下载 PBS ISO（在 PVE 宿主机操作）
cd /var/lib/vz/template/iso/
wget https://enterprise.proxmox.com/iso/proxmox-backup-server_3.3-1.iso

# 2. 创建虚拟机
qm create 200 \
  --name pbs \
  --memory 4096 \
  --cores 4 \
  --net0 virtio,bridge=vmbr0 \
  --scsihw virtio-scsi-single \
  --boot order=scsi0

# 3. 添加系统盘
qm set 200 --scsi0 local-lvm:32,ssd=1

# 4. 添加备份存储盘（建议独立 SSD）
qm set 200 --scsi1 local-lvm:256,ssd=1

# 5. 挂载 ISO 并启动
qm set 200 --ide2 local:iso/proxmox-backup-server_3.3-1.iso,media=cdrom
qm start 200
```

系统盘 32GB 足够，备份数据盘根据实际需求分配。如果 PVE 宿主机用的是 ZFS，可以在宿主机上先创建 dataset 再直通给 VM（性能更好但配置稍复杂）。

### 2.3 初始化配置

安装完成后，通过浏览器访问 `https://<PBS-IP>:8007`，使用 `root` 和安装时设置的密码登录。

首先获取订阅密钥（免费版）：

```bash
# SSH 到 PBS
ssh root@<PBS-IP>

# 注册产品（免费版即可用全部功能）
pbsubscription set standard
```

> ⚠️ **注意**：不注册也可以正常使用，只是每次登录会弹提示。免费版不限制备份数量和节点数。

---

## 3. 存储配置：ZFS Datastore 调优

PBS 对存储延迟敏感，强烈建议用 SSD。实测中，HDD 做 PBS datastore 时 GC 速度慢 5-10 倍。

### 3.1 创建 ZFS Pool（PBS VM 内部）

```bash
# 查看可用磁盘
lsblk

# 创建 ZFS pool（假设备份盘是 /dev/sdb）
zpool create -o ashift=12 pbsdata /dev/sdb

# 创建 dataset 作为 datastore 目录
zfs create -o mountpoint=/mnt/pbs pbsdata/datastore
```

### 3.2 ZFS 性能调优

```bash
# 压缩：lz4 速度快，对备份数据压缩率也不错
zfs set compression=lz4 pbsdata/datastore

# 关闭 atime：避免每次读取都产生写操作
zfs set atime=off pbsdata/datastore

# recordsize：PBS chunk 大小是 4MB，用 1M 对齐
zfs set recordsize=1M pbsdata/datastore

# xattr=sa：减少元数据碎片
zfs set xattr=sa pbsdata/datastore

# 关闭冗余去重（PBS 自己有去重，ZFS dedup 会吃大量内存）
zfs set dedup=off pbsdata/datastore
```

> 🏆 **关键优化**：`recordsize=1M` 是 PBS 在 ZFS 上最重要的调优参数。PBS 写入单个 chunk 大小为 1-4MB，recordsize 设为 1M 可以显著减少写放大和碎片。

### 3.3 创建 Datastore

```bash
# 在 PBS Web UI 或命令行创建
proxmox-backup-manager datastore create main /mnt/pbs

# 查看 datastore 信息
proxmox-backup-manager datastore list
```

---

## 4. 连接 PVE 到 PBS

### 4.1 在 PVE 上添加 PBS 存储

在 PVE Web UI：**数据中心 → 存储 → 添加 → Proxmox Backup Server**

| 字段 | 值 |
|------|-----|
| ID | `pbs-main` |
| 服务器 | `<PBS-IP>` |
| 用户名 | `root@pam` |
| 密码 | `<PBS root 密码>` |
| Datastore | `main` |
| 指纹 | PBS 仪表盘 → 显示指纹 获取 |

或者命令行：

```bash
pvesm add pbs pbs-main \
  --server <PBS-IP> \
  --datastore main \
  --username root@pam \
  --password '<password>' \
  --fingerprint '<SHA256-fingerprint>'
```

### 4.2 创建专用备份用户（可选，推荐）

不建议用 root 做日常备份。在 PBS 上创建专用账户：

```bash
# 在 PBS 上执行
proxmox-backup-manager user create backup-user@pbs --password '<secure-password>'

# 授予权限
proxmox-backup-manager acl update /datastore/main backup-user@pbs --roles DatastoreBackup,DatastoreReader
```

---

## 5. 备份策略设计

### 5.1 保留策略配置

PBS 的留存策略在 Datastore 级别设置，影响该 datastore 下所有备份。

推荐 Homelab 场景配置：

```bash
proxmox-backup-manager datastore update main \
  --prune-schedule 'daily' \
  --keep-last 7 \
  --keep-daily 7 \
  --keep-weekly 4 \
  --keep-monthly 3 \
  --keep-yearly 1
```

| 参数 | 含义 | Homelab 推荐 |
|------|------|-------------|
| `keep-last` | 保留最近 N 个快照 | 7 |
| `keep-daily` | 保留每日备份数 | 7 |
| `keep-weekly` | 保留每周备份数 | 4 |
| `keep-monthly` | 保留每月备份数 | 3 |
| `keep-yearly` | 保留每年备份数 | 1 |

这样设计，你总能恢复到过去 7 天内的任意时间点、过去 4 周的每周快照，以及过去 3 个月的月末状态。

### 5.2 在 PVE 上配置备份任务

在 PVE Web UI：**数据中心 → 备份 → 添加**

或命令行：

```bash
# 单个 VM 备份
vzdump 100 \
  --storage pbs-main \
  --mode snapshot \
  --compress zstd \
  --remove 0 \
  --notes-template "{{guestname}} - 每日自动备份"

# 创建定时任务（每天凌晨 2 点）
cat > /etc/cron.d/pbs-backup << 'EOF'
0 2 * * * root vzdump 100 101 102 --storage pbs-main --mode snapshot --compress zstd --quiet 1 --mailnotification failure
EOF
```

也可以在 PVE Web UI 中设置备份计划，更直观。

### 5.3 备份模式对比

| 模式 | 适用场景 | 热备份 | 一致性 |
|------|----------|--------|--------|
| `snapshot` | ✅ KVM/VM 首选 | 是 | 一致性快照 |
| `suspend` | 对停机敏感的 VM | 短暂暂停 | 内存 + 磁盘 |
| `stop` | 数据库类 VM | 完全停机 | 最一致 |

> 对于 MySQL/PostgreSQL 等数据库，snapshot 模式会创建一致性快照（通过 QEMU guest agent），通常足够。但如果对数据一致性要求极高，建议搭配数据库原生备份工具先做 dump，再备份 dump 文件。

---

## 6. 维护操作自动化

备份存进去了，不代表就高枕无忧。PBS 还需要定期执行两项关键维护。

### 6.1 Garbage Collection（垃圾回收）

当备份快照被 Prune 删除后，对应的 chunk 并不会立刻释放空间。GC 扫描所有索引，标记仍在使用的 chunk，删除无人引用的孤立 chunk。

```bash
# 手动执行 GC
proxmox-backup-manager garbage-collection start main

# 查看 GC 状态
proxmox-backup-manager garbage-collection status
```

### 6.2 Verify（完整性校验）

验证操作会读取所有备份 chunk，重新计算哈希与元数据对比，确保数据没有静默损坏。

```bash
# 校验最新快照
proxmox-backup-manager verify main

# 全量校验所有快照（慢但彻底）
proxmox-backup-manager verify main --all-versions
```

### 6.3 定时任务（systemd timer）

PBS 自带 systemd timer，建议直接使用，不用自己写 cron：

```bash
# 查看当前 GC 和 Prune 定时器状态
systemctl list-timers proxmox-backup-*

# 编辑 GC 调度（默认每天 02:30）
systemctl edit proxmox-backup-gc.timer
# 内容如下：
[Timer]
OnCalendar=*-*-* 03:30:00

# 编辑 Verify 调度（默认每周日）
systemctl edit proxmox-backup-verify.timer
[Timer]
OnCalendar=Sun 05:00:00
```

建议调度顺序：**Prune（删除过期快照）→ GC（回收空间）→ Verify（校验剩余数据）**，三者间隔至少 1 小时。

---

## 7. 恢复操作

备份的终极目的是恢复。PBS 的恢复体验非常好。

### 7.1 PVE Web UI 恢复

**VM → 备份 → 选择快照 → 还原**，全程图形化操作。可以恢复到原来的 VM（覆盖）或创建新 VM。

### 7.2 命令行恢复

```bash
# 列出 VM 100 的可用备份
proxmox-backup-client snapshot list \
  --repository root@pam@<PBS-IP>:main \
  host/<PVE-HOSTNAME> --ns vm/100

# 恢复 VM 磁盘到指定路径
proxmox-backup-client restore \
  vm/100/2026-05-08T02:00:00Z drive-scsi0.img \
  /var/lib/vz/images/100/vm-100-disk-0.raw \
  --repository root@pam@<PBS-IP>:main
```

### 7.3 单独恢复文件（文件级恢复）

PBS 支持从 VM 备份中单独提取文件，不需要恢复整个磁盘：

```bash
# 映射备份快照为块设备（在 PVE 宿主机上）
proxmox-backup-client map \
  vm/100/2026-05-08T02:00:00Z drive-scsi0.img \
  --repository root@pam@<PBS-IP>:main

# 挂载映射的设备
mount /dev/loop0p1 /mnt/recovery

# 复制需要的文件
cp /mnt/recovery/var/lib/mysql/xxx.sql /tmp/

# 卸载并取消映射
umount /mnt/recovery
proxmox-backup-client unmap /dev/loop0
```

---

## 8. 多节点集中备份架构

如果你有多台 PVE 主机，不需要每台都部署 PBS。一个 PBS 实例可以同时为多个 PVE 节点服务。

```
  PVE-Node-1 ──┐
  PVE-Node-2 ──┼── 10GbE Switch ── PBS (Datastore: main)
  PVE-Node-3 ──┘
```

建议：
- PBS 和数据盘部署在一台独立机器上（或 PVE 集群外的 VM）
- 为不同节点使用 **namespace** 隔离（PBS 3.0+）：

```bash
# 为不同 PVE 节点创建独立命名空间
proxmox-backup-manager datastore create-namespace main node1
proxmox-backup-manager datastore create-namespace main node2
proxmox-backup-manager datastore create-namespace main node3
```

然后在各 PVE 节点的存储配置中指定对应 namespace。

### PBS 异地备份

即使有 PBS，异地容灾仍然重要。可以用 PBS 的 **Remote Sync** 功能或直接同步 datastore：

```bash
# PBS 远程同步（需要两台 PBS）
proxmox-backup-manager remote create backup-remote \
  --host <remote-pbs-ip> \
  --user root@pam \
  --password '<password>' \
  --fingerprint '<fingerprint>'

# 创建同步任务
proxmox-backup-manager sync-job create \
  --remote backup-remote \
  --remote-datastore main \
  --local-datastore main \
  --schedule 'daily'
```

> 💡 Homelab 异地方案：如果只有一台 PBS，可以用 `rclone` 把 datastore 目录同步到云存储（Backblaze B2、阿里云 OSS 等），便宜又省心。

---

## 9. 踩坑记录

### ❌ 坑 1：用 HDD 做 PBS Datastore，GC 跑一整夜

PBS 的 GC 操作会产生大量随机 I/O（遍历 chunk 索引 + 删除孤立 chunk），HDD 的 IOPS 完全不够。

**现象**：`pbs-gc.service` 运行超过 6 小时，期间备份任务被阻塞。

**修复**：将 datastore 迁移到 SSD。至少 $GCTMPDIR 指向 SSD（PBS 3.1+ 支持）。

```bash
# PBS 3.1+: 将 GC 临时目录放到 SSD
mkdir -p /mnt/ssd/gc-tmp
proxmox-backup-manager datastore update main --gc-tmpdir /mnt/ssd/gc-tmp
```

### ❌ 坑 2：ZFS recordsize=128K 导致写放大

默认的 ZFS recordsize 是 128KB，但 PBS 的 chunk 大小在 1-4MB。128K 的 record 意味着一个 chunk 被拆成几十个 ZFS 记录，产生大量碎片和写放大。

**修复**：

```bash
zfs set recordsize=1M pbsdata/datastore
```

已存在的数据需要重写才能生效，建议创建 dataset 时就设置好。

### ❌ 坑 3：忘记定期 Verify，备份悄悄损坏

备份写进去了不等于就安全了。磁盘的静默数据损坏（bit rot）在 ZFS 上通常能被检测，但如果你用了 ext4/xfs 做 PBS datastore，静默损坏会一直累积。

**修复**：

```bash
# 开启自动 verify（每月一次全身检查）
systemctl edit proxmox-backup-verify.timer
```

同时，PBS datastore 底层文件系统**强烈建议用 ZFS**（自带校验和），ext4 是在冒险。

### ❌ 坑 4：Prune 后空间没回收

Prune 删除过期快照后，你会发现磁盘空间没有立刻释放。这是因为 GC 还没跑。

```bash
# 手动触发 GC 回收空间
proxmox-backup-manager garbage-collection start main
```

Prune → GC 的时间差是设计如此，不是 bug。确保 GC timer 正常工作即可。

---

## 总结

PBS 是 Proxmox VE 生态中最被低估的组件。它在 Homelab 场景下几乎是零成本的生产级备份方案。

| 维度 | 结论 |
|------|------|
| ✅ 推荐场景 | 所有 PVE Homelab（即便只有单节点） |
| ✅ 存储方案 | ZFS + SSD + recordsize=1M |
| ✅ 备份策略 | daily snapshot + keep-last 7 + keep-weekly 4 + keep-monthly 3 |
| ✅ 自动化 | Prune → GC → Verify 顺序执行，使用 systemd timer |
| ⚠️ 注意事项 | HDD 只适合存冷数据，活跃 datastore 务必 SSD |
| 🔗 关联阅读 | [Traefik 反向代理](/p/traefik-reverse-proxy-homelab/) · [ZFS 存储配置](/p/proxmox-zfs-storage-guide/) · [监控告警](/p/homelab-monitoring-stack-guide/) |

备份不是「一次性设置」的东西——它需要持续的验证、清理和监控。好消息是，PBS 让这些工作变得异常简单。
