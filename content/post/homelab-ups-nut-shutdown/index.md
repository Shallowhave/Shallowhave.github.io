---
title: "停电不再裸奔：Homelab UPS + NUT 自动关机实战"
description: "使用 Network UPS Tools（NUT）把 UPS 接入 Homelab，实现 Proxmox VE、Docker 主机与 NAS 的统一电源监控、告警和低电量自动关机，避免停电导致虚拟机、ZFS 与数据库损坏。"
date: 2026-05-17T01:01:41+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
slug: "homelab-ups-nut-shutdown"
---

## 前言

Homelab 里很多人愿意花时间折腾 2.5G 网卡、GPU 直通、ZFS、监控面板，却经常忽略一个最基础的问题：**突然停电时，谁来安全关机？**

如果 UPS 只是插在墙上当“超大号插排”，那它只能帮你多撑几分钟。真正有价值的做法是让 UPS 成为基础设施的一部分：电池状态能被监控，断电能发告警，电量低时能按顺序关停 Docker、虚拟机、NAS 和 Proxmox VE 宿主机。

本文用 Network UPS Tools（NUT）搭一套适合 Homelab 的 UPS 自动关机方案。它的特点是：

- 一台机器通过 USB 直连 UPS，作为 **NUT Server**；
- PVE、Docker 主机、NAS 等其他机器作为 **NUT Client**；
- 停电后先等待一段时间，避免短暂闪断误关机；
- 电量低或剩余时间不足时，自动执行安全关机；
- 可接入 Prometheus / Grafana 做可视化监控。

官方 NUT 文档里也明确了它的核心模式：`standalone`、`netserver`、`netclient`。Homelab 多主机环境推荐用 `netserver + netclient`，不要每台机器都各接一根 USB 线。

---

## 1. 架构设计

假设环境如下：

| 角色 | 主机 | IP | 说明 |
|------|------|----|------|
| NUT Server | pve01 | `192.168.1.10` | USB 直连 UPS，运行 `upsd` 和 `upsmon` |
| NUT Client | pve02 | `192.168.1.11` | 只运行 `upsmon` |
| NUT Client | docker01 | `192.168.1.20` | Docker 服务主机 |
| NUT Client | nas01 | `192.168.1.30` | NAS / 文件服务 |

整体链路：

```text
          USB
 UPS ─────────── pve01 / NUT Server
                  │
                  │ TCP 3493
        ┌─────────┼─────────┐
        │         │         │
      pve02    docker01   nas01
    netclient netclient netclient
```

关机策略建议：

| 事件 | 动作 |
|------|------|
| 停电 0-3 分钟 | 只告警，不关机，过滤短暂闪断 |
| 停电超过 3 分钟 | 关闭非核心 Docker 服务 |
| UPS 进入 LOWBATT | 所有 client 执行系统关机 |
| Server 最后关机 | PVE 停 VM/CT 后宿主机关机 |

这里最关键的一点是：**不要让所有机器同时暴力断电**。PVE 宿主机关机前会按 VM/CT 的启动/关机顺序执行 shutdown，但 Docker、数据库、NAS 最好也各自收到关机信号。

---

## 2. 在 PVE 上安装 NUT Server

以下命令在 USB 直连 UPS 的主机上执行，例如 `pve01`。

```bash
apt update
apt install -y nut nut-client nut-server usbutils
```

先确认系统能看到 UPS：

```bash
lsusb
```

常见输出类似：

```text
Bus 001 Device 004: ID 051d:0002 American Power Conversion Uninterruptible Power Supply
```

也可以用 NUT 自带扫描工具：

```bash
nut-scanner -U
```

如果能识别，通常会输出推荐 driver，例如：

```ini
[nutdev1]
    driver = "usbhid-ups"
    port = "auto"
    vendorid = "051D"
    productid = "0002"
```

