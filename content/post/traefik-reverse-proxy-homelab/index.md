---
title: "Homelab 必备：Traefik 反向代理 + SSL 证书自动化部署指南"
description: "在 Homelab 中使用 Docker Compose 部署 Traefik 反向代理，自动申请 Let's Encrypt SSL 证书，统一管理内外网服务入口，告别手动配置 Nginx 和续期证书的烦恼。"
date: 2024-07-15T17:12:00+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
slug: "traefik-reverse-proxy-homelab"
---

## 前言

Homelab 玩到一定程度，你就会面临一个经典问题：服务越来越多。

PVE 管理后台、Docker 上的 Portainer、Home Assistant、Jellyfin 媒体服务器、Grafana 监控面板……每个服务都有一个不同的端口号，每次访问都要记 `192.168.1.100:9000`、`:8123`、`:3000`，不仅不方便，还容易搞混。

解决方案就是**反向代理** —— 通过域名访问所有服务：

- `ha.yourlab.com` → Home Assistant
- `portainer.yourlab.com` → Portainer
- `grafana.yourlab.com` → Grafana
- `jellyfin.yourlab.com` → Jellyfin

而 Traefik 是目前最适合 Docker 环境下的反向代理方案之一。相比 Nginx 需要手动维护配置文件，Traefik 可以**自动**监听 Docker 容器，通过容器 Label 自动配置路由，还能自动申请和续期 Let's Encrypt SSL 证书。真正做到"加一个新服务，改几行 docker-compose 配置，Traefik 全自动搞定"。

本文带你从零搭建一套 Traefik 反向代理，完整覆盖 HTTPS、Let's Encrypt 自动证书、Dashboard 监控和常见服务接入。

---

## 1. 架构概览

```
        Internet
           │
        🔒 443 (HTTPS)
           │
    ┌──────▼──────┐
    │   Traefik   │  ← 自动申请 Let's Encrypt 证书
    │  (Proxy)    │
    └──┬──┬──┬───┘
       │  │  │
  ┌────▼┐ │ └──────┐
  │ HA  │ │        │
  └─────┘ │   ┌────▼──┐
          │   │Jelly..│
    ┌─────▼──┐└───────┘
    │Portn.. │
    └────────┘
```

Traefik 监听宿主机的 80 和 443 端口，通过 Docker Socket 自动发现容器，根据容器 Label 自动配置路由。Let's Encrypt 使用 HTTP-01 或 TLS-ALPN-01 验证域名所有权，自动签发和续期证书。

---

## 2. 准备工作

### 2.1 网络与域名

你需要：

1. **一个公网域名**（或内网 DNS）—— 推荐在 Cloudflare、阿里云、腾讯云等平台注册一个域名
2. **域名解析到你的 Homelab 公网 IP**（如果服务需要外网访问）
3. **开放防火墙的 80 和 443 端口**

如果你只在内网使用，也可以用内网 DNS 或修改 `/etc/hosts`，但 Traefik 的 Let's Encrypt 功能仍需要域名能公网解析。对于纯内网场景，可以使用 Traefik 的自签证书或跳过 TLS，后续我会另写文章介绍。

### 2.2 创建 Docker 网络

为了让 Traefik 能和所有后端容器通信，先创建一个共享网络：

```bash
docker network create traefik-net
```

> 所有需要经过 Traefik 反向代理的容器都要接入这个网络。

---

## 3. 部署 Traefik

### 3.1 目录结构

```bash
mkdir -p ~/docker/traefik && cd ~/docker/traefik
touch acme.json     # Let's Encrypt 证书存储文件
chmod 600 acme.json # 重要！权限必须是 600，否则 Traefik 拒绝启动
```

### 3.2 Docker Compose 配置

创建 `docker-compose.yml`：

```yaml
version: '3.8'

services:
  traefik:
    image: traefik:v3.1
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    networks:
      - traefik-net
    ports:
      - "80:80"     # HTTP 重定向到 HTTPS
      - "443:443"   # HTTPS
      # - "8080:8080"  # Dashboard（建议通过反向代理访问，不暴露端口）
    environment:
      - CF_API_EMAIL=${CF_API_EMAIL}
      - CF_DNS_API_TOKEN=${CF_DNS_API_TOKEN}
      # 使用其他 DNS 提供商时修改为对应的环境变量
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik.yml:/traefik.yml:ro
      - ./acme.json:/acme.json
      - ./config:/config
    labels:
      - "traefik.enable=true"
      # Dashboard 路由
      - "traefik.http.routers.dashboard.rule=Host(`traefik.yourlab.com`)"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.middlewares=auth"
      # 中间件：Dashboard 基础认证
      - "traefik.http.middlewares.auth.basicauth.users=${DASHBOARD_USER}:${DASHBOARD_PASS_HASH}"

networks:
  traefik-net:
    external: true
```

