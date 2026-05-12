---
title: "一台主机跑出三张网：Proxmox VE VLAN、Bond 与防火墙分区实战"
description: "面向 Homelab 的 Proxmox VE 网络分层方案：用 VLAN-aware bridge、链路聚合、宿主机管理网隔离和 VM 分区，把一台机器切成多张逻辑网络。包含 ifupdown2 配置、PVE 命令行、路由与防火墙踩坑。"
date: 2026-05-12T01:00:34+08:00
image:
math:
license:
hidden: false
comments: true
draft: false
slug: "pve-vlan-bond-firewall"
---

## 前言

Homelab 最常见的扩张路径不是“再买一台服务器”，而是“把现有一台机器榨干”。问题也随之出现：NAS、Docker、Home Assistant、测试环境、监控告警、下载机全挤在一张网里，既不安全，也不好排障。今天这篇不讲花活，直接讲一个可落地的方案：**Proxmox VE 上用 VLAN-aware bridge + bond + 宿主机分区，把管理网、业务网、存储网拆开**。

这套方案适合：
- 只有一台 PVE 主机，但交换机支持 VLAN
- 想把管理入口、虚拟机业务流量、NAS/备份流量分离
- 想保留后续扩展到多节点集群的空间

重点不是“能不能通”，而是“出了问题怎么定位”。

---

## 1. 设计目标：先分层，再上配置

最容易犯的错，是直接在 `vmbr0` 上混跑所有流量：SSH、Web UI、Docker、NAS、备份、监控全部一锅端。短期能用，长期必炸。

推荐按下面逻辑拆：

| 网络 | VLAN | 作用 | 是否给宿主机 IP |
|---|---:|---|---|
| 管理网 | 10 | PVE Web / SSH / SSH Jump | ✅ |
| 业务网 | 20 | VM、LXC 的对外服务 | ❌ 或按需 |
| 存储网 | 30 | NFS / SMB / PBS / rsync | ✅（可选） |
| 隔离测试网 | 40 | 实验 VM、脏服务 | ❌ |

如果你只有一块网卡，也能做 VLAN；如果你有两块或更多网卡，就上 bond，把带宽和容错一起拿下。

---

## 2. 宿主机网络：bond + VLAN-aware bridge

### 2.1 交换机侧前提

先确认上联交换机端口是 trunk，允许通过这些 VLAN。示例：
- native/untagged：VLAN 10（管理网）
- tagged：VLAN 20/30/40

如果交换机不配对，PVE 配得再漂亮也没用。

### 2.2 `/etc/network/interfaces` 示例

下面给一个能直接改的模板。假设两张物理网卡 `enp3s0`、`enp4s0`，做 LACP bond：

```ini
auto lo
iface lo inet loopback

iface enp3s0 inet manual
iface enp4s0 inet manual

auto bond0
iface bond0 inet manual
    bond-slaves enp3s0 enp4s0
    bond-miimon 100
    bond-mode 802.3ad
    bond-xmit-hash-policy layer3+4

auto vmbr0
iface vmbr0 inet static
    address 192.168.10.2/24
    gateway 192.168.10.1
    bridge-ports bond0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 10 20 30 40
```

几个关键点：
- `bond-mode 802.3ad` 需要交换机也启用 LACP
- `bridge-vlan-aware yes` 是核心，不开就别谈 VLAN
- `bridge-vids` 明确允许通过的 VLAN，少走很多弯路
- 宿主机 IP 放在 untagged 或管理 VLAN 的方式，要和交换机 native VLAN 保持一致

### 2.3 应用配置前先验证

先别急着重启网络，先看现有链路：

```bash
ip link show
cat /proc/net/bonding/bond0
bridge vlan show
```

如果 bond 没起来，先检查：

```bash
dmesg | grep -i bond
ethtool enp3s0
ethtool enp4s0
```

真正上线前，建议通过 PVE console 或实体机旁路管理，避免把自己锁外面。

---

## 3. 给虚拟机分配 VLAN

PVE Web UI 创建 VM 时，网卡桥接选 `vmbr0`，然后在 VLAN Tag 填对应数字即可。命令行也可以直接改：

```bash
qm set 101 --net0 virtio,bridge=vmbr0,tag=20
qm set 102 --net0 virtio,bridge=vmbr0,tag=30
qm set 103 --net0 virtio,bridge=vmbr0,tag=40
```

如果是 LXC：

```bash
pct set 200 -net0 name=eth0,bridge=vmbr0,tag=20,ip=dhcp
```

