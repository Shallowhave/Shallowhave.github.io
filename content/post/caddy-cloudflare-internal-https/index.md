---
title: "Caddy + Cloudflare DNS Challenge：给 Homelab 内网服务自动签发 HTTPS 证书"
description: "用 Caddy 和 Cloudflare DNS Challenge 为不暴露到公网的 Homelab 服务签发 Let's Encrypt 泛域名证书，包含 Docker Compose、Caddyfile、API Token 权限、反代配置和常见踩坑。"
date: 2026-05-23T09:23:23+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
slug: "caddy-cloudflare-internal-https"
---

## 前言

Homelab 里最烦人的事情之一，是每个服务都有自己的端口和自签证书：PVE 是 `https://pve:8006`，NAS 是 `https://nas:5001`，Grafana 是 `http://grafana:3000`，Home Assistant 又是另一个端口。浏览器天天提示“不安全”，移动端 App 还经常因为证书不受信任拒绝连接。

最直接的方案是做一个统一入口：**反向代理 + 正式 HTTPS 证书**。但很多 Homelab 服务只想在内网访问，不想把 80/443 暴露到公网。这时 HTTP-01 验证就不好用了，推荐改用 **DNS-01 Challenge**：由 Caddy 调用 Cloudflare API 写入 `_acme-challenge` TXT 记录，Let's Encrypt 验证 DNS 记录后签发证书。服务本身不需要公网开放，甚至可以只监听在内网。

本文用 Caddy 作为反向代理，Cloudflare 托管 DNS，目标是实现：

- `https://pve.lab.example.com` → PVE 管理页面
- `https://grafana.lab.example.com` → Grafana
- `https://ha.lab.example.com` → Home Assistant
- 自动申请、自动续期、统一日志、统一入口

> 下文把域名写成 `example.com`，实际操作时替换成你自己的域名。

## 1. 架构设计

推荐把 Caddy 放在一台稳定在线的 Linux/Docker 主机上，常见选择是：独立 LXC、Docker VM、小主机、NAS Docker。网络上只要内网客户端能访问它的 443 端口即可。

| 组件 | 作用 | 推荐配置 |
|---|---|---|
| Caddy | 反向代理、证书申请、自动续期 | Docker 部署，持久化 `/data` |
| Cloudflare DNS | DNS-01 验证 TXT 记录 | 只给最小 API Token 权限 |
| 内网 DNS | 让内网域名解析到 Caddy | AdGuard Home / 路由器 / Pi-hole |
| 后端服务 | PVE、Grafana、HA、NAS 等 | 尽量只监听内网 |

核心链路如下：

```text
浏览器 -> https://pve.lab.example.com:443 -> Caddy -> https://192.168.1.10:8006
                                         -> Caddy 自动用 Cloudflare DNS API 续证书
```

这里有两个关键点：

1. **证书签发靠公网 DNS，不靠公网端口**：DNS-01 只需要 Cloudflare 能写 TXT 记录，不要求你的家庭宽带有公网 IP。
2. **内网访问靠本地 DNS 覆盖**：`*.lab.example.com` 在内网解析到 Caddy 的内网 IP，比如 `192.168.1.9`。

## 2. 创建 Cloudflare API Token

不要直接使用 Global API Key。它权限太大，泄露后等于整个 Cloudflare 账号裸奔。创建一个最小权限 Token 即可：

Cloudflare 控制台路径：

```text
My Profile -> API Tokens -> Create Token -> Custom token
```

权限建议：

```text
Permissions:
  Zone / DNS / Edit
  Zone / Zone / Read

Zone Resources:
  Include / Specific zone / example.com
```

创建后得到类似下面的 Token：

```bash
CLOUDFLARE_API_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

这个 Token 只需要给 Caddy 使用，建议放进 `.env`，不要写进 `compose.yaml` 或 Git 仓库。

## 3. 构建带 Cloudflare 插件的 Caddy 镜像

官方 Caddy 镜像默认不带 `dns.providers.cloudflare` 插件，需要用 `xcaddy` 构建一个自定义镜像。目录结构建议如下：

```bash
mkdir -p /opt/caddy/{data,config,logs}
cd /opt/caddy
```

创建 `Dockerfile`：

```dockerfile
FROM caddy:2-builder AS builder

RUN xcaddy build \
    --with github.com/caddy-dns/cloudflare

FROM caddy:2
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

创建 `.env`：