### 3.3 Traefik 静态配置

创建 `traefik.yml`（注意不是 traefik.yaml，是 .yml）：

```yaml
# 全局配置
global:
  sendAnonymousUsage: false

# 日志级别: DEBUG, INFO, WARN, ERROR
log:
  level: INFO

# API 和 Dashboard
api:
  dashboard: true
  debug: true

# 入口点
entryPoints:
  web:
    address: :80
    http:
      redirections:
        entrypoint:
          to: websecure
          scheme: https
          permanent: true
  websecure:
    address: :443

# 证书配置（Let's Encrypt）
certificatesResolvers:
  letsencrypt:
    acme:
      email: your-email@example.com     # ← 改成你的邮箱
      storage: /acme.json
      # DNS-01 挑战（推荐，支持通配符证书）
      dnsChallenge:
        provider: cloudflare              # ← 根据你的 DNS 提供商修改
        resolvers:
          - "1.1.1.1:53"
          - "8.8.8.8:53"
      # 如果使用 HTTP-01 挑战，注释掉上面的 dnsChallenge，取消注释下面：
      # httpChallenge:
      #   entryPoint: web

# 开启 Docker 服务发现
providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false   # 只有打了 traefik.enable=true 标签的容器才被代理
  file:
    directory: /config        # 外部配置文件目录
    watch: true
```

### 3.4 环境变量

创建 `.env` 文件：

```bash
# DNS API 凭据（以 Cloudflare 为例）
CF_API_EMAIL=your-cloudflare-email@example.com
CF_DNS_API_TOKEN=your_cloudflare_api_token

# Dashboard 认证（用户名:密码的 bcrypt 哈希）
DASHBOARD_USER=admin
DASHBOARD_PASS_HASH=$(htpasswd -nb admin yourpassword | sed 's/\$/$$/g')
```

> 生成 bcrypt 密码哈希：`docker run --rm httpd:alpine htpasswd -nb admin 你的密码`

### 3.5 启动 Traefik

```bash
docker compose up -d
docker compose logs -f  # 查看启动日志
```

如果一切正常，你应该能看到 Let's Encrypt 证书成功申请的信息。访问 `https://traefik.yourlab.com` 就能看到 Traefik Dashboard 了。

---

## 4. 添加反向代理服务

### 4.1 例 1：反向代理 Portainer

Portainer 是管理 Docker 的利器，给它加上 Traefik 路由：

```yaml
version: '3.8'

services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    networks:
      - traefik-net    # 使用 Traefik 网络
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.portainer.rule=Host(`portainer.yourlab.com`)"
      - "traefik.http.routers.portainer.entrypoints=websecure"
      - "traefik.http.routers.portainer.tls=true"
      - "traefik.http.routers.portainer.tls.certresolver=letsencrypt"
      - "traefik.http.services.portainer.loadbalancer.server.port=9000"

volumes:
  portainer_data:

networks:
  traefik-net:
    external: true
```

`docker compose up -d` 之后，访问 `https://portainer.yourlab.com` 即可。Traefik 会自动为这个域名申请证书。

### 4.2 例 2：反向代理 Home Assistant

Home Assistant 运行在宿主机或 PVE 的 LXC 中时，可以用 file provider 方式代理。但如果 HA 也在 Docker 里，同样的 Label 方式：

```yaml
  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    container_name: homeassistant
    restart: unless-stopped
    networks:
      - traefik-net
    volumes:
      - ./ha_config:/config
      - /etc/localtime:/etc/localtime:ro
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.ha.rule=Host(`ha.yourlab.com`)"
      - "traefik.http.routers.ha.entrypoints=websecure"
      - "traefik.http.routers.ha.tls=true"
      - "traefik.http.routers.ha.tls.certresolver=letsencrypt"
      - "traefik.http.services.ha.loadbalancer.server.port=8123"
    # 如果 HA 需要通过 HomeKit 或 WebSocket，可以额外暴露端口
    # ports:
    #   - "8123:8123"
    devices:
      - /dev/ttyUSB0:/dev/ttyUSB0   # Zigbee 协调器直通
```

### 4.3 例 3：同时代理多个子域名（通配符证书）

如果使用 DNS-01 挑战，Traefik 可以申请通配符证书 `*.yourlab.com`。只需在路由上移除 `certresolver` 配置，Traefik 会自动使用已签发的通配符证书。

```yaml
  # 所有服务都可以这样配置，无需单独申请证书
  labels:
    - "traefik.enable=true"
    - "traefik.http.routers.xxx.rule=Host(`xxx.yourlab.com`)"
    - "traefik.http.routers.xxx.entrypoints=websecure"
    - "traefik.http.routers.xxx.tls=true"        # 隐式使用通配符证书
    - "traefik.http.services.xxx.loadbalancer.server.port=8080"
```

