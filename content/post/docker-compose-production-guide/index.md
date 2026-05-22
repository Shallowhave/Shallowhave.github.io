---
title: "Docker Compose 生产级配置完全指南"
description: "从 docker-compose.yml 基础编写到容器安全加固、健康检查、日志管理、网络规划，一篇涵盖 Homelab 场景下所有 Compose 配置最佳实践的深度指南"
date: 2026-05-22T09:25:32+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
slug: "docker-compose-production-guide"
---

## 前言

如果你玩 Homelab，Docker Compose 几乎是绕不过去的工具。从部署一个 Nginx 到编排完整的 Web 应用栈，Compose 让容器管理变得异常简单。

但简单的另一面是**坑多**。

你可能遇到过：

- 容器日志撑爆磁盘（默认**不轮转**，跑几天吃掉几十 GB）
- 服务一重启就挂，因为没有配置 `healthcheck` + `depends_on` 条件等待
- `deploy.resources` 设置了 CPU 内存限制却**完全无效**（V3 文件格式的经典陷阱）
- 容器以 root 运行，被反弹 Shell 后宿主机直接被控

这篇文章会把 Docker Compose 从"能跑"提升到"可靠、安全、可运维"的生产级水平。所有配置均来自实战踩坑，**直接复制就能用**。

## 1. Compose 文件格式：别再写 version 了

先看一个过时的例子：

```yaml
# ❌ 别这么写——这是 Docker Compose V1 时代的产物
version: '3.8'
services:
  web:
    image: nginx
```

**现状**：Docker Compose V2（命令是 `docker compose`，不带连字符）已全面取代 V1。V1 在 2023 年 7 月正式停止维护。V2 完全忽略顶层 `version` 字段，写了反而报警告。

**推荐写法（Compose 规范）**：

```yaml
# ✅ 当前推荐——无 version 行
name: myapp

services:
  web:
    image: nginx:alpine
```

| 方面 | V1（已弃用） | V2（当前） |
|---|---|---|
| 命令 | `docker-compose` | `docker compose` |
| 语言 | Python | Go |
| version 字段 | 必需 | 已忽略，有警告 |
| 推荐文件名 | `docker-compose.yml` | `compose.yaml` |
| `depends_on.condition` | 有限支持 | 完整支持 |
| 性能 | 慢 | 快 2-3 倍 |

**迁移步骤**：
1. 删除 `version:` 行
2. 将命令中的 `docker-compose` 替换为 `docker compose`
3. 文件名改为 `compose.yaml`（可选，旧名仍兼容）
4. 资源限制用 V2 风格的 `mem_limit`、`cpus` 等键（见下节）

## 2. 资源限制：一个值不对就白设

这是 Homelab 里最容易踩的坑——**没有之一**。

### ❌ 无效写法

```yaml
services:
  app:
    image: myapp
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
```

这段配置在 `docker compose up` 模式下**完全无效**。`deploy.resources` 只在 Docker Swarm 模式下生效。普通 `docker compose up` 会静默忽略它，容器照样能吃掉宿主机所有资源。

### ✅ 有效写法

```yaml
services:
  app:
    image: myapp
    mem_limit: 512m          # 硬性上限，超出即 OOM Kill
    mem_reservation: 256m    # 软性保留（尽力保证）
    memswap_limit: 768m      # 内存 + Swap 上限
    cpus: '1.5'              # 最多 1.5 个核心
    cpuset: '0,1'            # 绑定到 CPU 0 和 1
```

这些键在 Compose 规范下对所有启动模式生效。

### CPU 绑定实战

在 Homelab 中，CPU 绑定非常实用。例如 PVE 上跑一个 Jellyfin 转码容器和一个数据库容器：

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin
    cpuset: '2-5'            # 转码占用核 2-5
    cpus: '4'
    mem_limit: 4g

  postgres:
    image: postgres:16-alpine
    cpuset: '0-1'            # 数据库跑核 0-1
    cpus: '2'
    mem_limit: 2g
```

这样两个容器互不争抢，避免转码高峰期让数据库响应变慢。

## 3. 重启策略：unless-stopped 是 Homelab 首选

| 策略 | 行为 | 重启后 | 推荐场景 |
|---|---|---|---|
| `no` | 从不重启 | — | 一次性任务 |
| `always` | 始终重启 | 即使手动 `stop` 过也会启动 | 关键服务 |
| `unless-stopped` | 始终重启，但手动停止的除外 | 手动停止 → 不启动 | ✅ **Homelab 首选** |
| `on-failure` | 仅非零退出码时重启 | 非零退出 → 启动 | 有状态服务 |

```yaml
services:
  app:
    image: myapp
    restart: unless-stopped
    stop_grace_period: 30s   # 给优雅关闭留出时间
