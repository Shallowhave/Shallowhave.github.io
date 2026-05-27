---
title: "Docker 网络完全指南：从 Bridge 到 Macvlan，打通 Homelab 容器网络任督二脉"
description: "手把手玩转 Docker 六种网络模式：Bridge、Host、None、Macvlan、IPvlan、Overlay，配合真实 Homelab 场景的配置案例、命令实测与踩坑记录。"
date: 2026-05-27T09:00:00+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
slug: "docker-network-deep-dive"
---

## 前言

在 Homelab 里跑 Docker 容器多了，总会在某个深夜遇到这样的问题：

- 容器里的服务能 curl 到外面，但从局域网其他机器就是访问不了  
- 两个容器想互相通信，`localhost` 不行，`--link` 又提示 deprecated
- 想让某个容器拿一个和宿主机同网段的 IP，方便用局域网其他服务直接访问
- 跨两台 PVE 虚拟机上的容器之间该怎么互相发现？

**这些问题的本质，都是没搞懂 Docker 网络模型。**

Docker 提供了六种网络驱动（Network Driver），每种对应不同的隔离级别和通信方式。很多教程只讲 `bridge` 和 `host`，但在真实的 Homelab 环境中——特别是当你要整合 PVE、NAS、多台宿主机时——Macvlan、IPvlan、Overlay 才是真正解决问题的工具。

本文从一个 Homelab 常见拓扑出发，逐个拆解 Docker 网络模式，每个模式都附带可复现的配置命令和踩坑记录。

## 0. 先看拓扑

假设你的 Homelab 网络如下：

| 设备 | IP | 角色 |
|------|-----|------|
| PVE 宿主机 | 192.168.1.10 | 运行 Docker |
| PVE 宿主机 2 | 192.168.1.11 | 运行 Docker |
| NAS | 192.168.1.20 | NFS/Samba |
| 你的电脑 | 192.168.1.100 | 日常使用 |

局域网网段: `192.168.1.0/24`，网关 `192.168.1.1`。

我们要解决的核心问题是：**容器如何优雅地与这个局域网交互？**

## 1. Bridge — 默认模式，够用但有限

### 工作原理

当你 `docker run` 什么参数都不加网络时，Docker 会自动把容器挂到 `docker0` (默认) 或自定义 bridge 上。每个容器得到一个 `172.x.x.x` 的内部 IP，通过 NAT 访问外部网络。

```
容器A (172.17.0.2) → docker0 (NAT) → eth0 (192.168.1.10) → 局域网
```

### 最常见的坑：端口映射

```bash
docker run -d --name nginx -p 8080:80 nginx
```

从局域网其他机器访问 `192.168.1.10:8080` → 没问题。  
但如果 Nginx 要访问局域网另一台机器的某个端口？容器里拿到的 IP 是 `172.17.0.2`，外面不认识。

**解决方案一：用 `--network=host`（见后文）**  
**解决方案二：用自定义 bridge，指定 subnet**

### 自定义 Bridge 才是正确姿势

```bash
# 创建自定义 bridge，而不是用默认的 docker0
docker network create --driver bridge \
  --subnet=172.20.0.0/24 \
  --gateway=172.20.0.1 \
  --ip-range=172.20.0.0/25 \
  --label=env=homelab \
  homelab-net

# 运行容器时指定
docker run -d --name nginx \
  --network homelab-net \
  -p 8080:80 \
  nginx:alpine
```

自定义 bridge 比默认 `docker0` 好的地方：

1. **容器间 DNS 自动解析** — 在自定义 bridge 上的容器可以 `ping nginx` 用名字互相发现
2. **可自定义 subnet** — 避免与真实局域网或 VPN 冲突
3. **可以随时加容器** — 但一旦建好就不能改 subnet（重建即可）

### iptables 与端口映射的隐患

`-p 8080:80` 底层是通过 iptables DNAT 实现的。如果你在 PVE 上有其他 iptables 规则（比如之前文章提过的 WireGuard 规则），就可能出现端口映射不生效的情况：

```bash
# 排查端口映射是否生效
iptables -t nat -L -n | grep 8080

# Docker 默认会写入 DOCKER 链
iptables -t nat -L DOCKER -n

# 如果发现规则被其他策略覆盖，检查 FORWARD 链默认策略
iptables -L FORWARD -n
```

