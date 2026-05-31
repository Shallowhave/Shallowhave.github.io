---
title: "三台小主机也能玩分布式存储：Proxmox VE + Ceph 超融合实战"
description: "在 Homelab 中用 Proxmox VE 内置 pveceph 搭建三节点 Ceph/RBD 超融合存储，覆盖网络规划、MON/MGR/OSD 创建、Pool 参数、VM 迁移测试、性能验证、故障恢复与踩坑经验。"
date: 2026-05-31T09:22:06+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
slug: "pve-ceph-hyperconverged-storage"
---

## 前言：什么时候 Homelab 才值得上 Ceph？

很多 Homelab 玩家第一次看到 Proxmox VE 里的 Ceph 菜单，都会有一种「这不就是企业级分布式存储吗，我也要开！」的冲动。但 Ceph 不是魔法，它用**网络、CPU、内存和磁盘数量**换来分布式副本、高可用和在线迁移能力。硬件太弱、网络太慢、磁盘混搭太随意，最后得到的不是高可用，而是一套很难排障的慢存储。

如果你只有一台 PVE，优先选本地 ZFS；如果你有 NAS，NFS/iSCSI 会更简单；如果你已经有 **3 台以上 PVE 节点**，每台都有独立 SSD/NVMe，并且能给存储准备一张 2.5G/10G 网卡，那么 Ceph 才开始有意义。

本文记录一套适合 Homelab 的 Proxmox VE + Ceph 超融合落地方案：三台 PVE 既跑虚拟机，也贡献本地磁盘做 OSD，VM 磁盘放到 RBD Pool 中。任意一台节点维护或宕机时，虚拟机可以迁移到其他节点继续运行。

> 目标不是堆概念，而是跑通一套可维护的 Ceph：能部署、能看健康状态、能测试迁移、能处理 OSD 故障。

---

## 1. 架构与硬件规划

### 1.1 推荐拓扑

| 组件 | Homelab 最低建议 | 更推荐配置 |
|---|---:|---:|
| PVE 节点 | 3 台 | 3/5 台奇数节点 |
| 系统盘 | 独立 SSD | 独立 SSD + 备份 |
| Ceph OSD | 每节点 1 块 SSD/NVMe | 每节点 2 块以上同型号 SSD/NVMe |
| 内存 | 每节点 16GB | 每节点 32GB+ |
| 存储网络 | 2.5GbE 独立 VLAN | 10GbE 独立交换/网段 🏆 |
| 副本数 | size=3/min_size=2 | size=3/min_size=2 |

Ceph 的核心组件可以简单理解为：

- **MON（Monitor）**：保存集群地图，负责投票和一致性。三节点至少每节点一个 MON。
- **MGR（Manager）**：提供管理接口、性能统计、Dashboard 等能力。建议至少两个。
- **OSD（Object Storage Daemon）**：真正存数据的进程，一块磁盘通常对应一个 OSD。
- **Pool**：逻辑存储池，PVE VM 磁盘通常使用 RBD Pool。
- **PG（Placement Group）**：数据分布的中间层，新版本 Ceph 可以开启 autoscale，Homelab 不必手动硬算 PG。

### 1.2 网络规划：不要让 Ceph 和业务流量抢一根网线

Ceph 最怕网络抖动。VM 业务流量、Corosync 集群心跳、Ceph 副本同步如果全挤在一个网段，迁移或恢复时很容易出现延迟尖刺。推荐至少拆成三类网络：

| 网络 | 示例网段 | 用途 |
|---|---|---|
| 管理网络 | `192.168.10.0/24` | PVE Web、SSH、API |
| Corosync 网络 | `192.168.20.0/24` | PVE 集群心跳，低延迟优先 |
| Ceph 网络 | `10.10.10.0/24` | OSD 副本、恢复、客户端访问 |

如果只有双网口，可以把 Ceph 单独放到 VLAN：

```ini
# /etc/network/interfaces 示例：eno2 作为 Ceph 专用 VLAN Trunk

auto eno2
iface eno2 inet manual
    mtu 9000

auto eno2.30
iface eno2.30 inet static
    address 10.10.10.11/24
    vlan-raw-device eno2
    mtu 9000
```

三台节点分别使用：

```text
pve1: 10.10.10.11
pve2: 10.10.10.12
pve3: 10.10.10.13
```

如果开启 Jumbo Frame，**交换机端口、PVE 网卡、VLAN 子接口必须全部 MTU 一致**。否则 `ping` 小包正常，大量恢复数据时却随机卡顿。

验证 MTU：

