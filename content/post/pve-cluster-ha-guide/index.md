---
title: "Proxmox VE 集群与高可用实战：从零搭建三节点 HA 虚拟化平台"
description: "当一台 PVE 扛不住的时候，就该上集群了。本文从 Corosync 集群搭建、共享存储配置、Live Migration 热迁移机制，到 HA 高可用策略，完整覆盖三节点 PVE 集群的落地全过程，附带故障模拟和踩坑实录。"
date: 2026-05-29T09:00:00+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
slug: "pve-cluster-ha-guide"
---

## 前言

单机 PVE 能承载大部分 Homelab 需求——跑几个 LXC、十来台 VM，配上 PBS 定时备份，已经相当可靠。

但总有那么几个时刻让你意识到单机的天花板：

- 需要**滚动升级** PVE 内核，业务却不能停
- 宿主机硬件维护（换风扇、加内存），所有 VM 被迫关机
- 某个节点宕机后，所有服务直到你手动恢复才恢复
- 想**在线迁移** VM 到另一台性能更强的机器上做负载均衡

这就是 PVE 集群 + 高可用（HA）出场的时机。

本文以 **PVE 8.x 三节点集群**为例，从集群创建、共享存储、Live Migration 热迁移到 HA 故障自愈，完整覆盖生产级 Homelab 虚拟化平台的搭建流程。

---

## 1. 集群基础架构

PVE 集群的底层组件并不复杂，理解它们就能在故障时从容应对：

| 组件 | 作用 |
|------|------|
| **Corosync** | 集群通信引擎（组播/UDP），心跳检测与消息广播 |
| **pmxcfs** | Proxmox Cluster File System，FUSE 挂载于 `/etc/pve`，底层用 SQLite 同步集群配置 |
| **votequorum** | 投票仲裁模块，决定集群是否拥有法定人数 |
| **pve-ha-manager** | 高可用管理器，监控资源状态并触发故障恢复 |
| **watchdog** | 看门狗定时器，节点失联后自动重启以释放资源 |

**关于 Quorum（仲裁）：**

Quorum 是防止 **Split-Brain（脑裂）** 的核心机制。当集群网络分裂时，只有拥有多数票的"半区"能继续提供服务，失去 quorum 的半区会将 pmxcfs 切换为**只读模式**并停止 VM。

- 三节点集群：quorum = 2（需要≥2节点在线）
- 双节点集群：必须设置 `two_node: 1`，quorum 降为 1，但风险极高——单节点故障后集群直接不可用

> **Homelab 建议：** 至少 3 节点。双节点 + QDevice（第三方仲裁设备，如树莓派）也是可行方案，但管理复杂度显著增加。

---

## 2. 三节点集群搭建

### 2.1 前提条件

- 三台安装了 PVE 8.x 的主机，版本一致
- 节点时间同步（NTP）
- 节点之间能通过 **主机名** 互相解析（推荐 `/etc/hosts`）
- 防火墙放行集群通信端口：UDP 5404/5405（Corosync）和 TCP 22（SSH）

```bash
# 在每个节点上设置 hosts 解析
cat >> /etc/hosts << 'EOF'
192.168.1.10 pve1
192.168.1.11 pve2
192.168.1.12 pve3
EOF

# 确保 NTP 同步
apt install -y chrony
systemctl enable --now chrony
chronyc sources -v
```

### 2.2 创建集群

在 **第一个节点（pve1）** 上执行：

```bash
pvecm create homelab-cluster
```

成功后会看到集群已创建，`/etc/pve/` 目录自动变为集群文件系统。

### 2.3 其他节点加入

```bash
# 在 pve2 上执行
pvecm add pve1

# 在 pve3 上执行
pvecm add pve1
```

### 2.4 验证集群状态

```bash
pvecm status
pvecm nodes
corosync-quorumtool -s
```

正常输出应显示三个节点都在线，quorum 为 2（3节点中需2票）：

```
Membership information
-----------------------
    Nodeid      Votes Name
         1          1 pve1 (local)
         2          1 pve2
         3          1 pve3
Quorum: 2 (expected: 2)
```

生成的 `corosync.conf` 位于 `/etc/pve/corosync.conf`，由 pmxcfs 自动同步到所有节点：

```ini
logging {
  debug: off
  to_syslog: yes
}
nodelist {
  node {
    name: pve1
    nodeid: 1
    quorum_votes: 1
    ring0_addr: 192.168.1.10
  }
  node {
    name: pve2
    nodeid: 2
    quorum_votes: 1
    ring0_addr: 192.168.1.11
  }
  node {
    name: pve3
    nodeid: 3
    quorum_votes: 1
    ring0_addr: 192.168.1.12
  }
}
quorum {
  provider: corosync_votequorum
}
totem {
  cluster_name: homelab-cluster
  config_version: 1
  interface { ringnumber: 0 }
  ip_version: ipv4
  secauth: on
  transport: udpu
  version: 2
}
```

