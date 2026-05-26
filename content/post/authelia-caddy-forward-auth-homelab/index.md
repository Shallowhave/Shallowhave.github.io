---
title: "Homelab 入口别只靠弱密码：Authelia + Caddy 给内网服务加统一认证"
description: "用 Authelia、Caddy Forward Auth、PostgreSQL 和 Redis 给 Homelab 内网服务加统一登录、访问控制与二次验证，包含 Docker Compose、Caddyfile、ACL 规则、排障命令和踩坑记录。"
date: 2026-05-26T01:22:38+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
slug: "authelia-caddy-forward-auth-homelab"
---

## 前言

Homelab 服务越跑越多以后，真正危险的不是某一个服务本身，而是**入口失控**：Grafana 一个默认账号、qBittorrent 一个弱口令、NAS 面板一个旧版本、某个临时测试服务忘了下线。反向代理和 HTTPS 只能解决「怎么访问」和「传输是否加密」，不能解决「谁能访问」。

本文搭一套适合家庭实验室的统一认证入口：**Caddy + Authelia Forward Auth**。

目标很明确：

- Caddy 继续负责 HTTPS、反向代理和路由；
- Authelia 负责登录、Cookie Session、访问控制和 TOTP 二次验证；
- PostgreSQL 保存 Authelia 状态数据；
- Redis 保存会话，后续扩展多实例也更平滑；
- 对没有登录功能或登录很弱的内网服务，加一层统一身份认证。

> 注意：本文示例以 `example.com`、`auth.example.com`、`grafana.example.com` 作为占位域名。请替换成你自己的内网域名或公网域名。Authelia 可以保护入口，但不要把 PVE、NAS、路由器管理口无脑暴露到公网。

## 1. 架构设计

整体流量链路如下：

```text
Browser
  │
  ▼
Caddy :443
  │
  ├── auth.example.com    ──► Authelia :9091
  │
  └── grafana.example.com ──► forward_auth 到 Authelia 校验
                              │
                              └── 校验通过后 reverse_proxy 到 Grafana :3000
```

Forward Auth 的核心逻辑是：用户访问受保护服务时，Caddy 先把请求交给 Authelia 的 `/api/authz/forward-auth` 端点判断。Authelia 如果认为用户没有登录，就返回重定向到登录门户；如果已登录且 ACL 允许访问，就返回 `2xx`，Caddy 再把原始请求转发给后端服务。

组件建议如下：

| 组件 | 作用 | Homelab 建议 |
|---|---|---|
| Caddy | HTTPS、反向代理、Forward Auth | 复用已有 Caddy 入口 |
| Authelia | 认证门户、ACL、TOTP | 独立 Compose 栈，监听内网 |
| PostgreSQL | 持久化 Authelia 数据 | 比 SQLite 更适合长期使用 |
| Redis | Session 存储 | 单实例即可，别暴露端口 |
| Protected Apps | Grafana、Prometheus、qBittorrent 等 | 优先保护无认证或弱认证服务 |

## 2. 准备目录和密钥

先创建目录：

```bash
mkdir -p ~/authelia/{config,secrets,postgres,redis}
cd ~/authelia
```

生成几组密钥。Authelia 里最容易出问题的就是 secret 混用、太短或换了之后导致会话失效，所以单独保存到文件：

```bash
openssl rand -hex 64 > secrets/JWT_SECRET
openssl rand -hex 64 > secrets/SESSION_SECRET
openssl rand -hex 64 > secrets/STORAGE_ENCRYPTION_KEY
openssl rand -base64 36 > secrets/POSTGRES_PASSWORD

chmod 600 secrets/*
```

如果你用 Git 管理 Compose 配置，`secrets/` 和数据库目录必须加入 `.gitignore`：

```bash
cat > .gitignore <<'EOF'
secrets/
postgres/
redis/
EOF
```

## 3. Docker Compose 部署 Authelia

创建 `docker-compose.yml`：

