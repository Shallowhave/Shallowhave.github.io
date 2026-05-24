---
title: "Homelab 日志别再 SSH 上去 grep：Loki + Alloy 打造轻量集中日志系统"
description: "用 Grafana Loki、Grafana Alloy 和 Docker Compose 搭建 Homelab 集中日志平台，采集 Docker 容器日志与 systemd journal，包含可直接落地的配置、LogQL 查询、保留策略和踩坑记录。"
date: 2026-05-24T01:26:16+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
slug: "homelab-loki-alloy-logs"
---

## 前言

Homelab 服务跑多以后，排障最常见的动作不是重启，而是找日志：PVE 节点上看 `journalctl`，Docker 主机上看 `docker logs`，NAS 上再 SSH 一次，最后还要把几台机器的时间线手动对齐。服务少的时候还能忍，等你有了反向代理、DNS、监控、相册、Home Assistant、备份任务之后，**日志分散**会变成非常低效的运维方式。

本文搭一个轻量集中日志系统：**Grafana Loki + Grafana Alloy + Grafana**。

它适合 Homelab 的原因很直接：

- Loki 不像 Elasticsearch 那样默认把全文建索引，更省磁盘和内存；
- Alloy 是 Grafana 官方推荐的新一代采集 Agent，可替代已经进入生命周期末期的 Promtail；
- Docker Compose 即可部署，不需要 Kubernetes；
- 可以同时采集 Docker 容器日志和 Linux systemd journal；
- Grafana 里可以用 LogQL 按主机、容器、服务快速过滤日志。

> 注意：下面示例默认只在内网使用。Loki 示例配置 `auth_enabled: false`，不要把 `3100` 端口直接暴露到公网。

## 1. 架构设计

单机 Homelab 可以把 Loki、Grafana、Alloy 都放在同一个 Compose 栈里。多节点场景则建议中心节点跑 Loki + Grafana，每台需要采集日志的机器单独跑一个 Alloy Agent。

```text
Docker 主机 / PVE VM / NAS
  ├── Docker logs
  ├── systemd journal
  └── Grafana Alloy  --->  Loki :3100  --->  Grafana :3000
```

组件分工如下：

| 组件 | 作用 | Homelab 建议 |
|---|---|---|
| Loki | 日志存储与查询 | 单机模式 + 文件系统存储 |
| Alloy | 日志采集 Agent | 每台主机一个，读取 Docker socket 和 journal |
| Grafana | 查询、可视化、Explore | 复用已有 Grafana 或单独部署 |
| LogQL | 日志查询语言 | 用 label 过滤，用管道做文本/JSON 解析 |

为什么不用 Promtail？Grafana 官方已经将 Promtail 标记为生命周期末期，新部署建议直接用 Alloy。Promtail 老配置还能跑，但新 Homelab 没必要再背一套即将退场的 Agent。

## 2. 准备目录

这里把数据和配置都放在 `~/loki-stack`，方便备份和迁移。

```bash
mkdir -p ~/loki-stack/{loki,alloy,grafana/provisioning/datasources}
cd ~/loki-stack
```

目录结构如下：

```text
loki-stack/
├── docker-compose.yml
├── loki/
│   └── loki-config.yaml
├── alloy/
│   └── config.alloy
└── grafana/
    └── provisioning/
        └── datasources/
            └── loki.yaml
```

如果你已经有 Grafana，可以不部署本文里的 `grafana` 服务，只保留 Loki 和 Alloy，然后在现有 Grafana 里添加 Loki 数据源即可。

## 3. Docker Compose：一套跑起来

创建 `docker-compose.yml`：

