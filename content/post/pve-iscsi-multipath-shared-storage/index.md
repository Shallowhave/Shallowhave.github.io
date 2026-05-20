---
title: "PVE 共享存储进阶：用 iSCSI + Multipath 搭一套可迁移、不断链的 Homelab 存储底座"
description: "在 Homelab 中把 NAS/SAN 通过 iSCSI 接入 Proxmox VE，并用 multipath 做链路冗余，再在多路径设备上创建 LVM/LVM-Thin 存储池，实现虚拟机磁盘集中管理和节点间迁移。"
date: 2026-05-20T01:01:23+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
slug: "pve-iscsi-multipath-shared-storage"
---

## 前言

Homelab 里的 PVE 节点越跑越多以后，最先暴露的问题通常不是 CPU 不够，而是 **虚拟机磁盘被绑死在某一台机器上**：A 节点的 NVMe 很快，但机器要关机清灰；B 节点还有空闲资源，却因为磁盘不在本地而不能直接迁移。PBS 能解决备份恢复，但它不是在线共享存储；NFS 配置简单，但随机写延迟和文件锁在高 IO 场景下不一定舒服。

这篇文章记录一套更偏“准生产”的 Homelab 存储方案：**NAS/SAN 导出 iSCSI LUN，PVE 节点通过多张网卡访问，宿主机用 `multipath-tools` 聚合多条链路，然后在 multipath 设备上创建 LVM 或 LVM-Thin，最后交给 Proxmox VE 管理 VM 磁盘**。

它适合这些场景：

- 有 2 台以上 PVE 节点，希望 VM 磁盘可以随节点迁移；
- NAS 或存储服务器有双网口/多网口，想避免单根网线或单个交换机端口故障；
- 不想上 Ceph，但又希望比单机本地盘更灵活；
- 能接受“共享块存储需要更谨慎操作”的复杂度。

> 先说结论：Homelab 里最省心的顺序是 **单节点本地 ZFS → NFS 共享 → iSCSI 单路径 → iSCSI Multipath + LVM**。不要为了“看起来高级”直接跳到最后一档，除非你真的需要在线迁移和链路冗余。

## 1. 方案对比：NFS、iSCSI、Ceph 到底怎么选？

| 方案 | 优点 | 缺点 | 适合场景 |
| --- | --- | --- | --- |
| 本地 ZFS / LVM-Thin | 延迟低、简单、故障域清晰 ✅ | VM 磁盘绑定节点，迁移依赖备份/复制 | 单节点或性能优先 |
| NFS | 配置简单、文件级管理、快照友好 ✅ | 小 IO 延迟偏高，强依赖 NAS 稳定性 | ISO、备份、轻负载 VM |
| iSCSI 单路径 | 块设备语义，性能通常比 NFS 稳 | 单链路故障会直接影响 VM | 单节点接 NAS，链路可靠 |
| iSCSI + Multipath | 多链路冗余，适合 PVE 集群 🏆 | 配置复杂，误操作可能影响所有节点 | 多节点共享块存储 |
| Ceph | 分布式、高可用、PVE 原生集成 | 至少 3 节点，内存/网络/磁盘要求高 | 真正集群化 Homelab |

需要注意：**iSCSI 只是把远端 LUN 暴露成块设备**，它本身不懂“文件系统共享”。如果多个 PVE 节点同时把同一个普通文件系统挂起来，基本就是数据损坏倒计时。正确姿势是：把 iSCSI LUN 交给 PVE 的 LVM/LVM-Thin 存储插件，或者使用支持集群语义的文件系统/存储管理层。

## 2. 拓扑规划：先把存储网络隔离出来

示例环境：

| 角色 | 管理网 | iSCSI-A | iSCSI-B | 说明 |
| --- | --- | --- | --- | --- |
| PVE-1 | `192.168.10.11` | `10.10.10.11` | `10.10.20.11` | 双路径访问 LUN |
| PVE-2 | `192.168.10.12` | `10.10.10.12` | `10.10.20.12` | 双路径访问 LUN |
| NAS/SAN | `192.168.10.20` | `10.10.10.20` | `10.10.20.20` | 导出同一个 iSCSI target |

建议：

1. iSCSI 流量和管理网分开，不要和公网反代、下载、监控共用同一个 VLAN；
2. 有条件就两张独立网卡、两个独立交换机；没条件至少两个 VLAN；
3. MTU 先用 1500，稳定后再考虑 9000 Jumbo Frame；
4. 每个 PVE 节点看到的 LUN 必须是同一个 WWID，否则 multipath 无法正确聚合。

PVE 上可以这样配置两个存储网口（示例为 `/etc/network/interfaces` 片段）：