```bash
cat > .env <<'EOF'
CLOUDFLARE_API_TOKEN=替换成你的 Cloudflare API Token
TZ=Asia/Shanghai
EOF
chmod 600 .env
```

创建 `compose.yaml`：

```yaml
name: homelab-caddy

services:
  caddy:
    build: .
    container_name: caddy
    restart: unless-stopped
    env_file:
      - .env
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"   # HTTP/3，可选
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./data:/data
      - ./config:/config
      - ./logs:/var/log/caddy
    networks:
      - proxy

networks:
  proxy:
    name: proxy
```

注意 `/data` 必须持久化，Caddy 的账户密钥和证书都在里面。如果这个目录丢了，Caddy 会重新申请证书；频繁重建可能触发 Let's Encrypt 速率限制。

## 4. 编写 Caddyfile

先写一个可复用的 Cloudflare DNS Challenge 片段：

```caddyfile
{
    email admin@example.com
}

(tls_cloudflare) {
    tls {
        dns cloudflare {env.CLOUDFLARE_API_TOKEN}
        resolvers 1.1.1.1 8.8.8.8
    }
}

(common_headers) {
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "SAMEORIGIN"
        Referrer-Policy "strict-origin-when-cross-origin"
    }
}
```

然后为不同服务写反代。PVE 比较特殊：后端是 HTTPS，证书通常是自签名，需要跳过后端证书校验。

```caddyfile
pve.lab.example.com {
    import tls_cloudflare
    import common_headers

    reverse_proxy https://192.168.1.10:8006 {
        transport http {
            tls_insecure_skip_verify
        }
    }

    log {
        output file /var/log/caddy/pve.log
        format json
    }
}
```

Grafana 这类普通 HTTP 服务更简单：

```caddyfile
grafana.lab.example.com {
    import tls_cloudflare
    import common_headers

    reverse_proxy http://192.168.1.20:3000

    log {
        output file /var/log/caddy/grafana.log
        format json
    }
}
```

Home Assistant 如果通过反代访问，需要在 `configuration.yaml` 中信任代理 IP：

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 192.168.1.9  # Caddy 主机 IP
```

对应 Caddy 配置：

```caddyfile
ha.lab.example.com {
    import tls_cloudflare
    import common_headers

    reverse_proxy http://192.168.1.30:8123
}
```

完整配置写完后，先格式化检查：

```bash
docker compose build
docker compose run --rm caddy caddy fmt --overwrite /etc/caddy/Caddyfile
docker compose run --rm caddy caddy validate --config /etc/caddy/Caddyfile
```

如果 `validate` 通过，再启动：

```bash
docker compose up -d
docker compose logs -f caddy
```

看到类似 `certificate obtained successfully` 或 `serving initial configuration`，说明 Caddy 已经开始工作。

## 5. 配置内网 DNS 解析

证书签发完成后，还需要让内网客户端访问 `*.lab.example.com` 时命中 Caddy。最简单做法是在 AdGuard Home / Pi-hole 里加 DNS Rewrite：

```text
*.lab.example.com -> 192.168.1.9
```

如果你的 DNS 不支持通配符，就逐条添加：

```text
pve.lab.example.com     192.168.1.9
grafana.lab.example.com 192.168.1.9
ha.lab.example.com      192.168.1.9
```

Linux 客户端可以用下面命令验证：

```bash
dig pve.lab.example.com +short
curl -I https://pve.lab.example.com
```

预期结果：

```text
192.168.1.9
HTTP/2 200
server: Caddy
```

如果 `dig` 解析到了公网 Cloudflare IP，而不是你的 Caddy 内网 IP，说明客户端没有使用你的内网 DNS，先检查 DHCP 下发的 DNS 地址。

## 6. 证书续期和备份

Caddy 会自动续期证书，不需要 cron。你真正需要备份的是 `/opt/caddy/data` 和 `Caddyfile`。

推荐备份命令：

```bash
tar -czf /backup/caddy-$(date +%F).tar.gz \
  /opt/caddy/Caddyfile \
  /opt/caddy/compose.yaml \
  /opt/caddy/data \
  /opt/caddy/config
```

也可以用 Restic：

```bash
restic -r s3:https://minio.example.com/backup/caddy backup /opt/caddy \
  --exclude '/opt/caddy/logs'