```yaml
services:
  authelia:
    image: authelia/authelia:latest
    container_name: authelia
    restart: unless-stopped
    depends_on:
      - postgres
      - redis
    volumes:
      - ./config:/config
      - ./secrets:/secrets:ro
    environment:
      TZ: Asia/Shanghai
      AUTHELIA_JWT_SECRET_FILE: /secrets/JWT_SECRET
      AUTHELIA_SESSION_SECRET_FILE: /secrets/SESSION_SECRET
      AUTHELIA_STORAGE_ENCRYPTION_KEY_FILE: /secrets/STORAGE_ENCRYPTION_KEY
      AUTHELIA_STORAGE_POSTGRES_PASSWORD_FILE: /secrets/POSTGRES_PASSWORD
    ports:
      # 只给内网 Caddy 访问。若 Caddy 与 Authelia 在同一 Docker 网络，可删除 ports。
      - "127.0.0.1:9091:9091"
    networks:
      - auth

  postgres:
    image: postgres:16-alpine
    container_name: authelia-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: authelia
      POSTGRES_DB: authelia
      POSTGRES_PASSWORD_FILE: /run/secrets/postgres_password
      TZ: Asia/Shanghai
    secrets:
      - postgres_password
    volumes:
      - ./postgres:/var/lib/postgresql/data
    networks:
      - auth

  redis:
    image: redis:7-alpine
    container_name: authelia-redis
    restart: unless-stopped
    command: redis-server --save 60 1 --loglevel warning
    volumes:
      - ./redis:/data
    networks:
      - auth

secrets:
  postgres_password:
    file: ./secrets/POSTGRES_PASSWORD

networks:
  auth:
    name: authelia
```

这里把 Authelia 绑定到 `127.0.0.1:9091`，原因是 Caddy 和 Authelia 如果在同一台机器，没必要把认证端口暴露到局域网。Caddy 在宿主机运行时访问 `http://127.0.0.1:9091`；如果 Caddy 也在 Docker 里，可以把二者放到同一个 Docker network 后直接访问 `http://authelia:9091`。

## 4. Authelia 主配置

创建 `config/configuration.yml`：

```yaml
server:
  address: tcp://0.0.0.0:9091
  endpoints:
    authz:
      forward-auth:
        implementation: ForwardAuth

log:
  level: info

totp:
  issuer: example.com

authentication_backend:
  file:
    path: /config/users_database.yml
    watch: true

password_policy:
  standard:
    enabled: true
    min_length: 12
    require_uppercase: true
    require_lowercase: true
    require_number: true
    require_special: true

access_control:
  default_policy: deny
  networks:
    - name: local_lan
      networks:
        - 192.168.1.0/24
        - 10.0.0.0/8
        - 172.16.0.0/12

  rules:
    # Authelia 门户本身不需要再套自己
    - domain: auth.example.com
      policy: bypass

    # 只允许内网访问的服务，例如 PVE、NAS、路由器跳板页
    - domain:
        - pve.example.com
        - nas.example.com
      networks:
        - local_lan
      policy: two_factor

    # 普通管理服务：登录 + TOTP
    - domain:
        - grafana.example.com
        - prometheus.example.com
        - qbittorrent.example.com
      policy: two_factor

    # 只读类服务可以降级为一因子，按需调整
    - domain:
        - status.example.com
      policy: one_factor

session:
  cookies:
    - domain: example.com
      authelia_url: https://auth.example.com
      default_redirection_url: https://grafana.example.com
      name: authelia_session
      same_site: lax
      inactivity: 30m
      expiration: 8h
      remember_me: 1M
  redis:
    host: redis
    port: 6379

storage:
  postgres:
    address: tcp://postgres:5432
    database: authelia
    username: authelia

notifier:
  filesystem:
    filename: /config/notification.txt
```

几个关键点：

1. `default_policy: deny`：默认拒绝，靠规则显式放行。这比默认允许安全很多。
2. `session.cookies.domain` 必须是顶级共享域，例如 `example.com`，不要写成 `auth.example.com`，否则其他子域拿不到会话。
3. `authelia_url` 是用户未登录时跳转的认证门户地址，必须是浏览器可访问的 HTTPS 地址。
4. `notifier.filesystem` 适合 Homelab 初期调试，注册 TOTP、重置密码等通知会写到文件里；正式使用可以换 SMTP。

## 5. 创建用户数据库

先生成密码哈希。不要把明文密码写进配置：

```bash
docker run --rm authelia/authelia:latest \
  authelia crypto hash generate argon2 \
  --password '请换成你的强密码'
```

输出会类似：

```text
Digest: $argon2id$v=19$m=65536,t=3,p=4$......
```

创建 `config/users_database.yml`：

```yaml
users:
  admin:
    displayname: "Homelab Admin"
    password: "$argon2id$v=19$m=65536,t=3,p=4$请替换成上一步生成的哈希"
    email: admin@example.com
    groups:
      - admins
      - homelab

  guest:
    displayname: "Guest"
    password: "$argon2id$v=19$m=65536,t=3,p=4$另一个哈希"
    email: guest@example.com
    groups:
      - guests
```

启动服务：

```bash
docker compose up -d

docker compose ps
docker logs -f authelia
```

如果日志里看到数据库迁移完成、服务监听 `:9091`，说明 Authelia 侧基本正常。再用本机 curl 验证：