大多数 APC、Eaton、CyberPower 家用 UPS 都可以先试 `usbhid-ups`。如果你的 UPS 是串口、SNMP 或厂商私有协议，需要查 NUT Hardware Compatibility List 再换 driver。

---

## 3. 配置 NUT Server

### 3.1 `/etc/nut/nut.conf`

Server 端使用 `netserver` 模式：

```bash
nano /etc/nut/nut.conf
```

```ini
MODE=netserver
```

注意 NUT 官方手册特别强调：`nut.conf` 这种文件里 **等号两边不能有空格**。写成 `MODE = netserver` 可能导致服务脚本不按预期工作。

### 3.2 `/etc/nut/ups.conf`

定义 UPS 名称。这里把 UPS 命名为 `homelab-ups`：

```bash
nano /etc/nut/ups.conf
```

```ini
[homelab-ups]
    driver = usbhid-ups
    port = auto
    desc = "Homelab APC UPS"
    pollinterval = 5
```

如果你的 UPS 不稳定，可以指定 vendor/product：

```ini
[homelab-ups]
    driver = usbhid-ups
    port = auto
    vendorid = 051d
    productid = 0002
    desc = "APC Back-UPS"
```

### 3.3 `/etc/nut/upsd.conf`

让 `upsd` 监听局域网地址：

```bash
nano /etc/nut/upsd.conf
```

```ini
LISTEN 127.0.0.1 3493
LISTEN 192.168.1.10 3493
```

不建议直接 `LISTEN 0.0.0.0` 暴露到所有网卡，尤其是你的 PVE 还有公网、旁路由或实验 VLAN 时。UPS 控制接口应该只在管理网段可达。

### 3.4 `/etc/nut/upsd.users`

创建两个用户：一个用于监控，一个用于执行关机。

```bash
nano /etc/nut/upsd.users
```

```ini
[monuser]
    password = change-this-monitor-password
    upsmon slave

[admin]
    password = change-this-admin-password
    actions = SET
    instcmds = ALL
```

生产环境别照抄密码。至少用：

```bash
openssl rand -base64 24
```

生成随机密码。

### 3.5 `/etc/nut/upsmon.conf`

Server 自己也要监控 UPS，并在 LOWBATT 时关机：

```bash
nano /etc/nut/upsmon.conf
```

核心配置如下：

```ini
MONITOR homelab-ups@localhost 1 monuser change-this-monitor-password master

MINSUPPLIES 1
SHUTDOWNCMD "/sbin/shutdown -h +0"
POLLFREQ 5
POLLFREQALERT 2
HOSTSYNC 15
DEADTIME 15
POWERDOWNFLAG /etc/killpower

NOTIFYFLAG ONLINE     SYSLOG+WALL
NOTIFYFLAG ONBATT     SYSLOG+WALL+EXEC
NOTIFYFLAG LOWBATT    SYSLOG+WALL+EXEC
NOTIFYFLAG FSD        SYSLOG+WALL+EXEC
NOTIFYFLAG COMMOK     SYSLOG
NOTIFYFLAG COMMBAD    SYSLOG+WALL
```

几个关键参数解释：

| 参数 | 含义 | 建议 |
|------|------|------|
| `master` | 当前机器直连 UPS，有最终关机权 | Server 必须是 master |
| `MINSUPPLIES` | 至少几个 UPS 在线才认为供电正常 | 单 UPS 写 1 |
| `DEADTIME` | 多久收不到 UPS 数据认为通信中断 | 15-30 秒 |
| `HOSTSYNC` | 通知 client 关机后的等待时间 | 多机器可设 15-60 秒 |
| `POWERDOWNFLAG` | 系统关机后通知 UPS 断电的标记 | Debian 默认可用 |

---

## 4. 启动并验证 Server

重启服务：

```bash
systemctl restart nut-driver nut-server nut-monitor
systemctl enable nut-driver nut-server nut-monitor
```

检查状态：

```bash
systemctl status nut-driver --no-pager
systemctl status nut-server --no-pager
systemctl status nut-monitor --no-pager
```