```bash
# 8972 + 28 字节 ICMP/IP 头 = 9000 MTU
ping -M do -s 8972 10.10.10.12
ping -M do -s 8972 10.10.10.13
```

---

## 2. 部署前检查

先确认 PVE 集群健康：

```bash
pvecm status
pvecm nodes
```

重点看：

```text
Quorate:          Yes
Expected votes:   3
Total votes:      3
```

检查时间同步。Ceph 对时间漂移敏感，PVE 默认使用 chrony/systemd-timesyncd，但最好确认：

```bash
timedatectl
chronyc tracking 2>/dev/null || true
```

检查待加入 Ceph 的磁盘。**不要把系统盘拿去做 OSD**：

```bash
lsblk -o NAME,SIZE,TYPE,MODEL,SERIAL,FSTYPE,MOUNTPOINT
```

假设每台节点的 OSD 盘都是 `/dev/nvme1n1`。如果磁盘以前做过 ZFS/LVM，需要先清理签名：

```bash
# 危险操作：确认盘符无误再执行
wipefs -a /dev/nvme1n1
sgdisk --zap-all /dev/nvme1n1
```

---

## 3. 安装 Ceph 包

在每个 PVE 节点执行：

```bash
apt update
pveceph install --repository no-subscription
```

企业订阅环境可以改成：

```bash
pveceph install --repository enterprise
```

安装后确认命令可用：

```bash
ceph --version
pveceph status
```

如果你的 PVE 版本支持 Reef/Squid 多版本选择，建议整个集群保持一致，不要一台 Reef、一台 Squid 混跑。升级 Ceph 要按 Proxmox 官方文档分阶段做，别在生产 VM 正在写大量数据时直接滚动升级。

---

## 4. 初始化 Ceph 集群

只在第一台节点执行初始化：

```bash
pveceph init --network 10.10.10.0/24
```

如果你有更豪华的双 Ceph 网络，可以把 public 和 cluster network 拆开：

```bash
pveceph init \
  --network 10.10.10.0/24 \
  --cluster-network 10.10.20.0/24
```

- `public_network`：PVE/RBD 客户端访问 Ceph 的网络
- `cluster_network`：OSD 之间复制、恢复、心跳使用的网络

Homelab 多数情况下只有一张专用存储网卡，直接使用 `--network` 即可。初始化后配置会写到 `/etc/pve/ceph.conf`，并通过 PVE 的 pmxcfs 分发到所有节点：

```bash
cat /etc/pve/ceph.conf
```

典型配置片段：

```ini
[global]
    auth_client_required = cephx
    auth_cluster_required = cephx
    auth_service_required = cephx
    cluster_network = 10.10.10.0/24
    public_network = 10.10.10.0/24
    fsid = xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    mon_allow_pool_delete = true
```

---

## 5. 创建 MON 与 MGR

每台节点创建一个 MON：

```bash
# 在 pve1 上
pveceph mon create

# 在 pve2 上
pveceph mon create

# 在 pve3 上
pveceph mon create
```

创建 MGR，至少两个：

```bash
# pve1
pveceph mgr create

# pve2
pveceph mgr create
```

查看状态：

```bash
ceph -s
ceph mon stat
ceph mgr stat
```

健康状态至少应该接近：

```text
cluster:
  health: HEALTH_OK
services:
  mon: 3 daemons, quorum pve1,pve2,pve3
  mgr: pve1(active), standbys: pve2
```

如果 MON 没有 quorum，先不要继续创建 OSD。优先检查：

```bash
cat /etc/hosts
ip addr
ping 10.10.10.12
ping 10.10.10.13
journalctl -u ceph-mon@$(hostname) -n 100 --no-pager
```

---

## 6. 创建 OSD：一块盘一个 OSD

在每台节点执行：

```bash
pveceph osd create /dev/nvme1n1
```

创建完成后查看：

```bash
ceph osd tree
ceph osd df tree
ceph -s
```

示例结构：

```text
ID  CLASS  WEIGHT   TYPE NAME
-1         2.79474  root default
-3         0.93158      host pve1
 0    ssd  0.93158          osd.0
-5         0.93158      host pve2
 1    ssd  0.93158          osd.1
-7         0.93158      host pve3
 2    ssd  0.93158          osd.2
```

这里有两个 Homelab 常见坑：

1. **不要混用 HDD 和 SSD 做同一个 Pool**。慢盘会拖垮整体延迟。
2. **不要每台节点 OSD 数量差异过大**。例如 pve1 有 4 块盘，pve2/pve3 各 1 块，数据分布和恢复压力会很不均衡。

