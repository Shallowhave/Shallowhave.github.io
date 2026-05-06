---
title: "Homelab 必备：Prometheus + Grafana 全栈监控告警体系搭建指南"
description: "在 Proxmox VE Homelab 中，用 Docker Compose 一键部署 Prometheus、Grafana、Node Exporter、Uptime Kuma 和 Alertmanager，构建从宿主机指标到服务可用性的全方位监控告警系统，支持 Telegram 实时推送。"
date: 2026-05-06T09:01:07+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
slug: "homelab-monitoring-stack-guide"
---

## 前言

Homelab 跑了大半年，服务从 PVE 虚拟机到 Docker 容器越堆越多——Jellyfin、Home Assistant、Traefik、Portainer、Grafana……直到有一天 2TB 的 ZFS 池被日志撑爆，磁盘 IO 飙到 100% 系统卡成 PPT，我才意识到一个严肃的问题：**我根本不知道我的 Homelab 里正在发生什么。**

没有监控，就是盲人摸象。本文手把手教你搭建一套完整的 Homelab 监控体系，覆盖以下场景：

- **宿主机层**：PVE 宿主机的 CPU、内存、磁盘、网络、ZFS ARC 命中率
- **服务层**：Docker 中各容器的资源占用、运行状态、HTTP 可达性
- **告警层**：当磁盘快满、服务挂了、证书快过期时，Telegram 机器人直接通知到手机

整套方案基于 Docker Compose 一键部署，与之前搭建的 [Traefik 反向代理](/post/traefik-reverse-proxy-homelab/) 无缝集成。

## 整体架构

我们使用以下组件组成监控栈：

| 组件 | 作用 | 镜像 |
|------|------|------|
| **Prometheus** | 时序数据库，存储所有指标 | `prom/prometheus` |
| **Node Exporter** | 采集宿主机系统指标 | `prom/node-exporter` |
| **cAdvisor** | 采集 Docker 容器指标 | `gcr.io/cadvisor/cadvisor` |
| **Grafana** | 可视化仪表盘 | `grafana/grafana` |
| **Uptime Kuma** | 外部服务可用性监控 | `louislam/uptime-kuma` |
| **Alertmanager** | 告警聚合与路由分发 | `prom/alertmanager` |
| **Blackbox Exporter** | HTTP/HTTPS/TCP 探针 | `prom/blackbox-exporter` |

数据流：`Node Exporter / cAdvisor / Blackbox Exporter → Prometheus (抓取) → Grafana (展示) + Alertmanager (告警规则匹配) → Telegram Bot (推送)`

## 第一步：Docker Compose 部署监控栈

在 PVE 的 CT（推荐 Debian 或 Ubuntu LXC）或 VM 中，创建一个 `monitoring` 项目目录：

```bash
mkdir -p ~/monitoring && cd ~/monitoring
```

### docker-compose.yml