```ini
auto enp3s0
iface enp3s0 inet static
    address 10.10.10.11/24
    mtu 1500

auto enp4s0
iface enp4s0 inet static
    address 10.10.20.11/24
    mtu 1500
```

如果你使用 VLAN，则可以挂在 Linux bridge 或物理网卡子接口上：

```ini
auto enp3s0.110
iface enp3s0.110 inet static
    address 10.10.10.11/24
    vlan-raw-device enp3s0

auto enp4s0.120
iface enp4s0.120 inet static
    address 10.10.20.11/24
    vlan-raw-device enp4s0
```

改完网络后先验证连通性：

```bash
ping -c 3 10.10.10.20
ping -c 3 10.10.20.20
ip route get 10.10.10.20
ip route get 10.10.20.20
```

## 3. NAS 侧：创建 iSCSI Target 和 LUN

不同 NAS UI 名字不一样，但核心对象基本一致：

- **Portal / Network Portal**：监听哪些 IP，例如 `10.10.10.20`、`10.10.20.20`；
- **Target**：iSCSI 目标，通常形如 `iqn.2026-05.lab.nas:pve-vmstore`；
- **LUN**：实际块设备，可以来自 ZVOL、厚置备文件、稀疏文件、块设备；
- **ACL / Initiator**：允许哪些 PVE 节点访问。

如果是 TrueNAS SCALE，我更推荐用 **ZVOL 作为 LUN 后端**，并注意：

| 参数 | 建议值 | 原因 |
| --- | --- | --- |
| ZVOL block size | `16K` 或 `32K` | 兼顾 VM 随机 IO 和空间效率 |
| Sync | `STANDARD` | 除非你有断电保护和 SLOG，否则别乱关同步 |
| Compression | `lz4` | CPU 开销低，虚拟磁盘常有压缩收益 |
| Sparse | 谨慎开启 | 稀疏省空间，但更需要监控池剩余容量 |

创建完成后，在 PVE 节点扫描 target：

```bash
apt update
apt install -y open-iscsi multipath-tools lsscsi sg3-utils

iscsiadm -m discovery -t sendtargets -p 10.10.10.20
iscsiadm -m discovery -t sendtargets -p 10.10.20.20
```

登录两条路径：

```bash
iscsiadm -m node -p 10.10.10.20 --login
iscsiadm -m node -p 10.10.20.20 --login

# 设置开机自动登录
iscsiadm -m node -p 10.10.10.20 --op update -n node.startup -v automatic
iscsiadm -m node -p 10.10.20.20 --op update -n node.startup -v automatic
```

检查系统是否看到两个 SCSI 路径：

```bash
lsscsi -s
lsblk -o NAME,TYPE,SIZE,MODEL,SERIAL,WWN
```

此时通常会看到 `/dev/sdb`、`/dev/sdc` 之类两个设备，但它们其实指向同一个 LUN。**不要直接对 `/dev/sdb` 做 `pvcreate`**，下一步必须先交给 multipath。

## 4. PVE 侧：配置 multipath 聚合链路

先确认 LUN 的 WWID：

```bash
multipath -ll
/usr/lib/udev/scsi_id -g -u -d /dev/sdb
/usr/lib/udev/scsi_id -g -u -d /dev/sdc
```

如果两个路径返回同一个 WWID，就可以创建 `/etc/multipath.conf`：

```ini
defaults {
    user_friendly_names yes
    find_multipaths yes
    polling_interval 5
    path_selector "service-time 0"
    path_grouping_policy multibus
    path_checker tur
    failback immediate
    no_path_retry queue
}

blacklist {
    devnode "^(ram|raw|loop|fd|md|dm-|sr|scd|st)[0-9]*"
    devnode "^zd[0-9]*"
}

multipaths {
    multipath {
        wwid 3600c0ff0000000000000000000000001
        alias pve_vmstore
    }
}
```

> 把上面的 `wwid` 换成你实际查到的值。`alias pve_vmstore` 会生成 `/dev/mapper/pve_vmstore`，后面所有 LVM 操作都只对这个设备做。

启动并重新加载：

```bash
systemctl enable --now multipathd
multipath -F
multipath -r
multipath -ll
```

正常输出应该类似：

```text
pve_vmstore (3600c0ff0000000000000000000000001) dm-3 TrueNAS,iSCSI Disk
size=2.0T features='1 queue_if_no_path' hwhandler='0' wp=rw
|-+- policy='service-time 0' prio=1 status=active
| `- 8:0:0:0 sdb 8:16 active ready running
`-+- policy='service-time 0' prio=1 status=enabled
  `- 9:0:0:0 sdc 8:32 active ready running