读取 UPS 数据：

```bash
upsc homelab-ups@localhost
```

正常会看到类似：

```text
battery.charge: 100
battery.runtime: 2840
input.voltage: 229.0
ups.load: 18
ups.status: OL
```

常见 `ups.status`：

| 状态 | 含义 |
|------|------|
| `OL` | Online，市电供电 |
| `OB` | On Battery，电池供电 |
| `LB` | Low Battery，低电量 |
| `OL CHRG` | 市电供电并充电 |

从其他主机测试网络访问：

```bash
upsc homelab-ups@192.168.1.10
```

如果连接失败，优先检查：

```bash
ss -lntp | grep 3493
pve-firewall status
iptables -S | grep 3493
```

PVE 防火墙开启时，需要放行管理网段访问 TCP 3493。

---

## 5. 配置 NUT Client

在 `pve02`、`docker01`、`nas01` 上安装 client：

```bash
apt update
apt install -y nut-client
```

配置 `/etc/nut/nut.conf`：

```ini
MODE=netclient
```

配置 `/etc/nut/upsmon.conf`：

```ini
MONITOR homelab-ups@192.168.1.10 1 monuser change-this-monitor-password slave

MINSUPPLIES 1
SHUTDOWNCMD "/sbin/shutdown -h +0"
POLLFREQ 5
POLLFREQALERT 2
HOSTSYNC 15
DEADTIME 15
POWERDOWNFLAG /etc/killpower

NOTIFYFLAG ONLINE     SYSLOG+WALL
NOTIFYFLAG ONBATT     SYSLOG+WALL
NOTIFYFLAG LOWBATT    SYSLOG+WALL
NOTIFYFLAG FSD        SYSLOG+WALL
NOTIFYFLAG COMMBAD    SYSLOG+WALL
```

启动：

```bash
systemctl restart nut-monitor
systemctl enable nut-monitor
systemctl status nut-monitor --no-pager
```

验证 client 能读到 UPS：

```bash
upsc homelab-ups@192.168.1.10 battery.charge
upsc homelab-ups@192.168.1.10 ups.status
```

---

## 6. 给 Docker 主机加优雅停机脚本

默认 `shutdown -h +0` 会让 systemd 停止 Docker 服务，但复杂场景下我更建议显式关停关键 compose 项目，尤其是数据库、媒体库、下载器这类写盘频繁的服务。

创建脚本：

```bash
mkdir -p /usr/local/sbin
nano /usr/local/sbin/homelab-ups-shutdown.sh
chmod +x /usr/local/sbin/homelab-ups-shutdown.sh
```

内容示例：

```bash
#!/usr/bin/env bash
set -euo pipefail

logger -t ups-shutdown "UPS low battery: stopping Docker compose stacks"

for dir in \
  /opt/compose/immich \
  /opt/compose/homeassistant \
  /opt/compose/monitoring \
  /opt/compose/media; do
  if [ -f "$dir/docker-compose.yml" ] || [ -f "$dir/compose.yml" ]; then
    logger -t ups-shutdown "Stopping compose stack: $dir"
    docker compose -f "$dir/docker-compose.yml" down --timeout 60 || true
  fi
done

logger -t ups-shutdown "Stopping docker service"
systemctl stop docker || true

logger -t ups-shutdown "Powering off host"
/sbin/shutdown -h +0
```

然后把 client 的 `upsmon.conf` 改为：

```ini
SHUTDOWNCMD "/usr/local/sbin/homelab-ups-shutdown.sh"
```

如果你的 compose 文件统一叫 `compose.yml`，脚本里可以改成：

```bash
docker compose -f "$dir/compose.yml" down --timeout 60 || true
```

---

## 7. Proxmox VE 里的关机顺序

PVE 宿主机收到 shutdown 后，会尝试停止 VM/CT。建议给关键虚拟机设置启动/关机顺序和 timeout：