```

查看证书和自动化状态：

```bash
# 查看 Caddy 当前配置适配结果
docker exec caddy caddy adapt --config /etc/caddy/Caddyfile --pretty

# 查看最近的证书申请/续期日志
docker compose logs caddy | grep -Ei 'certificate|renew|acme|challenge'

# 查看证书有效期
openssl s_client -connect pve.lab.example.com:443 -servername pve.lab.example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

## 7. 常见踩坑

### ❌ 1. Cloudflare Token 权限太小或选错 Zone

日志通常会出现 `Error presenting token`、`Invalid request headers` 或 `Actor requires permission`。重点检查：

```text
Zone / DNS / Edit
Zone / Zone / Read
Zone Resources 必须包含你的域名
```

如果你有多个域名，确认 Token 绑定的是 `example.com`，不是别的 Zone。

### ❌ 2. Caddy 容器里读不到环境变量

如果 Caddyfile 写了：

```caddyfile
dns cloudflare {env.CLOUDFLARE_API_TOKEN}
```

但日志提示 token 为空，检查：

```bash
docker compose config | grep CLOUDFLARE -n
docker exec caddy printenv | grep CLOUDFLARE
```

`.env` 默认只用于 Compose 变量替换，不一定自动注入容器；本文使用 `env_file` 是为了确保变量进入容器环境。

### ❌ 3. 后端 HTTPS 自签证书导致 502

PVE、部分 NAS、ESXi 这类服务经常用自签证书。Caddy 默认会校验证书，失败就返回 502。解决方式是在对应 `reverse_proxy` 上加：

```caddyfile
transport http {
    tls_insecure_skip_verify
}
```

这只影响 Caddy 到后端的内网连接，不影响浏览器到 Caddy 的公网可信证书。但如果你的后端能部署内网 CA，更推荐导入 CA 而不是跳过校验。

### ❌ 4. WebSocket 服务异常

Caddy 的 `reverse_proxy` 默认支持 WebSocket，通常不需要手动设置 `Upgrade` 头。如果 WebSocket 仍然异常，多半是后端应用需要知道真实域名或 HTTPS 协议，例如 Grafana：

```ini
[server]
domain = grafana.lab.example.com
root_url = https://grafana.lab.example.com/
serve_from_sub_path = false
```

Home Assistant 则重点检查 `trusted_proxies`。

### ❌ 5. 证书申请成功，但浏览器还是访问旧证书

常见原因是访问没有命中 Caddy，而是直连后端。用下面命令确认：

```bash
curl -vk https://pve.lab.example.com 2>&1 | grep -E 'subject:|issuer:|server:'
```

如果 `server` 不是 Caddy，或者证书 issuer 不是 Let's Encrypt，说明 DNS 或端口转发走错了。

## 8. 安全建议

1. **Caddy 主机只开放必要端口**：内网场景只需要 443，80 可用于跳转或关闭。
2. **管理后台加二次认证**：PVE、NAS、Grafana 都建议开启 2FA。
3. **不要把所有服务暴露到公网**：DNS-01 能签证书，不代表必须公网开放。
4. **API Token 定期轮换**：Cloudflare Token 泄露后可被用来改 DNS 记录。
5. **反代日志要轮转**：否则 JSON 日志也会慢慢吃掉磁盘。

一个简单的 logrotate 配置：

```conf
/opt/caddy/logs/*.log {
    daily
    rotate 14
    compress
    missingok
    notifempty
    copytruncate
}
```

## 总结

如果你的 Homelab 服务主要在内网使用，我更推荐 **Caddy + Cloudflare DNS-01**，而不是把 80/443 暴露出去做 HTTP 验证。它的优势很明显：

| 方案 | 是否需要公网端口 | 自动续期 | 配置复杂度 | 适合内网服务 |
|---|---:|---:|---:|---:|
| 自签证书 | 否 | ❌ | 低 | 勉强可用 |
| Nginx + acme.sh | 否 | ✅ | 中高 | ✅ |
| Traefik + DNS Challenge | 否 | ✅ | 中 | ✅ |
| Caddy + Cloudflare DNS Challenge | 否 | ✅ | 低 | 🏆 |

一句话：**把 Caddy 当成 Homelab 的 HTTPS 入口，用 DNS-01 解决证书，用内网 DNS 解决访问路径**。这样既不牺牲安全性，也能让 PVE、Grafana、Home Assistant 这些服务拥有和公网网站一样干净的 HTTPS 体验。