如果确实有 SSD/HDD 混合，至少用 device class 和 CRUSH rule 分开：

```bash
ceph osd crush rule create-replicated replicated_ssd default host ssd
ceph osd crush rule create-replicated replicated_hdd default host hdd
```

---

## 7. 创建 RBD Pool 并接入 PVE

VM 磁盘使用 RBD Pool。创建一个名为 `rbd-ssd` 的池：

```bash
pveceph pool create rbd-ssd \
  --size 3 \
  --min_size 2 \
  --pg_autoscale_mode on \
  --application rbd \
  --add_storages 1
```

参数解释：

| 参数 | 建议值 | 说明 |
|---|---:|---|
| `size` | 3 | 每份数据保留 3 个副本 |
| `min_size` | 2 | 至少 2 个副本在线才允许写入 |
| `pg_autoscale_mode` | on | 让 Ceph 自动调整 PG |
| `application` | rbd | 标记用于块设备 |
| `add_storages` | 1 | 自动加入 PVE Storage |

确认 PVE 存储已加入：

```bash
pvesm status
cat /etc/pve/storage.cfg
```

典型 `storage.cfg`：

```ini
rbd: rbd-ssd
    content images,rootdir
    krbd 0
    pool rbd-ssd
```

如果想手动添加，也可以：

```bash
pvesm add rbd rbd-ssd \
  --pool rbd-ssd \
  --content images,rootdir \
  --krbd 0
```

---

## 8. 把虚拟机磁盘迁移到 Ceph

创建新 VM 时直接选择 `rbd-ssd` 存储即可。已有 VM 可以在线迁移磁盘：

```bash
# 查看 VM 磁盘
qm config 101

# 将 scsi0 迁移到 Ceph RBD
qm move_disk 101 scsi0 rbd-ssd --delete 1
```

推荐 VM 磁盘配置：

```bash
qm set 101 \
  --scsihw virtio-scsi-single \
  --scsi0 rbd-ssd:vm-101-disk-0,discard=on,iothread=1 \
  --agent enabled=1
```

开启 discard 后，Guest 内部删除数据可以回收 RBD 空间。Linux VM 内执行：

```bash
systemctl enable --now fstrim.timer
fstrim -av
```

Windows VM 则确认「优化驱动器」能识别精简置备磁盘，并安装 VirtIO 驱动。

---

## 9. 验证迁移和故障恢复

### 9.1 Live Migration 测试

```bash
# 从 pve1 迁移 VM 101 到 pve2
qm migrate 101 pve2 --online
```

如果 VM 磁盘在 Ceph RBD 上，迁移时不需要复制整块磁盘，只迁移内存和运行状态，速度会明显快于本地盘迁移。

观察 Ceph 状态：

```bash
watch -n 2 'ceph -s; echo; ceph osd df tree'
```

### 9.2 模拟一块 OSD 下线

先找一个 OSD ID：

```bash
ceph osd tree
```

模拟下线：

```bash
ceph osd out osd.0
ceph -s
```

此时会看到 PG remapped/backfill。等恢复完成后再放回：

```bash
ceph osd in osd.0
ceph -s
```

如果是真实坏盘，流程应该是：

```bash
# 1. 标记 out，等待数据迁移
ceph osd out osd.0
watch -n 5 ceph -s

# 2. 停止 OSD
systemctl stop ceph-osd@0

# 3. 从集群移除
pveceph osd destroy 0

# 4. 换盘后重新创建
wipefs -a /dev/nvme1n1
sgdisk --zap-all /dev/nvme1n1
pveceph osd create /dev/nvme1n1
```

不要在 `HEALTH_WARN` 且还有大量 backfill 时连续拔多块盘。三副本不是免死金牌，`size=3/min_size=2` 只能容忍一个故障域丢失；如果第二台节点也出问题，写入会被阻塞，严重时数据不可用。

---

## 10. 简单性能测试

在测试 VM 里安装 fio：

```bash
apt update
apt install -y fio
```

随机 4K 读写测试：

```bash
fio --name=rbd-4k-randrw \
  --filename=/tmp/fio-test \
  --size=8G \
  --direct=1 \
  --rw=randrw \
  --rwmixread=70 \
  --bs=4k \
  --iodepth=32 \
  --numjobs=4 \
  --runtime=120 \
  --time_based \
  --group_reporting
```

顺序写测试：

```bash
fio --name=rbd-seqwrite \
  --filename=/tmp/fio-seq \
  --size=16G \
  --direct=1 \
  --rw=write \
  --bs=1M \
  --iodepth=16 \
  --numjobs=1 \
  --runtime=120 \
  --time_based \
  --group_reporting
```