```bash
# 查看 VM 配置
qm config 100

# 设置 VM 启动顺序、延迟、关机超时
qm set 100 --startup order=10,up=60,down=120
qm set 101 --startup order=20,up=30,down=90

# LXC 同理
pct set 200 --startup order=30,up=20,down=60
```

建议顺序：

| 类型 | order | 说明 |
|------|-------|------|
| DNS / DHCP / ADGuard | 10 | 网络基础服务先启动、后关闭 |
| NAS / 数据库 | 20 | 写盘服务给更长 shutdown timeout |
| Home Assistant | 30 | 智能家居服务 |
| 下载器 / 测试机 | 90 | 可最先关闭 |

PVE 里还要检查 VM 是否安装了 QEMU Guest Agent：

```bash
qm agent 100 ping
```

如果返回失败，进入虚拟机安装：

```bash
# Debian / Ubuntu VM
apt install -y qemu-guest-agent
systemctl enable --now qemu-guest-agent
```

然后在 PVE 开启：

```bash
qm set 100 --agent enabled=1
```

没有 Guest Agent 的 VM，PVE 只能发 ACPI shutdown，遇到系统卡死或 ACPI 不响应时，最终会 timeout 后强制停止，数据风险更高。

---

## 8. 告警：接入 Gotify / Telegram / Webhook

NUT 可以通过 `NOTIFYCMD` 执行自定义脚本。下面以 Gotify 为例。

创建通知脚本：

```bash
nano /usr/local/sbin/nut-notify.sh
chmod +x /usr/local/sbin/nut-notify.sh
```

```bash
#!/usr/bin/env bash
set -euo pipefail

EVENT="${NOTIFYTYPE:-UNKNOWN}"
MESSAGE="${UPSNAME:-UPS}: ${EVENT} - ${1:-no message}"

curl -fsS \
  -X POST "https://gotify.example.com/message?token=YOUR_GOTIFY_TOKEN" \
  -F "title=Homelab UPS" \
  -F "message=${MESSAGE}" \
  -F "priority=8" || true

logger -t nut-notify "$MESSAGE"
```

在 Server 的 `/etc/nut/upsmon.conf` 添加：

```ini
NOTIFYCMD "/usr/local/sbin/nut-notify.sh"

NOTIFYFLAG ONLINE     SYSLOG+WALL+EXEC
NOTIFYFLAG ONBATT     SYSLOG+WALL+EXEC
NOTIFYFLAG LOWBATT    SYSLOG+WALL+EXEC
NOTIFYFLAG FSD        SYSLOG+WALL+EXEC
NOTIFYFLAG COMMBAD    SYSLOG+WALL+EXEC
```

重启：

```bash
systemctl restart nut-monitor
```

如果你已经有 Prometheus + Grafana 监控栈，可以再加一个 `nut-exporter`。Docker Compose 示例：

```yaml
services:
  nut-exporter:
    image: hon95/prometheus-nut-exporter:latest
    container_name: nut-exporter
    restart: unless-stopped
    ports:
      - "9199:9199"
    environment:
      - HTTP_PATH=/metrics
      - NUT_EXPORTER_SERVER=192.168.1.10
      - NUT_EXPORTER_USERNAME=monuser
      - NUT_EXPORTER_PASSWORD=change-this-monitor-password
```

Prometheus 抓取：

```yaml
scrape_configs:
  - job_name: "nut"
    static_configs:
      - targets: ["192.168.1.20:9199"]
```

Grafana 里重点看：

- `battery_charge`：电量百分比；
- `battery_runtime`：预计剩余秒数；
- `ups_load`：负载比例；
- `ups_status`：OL / OB / LB 状态。

---

## 9. 测试流程：不要第一次停电才知道配置错了

测试分三层做，不要一上来拔 UPS 市电。

### 9.1 通信测试

```bash
upsc homelab-ups@localhost
upsc homelab-ups@192.168.1.10 ups.status
```

