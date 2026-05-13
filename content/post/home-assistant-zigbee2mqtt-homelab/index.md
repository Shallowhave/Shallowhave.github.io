---
title: "把智能家居装进 Homelab：Docker 部署 Home Assistant、MQTT 与 Zigbee2MQTT 实战"
description: "在 Homelab 中用 Docker Compose 部署 Home Assistant、Mosquitto 和 Zigbee2MQTT，打通 Zigbee 设备、本地自动化、备份与安全访问，包含串口映射、MQTT 权限、踩坑排查和运维命令。"
date: 2026-05-13T09:01:19+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
slug: "home-assistant-zigbee2mqtt-homelab"
---

## 前言

Homelab 玩到一定阶段，除了虚拟化、存储、网络和监控，智能家居通常也会被纳入统一运维体系。原因很简单：Home Assistant 本质上就是一个家庭自动化控制平面，和我们平时管理的服务一样，需要稳定运行、可观测、可备份、可恢复，还要尽量避免云端依赖。

很多人第一次部署 Home Assistant 会直接用官方 HAOS 镜像，但在已经有 Proxmox VE、Docker 主机或者 NAS 的 Homelab 里，我更推荐先从 **Docker Compose 方案** 开始：依赖关系清晰、迁移简单、配置文件可版本化，后续接入 Zigbee2MQTT、Mosquitto、Node-RED、InfluxDB/Grafana 也更自然。

本文目标是搭一套本地优先的智能家居基础栈：

- Home Assistant：自动化核心与 Web UI
- Mosquitto：本地 MQTT Broker
- Zigbee2MQTT：把 Zigbee 设备转换成 MQTT 实体
- Docker Compose：声明式部署与日常运维
- 备份与排错：避免升级、换主机、USB 设备漂移时翻车

> 如果你已经在 PVE 里跑了 Docker LXC 或 VM，这套方案可以直接落地。若还没规划容器承载环境，可以先参考站内的 [Proxmox VE 下 LXC 容器完整实战指南](/p/pve-lxc-container-guide/) 和 [PVE VLAN、Bond 与防火墙分区实战](/p/pve-vlan-bond-firewall/)。

---

## 1. 架构设计：为什么拆成三个容器？

很多初学者会把所有插件都塞进 Home Assistant Add-on，但 Add-on 机制主要服务 HAOS/Supervised 环境。Docker Compose 模式下，更运维友好的做法是把核心组件拆开：

| 组件 | 作用 | 是否有状态 | 推荐暴露端口 |
|------|------|------------|--------------|
| Home Assistant | 自动化编排、Dashboard、设备实体管理 | ✅ `/config` | `8123/tcp` |
| Mosquitto | MQTT 消息中转、认证、ACL | ✅ `/mosquitto` | `1883/tcp`，可选 `9001/ws` |
| Zigbee2MQTT | Zigbee 协调器管理、设备入网、MQTT 转换 | ✅ `/app/data` | `8080/tcp` Web UI |

拆分的好处：

1. **故障域隔离**：Zigbee2MQTT 升级失败不会直接破坏 Home Assistant。
2. **备份边界清晰**：三个目录就是全部状态，迁移时复制目录即可。
3. **权限更可控**：MQTT 账户、ACL、串口设备映射都能单独管理。
4. **更符合 Homelab 运维习惯**：可以被 Watchtower、Portainer、Prometheus、备份脚本统一纳管。

推荐网络拓扑如下：

```text
Zigbee 设备
   │
   ▼
Zigbee USB 协调器 ── /dev/serial/by-id/... ── Zigbee2MQTT
                                           │
                                           ▼
                                      Mosquitto MQTT
                                           │
                                           ▼
                                    Home Assistant
                                           │
                                           ▼
                                  自动化 / Dashboard
```

核心原则：**Zigbee2MQTT 不直接依赖 Home Assistant，二者只通过 MQTT 解耦**。

---

## 2. 环境准备

示例环境：