> **踩坑记录**：某次我在 PVE 上启用了 `ufw` 并设置了 `DEFAULT_FORWARD_POLICY=DROP`，所有 `-p` 映射全部失效，容器能出不能进。解决方法：`sysctl net.ipv4.ip_forward=1` + 确保 FORWARD ACCEPT。

### 什么时候用 Bridge

- 绝大多数 Web 服务（Grafana、Traefik、Home Assistant）
- 不需要局域网直接访问，走反向代理统一入口
- 容器间通过名字发现即可

## 2. Host — 性能为王，隔离牺牲

```bash
docker run -d --name nginx \
  --network host \
  nginx:alpine
```

容器直接共享宿主机的网络栈，没有 NAT，没有虚拟网桥，性能最好。

```bash
# Host 模式下，容器里的 80 端口就是宿主机的 80 端口
# 不需要 -p 映射
ss -tlnp | grep :80
# 你会看到 nginx 的进程直接监听在 0.0.0.0:80
```

### 优点

- 零网络开销，延迟最低
- 容器可以访问宿主机所有网络接口
- 端口不冲突时部署最简单

### 坑点

- **端口冲突**：一个 Host 模式容器占用了 80 端口，第二个就没法用了
- **丧失了网络隔离**：容器可以监听任意端口，权限等同于宿主机进程
- **Docker DNS 失效**：如果依赖 `docker-compose` 的 service name 解析，Host 模式下不生效

### 什么时候用 Host

- 网络性能敏感的服务（如 Jellyfin 硬件转码 + 高码率流）
- 需要直接操作宿主机网络接口的工具（如 WireGuard、netdata 的某些插件）
- 单容器单服务的极简部署

## 3. None — 彻底隔离

```bash
docker run -d --name isolated \
  --network none \
  alpine sleep 3600
```

容器里只有 loopback 接口，没有任何网络能力。用于：

- 只处理本地文件，不需要网络的批处理任务
- 安全敏感的离线计算（如 GPG 签名）
- 自定义网络的起点：先 `--network none`，再手动把容器进程挪进自定义 netns

## 4. Macvlan — 让容器拥有「真实」IP

这是 Homelab 里最实用也最容易被误解的模式。

### 为什么需要 Macvlan

Bridge 模式下，容器走 NAT，局域网其他设备无法直接访问容器——除非你为每个服务配置 `-p` 映射并记住一堆端口号。

Macvlan 给每个容器分配一个 **宿主机同网段的真实 IP**，以及一个 **虚拟 MAC 地址**。对局域网来说，容器就像一台独立的物理机。

```bash
# 创建 macvlan 网络（连接到宿主机 eth0）
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  --ip-range=192.168.1.200/28 \
  -o parent=eth0 \
  macvlan-net

# 启动容器，自动分配 192.168.1.200-192.168.1.215 之间的 IP
docker run -d --name nginx-macvlan \
  --network macvlan-net \
  --ip=192.168.1.201 \
  nginx:alpine

# 现在从你的电脑 192.168.1.100 直接访问 192.168.1.201:80
# 不需要任何端口映射！
```

### Macvlan 模式 VS Bridge 模式

| 特性 | Bridge + -p | Macvlan |
|------|-------------|---------|
| 容器 IP | 172.x.x.x 内网 | 192.168.1.x 同网段 |
| 端口映射 | 需要 `-p` | 不需要 |
| 局域网直接访问 | 需端口转发 | ✅ 就像独立设备 |
| 性能 | 有 NAT 开销 | 接近原生 |
| 容器内 ping 网关 | ✅ | ✅ |
| 宿主机访问容器 | ❌ 默认不能 | ❌ 默认不能 |

### 最大的坑：宿主机无法访问 Macvlan 容器

这一点踩的人最多。

创建 Macvlan 网络后，**宿主机无法直接通过 Macvlan 容器的 IP 访问它**。原因：宿主机网卡 `eth0` 被配置了 Macvlan 子接口，Linux 网络栈会把发往容器 MAC 地址的包直接交给子接口，绕过了主接口的 IP 栈。

```bash
# 从宿主机访问 macvlan 容器会失败
curl http://192.168.1.201:80  # ❌ 超时

# 但从另一台机器（包括其他 PVE 虚拟机）可以访问
```