```bash
curl -I http://127.0.0.1:9091/
```

正常会返回 `200` 或重定向相关响应，至少不能是连接失败。

## 6. Caddyfile：认证门户 + 受保护服务

下面是 Caddyfile 示例：

```caddyfile
{
    email admin@example.com
}

auth.example.com {
    reverse_proxy 127.0.0.1:9091
}

grafana.example.com {
    forward_auth 127.0.0.1:9091 {
        uri /api/authz/forward-auth
        copy_headers Remote-User Remote-Groups Remote-Email Remote-Name
    }

    reverse_proxy 192.168.1.20:3000
}

prometheus.example.com {
    forward_auth 127.0.0.1:9091 {
        uri /api/authz/forward-auth
        copy_headers Remote-User Remote-Groups Remote-Email Remote-Name
    }

    reverse_proxy 192.168.1.20:9090
}
```

重载 Caddy：

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl reload caddy
sudo journalctl -u caddy -f
```

如果 Caddy 也跑在 Docker 里，`127.0.0.1:9091` 指的是 Caddy 容器自己，不是宿主机。这种情况下要把 Caddy 加入 `authelia` 网络，然后改成：

```caddyfile
grafana.example.com {
    forward_auth authelia:9091 {
        uri /api/authz/forward-auth
        copy_headers Remote-User Remote-Groups Remote-Email Remote-Name
    }

    reverse_proxy grafana:3000
}
```

对应 Compose 片段：

```yaml
services:
  caddy:
    image: caddy:2
    networks:
      - authelia
      - proxy

networks:
  authelia:
    external: true
  proxy:
    external: true
```

## 7. 分组 ACL：不要所有人都是管理员

只有一个管理员账号时，规则可以简单；但一旦你想给家人开放相册、给访客开放状态页，就应该用 group 分层。

示例：

```yaml
access_control:
  default_policy: deny
  rules:
    - domain: auth.example.com
      policy: bypass

    - domain: photos.example.com
      subject:
        - "group:family"
        - "group:admins"
      policy: one_factor

    - domain: grafana.example.com
      subject:
        - "group:admins"
      policy: two_factor

    - domain: qbittorrent.example.com
      subject:
        - "user:admin"
      policy: two_factor
```

这种写法的好处是权限边界清楚：相册不等于监控，监控不等于下载器，下载器更不等于 PVE 管理入口。

修改配置后重启 Authelia：

```bash
docker compose restart authelia
docker logs --tail=100 authelia
```

## 8. 让后端服务识别用户

Caddy 的 `copy_headers` 会把 Authelia 返回的用户信息传给后端：

```caddyfile
copy_headers Remote-User Remote-Groups Remote-Email Remote-Name
```

某些应用支持「反向代理认证」或「Header Auth」，可以直接读取 `Remote-User`。例如内部工具可以按 `Remote-Groups` 做权限判断。但这里有一个安全边界：**后端服务必须只允许来自 Caddy 的流量**。

否则攻击者可以绕过 Caddy 直接访问后端端口，并伪造 `Remote-User` 头。

最简单的做法是不要暴露后端端口，只在 Docker 内网里互通：

```yaml
services:
  grafana:
    image: grafana/grafana:latest
    expose:
      - "3000"
    networks:
      - proxy
```

如果后端是局域网其他机器，至少用防火墙限制只允许 Caddy 服务器访问：

```bash
# 在后端机器上，仅允许 Caddy 服务器 192.168.1.9 访问 Grafana 3000
sudo ufw allow from 192.168.1.9 to any port 3000 proto tcp
sudo ufw deny 3000/tcp
sudo ufw status numbered
```

## 9. 排障命令

### 9.1 看 Authelia 是否正常启动

```bash
docker compose ps
docker logs --tail=200 authelia
```

重点看这几类错误：

- PostgreSQL 连接失败：检查 `POSTGRES_PASSWORD_FILE` 和 `storage.postgres.address`；
- Redis 连接失败：检查 `session.redis.host` 是否是 Compose 服务名；
- session cookie 报错：检查 `session.cookies.domain` 和 `authelia_url`；
- ACL 不生效：检查域名是否和浏览器访问域名完全一致。

### 9.2 看 Caddy 是否调用了 forward_auth

```bash
sudo journalctl -u caddy -n 100 --no-pager
```

如果访问受保护域名时完全没有 Authelia 日志，大概率是 Caddyfile 没命中当前站点块，或 DNS 指到了别的入口。

### 9.3 验证 HTTP 状态

未登录访问受保护服务，应该得到跳转：

```bash
curl -I https://grafana.example.com
```

一般会看到 `302` 或与 Authelia 登录流程相关的响应。如果直接 `200` 打开了 Grafana，说明 Forward Auth 没套上；如果是 `502`，优先检查 Caddy 到 Authelia 或 Caddy 到后端的网络连通性。

### 9.4 检查时间同步

TOTP 对时间敏感。所有机器建议启用 NTP：

```bash
timedatectl
sudo timedatectl set-ntp true
```

如果手机验证码总是不对，先看服务器和手机时间是否偏差过大。

## 10. 踩坑记录

### ❌ 1. `session.cookies.domain` 写成 `auth.example.com`

这是最常见坑。写成 `auth.example.com` 后，登录 Cookie 只对认证门户有效，访问 `grafana.example.com` 时仍然像没登录一样循环跳转。正确写法是共享父域：

```yaml
session:
  cookies:
    - domain: example.com