---

## 3. 共享存储配置

集群的威力需要共享存储来释放。没有共享存储时，VM 磁盘只在本地，迁移时连数据一起搬——速度慢、占用网络带宽。有了共享存储，迁移只需传输内存状态，快如闪电。

Homelab 常见共享存储方案对比：

| 方案 | 性能 | 配置复杂度 | 成本 | 适用规模 |
|------|------|-----------|------|---------|
| **NFS** | 中 | ⭐ 低 | 免费 | 小型 Homelab |
| **Ceph** | 高 | ⭐⭐⭐ 很高 | 免费（需三节点） | 中型以上 |
| **iSCSI + LVM** | 高 | ⭐⭐ 中 | 免费 | 中型 |
| **Starwind VSAN** | 高 | ⭐⭐ 中 | 免费（2节点） | 小型 |

> 注：PVE 已内置 Ceph 支持（`pveceph install`），3节点+万兆网络能跑出不错的性能。但 Ceph 配置维护复杂度较高，Homelab 入门推荐从 NFS 起步。

这里以 **NFS 共享存储**为例（将一台 NAS 或任意节点作为 NFS Server）：

```bash
# 在 NFS Server 上安装并配置
apt install -y nfs-kernel-server
mkdir -p /srv/nfs/pve-share

cat >> /etc/exports << 'EOF'
/srv/nfs/pve-share 192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)
EOF

exportfs -ra
```

在 PVE 集群的任一节点的 Web UI 中：

**Datacenter → Storage → Add → NFS：**

- ID: `nfs-share`
- Server: NFS Server IP
- Export: `/srv/nfs/pve-share`
- Content: 勾选 `VZDump backup`, `Container`, `Snippets`, `ISO images`

或者在 CLI 中执行：

```bash
pvesm add nfs nfs-share --server 192.168.1.20 --path /srv/nfs/pve-share --export /srv/nfs/pve-share --content iso,vztmpl,backup,snippets
```

存储添加后，集群中所有节点的 `/etc/pve/storage.cfg` 会自动同步——这就是 pmxcfs 的魅力。

---

## 4. Live Migration 热迁移机制

热迁移（Live Migration）是 PVE 集群最令人舒适的特性之一。理解它的底层原理，有助于在遇到迁移失败时定位问题。

### 4.1 Pre-Copy 迁移算法

PVE 基于 QEMU/KVM 的 **Pre-Copy** 内存迁移算法，分为五个阶段：

1. **初始同步：** 将 VM 全量内存从源节点复制到目标节点
2. **脏页迭代：** VM 持续运行产生的脏页（Dirty Pages）被反复追踪并重传
3. **收敛检测：** 当脏页率低于传输率阈值，或达到最大迭代次数，进入停顿时段
4. **暂停与最终复制：** 暂停 VM（Downtime），复制最后一批脏页 + CPU/设备状态
5. **切换执行：** 在目标节点恢复 VM，源节点释放资源

整个过程中，VM 感知到的停机窗口通常只有 **20~200ms**——一次网络 Ping 的时间。

```
迁移进度 ≈ 全量内存 / 传输带宽 + (迭代轮数 × 脏页量 / 传输带宽)
```

### 4.2 影响迁移速度的关键因素

| 因素 | 影响 |
|------|------|
| **内存大小** | 256GB VM 比 4GB VM 迁移慢几十倍 |
| **内存脏化速率** | 数据库、缓存类 VM 脏页率高，可能导致永不收敛 |
| **网络带宽** | 推荐 10GbE 或专用迁移 VLAN |
| **加密开销** | 默认 `secure` 模式加密传输，约消耗 20-30% 带宽 |
| **存储类型** | 共享存储只需传内存；本地存储需同时传磁盘 |

### 4.3 迁移调优

在生产环境迁移前，建议先调优以下参数：

```bash
# 修改 datacenter 默认迁移配置
# 编辑 /etc/pve/datacenter.cfg，添加：
migration: type=insecure, network=10.0.0.0/24

# 说明：
#   insecure = 不加密（私有内网环境安全，速度提升明显）
#   network  = 使用专用迁移网络 CIDR
```

**为指定 VM 设置迁移参数：**

```bash
# 设置迁移网络（必须在 datacenter.cfg 的 migration network CIDR 范围内）
qm set 100 --migration_network 10.0.0.0/24

# 设置最大停机时间（默认 100ms，高脏页率 VM 可调大到 500ms）
qm set 100 --migrate_downtime 500

# 限制迁移带宽（MB/s），避免占满业务带宽
qm set 100 --migration_speed 500
```