```

**为什么不用 `always`**？假设你调试时手动 `docker stop` 了一个服务，Docker 守护进程重启后 `always` 策略会把它又拉起来——你可能已经不想要它了。`unless-stopped` 尊重你的手动操作，更可控。

## 4. 健康检查：让你的服务链真正可靠

没有 `healthcheck` 的 `depends_on` 是**假依赖**——它只保证容器启动了，不保证服务可用了。

### 常见服务的健康检查

```yaml
services:
  postgres:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 30s    # 启动期内不计算重试次数

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

  nginx:
    image: nginx:alpine
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:80/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

### 依赖等待——按条件启动

```yaml
services:
  app:
    image: myapp
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
```

这样 `app` 会等到 PostgreSQL 和 Redis 都通过健康检查后才启动，彻底避免连接拒绝错误。

**注意**：旧版 V3 的 `depends_on` 不支持 `condition` 字段，只有 Compose 规范（无 version 行）才能这样写。

### 调试健康检查

```bash
docker inspect --format='{{json .State.Health}}' <容器名>
docker inspect --format='{{range .State.Health.Log}}{{.Output}}{{end}}' <容器名>
```

## 5. 日志管理：不配轮转 = 磁盘定时炸弹

这是我在 Homelab 里吃过最大的亏——一个容器跑了三天，`/var/lib/docker/containers/` 吃掉了 47 GB。

### 日志驱动对比

| 驱动 | 轮转支持 | 性能 | 推荐场景 |
|---|---|---|---|
| `json-file`（默认） | 手动配 `max-size`/`max-file` | 中等 | 兼容性好 |
| `local` | **内置自动轮转** | ✅ 最高 | ✅ **Homelab 首选** |
| `journald` | 由 journald 控制 | 好 | systemd 环境 |
| `none` | 无 | 最高 | 极少用（不落盘） |

### 推荐配置：每个服务独立设置

```yaml
services:
  app:
    image: myapp
    logging:
      driver: local
      options:
        max-size: "10m"
        max-file: "3"
```

`local` 驱动性能优于 `json-file`（二进制格式写入更快），且默认自动轮转。

### 全局配置（对所有容器生效）

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3",
    "compress": "true"
  }
}
```

写入 `/etc/docker/daemon.json` 后执行 `systemctl restart docker`。

## 6. 网络规划：别只用默认 bridge

### 驱动选择

| 驱动 | 隔离性 | 外部访问 | 适用场景 |
|---|---|---|---|
| **bridge**（默认） | ✅ 好 | 通过 `ports` 映射 | ✅ **大部分 Homelab 服务** |
| **host** | ❌ 无隔离 | 共享宿主机网络 | 性能敏感型（Nginx 代理、P2P） |
| **macvlan** | 中 | 每个容器独立 LAN IP | 需要直连 LAN（Pi-hole DHCP、Home Assistant） |
| **ipvlan** | 中 | 同上，MAC 地址少 | Macvlan 替代方案，交换机不撑 |
| **none** | 完全隔离 | 无 | 仅 localhost 服务 |

### 自定义桥接网络——核心技巧

```yaml
networks:
  frontend:
    driver: bridge
    ipam:
      config:
        - subnet: "172.20.0.0/16"
          gateway: "172.20.0.1"
  backend:
    driver: bridge
    internal: true     # 无外部网络访问，仅容器间通信

services:
  nginx:
    image: nginx:alpine
    networks:
      frontend:
        aliases:
          - web-proxy
  app:
    image: myapp
    networks:
      - frontend
      - backend
  db:
    image: postgres:16-alpine
    networks:
      backend:
        ipv4_address: 172.20.1.10   # 分配固定 IP
```

关键点：

- **自定义桥接网络自带 DNS 服务发现**——容器之间可以直接用服务名通信，不需要 `links`（已废弃）
- `internal: true` 的网络完全隔离外网，数据库和 Redis 放在这样的网络里更安全
- `ipv4_address` 给关键服务分配固定 IP，方便防火墙规则

### Macvlan 实战：让容器拥有独立 LAN IP

```yaml
networks:
  macvlan_lan:
    driver: macvlan
    driver_opts:
      parent: eth0
    ipam:
      config:
        - subnet: "192.168.1.0/24"
          gateway: "192.168.1.1"
          ip_range: "192.168.1.200/28"

services:
  pihole:
    image: pihole/pihole
    networks:
      macvlan_lan:
        ipv4_address: 192.168.1.210