```

可以做一个非破坏性的路径测试：拔掉其中一根存储网线，或者在交换机上临时关闭一个端口，然后观察：

```bash
watch -n 1 'multipath -ll pve_vmstore; echo; iscsiadm -m session'
```

只要还有一条路径是 `active ready running`，VM IO 就不应该立刻中断。恢复链路后，multipath 会重新把路径加回来。

## 5. 在 multipath 设备上创建 LVM / LVM-Thin

如果这是全新的 LUN，可以初始化成 LVM。再次强调：**确认设备名是 `/dev/mapper/pve_vmstore`，不是 `/dev/sdb` 或 `/dev/sdc`**。

```bash
# 危险操作：只对全新 LUN 执行，已有数据会被清理
wipefs -n /dev/mapper/pve_vmstore   # 先预览
wipefs -a /dev/mapper/pve_vmstore

pvcreate /dev/mapper/pve_vmstore
vgcreate vg_iscsi_vmstore /dev/mapper/pve_vmstore
vgs
pvs
```

### 5.1 方案 A：普通 LVM（更稳，适合多节点共享）

普通 LVM 不支持精细的 thin provisioning，但语义更直观，适合 PVE 集群共享块存储：

```bash
pvesm add lvm iscsi-lvm \
  --vgname vg_iscsi_vmstore \
  --content images \
  --shared 1
```

查看 `/etc/pve/storage.cfg` 会出现类似配置：

```ini
lvm: iscsi-lvm
        vgname vg_iscsi_vmstore
        content images
        shared 1
```

### 5.2 方案 B：LVM-Thin（省空间，但要更严格监控）

如果你明确知道自己需要精简配置，可以创建 thin pool：

```bash
lvcreate -L 1.8T -T vg_iscsi_vmstore/vmthin

pvesm add lvmthin iscsi-lvmthin \
  --vgname vg_iscsi_vmstore \
  --thinpool vmthin \
  --content images \
  --shared 1
```

但是我不建议在所有 Homelab 场景都默认上 LVM-Thin，原因是：thin pool 一旦写满，故障表现会非常难看；如果底层 NAS 也是稀疏 ZVOL，相当于 **上层 thin + 下层 sparse**，空间超卖风险会叠加。

至少加上监控：

```bash
lvs -a -o+seg_monitor,metadata_percent,data_percent
vgs
pvesm status
```

可以写一个简单的告警脚本：

```bash
#!/usr/bin/env bash
set -euo pipefail
THRESHOLD=80
lvs --noheadings --separator ',' -o lv_name,data_percent,metadata_percent vg_iscsi_vmstore \
  | while IFS=',' read -r lv data meta; do
      data=${data%.*}; meta=${meta%.*}
      if [ "${data:-0}" -ge "$THRESHOLD" ] || [ "${meta:-0}" -ge "$THRESHOLD" ]; then
        echo "WARN: $lv thin usage data=${data}% meta=${meta}%"
      fi
    done
```

## 6. PVE 集群里的关键细节

### 6.1 所有节点都要看到同一个 multipath alias

在 PVE-1 配好 `/etc/multipath.conf` 后，不要只复制一半。PVE-2、PVE-3 上也要安装同样组件、登录同样 target，并确保：

```bash
multipath -ll pve_vmstore
ls -l /dev/mapper/pve_vmstore
pvscan --cache
vgscan
vgs
```

如果某个节点看不到 VG，可以执行：

```bash
pvscan --cache
vgchange -ay vg_iscsi_vmstore
systemctl restart pvedaemon pvestatd
```

### 6.2 `shared 1` 不是魔法高可用

`shared 1` 的意思是：PVE 知道这个存储在多个节点可见，可以用于迁移和跨节点调度。它不代表：

- NAS 故障时 VM 还能继续运行；
- iSCSI 网络抖动不会影响数据库；
- 多个节点可以随便同时挂载同一个普通文件系统；
- 不需要备份。

真正的高可用至少还需要：独立供电、UPS、存储链路冗余、PVE HA 策略、PBS 备份和恢复演练。

### 6.3 迁移测试

创建一个测试 VM，把磁盘放到新存储池：

```bash
qm create 9000 --name test-iscsi --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0
qm set 9000 --scsihw virtio-scsi-single
qm set 9000 --scsi0 iscsi-lvm:32,discard=on,iothread=1
qm config 9000
```

在 Web UI 或 CLI 做迁移：

```bash
qm migrate 9000 pve-2 --online
```

迁移成功后确认磁盘没有被复制，而是直接在共享存储上切换归属：

```bash
qm status 9000
pvesm list iscsi-lvm
```

## 7. 性能测试：不要只看顺序读写

iSCSI 共享存储最怕“看起来带宽很高，实际 VM 卡顿”。建议至少测三类：4K 随机读写、混合读写、延迟分布。

在测试 VM 内执行：

```bash
apt update && apt install -y fio