**解决方案：创建一个 macvlan bridge 子接口**

```bash
# 在宿主机上创建一个 macvlan slave 接口
ip link add macvlan-host link eth0 type macvlan mode bridge
ip addr add 192.168.1.199/32 dev macvlan-host
ip link set macvlan-host up
ip route add 192.168.1.200/28 dev macvlan-host

# 现在宿主机可以访问 macvlan 容器了
curl http://192.168.1.201:80  # ✅ 成功
```

**更简单的方案**：把这条命令写到 `systemd` 服务自动执行，或者直接放进 PVE 宿主机开机脚本。

### 什么时候用 Macvlan

- DHCP 需要分配真实 IP（如 DNS 服务器、DHCP 服务器）
- 局域网其他设备（如 NAS、电视、手机 App）需要直接访问容器 IP
- 服务本身不支持端口映射或需要固定端口映射但不想管理端口表

## 5. IPvlan — 更轻量，没有 MAC 地址风暴

IPvlan 是 Macvlan 的进化版：所有容器共享宿主机同一个 MAC 地址，但各自拥有独立的 IP 地址。

```bash
# 创建 IPvlan（L2 模式）
docker network create -d ipvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  --ip-range=192.168.1.220/28 \
  -o parent=eth0 \
  -o ipvlan_mode=l2 \
  ipvlan-net

docker run -d --name nginx-ipvlan \
  --network ipvlan-net \
  --ip=192.168.1.221 \
  nginx:alpine
```

### 为什么需要 IPvlan

高密度容器场景（几十上百个容器），Macvlan 会消耗大量 MAC 地址，可能导致交换机 MAC 表溢出。IPvlan 只用一个物理 MAC，从根源上解决这个问题。

### IPvlan L2 vs L3 模式

- **L2 模式**：容器与宿主机在同一广播域，ARP 解析由本地交换机处理
- **L3 模式**：Docker 做三层路由，容器之间跨子网通信经过宿主机路由表

```bash
# L3 模式示例：不同子网
docker network create -d ipvlan \
  --subnet=10.0.1.0/24 \
  --subnet=10.0.2.0/24 \
  -o parent=eth0 \
  -o ipvlan_mode=l3 \
  ipvlan-l3-net

# L3 模式下，容器之间的跨子网通信走宿主机路由
```

### Macvlan vs IPvlan 选择指南

| 场景 | 推荐 |
|------|------|
| < 10 个容器，需要独立 MAC | Macvlan |
| 10+ 容器，只想拿独立 IP | IPvlan L2 |
| 跨子网路由，没有 VLAN 隔离需求 | IPvlan L3 |
| 宿主机需要访问容器 | Macvlan + 子接口或 IPvlan 都行，都有类似限制 |

## 6. Overlay — 跨宿主机容器通信

当你的 Homelab 扩展到两台以上 PVE 宿主机，每台都运行 Docker，容器之间需要互通时，Overlay 就是答案。

```bash
# 第一步：需要初始化 Swarm 模式（即便只用一个 manager 节点）
docker swarm init --advertise-addr=192.168.1.10

# 第二步：创建 overlay 网络
docker network create -d overlay \
  --subnet=10.0.100.0/24 \
  --attachable \
  homelab-overlay

# 在其他节点加入 swarm
docker swarm join --token <manager-token> 192.168.1.10:2377

# 第三步：启动服务 —— 注意 overlay 只支持 swarm service 或 attachable 模式
docker service create \
  --name whoami \
  --network homelab-overlay \
  --replicas 2 \
  traefik/whoami

# 或者用 compose + attachable overlay
docker run -d --name alpine-overlay \
  --network homelab-overlay \
  alpine sleep 3600
```

### Overlay 网络关键踩坑

**1. 防火墙必须放行 VXLAN 端口**

Overlay 底层用 VXLAN 封装（UDP 4789），跨宿主机通信必须放通：

```bash
# PVE 宿主机上
iptables -A INPUT -p udp --dport 4789 -j ACCEPT
iptables -A FORWARD -p udp --dport 4789 -j ACCEPT
```

**2. 需要全连接的网络拓扑**

Overlay 依赖底层网络全连接——每对 Docker 宿主机之间都能通信。如果两台 PVE 宿主机之间做了 VLAN 隔离或防火墙限制，VXLAN 包传不过去。