```

### ❌ 2. Caddy 在容器里却写 `127.0.0.1:9091`

容器里的 `127.0.0.1` 是容器自身。Caddy 容器访问 Authelia 容器，应使用同一个 Docker 网络里的服务名：

```caddyfile
forward_auth authelia:9091 {
    uri /api/authz/forward-auth
}
```

### ❌ 3. 后端端口仍然暴露在局域网

如果 `grafana:3000`、`prometheus:9090` 仍然可以被局域网直接访问，Authelia 只保护了域名入口，没有保护服务本身。后端端口应该只对 Caddy 可见。

### ❌ 4. 默认策略用 `one_factor` 或 `bypass`

`default_policy: deny` 是更安全的 Homelab 默认值。新增服务时显式写规则，虽然麻烦一点，但不会因为漏配把服务直接放出去。

### ❌ 5. 保护会回调自己的服务

某些应用有 Webhook、OAuth callback 或 API 路径，如果被二次认证拦住会异常。可以按路径做细粒度规则，例如只放行健康检查：

```yaml
access_control:
  rules:
    - domain: app.example.com
      resources:
        - "^/healthz$"
      policy: bypass

    - domain: app.example.com
      policy: two_factor
```

## 11. 备份与恢复

Authelia 的关键数据有三类：

| 数据 | 路径 | 重要性 |
|---|---|---|
| 配置 | `~/authelia/config` | 高 |
| 密钥 | `~/authelia/secrets` | 极高，丢失会影响解密和会话 |
| 数据库 | `~/authelia/postgres` | 高，包含注册状态等数据 |

简单备份命令：

```bash
cd ~
tar --xattrs --acls -czf authelia-backup-$(date +%F).tar.gz authelia/config authelia/secrets

docker exec authelia-postgres pg_dump -U authelia authelia \
  | gzip > authelia-postgres-$(date +%F).sql.gz
```

恢复 PostgreSQL 时先启动数据库，再导入：

```bash
gunzip -c authelia-postgres-2026-05-26.sql.gz \
  | docker exec -i authelia-postgres psql -U authelia authelia
```

## 12. 安全加固建议

1. **Authelia 管理面只走 HTTPS**：`auth.example.com` 不要提供明文 HTTP 入口。
2. **开启 TOTP**：管理类服务使用 `two_factor`，不要只靠密码。
3. **后端端口不裸奔**：Docker 用 `expose`，跨主机用防火墙限制来源。
4. **默认拒绝**：`default_policy: deny`，新增服务手工加规则。
5. **日志接入集中日志系统**：把 Authelia 和 Caddy 日志送到 Loki，方便审计谁访问了什么服务。
6. **不要保护所有东西**：SSH、WireGuard、PVE 管理口这类高危入口，优先保持内网/VPN 访问，不建议仅靠 Web SSO 暴露公网。

## 总结

Authelia + Caddy Forward Auth 很适合 Homelab：部署成本不高，但能把分散在各个服务里的弱认证统一收口。它不是万能安全方案，不能替代防火墙、VPN、最小暴露原则；但它能显著降低「某个面板弱口令被撞开」的风险。

我的建议是：

| 场景 | 建议策略 |
|---|---|
| Grafana、Prometheus、qBittorrent | Authelia `two_factor` |
| 相册、只读状态页 | `one_factor` 或按用户组放行 |
| PVE、NAS、路由器后台 | 优先内网/VPN，必要时 `two_factor` + IP 限制 |
| Webhook、健康检查 | 单独路径 `bypass` |
| 所有新服务 | 默认拒绝，显式加 ACL |

一句话：**HTTPS 解决加密，反向代理解决入口，Authelia 解决身份；三者组合起来，Homelab 才像一个可维护的内网平台。**