一个很实用的判断方式：

| 现象 | 可能原因 | 排查命令 |
|---|---|---|
| 延迟几十毫秒以上 | 网络拥塞/OSD 慢盘 | `ceph osd perf`、`iftop` |
| 写入速度只有单盘一半 | 2.5G/1G 网络瓶颈 | `iperf3 -c 10.10.10.12` |
| 迁移时 VM 卡顿 | Ceph 与业务网络混用 | `ceph -s`、交换机端口统计 |
| HEALTH_WARN 长期不恢复 | PG/OSD 异常 | `ceph health detail` |

测试完删除临时文件：

```bash
rm -f /tmp/fio-test /tmp/fio-seq
fstrim -av
```

---

## 11. 常用运维命令清单

```bash
# 集群总览
ceph -s
ceph health detail

# OSD 状态
ceph osd tree
ceph osd df tree
ceph osd perf

# Pool 与容量
ceph df
ceph osd pool ls detail
rbd -p rbd-ssd ls

# 查看某个 VM 磁盘映像
rbd -p rbd-ssd info vm-101-disk-0

# PVE 存储状态
pvesm status
pvesm list rbd-ssd

# 日志
journalctl -u 'ceph-*' -n 200 --no-pager
```

建议把 `ceph -s` 接入监控。最简单可以用 Node Exporter textfile collector，也可以使用 Prometheus 的 ceph exporter 或 Ceph MGR prometheus 模块：

```bash
ceph mgr module enable prometheus
ceph config set mgr mgr/prometheus/server_addr 0.0.0.0
ceph config set mgr mgr/prometheus/server_port 9283
```

然后在 Prometheus 里抓取：

```yaml
scrape_configs:
  - job_name: ceph
    static_configs:
      - targets:
          - 10.10.10.11:9283
          - 10.10.10.12:9283
          - 10.10.10.13:9283
```

---

## 12. 踩坑记录

### ❌ 坑 1：三节点但只有两个 OSD

Ceph MON 可以三节点 quorum，但数据副本靠 OSD。`size=3` 至少需要三个不同故障域的 OSD。只有两个 OSD 时，Pool 不是降级就是无法满足副本策略。

### ❌ 坑 2：用 USB 硬盘做 OSD

USB 控制器重置、线材松动、省电休眠都会让 OSD 掉线。Ceph 会把它当成真实故障并触发恢复，恢复又继续打爆 USB 链路。Homelab 测试可以，长期运行不建议。

### ❌ 坑 3：把 Ceph 放在 1GbE 管理网

Ceph 三副本写入不是「写一份就完事」，它还要复制到其他 OSD。1GbE 理论上限 125MB/s，实际多 VM 并发时很容易被打满。能上 10GbE 最好，最低也建议 2.5GbE 独立网络。

### ❌ 坑 4：节点容量差异太大

CRUSH 会按权重分布数据。某台节点容量明显更大并不代表它更快，恢复时反而可能承担更多数据迁移压力。Homelab 最省心的做法是：每节点同数量、同容量、同类型 SSD。

### ❌ 坑 5：误以为 Ceph 等于备份

Ceph 是高可用存储，不是备份。误删 VM、勒索软件、数据库逻辑损坏会被三副本同步得非常可靠。重要数据仍然要用 PBS、Restic、Borg 或异地备份做离线/版本化保护。

---

## 总结：Ceph 很强，但别把它当默认答案

| 场景 | 推荐方案 |
|---|---|
| 单节点 PVE | 本地 ZFS 🏆 |
| 1-2 台 PVE + NAS | NFS/iSCSI |
| 3 台以上 PVE、需要热迁移/HA | Ceph RBD ✅ |
| 低功耗小主机、只有 1GbE | 不建议上 Ceph ❌ |
| 追求简单备份恢复 | PBS/Restic，而不是 Ceph |

Proxmox VE 把 Ceph 集成得很顺手，`pveceph install/init/osd/pool` 几条命令就能搭出分布式存储。但真正决定体验的不是按钮，而是前期规划：**独立高速网络、均衡 OSD、三副本策略、监控告警和外部备份**。

如果你的 Homelab 已经从单机 PVE 走到三节点集群，Ceph 是下一块很值得折腾的拼图；如果还停留在一台小主机加几块盘，那就先把 ZFS、PBS、UPS 和监控做好。高可用的第一原则不是复杂，而是可理解、可恢复、可验证。