```yaml
services:
  loki:
    image: grafana/loki:3.5.12
    container_name: loki
    command:
      - -config.file=/etc/loki/loki-config.yaml
    ports:
      - "3100:3100"
    volumes:
      - ./loki/loki-config.yaml:/etc/loki/loki-config.yaml:ro
      - ./data/loki:/loki
    restart: unless-stopped

  grafana:
    image: grafana/grafana:13.0.1
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: change-this-password
      GF_USERS_ALLOW_SIGN_UP: "false"
    volumes:
      - ./data/grafana:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
    depends_on:
      - loki
    restart: unless-stopped

  alloy:
    image: grafana/alloy:v1.16.1
    container_name: alloy
    user: "0:0"
    command:
      - run
      - --server.http.listen-addr=0.0.0.0:12345
      - --storage.path=/var/lib/alloy/data
      - /etc/alloy/config.alloy
    environment:
      ALLOY_HOSTNAME: ${HOSTNAME}
      LOKI_URL: http://loki:3100/loki/api/v1/push
    ports:
      - "12345:12345"
    volumes:
      - ./alloy/config.alloy:/etc/alloy/config.alloy:ro
      - ./data/alloy:/var/lib/alloy
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /var/log/journal:/var/log/journal:ro
      - /etc/machine-id:/etc/machine-id:ro
    depends_on:
      - loki
    restart: unless-stopped
```

几个关键点：

1. `user: "0:0"` 是为了读取宿主机 journal 文件，权限不够时 Alloy 会采集不到 systemd 日志；
2. Docker socket 即使只读也很敏感，Alloy 这台机器要当成可信组件；
3. Loki 数据用 `./data/loki:/loki` 这种显式目录，比 Docker named volume 更容易被 Restic、PBS 或 NAS 备份；
4. `grafana/loki`、`grafana/alloy`、`grafana/grafana` 都建议固定版本，不要生产环境裸用 `latest`。

## 4. Loki 配置：单机存储 + 7 天保留

创建 `loki/loki-config.yaml`：

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  instance_addr: 127.0.0.1
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  retention_period: 168h

compactor:
  working_directory: /loki/compactor
  retention_enabled: true
  delete_request_store: filesystem

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100
```

`retention_period: 168h` 表示保留 7 天。日志系统最容易被忽略的坑就是**只采集不清理**，最后磁盘爆掉。Loki 的保留清理依赖 compactor，所以上面同时开启了：

```yaml
compactor:
  retention_enabled: true
```

如果你的日志量很小，可以改成 30 天：

```yaml
limits_config:
  retention_period: 720h
```

但不要一上来就永久保留。Homelab 的日志价值通常集中在最近几天：服务为什么重启、证书为什么续签失败、备份任务昨晚有没有报错，这些问题 7~30 天足够覆盖。

## 5. Grafana 自动配置 Loki 数据源

创建 `grafana/provisioning/datasources/loki.yaml`：

```yaml
apiVersion: 1

datasources:
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    isDefault: false
    editable: true
```

Grafana 启动后会自动加载这个数据源。进入 `Explore` 页面选择 `Loki`，就可以开始查日志。

如果你复用已有 Grafana，手动添加数据源也可以：

```text
Connections -> Data sources -> Add data source -> Loki
URL: http://<loki-ip>:3100
```

## 6. Alloy 配置：采集 Docker + systemd journal

创建 `alloy/config.alloy`：

```hcl
logging {
  level = "info"
}

loki.write "default" {
  endpoint {
    url = env("LOKI_URL")
  }
}

discovery.docker "containers" {
  host = "unix:///var/run/docker.sock"
}

discovery.relabel "docker" {
  targets = []

  rule {
    target_label = "job"
    replacement  = "docker"
  }

  rule {
    target_label = "host"
    replacement  = env("ALLOY_HOSTNAME")
  }

  rule {
    source_labels = ["__meta_docker_container_name"]
    regex         = "/(.*)"
    target_label  = "container"
  }

  rule {
    source_labels = ["__meta_docker_container_log_stream"]
    target_label  = "stream"
  }

  rule {
    source_labels = ["__meta_docker_container_label_com_docker_compose_project"]
    target_label  = "compose_project"
  }

  rule {
    source_labels = ["__meta_docker_container_label_com_docker_compose_service"]
    target_label  = "compose_service"
  }
}

loki.source.docker "containers" {
  host             = "unix:///var/run/docker.sock"
  targets          = discovery.docker.containers.targets
  relabel_rules    = discovery.relabel.docker.rules
  refresh_interval = "5s"
  forward_to       = [loki.write.default.receiver]
}