### 4.4 中间件：给服务加 IP 白名单

不想让某些服务暴露在公网？加个 IP 白名单中间件：

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.grafana.rule=Host(`grafana.yourlab.com`)"
  - "traefik.http.routers.grafana.middlewares=whitelist"
  - "traefik.http.middlewares.whitelist.ipwhitelist.sourcerange=192.168.0.0/16,10.0.0.0/8"
```

这样只有内网 IP 才能访问 Grafana，外网用户即使知道域名也打不开。

---

## 5. 踩坑记录

### ❌ 坑 1：acme.json 权限问题

Traefik 出于安全考虑，拒绝使用权限过于开放的 acme.json。

```bash
# 正确姿势
touch acme.json
chmod 600 acme.json
# 如果 chmod 600 后还报错，确认不是 root:root 以外属主
```

错误日志：`permission denied for /acme.json`

### ❌ 坑 2：DNS 挑战失败（Cloudflare）

Traefik 使用 Cloudflare DNS 挑战时，需要 API Token 有 `Zone:DNS:Edit` 权限。如果只给了 Read，会一直报错：

```
acme: error: 403 :: POST :: https://api.cloudflare.com/client/v4/zones/xxx/dns_records
```

需要去 Cloudflare Dashboard → My Profile → API Tokens，创建 token 时勾选 **Zone → DNS → Edit** 权限。

### ❌ 坑 3：Dashboard 暴露到公网

默认 Dashboard 绑定在 `:8080`，如果不加认证直接暴露到公网非常危险。我上面的配置通过反向代理访问 Dashboard，并加了 Basic Auth 中间件。或者你也可以在 `traefik.yml` 中只监听本地：

```yaml
api:
  dashboard: true
  # 只监听本地
  # （不配置 entrypoint，仅在反向代理路由中暴露）
```

### ❌ 坑 4：容器重启后 IP 变化

Docker 容器重启后 IP 会变，但 Traefik 通过 Docker provider 自动感知容器变化，所以不用担心。但如果你用 `file provider` 配置了后端地址，记得用容器名而不是 IP：

```yaml
# /config/dynamic.yml
http:
  services:
    my-service:
      loadBalancer:
        servers:
          - url: "http://my-container-name:8080"  # ✅ Docker DNS 解析
```

---

## 6. 进阶：Uptime Kuma 监控你的所有服务

有了 Traefik 后，再部署一个 Uptime Kuma 来监控所有服务的可用性：

```yaml
  uptime-kuma:
    image: louislam/uptime-kuma:latest
    container_name: uptime-kuma
    restart: unless-stopped
    networks:
      - traefik-net
    volumes:
      - ./uptime-kuma-data:/app/data
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.kuma.rule=Host(`status.yourlab.com`)"
      - "traefik.http.routers.kuma.entrypoints=websecure"
      - "traefik.http.routers.kuma.tls=true"
      - "traefik.http.routers.kuma.tls.certresolver=letsencrypt"
      - "traefik.http.services.kuma.loadbalancer.server.port=3001"
```

Uptime Kuma 可以监控 HTTP、TCP、Ping 等，一旦某个服务挂了，会通过 Telegram、邮件、Webhook 等方式告警，是 Homelab 运维的好帮手。

---

## 总结

有了 Traefik 之后，你的 Homelab 服务管理会变得无比清爽：

| 功能 | 效果 |
|------|------|
| ✅ 统一入口 | 所有服务通过域名访问，告别记端口号 |
| ✅ 自动 HTTPS | Let's Encrypt 自动签发和续期，零维护 |
| ✅ 通配符证书 | 一个证书覆盖所有子域名 |
| ✅ Docker 自动发现 | 加新服务只需加 Label，无需重启 Traefik |
| ✅ 中间件生态 | IP 白名单、Basic Auth、Rate Limit 开箱即用 |

**下一步可以做的事情：**

1. 部署 **Uptime Kuma** 实现服务可用性监控
2. 集成 **Grafana + Prometheus** 监控 PVE 和 Docker 主机资源
3. 使用 **Traefik Plugin** 实现更丰富的中间件功能（如 OAuth 代理、GeoIP 限制）
4. 配合 **Cloudflare Tunnel** 实现无公网 IP 的内网穿透

如果你目前还在用 Nginx 手动维护反代配置，强烈建议试试 Traefik — 尤其是在 Docker 环境下，它带来的"声明式配置"体验是革命性的。

> **预告：** 下一篇文章将介绍如何在 PVE 中部署 Grafana + Prometheus + Node Exporter 全面监控你的 Homelab 集群资源使用情况。