### 4.4 执行热迁移

```bash
# 在线热迁移（需要共享存储）
qm migrate 100 pve2 --online

# 观察迁移进度
watch 'qm monitor 100 <<< "info migrate" 2>/dev/null || echo "Migration finished"'

# 带存储迁移（本地存储场景）
qm migrate 100 pve2 --online --storage nfs-share
```

> ⚠️ **注意：** 迁移中的 VM 会短暂暂停（~100ms 左右），对延迟敏感的应用（如游戏服务器、实时音频）可能造成可感知的卡顿。建议在低峰期执行。

---

## 5. HA 高可用配置

有了集群和共享存储，HA 高可用才能完整生效。HA 的核心逻辑很简单：**检测到节点故障 → 在其他节点自动拉起 VM**。

### 5.1 HA 架构

PVE 的 HA 栈由三部分组成：

- **CRM（Cluster Resource Manager）**：集群资源管理器，运行在 master 节点，决策资源调度
- **LRM（Local Resource Manager）**：本地资源管理器，每个节点上运行，执行具体操作
- **Watchdog**：硬件/软件看门狗，节点故障后触发自动重启以释放资源（fencing）

故障恢复流程：

```
节点宕机 → CRM 检测到心跳丢失 → 等待 watchdog 超时（默认 60s）→
节点被 fencing（重启）→ CRM 在其他节点启动受影响的 VM → 
VM 状态变为 "started"，对外服务恢复
```

### 5.2 配置 HA 组

HA 组定义了 VM 可以在哪些节点上运行，以及优先级：

```bash
# 创建 HA 组，节点按优先级排列
ha-manager groupadd prod-group \
  --nodes "pve1:1,pve2:2,pve3:3" \
  --restricted 1 \
  --nofailback 0

# 参数说明：
#   restricted: 1 = VM 只能在组内节点运行，不会跑到组外节点
#   nofailback: 0 = 故障节点恢复后，VM 自动回迁到优先级更高的节点
#              1 = 不自动回迁，保持当前运行位置
```

### 5.3 设置 HA 资源

```bash
# 将 VM 加入 HA 管理
ha-manager set 100 --group prod-group --state started

# 查看 HA 状态
ha-manager status
ha-manager resources
ha-manager resource-status
```

HA 的 **state** 有四个常见值：

| State | 含义 |
|-------|------|
| `started` | VM 应始终运行，故障时自动在另一节点启动 |
| `stopped` | VM 应保持停止 |
| `disabled` | 不被 HA 管理，手动控制 |
| `freeze` | 冻结，不自动迁移但保留资源定义 |

### 5.4 CPU 类型兼容性

HA 多节点之间必须确保 CPU 指令集兼容，否则迁移后的 VM 可能直接崩溃：

```bash
# ❌ 不推荐的迁移配置：使用 host 模式
qm set 100 --cpu host
# host 模式暴露全部 CPU 特性，如果两个节点 CPU 不同代，迁移后 Guest OS 可能遇到 SIGILL

# ✅ 推荐的迁移配置：使用标准化 CPU 类型
qm set 100 --cpu x86-64-v2-AES
# x86-64-v2 对应约 Intel Haswell (2013) 及以上，AES 是硬件加速加密指令
# 更保守的选择：kvm64（最兼容，但性能损失约 10-15%）
```

> **Homelab 实践：** 如果所有节点 CPU 同代同型号，大胆用 `host` 模式获取最大性能。如果混合了不同代 CPU（如 i5-12500 和 N100），统一设置为 `x86-64-v2-AES` 是最优平衡。

---

## 6. 故障模拟与验证

搭建完成后，一定要做故障模拟测试，否则高可用只是心理安慰。

### 6.1 模拟节点宕机

```bash
# 在 pve1 上直接关机
pve1# poweroff

# 或者模拟网络断开（更真实的 Split-brain 测试）
pve1# iptables -A INPUT -s 192.168.1.0/24 -j DROP
```

在 Web UI 上观察：

1. pve1 状态变为未知/离线
2. HA 资源状态变为 `stoped` 并等待 fencing
3. Watchdog 超时后（约 60 秒），CRM 在其他节点自动启动 VM

查看 HA 日志：

```bash
journalctl -u pve-ha-crm -f
journalctl -u pve-ha-lrm -f
```

### 6.2 验证业务连续性

测试期间，从外部持续 Ping VM 的 IP 地址，观察断连持续时间：

```bash
# 在外部机器上
ping -D 192.168.1.100
```