| 项目 | 示例值 |
|------|--------|
| 宿主机 | Debian 12 / Ubuntu 22.04 / PVE VM |
| Docker | 24.x 或更新 |
| Compose | Docker Compose v2 |
| Zigbee 协调器 | Sonoff ZBDongle-P/E、CC2652P、EFR32MG21 等 |
| 数据目录 | `/opt/smarthome` |

安装 Docker 的过程不展开，先确认运行环境：

```bash
docker version
docker compose version
uname -a
```

创建目录：

```bash
sudo mkdir -p /opt/smarthome/{homeassistant,mosquitto/{config,data,log},zigbee2mqtt}
sudo chown -R $USER:$USER /opt/smarthome
cd /opt/smarthome
```

插入 Zigbee USB 协调器后，不要直接使用 `/dev/ttyUSB0`，因为重启或插拔后编号可能变化。正确方式是查找稳定路径：

```bash
ls -l /dev/serial/by-id/
```

输出类似：

```text
usb-ITead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_123456-if00-port0 -> ../../ttyUSB0
```

后续 Compose 中应使用 `/dev/serial/by-id/usb-...`，这是避免串口漂移的关键。

如果你跑在 Proxmox VE 虚拟机里，需要把 USB 设备透传给 VM：

```bash
# 在 PVE 宿主机查看 USB 设备
lsusb

# 假设 VMID 为 110，把指定 USB 设备透传给虚拟机
qm set 110 -usb0 host=10c4:ea60

# 或者透传整个 USB 端口，适合固定插口
qm set 110 -usb0 host=1-2
```

LXC 容器也能映射 USB，但权限处理更麻烦。稳定性优先时，我更建议用一个轻量 VM 跑智能家居栈。

---

## 3. Mosquitto：先把 MQTT 权限做好

先写 Mosquitto 配置：

```bash
cat > /opt/smarthome/mosquitto/config/mosquitto.conf <<'EOF'
persistence true
persistence_location /mosquitto/data/
log_dest file /mosquitto/log/mosquitto.log
log_dest stdout

listener 1883
allow_anonymous false
password_file /mosquitto/config/passwd

# 如果需要 WebSocket，可打开以下配置
# listener 9001
# protocol websockets
EOF
```

创建 MQTT 用户。建议至少拆成两个账号：`homeassistant` 和 `zigbee2mqtt`，后续做 ACL 时更细：

```bash
docker run --rm -it \
  -v /opt/smarthome/mosquitto/config:/mosquitto/config \
  eclipse-mosquitto:2 \
  mosquitto_passwd -c /mosquitto/config/passwd homeassistant

docker run --rm -it \
  -v /opt/smarthome/mosquitto/config:/mosquitto/config \
  eclipse-mosquitto:2 \
  mosquitto_passwd /mosquitto/config/passwd zigbee2mqtt
```

如果是自动化部署，不想交互输入，可以用 `-b`，但注意命令历史会留下明文密码：

```bash
mosquitto_passwd -b /mosquitto/config/passwd homeassistant '请替换成强密码'
```

生产一点的做法是增加 ACL：

```bash
cat > /opt/smarthome/mosquitto/config/acl <<'EOF'
user homeassistant
topic readwrite #

user zigbee2mqtt
topic readwrite zigbee2mqtt/#
topic readwrite homeassistant/#
EOF
```

然后在 `mosquitto.conf` 中追加：

```ini
acl_file /mosquitto/config/acl
```

HomeLab 内部网络虽然可信度更高，但 MQTT 一旦被匿名访问，任何人都可以伪造传感器状态、触发自动化，甚至控制插座和门锁。所以第一步必须关闭匿名访问。

---

## 4. Zigbee2MQTT 配置

创建 `/opt/smarthome/zigbee2mqtt/configuration.yaml`：