loki.source.journal "system" {
  path    = "/var/log/journal"
  max_age = "24h"

  labels = {
    job  = "systemd-journal"
    host = env("ALLOY_HOSTNAME")
  }

  forward_to = [loki.write.default.receiver]
}
```

这里的 label 设计很重要。Loki 的原则是：**低基数信息放 label，高基数信息留在日志正文里查询**。

推荐 label：

| Label | 例子 | 说明 |
|---|---|---|
| `host` | `pve1` | 主机名，基数低 |
| `job` | `docker` / `systemd-journal` | 日志来源 |
| `container` | `caddy` | 容器名 |
| `compose_service` | `grafana` | Compose 服务名 |
| `stream` | `stdout` / `stderr` | Docker 输出流 |

不要把这些字段做成 label：

```text
request_id、trace_id、user_id、email、完整 URL、客户端 IP、时间戳
```

这些字段变化太快，会制造高基数，Loki 查询会变慢，内存压力也会明显增加。

## 7. 启动与验证

启动服务：

```bash
cd ~/loki-stack
docker compose up -d
```

查看容器状态：

```bash
docker compose ps
```

验证 Loki：

```bash
curl -f http://localhost:3100/ready
```

正常会返回：

```text
ready
```

验证 Alloy：

```bash
curl -f http://localhost:12345/-/ready
```

查看 Alloy 自身日志：

```bash
docker logs -f alloy
```

Grafana 访问：

```text
http://你的服务器IP:3000
```

默认账号密码来自 Compose：

```text
admin / change-this-password
```

首次登录后立刻改密码，或者在 Compose 里提前改掉 `GF_SECURITY_ADMIN_PASSWORD`。

## 8. 常用 LogQL 查询

进入 Grafana -> Explore -> Loki，先试下面这些查询。

查看所有 Docker 日志：

```logql
{job="docker"}
```

查看某台主机：

```logql
{job="docker", host="pve1"}
```

查看某个 Compose 服务：

```logql
{job="docker", compose_service="caddy"}
```

只看 stderr：

```logql
{job="docker", stream="stderr"}
```

查看 systemd journal：

```logql
{job="systemd-journal"}
```

查找错误关键词：

```logql
{job="systemd-journal"} |= "error"
```

如果应用输出 JSON 日志，可以临时解析：

```logql
{job="docker", compose_service="app"} | json
```

统计 5 分钟内每个服务的错误数量：

```logql
sum by (host, compose_service) (
  count_over_time({job="docker"} |= "error" [5m])
)
```

这个查询很适合做 Grafana 面板：某个服务突然刷错误时，一眼就能看到是哪台机器、哪个 Compose 服务在出问题。

## 9. 多节点接入：每台机器跑一个 Alloy

中心节点只需要开放 Loki 的内网地址，比如：

```text
http://192.168.1.20:3100/loki/api/v1/push
```

其他 Docker 主机创建一个精简 Alloy Agent：

```bash
mkdir -p ~/alloy-agent/alloy
cd ~/alloy-agent
cp ~/loki-stack/alloy/config.alloy ./alloy/config.alloy
```

`docker-compose.yml`：

```yaml
services:
  alloy:
    image: grafana/alloy:v1.16.1
    container_name: alloy
    user: "0:0"
    command:
      - run
      - --server.http.listen-addr=0.0.0.0:12345
      - --storage.path=/var/lib/alloy/data
      - /etc/alloy/config.alloy
    environment:
      ALLOY_HOSTNAME: ${ALLOY_HOSTNAME}
      LOKI_URL: ${LOKI_URL}
    ports:
      - "12345:12345"
    volumes:
      - ./alloy/config.alloy:/etc/alloy/config.alloy:ro
      - ./data/alloy:/var/lib/alloy
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /var/log/journal:/var/log/journal:ro
      - /etc/machine-id:/etc/machine-id:ro
    restart: unless-stopped