```yaml
version: '3.8'

networks:
  monitoring:
    name: monitoring
    driver: bridge
  traefik:
    external: true
    name: traefik

volumes:
  prometheus_data:
  grafana_data:
  alertmanager_data:
  uptime-kuma_data:

services:
  prometheus:
    image: prom/prometheus:v2.53.0
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/rules:/etc/prometheus/rules:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
    networks:
      - monitoring
    ports:
      - "9090:9090"

  node_exporter:
    image: prom/node-exporter:v1.8.0
    container_name: node_exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    networks:
      - monitoring
    ports:
      - "9100:9100"

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.49.1
    container_name: cadvisor
    restart: unless-stopped
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    privileged: true
    devices:
      - /dev/kmsg:/dev/kmsg
    networks:
      - monitoring
    ports:
      - "8080:8080"

  blackbox_exporter:
    image: prom/blackbox-exporter:v0.25.0
    container_name: blackbox_exporter
    restart: unless-stopped
    volumes:
      - ./blackbox/blackbox.yml:/etc/blackbox_exporter/config.yml:ro
    networks:
      - monitoring
    ports:
      - "9115:9115"

  grafana:
    image: grafana/grafana:11.0.0
    container_name: grafana
    restart: unless-stopped
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_INSTALL_PLUGINS=grafana-piechart-panel
      - GF_SERVER_ROOT_URL=https://grafana.你的域名.com
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - ./grafana/dashboards:/var/lib/grafana/dashboards:ro
    networks:
      - monitoring
      - traefik
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.grafana.rule=Host(`grafana.你的域名.com`)"
      - "traefik.http.services.grafana.loadbalancer.server.port=3000"

  alertmanager:
    image: prom/alertmanager:v0.27.0
    container_name: alertmanager
    restart: unless-stopped
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
      - alertmanager_data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
    networks:
      - monitoring
    ports:
      - "9093:9093"

  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    restart: unless-stopped
    volumes:
      - uptime-kuma_data:/app/data
    networks:
      - monitoring
      - traefik
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.uptime-kuma.rule=Host(`status.你的域名.com`)"
      - "traefik.http.services.uptime-kuma.loadbalancer.server.port=3001"
```

> **注意**：`cadvisor` 需要 `privileged: true` 和 `/dev/kmsg` 才能正常采集容器指标。如果用 LXC 容器部署，需要确保 LXC 的 `features: keyctl=1,nesting=1`。

### 第二步：Prometheus 配置文件

创建 `prometheus/prometheus.yml`：

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  scrape_timeout: 10s