```

这样 Pi-hole 在局域网中看起来就像一台独立机器，可以直接用 `192.168.1.210` 作为 DNS 服务器。

## 7. Secret 管理：别再写死密码

### 推荐方式：文件式 Secrets

```yaml
secrets:
  db_password:
    file: ./secrets/db_password.txt

services:
  app:
    image: myapp
    secrets:
      - db_password
```

容器内会挂载到 `/run/secrets/db_password`，应用从文件读取。

### 为什么优于环境变量？

| 方式 | `docker inspect` 暴露 | 日志暴露 | 子进程可见 |
|---|---|---|---|
| 环境变量 `POSTGRES_PASSWORD=xxx` | ✅ 是 | ✅ 是（启动日志） | ✅ 是 |
| Secret 文件 `/run/secrets/*` | ❌ 否 | ❌ 否 | ❌ 需显式读取 |

```yaml
# 安全的做法：使用 _FILE 后缀（PostgreSQL 原生支持）
services:
  postgres:
    image: postgres:16-alpine
    secrets:
      - db_password
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
```

### .env 文件的正确用法

`docker-compose.yml` 同目录下的 `.env` 文件会自动加载，用于注入非敏感变量：

```env
# .env
NGINX_PORT=8080
LOG_LEVEL=info
COMPOSE_PROJECT_NAME=myapp
```

```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "${NGINX_PORT}:80"
```

## 8. 容器安全加固

### 最小权限原则

每个容器都应该遵循：**放弃所有权限，只添加需要的**。

```yaml
services:
  nginx:
    image: nginx:alpine
    user: "101:101"              # Nginx 的默认 UID/GID
    read_only: true              # 文件系统只读
    tmpfs:
      - /tmp
      - /var/cache/nginx
      - /var/run
    cap_drop:
      - ALL                      # 放弃所有内核能力
    cap_add:
      - CHOWN
      - NET_BIND_SERVICE
      - SETGID
      - SETUID
    security_opt:
      - no-new-privileges:true   # 禁止子进程提权
```

### 常见服务权限清单

| 服务 | 可用权限 | 备选方案 |
|---|---|---|
| Nginx（80/443） | `NET_BIND_SERVICE` + `CHOWN` + `SETGID` + `SETUID` | user: 101:101 |
| Redis | `cap_drop: ALL` | user: 999:999 |
| PostgreSQL | 较难严格限制，至少 `no-new-privileges` | 先测试 `cap_drop: ALL` |
| Jellyfin 转码 | 需 `SYS_NICE` + 挂载 `/dev/dri` | 酌情放宽权限 |

## 9. 多环境配置：dev/prod 分离

### 文件命名规则

```
项目/
├── compose.yaml            # 基础配置
├── compose.override.yaml   # 🔄 自动加载（若有，docker compose up 自动合并）
├── compose.dev.yaml        # 开发环境
└── compose.prod.yaml       # 生产环境
```

### 基础配置（compose.yaml）

```yaml
services:
  app:
    image: myapp
    environment:
      - DB_HOST=db
      - DB_PORT=5432
  db:
    image: postgres:16-alpine
```

### 开发覆盖（compose.dev.yaml）

```yaml
services:
  app:
    ports:
      - "3000:3000"
      - "9229:9229"         # Node.js debugger
    volumes:
      - .:/app              # 代码热更新
    environment:
      - NODE_ENV=development
  db:
    ports:
      - "5432:5432"         # 暴露给本地数据库客户端
```

### 生产覆盖（compose.prod.yaml）

```yaml
services:
  app:
    ports:
      - "127.0.0.1:3000:3000"    # 仅监听本地，前面放 Nginx/Traefik
    restart: unless-stopped
    mem_limit: 512m
    cpus: '1'
    logging:
      driver: local
      options:
        max-size: "10m"
        max-file: "3"
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
```

### 启动命令

```bash
# 开发
docker compose -f compose.yaml -f compose.dev.yaml up -d

# 生产
docker compose -f compose.yaml -f compose.prod.yaml up -d

# 或用环境变量
export COMPOSE_FILE=compose.yaml:compose.prod.yaml
docker compose up -d
```

## 10. 完整 Homelab 生产级示例

以下是一个涵盖所有技巧的完整示例，可直接用于 Homelab 中的 Web 应用部署：

```yaml
name: homelab-webapp

services:
  traefik:
    image: traefik:v3.1
    command:
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
    ports:
      - "80:80"
      - "443:443"
    networks:
      - proxy
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    user: "1000:1000"
    read_only: true
    tmpfs:
      - /tmp
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
      - CHOWN
    security_opt:
      - no-new-privileges:true
    restart: unless-stopped
    mem_limit: 256m
    cpus: '0.5'
    logging:
      driver: local
      options:
        max-size: "10m"
        max-file: "3"
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:80/ping"]
      interval: 30s
      timeout: 10s
      retries: 3

  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    networks:
      backend:
        ipv4_address: 172.20.1.10
    volumes:
      - postgres_data:/var/lib/postgresql/data
    env_file:
      - ./postgres.env
    mem_limit: 1g
    mem_reservation: 512m
    cpus: '1'
    cpuset: '0,1'
    logging:
      driver: local
      options:
        max-size: "10m"
        max-file: "3"
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "myapp"]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 30s

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    networks:
      - backend
    volumes:
      - redis_data:/data
    command: ["redis-server", "--appendonly", "yes"]
    user: "999:999"
    read_only: true
    tmpfs:
      - /tmp
    cap_drop:
      - ALL
    mem_limit: 256m
    logging:
      driver: local
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

  app:
    image: myapp:latest
    restart: unless-stopped
    networks:
      - proxy
      - backend
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    env_file:
      - ./app.env
    mem_limit: 512m
    cpus: '1'
    logging:
      driver: local
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  cron-backup:
    image: alpine:3.20
    restart: unless-stopped
    networks:
      - backend
    volumes:
      - postgres_data:/data/postgres:ro
      - ./backups:/backups
    entrypoint: |
      sh -c "
      echo '0 3 * * * pg_dump -h postgres -U myapp myapp > /backups/db_\$(date +\\%Y\\%m\\%d).sql' > /var/spool/cron/crontabs/root
      crond -f
      "
    depends_on:
      postgres:
        condition: service_healthy
    mem_limit: 128m
    cpus: '0.2'

volumes:
  postgres_data:
  redis_data:

networks:
  proxy:
    driver: bridge
  backend:
    driver: bridge
    internal: true
    ipam:
      config:
        - subnet: "172.20.0.0/16"
          gateway: "172.20.0.1"
```

## 踩坑记录

### ❌ 踩坑 1：`deploy.resources` 在非 Swarm 模式下不生效

**后果**：限制形同虚设，容器 OOM 杀死的不是容器而是整个宿主机的可用内存。

**解决**：用 `mem_limit`、`cpus` 等 V2 风格键。

### ❌ 踩坑 2：没配日志轮转，磁盘被撑爆

**后果**：一个容器几天产生 40+ GB 日志文件，导致 Docker 守护进程 hang 死。

**解决**：每个服务配 `logging: driver: local` 或全局设 `daemon.json`。

### ❌ 踩坑 3：`depends_on` 不加条件

**后果**：应用启动时数据库还没就绪，疯狂报错重试，进入崩溃循环。

**解决**：`depends_on: db: condition: service_healthy` + 数据库侧配 `healthcheck`。

### ❌ 踩坑 4：容器以 root 运行

**后果**：容器被攻破后，攻击者直接获得宿主机的 root 权限（容器逃逸）。

**解决**：`user: "1000:1000"` + `cap_drop: ALL` + `security_opt: no-new-privileges:true`。

### ❌ 踩坑 5：使用默认 bridge 网络

**后果**：容器之间不能通过服务名自动解析（需 `--link`，已废弃），且所有容器在同一个广播域中。

**解决**：自定义 bridge 网络，自带 DNS 服务发现。

## 总结

写好一个生产级 `compose.yaml`，核心是做到三点：**限制资源、管理日志、加固安全**。

| 维度 | 核心命令/配置 | 一句话总结 |
|---|---|---|
| 文件格式 | 无 `version` 行，文件名 `compose.yaml` | 跟上时代，删除弃用字段 |
| 资源限制 | `mem_limit`/`cpus`/`cpuset` | 别用 `deploy.resources`，它只在 Swarm 下有效 |
| 重启策略 | `restart: unless-stopped` | 手动停的不重启，Docker 重启的自动恢复 |
| 健康检查 | `healthcheck` + `condition: service_healthy` | 让依赖链真正可靠 |
| 日志管理 | `driver: local`, `max-size: 10m` | 不配轮转等于给磁盘装了个定时炸弹 |
| 网络 | 自定义 bridge，`internal: true` | 用服务名通信，数据库放内网 |
| 安全 | `user` + `cap_drop` + `read_only` | 最小权限原则，能不给的权限一律不给 |
| Secrets | `secrets: file:` | 敏感信息走文件，别用环境变量 |

把这份配置保存为你的 `compose.yaml` 模板，每次开新项目时对照着写，你的 Homelab 能少踩 80% 的坑。