### 9.2 通知测试

NUT 可以模拟事件：

```bash
upsmon -c fsd
```

注意：`fsd` 是 Forced Shutdown，会触发关机流程。测试前要确认你真的准备好了。更安全的做法是先单独运行通知脚本：

```bash
NOTIFYTYPE=ONBATT UPSNAME=homelab-ups /usr/local/sbin/nut-notify.sh "test message"
```

### 9.3 真实断电演练

建议第一次演练按这个顺序：

1. 暂停重要下载、备份、数据库迁移任务；
2. 打开多个 SSH 窗口观察日志；
3. 拔掉 UPS 市电输入，不要拔服务器电源；
4. 观察状态是否从 `OL` 变成 `OB`；
5. 等 1-2 分钟确认告警；
6. 插回市电，确认状态恢复 `OL`。

观察日志：

```bash
journalctl -u nut-monitor -f
journalctl -u nut-server -f
```

确认状态：

```bash
watch -n 2 'upsc homelab-ups@localhost ups.status battery.charge battery.runtime ups.load'
```

---

## 10. 踩坑记录

### ❌ 1. USB 权限导致 driver 起不来

症状：

```text
Can't claim USB device: could not detach kernel driver from interface
```

处理：

```bash
systemctl stop nut-driver
udevadm control --reload-rules
udevadm trigger
systemctl start nut-driver
```

如果仍然失败，检查 `/lib/udev/rules.d/` 里是否有 NUT 规则，或者临时重插 UPS USB。

### ❌ 2. PVE 防火墙拦了 3493

client 上执行：

```bash
upsc homelab-ups@192.168.1.10
```

如果超时，Server 上检查监听：

```bash
ss -lntp | grep 3493
```

PVE 防火墙放行示例：

```bash
pve-firewall status
```

在 Datacenter 或 Node 防火墙里添加规则：

```text
IN ACCEPT -source 192.168.1.0/24 -p tcp -dport 3493
```

### ❌ 3. `upsmon.conf` 里 master/slave 写反

Server 直连 UPS，必须是：

```ini
MONITOR homelab-ups@localhost 1 monuser password master
```

Client 必须是：

```ini
MONITOR homelab-ups@192.168.1.10 1 monuser password slave
```

写反后可能导致 client 试图执行不该执行的电源控制，或者 Server 不负责最终关机。

### ❌ 4. 只关 PVE，不关 NAS

如果 NAS 是独立机器，但没有配置 NUT client，停电后 PVE 安全关机了，NAS 仍然硬扛到 UPS 没电。ZFS、Btrfs、数据库最怕这种断电。凡是接在同一台 UPS 上的主机，都应该纳入 NUT。

### ❌ 5. UPS 负载过高，runtime 虚高

UPS 标称 1000VA 不等于可以长时间带 1000W。建议平时让 `ups.load` 控制在 40%-60% 以下。负载越高，电池老化后 runtime 掉得越快。

查看负载：

```bash
upsc homelab-ups@localhost ups.load
upsc homelab-ups@localhost battery.runtime
```

---

## 总结

Homelab 的电源保护不是“买个 UPS 就完事”，而是要把 UPS 接入自动化体系：监控、告警、关机顺序、演练，一个都不能少。

推荐落地方案：

| 场景 | 推荐做法 |
|------|----------|
| 单台 PVE + UPS | NUT `standalone`，本机自动关机 |
| 多台服务器共用 UPS | ✅ NUT `netserver + netclient` |
| Docker 服务多 | 自定义 `SHUTDOWNCMD`，先停 compose 再关机 |
| 有监控栈 | 接入 `nut-exporter` + Grafana |
| 有 NAS / 数据库 | 必须作为 client 纳入关机链路 |

一句话：**UPS 负责争取时间，NUT 负责用好这段时间。** 真正可靠的 Homelab，不是永不断电，而是断电时也能按预期收尾。