**3. 性能损耗**

VXLAN 封装/解封装有约 5-10% 的性能损耗。IO 敏感的服务（如数据库）不要走 Overlay，用 Macvlan 或直接 Host 模式更好。

### 什么时候用 Overlay

- 多台 Docker 宿主机上的容器需要服务发现
- 不想暴露端口到宿主机网络
- 使用 Docker Swarm 或 Compose + attachable overlay

## 7. 实战：Homelab Docker 网络统一方案

结合以上所有模式，这里给出一个我在用的实际配置方案：

```yaml
# docker-compose.yml
version: "3.8"

networks:
  # 1. 公共服务网 — Bridge，用于 Web 服务通过反向代理访问
  webnet:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24

  # 2. 内部后端网 — Bridge，数据库和中间件之间互通
  backend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.1.0/24

  # 3. 局域网直连网 — Macvlan，让某些容器获得独立局域网 IP
  macvlan-net:
    driver: macvlan
    driver_opts:
      parent: eth0
    ipam:
      config:
        - subnet: 192.168.1.0/24
          ip_range: 192.168.1.230/28
          gateway: 192.168.1.1

services:
  traefik:
    image: traefik:v3.0
    networks:
      - webnet
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

  postgres:
    image: postgres:16-alpine
    networks:
      - backend
    # 数据库只暴露给 backend 网，不向外

  whoami:
    image: traefik/whoami
    networks:
      - macvlan-net
      - webnet
    # 这个服务同时在两个网里：
    # - 通过 macvlan 获得 192.168.1.x IP 供局域网直接访问
    # - 通过 webnet 被 Traefik 反向代理
```

## 8. 网络排障速查表

遇到 Docker 网络问题时，按下面的顺序排查：

```bash
# 1. 容器自己在不在？
docker exec -it <container> ip addr
docker exec -it <container> ip route
docker exec -it <container> ping 8.8.8.8
docker exec -it <container> ping 192.168.1.1

# 2. 网关能到容器吗？
ping <容器IP>
# 如果宿主机能 ping 但局域网不能 → 检查 iptables FORWARD

# 3. DNS 解析正常吗？
docker exec -it <container> cat /etc/resolv.conf
docker exec -it <container> nslookup google.com

# 4. 端口映射生效吗？
ss -tlnp | grep <映射端口>
iptables -t nat -L DOCKER -n | grep <映射端口>

# 5. 容器间能用名字访问吗？
# 自定义 bridge ✅
# 默认 docker0 ❌
# Host 模式 ❌
docker exec -it containerA ping containerB

# 6. 跨宿主机通信？
# Overlay：检查 VXLAN (udp 4789) 是否放通
nc -uvz 192.168.1.11 4789

# Macvlan：检查宿主机上是否有子接口
ip link show type macvlan
```

## 9. 总结

| 模式 | 隔离度 | 性能 | 局域网直接访问 | 跨宿主机 | 适用场景 |
|------|--------|------|---------------|---------|---------|
| Bridge | 中 | 中 | 需 -p 映射 | ❌ | 大多数 Web 服务 |
| Host | 低 | 最高 | 直接占用宿主机端口 | ❌ | 高性能/硬件直通 |
| None | 最高 | N/A | ❌ | ❌ | 离线任务 |
| Macvlan | 低 | 高 | ✅ 独立 IP | ❌ | 需要真实 IP 的场景 |
| IPvlan | 低 | 最高 | ✅ 无 MAC 风暴 | ❌ | 高密度容器 |
| Overlay | 中 | 中(5-10%损耗) | 需转发 | ✅ | 多宿主机集群 |

刚开始玩 Docker 时，Bridge 模式加几条 `-p` 映射就能跑起来，这没错。但一旦你的 Homelab 开始扩张——加第二台宿主机、接 NAS、跑跨容器服务发现——你就需要理解这些网络模式的真实差异。

**最推荐的 Homelab 入门方案**：主服务用 Bridge + Traefik 反向代理统一入口，给 DNS、DHCP、媒体服务器等需要真实 IP 的服务用 Macvlan 单独分配 IP，等上了两台宿主机再考虑 Overlay。

这些命令和配置我都一句一句在 PVE 宿主机上跑过验证过。如果你在配置中踩了坑，或者有更好的方案想交流，欢迎在评论区讨论。