正常 HA 切换的停机时间 = Watchdog 超时（60s）+ QEMU 启动时间（10-30s）= **约 70-90 秒**。这不是"零宕机"，但对大部分 Homelab 场景已经足够。

> **加速建议：** 如果业务对中断容忍度较低，可以缩短 Watchdog 超时：
> ```bash
# 修改 watchdog 超时时间（单位：秒，最小值 10）
echo 30 > /sys/devices/virtual/watchdog/watchdog0/timeout
# 持久化：在 /etc/systemd/system.conf 中设置 RuntimeWatchdogSec=30
```

### 6.3 恢复节点后自动回迁

将离线节点重新上线后，如果 HA 组设置了 `nofailback: 0`，VM 会自动从 pve2/pve3 迁移回 pve1：

```bash
# 检查回迁是否正常
ha-manager status
# 观察 VM 状态从 "migrate" → "started"
```

---

## 7. 踩坑记录

### ❌ 坑 1：迁移失败 — "could not start migration tunnel"

**现象：** 迁移进度卡在 0%，日志报 SSH 隧道建立失败。

**根因：** SSH 主机密钥认证失败，或节点间无法通过主机名解析。

**解决：**

```bash
# 检查节点之间的 SSH 连通性
ssh root@pve2 "hostname"

# 检查 hosts 解析
getent hosts pve2

# 如果缺失，重新建立集群 SSH 互信
pvecm updatecerts --force
```

### ❌ 坑 2：双节点集群 + NFS 存储，关机维护时集群瘫痪

**现象：** 维护时关闭一个节点，剩余节点显示"no quorum"，所有 VM 停止。

**根因：** 双节点集群 quorum = 2。一节点下线后只剩 1 票，失去 quorum。pmxcfs 变为只读，无法启动/迁移 VM。

**解决方案：**

```bash
# 维护前执行，临时降低 quorum 预期
pvecm expected 1

# 维护结束后恢复
pvecm expected 3
```

**更好的方案：** 直接升级到三节点，避免这个先天缺陷。

### ❌ 坑 3：HA Watchdog 在系统更新时误触发 Fencing

**现象：** 执行 `apt upgrade` 或重启 pve-ha-lrm 服务后，节点被 watchdog 重启。

**根因：** HA 模式下 watchdog 持续运行，如果 pve-ha-lrm 停止响应（升级过程中短暂暂停），watchdog 判定节点失联并触发重启。

**解决：** 维护操作前，先将 VM 移出 HA 管理：

```bash
# 方法一：将 VM 从 HA 中临时移除
ha-manager set 100 --state disabled

# 方法二：暂停 HA 服务
systemctl stop pve-ha-lrm
systemctl stop pve-ha-crm

# 更新完成后恢复
systemctl start pve-ha-crm
systemctl start pve-ha-lrm
ha-manager set 100 --state started
```

### ❌ 坑 4：高内存脏化率的 VM 迁移永不收敛

**现象：** 迁移进度达到 90%+ 后反复回退，VM 无法完成迁移。

**根因：** VM 内存写入速率超过了网络传输速率，脏页永远追不上。

**解决：**

```bash
# 1. 增大允许的停机时间
qm set 100 --migrate_downtime 2000

# 2. 尝试离线迁移（有停机时间但一定能完成）
qm migrate 100 pve2
# 不加 --online 即为离线迁移，速度更快

# 3. 内存超分配场景下，适当限制 VM 的 balloon 上限
qm set 100 --balloon 16384 --shares 8000
```

---

## 8. 总结

| 维度 | 结论 |
|------|------|
| ✅ **集群规模** | 最少 3 节点，避免双节点的 quorum 缺陷 |
| ✅ **共享存储** | NFS 入门，Ceph 进阶；共享存储是热迁移的关键前提 |
| ✅ **迁移模式** | 私有内网使用 `insecure` 模式，性能提升 30%+ |
| ✅ **CPU 类型** | 异构节点统一用 `x86-64-v2-AES`，同代可用 `host` |
| ✅ **HA 策略** | `restricted=1` 限制 VM 范围，`nofailback=0` 自动回迁 |
| ⚠️ **维护流程** | 维护前必须 `ha-manager set <vmid> --state disabled` |
| ⚠️ **定期演练** | 每月拔一次网线测试 HA 切换，确保故障恢复路径走通 |

**最后一条忠告：** HA 保护的是硬件故障，不保护数据损坏。HA 不是备份的替代品。即使上了集群，PBS 定时备份依然不可少。集群 + 共享存储 + HA + PBS = Homelab 虚拟化平台的终极形态。

---

*有没有在 PVE 集群上踩过什么神奇的坑？欢迎在评论区分享你的经历。*