rule_files:
  - /etc/prometheus/rules/*.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets:
        - 'node_exporter:9100'

  - job_name: 'cadvisor'
    static_configs:
      - targets:
        - 'cadvisor:8080'

  - job_name: 'blackbox_http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - https://你的域名.com
        - https://grafana.你的域名.com
        - https://status.你的域名.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox_exporter:9115
```

### 第三步：告警规则

创建 `prometheus/rules/homelab.yml`：

```yaml
groups:
  - name: homelab
    rules:
      - alert: 磁盘即将用满
        expr: (1 - node_filesystem_avail_bytes{mountpoint="/"}
              / node_filesystem_size_bytes{mountpoint="/"}) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "宿主机磁盘使用率超过 85%"
          description: "当前使用率 {{ $value | humanizePercentage }}"

      - alert: 宿主机宕机
        expr: up{job="node_exporter"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Node Exporter 无响应"
          description: "宿主机可能已宕机或网络不可达"

      - alert: 服务不可达
        expr: probe_success{job="blackbox_http"} == 0
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "服务 {{ $labels.instance }} 不可访问"
          description: "HTTP 探测失败，请检查服务状态"

      - alert: 容器异常重启
        expr: rate(container_restarts_total[5m]) > 0
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "容器频繁重启"
          description: "容器 {{ $labels.name }} 在 5 分钟内重启了多次"

      - alert: CPU 负载过高
        expr: (100 - (avg by(instance) (rate(node_cpu_seconds_total
              {mode="idle"}[5m])) * 100)) > 90
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "CPU 使用率超过 90%"
          description: "当前 CPU 使用率 {{ $value | humanizePercentage }}"

      - alert: 内存不足
        expr: (1 - node_memory_MemAvailable_bytes
              / node_memory_MemTotal_bytes) * 100 > 90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "内存使用率超过 90%"
          description: "当前内存使用率 {{ $value | humanizePercentage }}"
```

### 第四步：Alertmanager 配置与 Telegram 通知

创建 `alertmanager/alertmanager.yml`：

```yaml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'telegram'

receivers:
  - name: 'telegram'
    telegram_configs:
      - bot_token: '你的BOT_TOKEN'
        chat_id: 你的CHAT_ID
        message: |
          {{ range .Alerts }}
          🔴 *{{ .Labels.alertname }}*
          {{ .Annotations.description }}
          Severity: `{{ .Labels.severity }}`
          Started: {{ .StartsAt.Format "2006-01-02 15:04:05" }}
          {{ end }}
        parse_mode: 'Markdown'
        send_resolved: true
```

> **踩坑记录**：Telegram Bot Token 是敏感信息。建议用环境变量替代硬编码：
> `bot_token: '{{ external.url "https://vault.你的域名.com/telegram-token" }}'`
>
> 或者用 Docker secrets 注入。千万**不要**把 Bot Token 提交到 Git 仓库。

### 第五步：Blackbox Exporter 配置

创建 `blackbox/blackbox.yml`：

```yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200, 301, 302]
      follow_redirects: true
      preferred_ip_protocol: "ip4"

  tcp_connect:
    prober: tcp
    timeout: 5s

  icmp:
    prober: icmp
    timeout: 5s
    icmp:
      preferred_ip_protocol: "ip4"
```

### 第六步：Grafana 自动配置

创建 `grafana/provisioning/datasources/datasource.yml` 实现数据源自动注册：

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
```

然后安装社区流行的仪表盘。推荐以下几个 ID（在 Grafana Web UI 中通过 Import 导入）：

| 仪表盘 | ID | 描述 |
|--------|-----|------|
| Node Exporter Full | 1860 | 最全面的宿主机监控面板 |
| Docker Containers | 179 | cAdvisor 容器监控面板 |
| Prometheus 2.0 Stats | 3662 | Prometheus 自身健康状态 |

### 一键启动

配置完成后，启动所有服务：

```bash
cd ~/monitoring
docker compose up -d
```

检查服务状态：

```bash
docker compose ps
docker compose logs -f
```

确认 Prometheus Target 全部 UP：

```bash
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health}'
```

## 踩坑与优化

### 1. 存储 Retention 策略

Prometheus 默认只保留 15 天数据。在 Homelab 场景，我建议根据磁盘空间调整：

```bash
# 查看当前 Prometheus 数据占用
du -sh /var/lib/docker/volumes/monitoring_prometheus_data/_data

# 如果想保留更长时间，修改 retention 参数
# --storage.tsdb.retention.time=60d
```

我跑了 6 个月实验后的结论：**对于 Homelab 场景，30 天 retention + 15s scrape interval** 是存储和查询速度的最佳平衡点。更长的保留期会让 Grafana 查询变慢，而且旧数据的实用价值急剧下降。

### 2. Node Exporter 在 PVE LXC 中的注意事项

如果你把 Node Exporter 直接跑在 PVE 宿主上（而不是 Docker 里），要注意：

- 在 LXC 容器中，Node Exporter 看到的 `/proc` 和 `/sys` 是容器 namespace 的，**不反映宿主机真实指标**
- 如果需要在 LXC 中获取宿主机级指标，要么把 Node Exporter 直接安装在 PVE 宿主机上，要么用 PVE 的 `pve-node-exporter`（通过 `apt install prometheus-pve-exporter`）

在 PVE 宿主机上直接运行：

```bash
# PVE 宿主机安装 Node Exporter
apt update && apt install -y prometheus-node-exporter
systemctl enable --now prometheus-node-exporter
```

然后在 prometheus.yml 的静态目标中直接添加 PVE 宿主机的 IP：

```yaml
  - job_name: 'pve_host'
    static_configs:
      - targets: ['192.168.1.100:9100']  # PVE 宿主 IP
```

### 3. Grafana 内存占用过高

Grafana 默认配置下可能占用 300-500MB 内存。对于 4GB 的 LXC 容器有些吃力。优化方案：

```yaml
# 在 docker-compose 的 grafana environment 中添加
environment:
  - GF_PANELS_DISABLE_SANITIZE_HTML=true
  # 限制 Grafana 内存
  - GF_SERVER_ENABLE_GZIP=true
  # 关闭不必要的功能
  - GF_EXPLORE_ENABLED=false
  - GF_ALERTING_ENABLED=false  # 由 Prometheus Alertmanager 处理告警
```

### 4. Uptime Kuma 与 Prometheus 数据源

很多人会问：Uptime Kuma 和 Prometheus Blackbox Exporter 重复了吗？我的观点是**互补**：

- **Uptime Kuma**：面向人的监控，友好的 Web 界面、状态页、可视化监控历史饼图，适合分享给家人展示网络状况
- **Blackbox Exporter**：面向机器的监控，数据进 Prometheus，与已有告警体系集成

两者并存互不冲突。Uptime Kuma 也支持反向代理暴露为 `status.你的域名.com`，做成公开状态页。

## 集成 Traefik

如果你之前跟着本站的 [Traefik 部署指南](/post/traefik-reverse-proxy-homelab/) 搭建了反向代理，集成非常简单。

确保 Docker Compose 中 Grafana 和 Uptime Kuma 都连接了 `traefik` 网络，并添加了对应的 labels。在 Traefik 的 `dynamic.yml`（或 Provider 配置）中，不需要额外配置——Traefik 会自动发现 Docker 容器的 label。

然后就可以通过以下域名访问：

| 服务 | 地址 |
|------|------|
| Grafana 面板 | `https://grafana.你的域名.com` |
| Uptime Kuma 状态页 | `https://status.你的域名.com` |
| Prometheus Web UI | `https://prometheus.你的域名.com`（需额外配置） |
| Alertmanager UI | `https://alertmanager.你的域名.com`（需额外配置） |

> **安全建议**：Prometheus 和 Alertmanager 的 Web UI **不建议**暴露到公网。它们没有认证功能，暴露后任何人都可以查询你的所有指标。要么通过 Traefik Basic Auth 中间件保护，要么只在内网访问。

在 Traefik 中添加 Basic Auth 中间件的例子：

```yaml
# Traefik dynamic config 或 Docker label
labels:
  - "traefik.http.middlewares.prometheus-auth.basicauth.users=admin:$$2y$$10$$..."
  - "traefik.http.routers.prometheus.middlewares=prometheus-auth"
```

## 告警效果

告警配置完成后，当发生以下情况，Telegram 会在 1-5 分钟内收到通知：

- 磁盘使用率超过 85%
- 任意监控服务无法访问（HTTP 50x / 超时）
- Docker 容器异常重启
- 宿主机 CPU 或内存负载过高
- 证书即将过期（可结合 Blackbox Exporter 的 SSL 证书过期探测）

Telegram 通知示例：

```
🔴 磁盘即将用满
宿主机磁盘使用率超过 85%
当前使用率 91.2%
Severity: warning
Started: 2026-05-06 08:45:00
```

## 总结

这套监控栈跑了大半年，帮我抓到了不少问题：

1. Docker 日志未轮转导致 `/var/lib/docker` 撑爆宿主机 root —— 告警在磁盘 85% 时触发，及时清理
2. Jellyfin 硬件转码异常导致容器疯狂重启 —— Alertmanager 检测到容器重启频率异常
3. Traefik 证书有一次没自动续期 —— Blackbox Exporter 探测到 HTTPS 证书错误

有了监控，才真正感觉到 Homelab 是**可控的**。从盲人摸象到一目了然，这套组合是每个认真玩 Homelab 的人必备的基础设施。

对应的 Docker Compose 配置文件和告警规则模板已整理好，可以直接 clone 使用：

```bash
git clone https://github.com/你的用户名/homelab-monitoring.git
cd homelab-monitoring
# 编辑 alertmanager/alertmanager.yml 填入 Telegram Token
docker compose up -d
```

下一步可以探索：
- **Loki + Promtail**：Docker 日志聚合，与 Grafana 无缝集成
- **PVE Exporter**：采集 PVE 集群指标（虚拟机状态、存储池使用率、Ceph 状态）
- **SNMP Exporter**：监控交换机、UPS 等网络设备
- **Grafana OnCall**：更完善的告警排班和升级策略