```yaml
homeassistant: true
permit_join: false

mqtt:
  base_topic: zigbee2mqtt
  server: mqtt://mosquitto:1883
  user: zigbee2mqtt
  password: "请替换成 zigbee2mqtt 的 MQTT 密码"

serial:
  port: /dev/ttyUSB-zigbee
  adapter: zstack

frontend:
  enabled: true
  port: 8080

advanced:
  channel: 15
  network_key: GENERATE
  pan_id: GENERATE
  ext_pan_id: GENERATE
  log_level: info

devices: {}
groups: {}
```

几个关键点：

- `homeassistant: true`：自动发布 MQTT Discovery，HA 会自动出现实体。
- `permit_join: false`：默认禁止入网，避免邻居设备误加入。
- `network_key: GENERATE`：首次启动生成随机密钥；生成后不要随便改，否则所有设备要重新配对。
- `channel: 15`：避开常见 Wi-Fi 2.4G 干扰，具体要结合 AP 信道规划。

如果你使用 Sonoff ZBDongle-E/EFR32，`adapter` 可能要改成：

```yaml
serial:
  adapter: ember
```

如果是 ZBDongle-P/CC2652P，通常使用：

```yaml
serial:
  adapter: zstack
```

---

## 5. Docker Compose 一次拉起

创建 `/opt/smarthome/docker-compose.yml`：

```yaml
services:
  mosquitto:
    image: eclipse-mosquitto:2
    container_name: mosquitto
    restart: unless-stopped
    ports:
      - "1883:1883"
    volumes:
      - ./mosquitto/config:/mosquitto/config
      - ./mosquitto/data:/mosquitto/data
      - ./mosquitto/log:/mosquitto/log
    networks:
      - smarthome

  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    container_name: homeassistant
    restart: unless-stopped
    privileged: true
    environment:
      - TZ=Asia/Shanghai
    volumes:
      - ./homeassistant:/config
      - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro
    ports:
      - "8123:8123"
    depends_on:
      - mosquitto
    networks:
      - smarthome

  zigbee2mqtt:
    image: koenkk/zigbee2mqtt:latest
    container_name: zigbee2mqtt
    restart: unless-stopped
    environment:
      - TZ=Asia/Shanghai
    volumes:
      - ./zigbee2mqtt:/app/data
      - /run/udev:/run/udev:ro
    devices:
      - /dev/serial/by-id/usb-ITead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_123456-if00-port0:/dev/ttyUSB-zigbee
    ports:
      - "8080:8080"
    depends_on:
      - mosquitto
    networks:
      - smarthome

networks:
  smarthome:
    driver: bridge
```

把 `devices` 左侧路径替换成你自己的 `/dev/serial/by-id/...`。然后启动：

```bash
cd /opt/smarthome
docker compose pull
docker compose up -d
docker compose ps
```

看日志：

```bash
docker logs -f mosquitto
docker logs -f zigbee2mqtt
docker logs -f homeassistant
```

访问入口：

- Home Assistant：`http://你的服务器IP:8123`
- Zigbee2MQTT：`http://你的服务器IP:8080`

进入 Home Assistant 后，添加 MQTT 集成：

```text
设置 → 设备与服务 → 添加集成 → MQTT
Broker: mosquitto
Port: 1883
Username: homeassistant
Password: 对应密码
```

如果 Home Assistant 和 Mosquitto 在同一个 Compose 网络里，Broker 可以填 `mosquitto`；如果从外部访问，填服务器 IP。

---

## 6. 设备入网与自动化示例

临时允许 Zigbee 设备入网：

```bash
# 修改 configuration.yaml
permit_join: true

# 重启 Zigbee2MQTT
docker compose restart zigbee2mqtt
```

更推荐使用 Zigbee2MQTT 前端 Web UI 临时开启，配对完马上关闭。入网成功后，设备会出现在：

```text
zigbee2mqtt/<device_name>
homeassistant/sensor/<device_name>/...
```

可以用 MQTT 客户端验证消息：