# 4K 随机读
fio --name=randread --filename=/tmp/fio.test --size=8G \
  --rw=randread --bs=4k --iodepth=32 --numjobs=4 \
  --direct=1 --runtime=60 --time_based --group_reporting

# 4K 随机写
fio --name=randwrite --filename=/tmp/fio.test --size=8G \
  --rw=randwrite --bs=4k --iodepth=32 --numjobs=4 \
  --direct=1 --runtime=60 --time_based --group_reporting

# 70/30 混合负载，更接近数据库/容器盘
fio --name=randrw --filename=/tmp/fio.test --size=8G \
  --rw=randrw --rwmixread=70 --bs=4k --iodepth=32 --numjobs=4 \
  --direct=1 --runtime=60 --time_based --group_reporting
```

同时在 PVE 宿主机观察：

```bash
iostat -x 1
multipath -ll pve_vmstore
iftop -i enp3s0
iftop -i enp4s0
```

经验值：

| 指标 | 更应该关注什么 | 说明 |
| --- | --- | --- |
| 顺序吞吐 | 是否接近网卡上限 | 10GbE 下单流能跑满才有意义 |
| 4K 随机 IOPS | 数据库/容器响应 | 比大文件复制更重要 |
| `clat` 延迟 | P95/P99 是否飙升 | 卡顿往往来自尾延迟 |
| 路径分布 | 两条链路是否都有流量 | `service-time` 不等于简单 50/50 |

## 8. 踩坑记录

### ❌ 坑 1：对 `/dev/sdb` 初始化，重启后设备名变了

现象：一开始能用，重启后 VG 丢失，或者 multipath 里出现异常路径。

原因：`/dev/sdb`、`/dev/sdc` 是内核动态分配名，不稳定；多路径场景下它们只是同一个 LUN 的不同路径。

修复：只使用 `/dev/mapper/<alias>` 或 `/dev/disk/by-id/dm-uuid-mpath-*`，并把 LVM 建在 multipath 设备上。

### ❌ 坑 2：NAS 上开了稀疏 LUN，PVE 又开 LVM-Thin，结果空间被写爆

现象：PVE 里看 thin pool 还有空间，NAS 存储池却满了，VM 突然 IO error。

解决：不要双层超卖；如果必须 thin，就给 NAS 池和 LVM thin pool 都做 70%/80% 告警，并定期检查：

```bash
pvesm status
lvs -a -o+data_percent,metadata_percent
```

### ❌ 坑 3：所有路径都断开后，VM 卡死很久

`no_path_retry queue` 可以让 IO 在路径消失时排队，适合短暂链路抖动；但如果 NAS 真挂了，VM 会一直等。更保守的做法是设置有限重试，例如：

```ini
defaults {
    no_path_retry 12
    polling_interval 5
}
```

这代表大约等待 60 秒后返回 IO 错误。选择排队还是失败，要看你的业务：数据库可能宁愿等，某些无状态服务则宁愿快速失败后由 HA 拉起。

### ❌ 坑 4：Jumbo Frame 只改了一端

MTU 9000 必须链路两端和中间交换机都支持。否则会出现小包正常、大包丢失、iSCSI 随机超时的诡异问题。

验证方法：

```bash
ping -M do -s 8972 10.10.10.20
ping -M do -s 8972 10.10.20.20
```

如果不通，先退回 MTU 1500。Homelab 里稳定通常比多挤出一点吞吐更重要。

## 9. 我的推荐配置

| 场景 | 推荐方案 | 理由 |
| --- | --- | --- |
| 单 PVE 节点 | 本地 ZFS / LVM-Thin ✅ | 简单、低延迟、好排障 |
| 单节点 + NAS | NFS 放 ISO/备份，iSCSI 放少量高 IO VM | 分工清晰 |
| 2-3 节点轻量集群 | iSCSI + Multipath + 普通 LVM 🏆 | 可迁移，复杂度可控 |
| 需要节省空间 | LVM-Thin，但必须做告警 | 防止 thin pool 写满 |
| 追求真正高可用 | Ceph 或商业 SAN | 单 NAS 不是 HA |

## 总结

iSCSI + Multipath 不是 Homelab 的必选项，但它能把 PVE 的存储能力从“每台机器各管各的”推进到“多节点共享一块可靠的块存储”。这套方案的核心不是某一条命令，而是三条原则：**存储网络隔离、所有路径统一进 multipath、PVE 只管理 multipath 之上的 LVM**。

如果你已经完成了本地 ZFS、PBS 备份和基础监控，再考虑把 NAS/SAN 作为共享块存储接入 PVE，会比一开始就堆复杂架构更稳。最后记住一句话：共享存储提高的是迁移和调度能力，不会替代备份；上线前一定要做断链、迁移和恢复测试。