```

启动：

```bash
export ALLOY_HOSTNAME="$(hostname -s)"
export LOKI_URL="http://192.168.1.20:3100/loki/api/v1/push"
docker compose up -d
```

如果 Loki 放在反向代理后面，例如 `https://logs.example.com`，则：

```bash
export LOKI_URL="https://logs.example.com/loki/api/v1/push"
docker compose up -d
```

生产一点的做法是把 `LOKI_URL` 写入 `.env`：

```bash
cat > .env <<'EOF'
ALLOY_HOSTNAME=pve1
LOKI_URL=http://192.168.1.20:3100/loki/api/v1/push
EOF
```

## 10. Docker 日志轮转也要做

集中日志不等于本机日志可以无限增长。Docker 默认日志如果不限制，某些刷屏容器能把系统盘写满。建议在所有 Docker 主机设置日志轮转。

```bash
sudo tee /etc/docker/daemon.json >/dev/null <<'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF

sudo systemctl restart docker
```

注意：重启 Docker 会影响正在运行的容器，建议维护窗口执行。执行后检查：

```bash
docker info --format '{{.LoggingDriver}}'
```

查看单个容器是否还能被 Docker API 读取日志：

```bash
docker logs --tail=20 <container_name>
```

只要 `docker logs` 能正常看到日志，Alloy 的 `loki.source.docker` 通常就能采集。

## 11. 踩坑记录

### ❌ 1. `/var/log/journal` 不存在

有些系统默认使用 volatile journal，日志在 `/run/log/journal`，重启就没了。先检查：

```bash
ls -ld /var/log/journal
```

如果不存在，开启持久化 journal：

```bash
sudo mkdir -p /var/log/journal
sudo systemctl restart systemd-journald
```

然后重启 Alloy：

```bash
docker compose restart alloy
```

### ❌ 2. Alloy 没权限读 journal

表现是 Docker 日志有，systemd 日志没有。优先检查 Alloy 日志：

```bash
docker logs alloy | grep -i journal
```

解决思路：

- Compose 里保留 `user: "0:0"`；
- 确认挂载了 `/var/log/journal`；
- 确认挂载了 `/etc/machine-id`；
- 如果系统实际日志在 `/run/log/journal`，需要改挂载路径和 Alloy 配置里的 `path`。

### ❌ 3. Loki 暴露到公网

本文 Loki 配置关闭了认证：

```yaml
auth_enabled: false
```

这只适合内网。如果必须跨公网推送日志，至少用 WireGuard/Tailscale，或者在 Caddy/Traefik 上加 Basic Auth、mTLS、访问源 IP 限制。Grafana 可以放在反代后做登录，Loki push endpoint 不建议裸奔。

### ❌ 4. Label 设计太激进

很多人一开始会把 `ip`、`path`、`request_id` 都提成 label，短期看查询很方便，长期就是灾难。Loki 的索引对象是 label 组合，label 基数过高会导致查询和写入都变慢。

正确思路：

- `host`、`service`、`container` 这种稳定字段做 label；
- `request_id`、`user_id` 留在日志正文；
- 查询时用 `|= "关键词"`、`| json`、`| logfmt` 临时解析。

## 12. 总结

这套 Loki + Alloy 日志方案不追求大而全，目标是解决 Homelab 里最实际的问题：**不用 SSH 到每台机器上手动 grep 日志**。

| 场景 | 推荐做法 |
|---|---|
| 单台 Docker 主机 | Loki + Grafana + Alloy 同机 Compose 部署 |
| 多台 Linux/Docker 主机 | 中心 Loki，每台主机一个 Alloy |
| 日志保留 | 先从 7 天开始，确认容量后再调到 30 天 |
| 安全 | Loki 只走内网/VPN，不公网裸露 |
| 查询 | 低基数 label 过滤，高基数字段用 LogQL 管道解析 |

如果你已经有 Prometheus + Grafana 监控，Loki 正好补上“日志”这一块：指标告诉你**哪里异常**，日志告诉你**为什么异常**。两者结合起来，Homelab 排障效率会比纯 SSH 查日志高很多。