```bash
docker exec -it mosquitto mosquitto_sub \
  -h localhost \
  -u homeassistant \
  -P '你的密码' \
  -t 'zigbee2mqtt/#' -v
```

一个最小自动化示例：人体传感器检测到移动后打开灯，2 分钟无人后关闭。

```yaml
alias: 厨房有人自动开灯
mode: restart
trigger:
  - platform: state
    entity_id: binary_sensor.kitchen_motion_occupancy
    to: "on"
action:
  - service: light.turn_on
    target:
      entity_id: light.kitchen_light
  - wait_for_trigger:
      - platform: state
        entity_id: binary_sensor.kitchen_motion_occupancy
        to: "off"
        for: "00:02:00"
  - service: light.turn_off
    target:
      entity_id: light.kitchen_light
```

如果想减少误触发，可以加光照条件：

```yaml
condition:
  - condition: numeric_state
    entity_id: sensor.kitchen_illuminance
    below: 30
```

这就是本地自动化的价值：即使外网断了，传感器、MQTT、HA 都在内网，灯光控制仍然可用。

---

## 7. 运维：备份、升级与监控

### 7.1 备份目录

这套方案最重要的状态都在 `/opt/smarthome`。可以直接用 `rsync`、`restic`、`borg` 或 Proxmox Backup Server 备份。

简单冷备脚本：

```bash
#!/usr/bin/env bash
set -euo pipefail

SRC="/opt/smarthome"
DST="/backup/smarthome"
DATE="$(date +%F-%H%M)"

mkdir -p "$DST"
cd /opt

docker compose -f /opt/smarthome/docker-compose.yml stop

tar --xattrs --acls -czf "$DST/smarthome-$DATE.tar.gz" smarthome
sha256sum "$DST/smarthome-$DATE.tar.gz" > "$DST/smarthome-$DATE.tar.gz.sha256"

docker compose -f /opt/smarthome/docker-compose.yml up -d

find "$DST" -name 'smarthome-*.tar.gz' -mtime +30 -delete
```

如果你已经部署 PBS，可以把运行该栈的 VM 整机纳入 PBS，同时在应用层额外保留 `/opt/smarthome` 的压缩包。这样既能快速整机恢复，也能单独取回某个 YAML 配置。

### 7.2 升级策略

不要无脑 `latest` 自动更新。Zigbee2MQTT 与 Home Assistant 都可能存在破坏性变更。更稳的升级流程：

```bash
cd /opt/smarthome

# 1. 先备份
tar -czf ~/smarthome-before-upgrade-$(date +%F).tar.gz .

# 2. 拉取镜像
docker compose pull

# 3. 重启
docker compose up -d

# 4. 观察日志
docker compose logs -f --tail=100 homeassistant zigbee2mqtt mosquitto
```

如果升级后异常，回滚：

```bash
docker compose down
rm -rf /opt/smarthome
mkdir -p /opt/smarthome
tar -xzf ~/smarthome-before-upgrade-YYYY-MM-DD.tar.gz -C /opt/smarthome --strip-components=1
docker compose up -d
```

更严谨的做法是固定镜像版本，例如：

```yaml
image: ghcr.io/home-assistant/home-assistant:2026.5
image: koenkk/zigbee2mqtt:2.3.0
```

### 7.3 健康检查命令

常用排查命令：

```bash
# 容器状态
docker compose ps

# MQTT 是否可登录
docker exec -it mosquitto mosquitto_sub -h localhost -u homeassistant -P '密码' -t '$SYS/#' -C 5 -v

# Zigbee 串口是否存在
ls -l /dev/serial/by-id/

# Zigbee2MQTT 是否占用串口
lsof /dev/ttyUSB0 || true

# Home Assistant 配置检查
docker exec -it homeassistant python -m homeassistant --script check_config --config /config
```

如果已有 Prometheus/Grafana，可以把容器指标接入 cAdvisor，把 Home Assistant 的 `/api/prometheus` 暴露出来，统一看自动化系统状态。监控栈可参考 [Prometheus + Grafana 全栈监控告警体系搭建指南](/p/homelab-monitoring-stack-guide/)。