这里有个很实用的原则：
- **管理类服务尽量放宿主机可达的 VLAN 10**
- **业务 VM 只拿它需要的 VLAN**
- **测试环境不要和生产共用广播域**

这样做以后，排障会轻很多。比如 Home Assistant 挂了，你能明确知道是 20 网段的 Docker 问题，不会怀疑 30 网段的 NAS。

---

## 4. 宿主机访问各 VLAN：不要乱开桥，优先加子接口

很多人为了让 PVE 宿主机也能访问存储网，直接再建一堆 bridge。我的建议是：**能用 VLAN 子接口就别额外造桥**。

例如给宿主机加一个 VLAN 30 的地址，用于 NFS/PBS：

```ini
auto vmbr0.30
iface vmbr0.30 inet static
    address 192.168.30.2/24
    vlan-raw-device vmbr0
```

这样宿主机就能直接访问 192.168.30.0/24 的存储网络，同时保持 `vmbr0` 作为统一承载层。

验证：

```bash
ip addr show vmbr0.30
ping -c 3 192.168.30.1
```

如果 ping 不通，优先查三件事：
1. 交换机 trunk 口是否允许 VLAN 30
2. `bridge vlan show` 是否包含 VLAN 30
3. 对端设备是否真的在 30 网段

---

## 5. PVE 防火墙分区：别把 VLAN 当安全边界的全部

VLAN 解决的是二层隔离，不是完整安全方案。真正稳妥的做法是：**VLAN + PVE Firewall + 服务端防火墙** 三层叠加。

### 5.1 在 PVE 上启用 Datacenter 防火墙

Web UI：
- Datacenter → Firewall → Enable
- Node → Firewall → Enable
- VM/CT → Firewall → Enable

命令行检查：

```bash
pve-firewall status
```

### 5.2 一个简单的分区策略

- 管理网 VLAN 10：只允许管理员 IP 访问 8006/22
- 业务网 VLAN 20：只允许 80/443/必要端口
- 存储网 VLAN 30：只允许 NFS/SMB/PBS 源地址
- 测试网 VLAN 40：默认拒绝，按需放行

示例：限制 PVE Web UI 只给管理网访问，可以在宿主机上配 `iptables`/`nftables`，也可以在上游防火墙做 ACL。PVE 自带防火墙更适合 VM 粒度控制。

一个最小示意：

```bash
# 只允许 192.168.10.0/24 访问 8006
pve-firewall compile
```

具体规则建议在 UI 里做，便于可视化维护。核心思想是：**不要让所有 VLAN 都能直接碰 PVE 管理面**。

---

## 6. 踩坑记录

### ❌ 1）bond 起来了，但流量不均衡

802.3ad 不等于单连接加速。LACP 以流会话哈希分发，单个 TCP 连接通常只走一条链路。不要指望单机 fio 立刻翻倍。

排查：
```bash
cat /proc/net/bonding/bond0
```

看 active slave 和 hash policy 是否符合预期。

### ❌ 2）VLAN-aware 开了，VM 还是不通

常见原因不是 PVE，而是交换机 trunk 没放行 VLAN，或者 native VLAN 配错。

排查顺序：
1. `bridge vlan show`
2. `ip link show vmbr0`
3. 交换机端口 VLAN 配置
4. VM 内是否正确拿到 tag

### ❌ 3）宿主机把自己锁死

改网络前，尽量保留实体控制台或 iDRAC/IPMI。PVE 网络配置错误会直接断掉 Web UI 和 SSH。

建议：先把配置写好，再 `ifreload -a`，别手工 `systemctl restart networking` 硬重启。

### ❌ 4）把宿主机和业务 VM 放同一网段

这会让故障范围扩大。宿主机是管理平面，业务 VM 是数据平面，尽量不要混。

---

## 7. 最后给一份落地建议

| 场景 | 推荐方案 | 评价 |
|---|---|---|
| 单网卡入门 | VLAN-aware bridge | ✅ 简单，够用 |
| 双网卡以上 | bond + VLAN-aware bridge | 🏆 更稳，更适合 Homelab |
| 需要宿主机访问存储网 | VLAN 子接口 | ✅ 易维护 |
| 追求细粒度隔离 | VLAN + PVE Firewall | 🏆 安全性更高 |
| 只想“能通” | 纯平面网络 | ❌ 后期排障痛苦 |

一句话总结：**把 PVE 主机做成“多逻辑网络汇聚点”，比把所有服务塞进一张网里更适合长期运维。**

如果你已经在跑 Proxmox VE，下一步最值得做的不是再装一个花哨服务，而是先把网络边界切清楚。网络一分层，整个 Homelab 的可维护性会直接上一个台阶。