---

## 8. 踩坑记录

### ❌ 坑 1：使用 `/dev/ttyUSB0`，重启后 Zigbee2MQTT 找不到协调器

现象：

```text
Error: Error while opening serialport 'Error: No such file or directory, cannot open /dev/ttyUSB0'
```

根因：USB 设备编号不稳定。解决：使用 `/dev/serial/by-id/...` 并映射成容器内固定路径：

```yaml
devices:
  - /dev/serial/by-id/usb-xxx:/dev/ttyUSB-zigbee
```

### ❌ 坑 2：Mosquitto 开了匿名访问

现象：任何内网主机都可以发布 MQTT 消息，Home Assistant 出现奇怪实体或自动化被误触发。

解决：

```ini
allow_anonymous false
password_file /mosquitto/config/passwd
acl_file /mosquitto/config/acl
```

然后重启：

```bash
docker compose restart mosquitto
```

### ❌ 坑 3：Zigbee 信道和 Wi-Fi 2.4G 互相干扰

Zigbee 和 2.4G Wi-Fi 同频段。常见建议：

| Wi-Fi 信道 | Zigbee 推荐信道 |
|------------|-----------------|
| Wi-Fi 1 | Zigbee 15 / 20 |
| Wi-Fi 6 | Zigbee 20 / 25 |
| Wi-Fi 11 | Zigbee 15 / 20 |

注意：Zigbee 网络建好后改信道，很多设备需要重新入网，所以一开始就要规划好。

### ❌ 坑 4：PVE 里 USB 省电导致协调器掉线

如果日志里频繁出现串口断开，可以在宿主机检查：

```bash
dmesg -T | grep -iE 'usb|ttyUSB|cp210x|ch341'
```

可以尝试关闭 USB autosuspend：

```bash
cat /sys/module/usbcore/parameters/autosuspend

# 临时关闭
echo -1 | sudo tee /sys/module/usbcore/parameters/autosuspend
```

永久配置可在 GRUB 里加入：

```ini
GRUB_CMDLINE_LINUX_DEFAULT="quiet usbcore.autosuspend=-1"
```

然后：

```bash
update-grub
reboot
```

### ❌ 坑 5：把 `network_key` 重新生成导致全屋设备掉线

`network_key` 是 Zigbee 网络的身份根密钥。首次 `GENERATE` 后，Zigbee2MQTT 会写入具体值。迁移或恢复时必须保留该文件，不要重新生成。否则所有设备都认为这是一个新网络，需要重新配对。

---

## 9. 最佳实践总结

| 场景 | 推荐做法 | 理由 |
|------|----------|------|
| 部署方式 | Docker Compose | 可迁移、可版本化、便于和 Homelab 其他服务统一管理 |
| MQTT 安全 | 禁止匿名 + 独立账号 + ACL | 防止伪造设备状态和误触发自动化 |
| 串口映射 | `/dev/serial/by-id` | 避免重启后 `/dev/ttyUSB0` 漂移 |
| Zigbee 入网 | 默认关闭 `permit_join` | 减少误入网和安全风险 |
| 升级策略 | 先备份，后升级，观察日志 | HA/Z2M 更新频繁，避免破坏性变更翻车 |
| 备份策略 | 应用目录 + VM 整机双备份 | 同时满足快速恢复和精细恢复 |

## 总结

Home Assistant 不是一个“装完就算”的玩具服务，而是 Homelab 里一个非常典型的本地控制平面。真正稳定的智能家居栈，关键不在于接了多少设备，而在于：MQTT 权限是否收紧、Zigbee 协调器路径是否稳定、配置是否可备份、升级是否可回滚。

如果只记住一句话：**把 Home Assistant 当作生产服务来运维，用 Docker Compose 固化组件边界，用 MQTT 解耦设备通信，用备份和日志兜底升级风险。**
