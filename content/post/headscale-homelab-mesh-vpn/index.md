---
title: "把 Homelab 安全带出门：Headscale 自建 Tailscale 控制平面实战"
description: "用 Headscale 自建 Tailscale 兼容控制平面，覆盖 Docker Compose 部署、Caddy 反代、客户端注册、Subnet Router、ACL、MagicDNS、备份迁移与常见排障。"
date: 2026-06-01T01:22:03+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
slug: "headscale-homelab-mesh-vpn"
---

## 前言：为什么有了 WireGuard 还要折腾 Headscale？

Homelab 远程访问最常见的方案是 WireGuard：性能好、配置简单、稳定。但随着设备变多，单纯的点对点 VPN 会遇到几个现实问题：手机、笔记本、云服务器、家里的 PVE 节点、NAS、Docker 主机都要互通；有些设备在双 NAT 或校园网后面；还希望按用户、标签、网段做访问控制。这时候 Tailscale 这类基于 WireGuard 的 Mesh VPN 就很香。

Headscale 是一个开源的 Tailscale 控制平面实现。简单说：**数据面仍然走 WireGuard，控制面由你自己托管**。客户端继续使用官方 Tailscale 客户端，只是登录服务器改成自己的 Headscale。对于 Homelab 来说，它很适合做一条「只暴露一个 HTTPS 入口，却能访问内网所有服务」的安全通道。

本文搭一套可直接落地的 Headscale：

- 使用 Docker Compose 部署 Headscale；
- 通过 Caddy 提供 HTTPS 反向代理；
- 注册 Linux、Windows、手机等客户端；
- 配置一台 Subnet Router，把 `192.168.10.0/24` 内网带出门；
- 用 ACL 限制不同设备能访问的网段和端口；
- 给出备份、迁移、升级和排障命令。

> 如果你只需要极简远程接入，可以先看站内的 [WireGuard VPN 部署](/p/wireguard-homelab-vpn-guide/)；如果你已经有多台设备、多用户、多网段，Headscale 的集中管控会更省心。

---

## 1. 架构设计：控制面、数据面和 DERP

Tailscale/Headscale 的关键点是分清两条路径：

| 组件 | 作用 | 是否经过 Headscale |
|---|---|---|
| Headscale 控制面 | 设备注册、密钥分发、节点列表、ACL、MagicDNS | ✅ 是 |
| WireGuard 数据面 | 设备之间真实业务流量 | ❌ 正常不经过 |
| DERP 中继 | NAT 穿透失败时的中继通道 | ❌ 默认用 Tailscale 公共 DERP |
| Subnet Router | 把某个内网网段宣告进 Tailnet | ❌ 只转发数据流量 |

很多人误以为 Headscale 会成为所有流量的代理瓶颈。实际上只要两台设备能 UDP 打洞成功，业务流量就是点对点直连；只有打洞失败时才会走 DERP 中继。Headscale 本身主要承载控制面，请求量不大，一台 1C/512MB 的 VPS 都能跑。

推荐部署位置：

| 部署位置 | 优点 | 缺点 | 适合场景 |
|---|---|---|---|
| 公网 VPS | 客户端随时可注册，证书好签发 | 多一台云主机 | 🏆 最推荐 |
| 家中公网 IP | 低成本，完全自托管 | 需要端口转发，家宽变更风险 | 有公网 IPv4/IPv6 |
| Cloudflare Tunnel 后面 | 不开端口 | gRPC/长连接兼容性要仔细测 | 临时实验 |

本文假设 Headscale 跑在一台公网 VPS 或有公网入口的 Docker 主机上，域名为：

```text
hs.example.com
```

请替换成你自己的域名。

---

## 2. 准备目录和 Docker Compose

先创建目录：

```bash
sudo mkdir -p /opt/headscale/{config,data}
cd /opt/headscale
```

写入 `compose.yml`：

```yaml
services:
  headscale:
    image: headscale/headscale:0.26.0
    container_name: headscale
    restart: unless-stopped
    command: serve
    volumes:
      - ./config:/etc/headscale
      - ./data:/var/lib/headscale
    ports:
      # 只监听本机，交给 Caddy 反代到公网
      - "127.0.0.1:8080:8080"
    environment:
      - TZ=Asia/Shanghai
    healthcheck:
      test: ["CMD", "headscale", "version"]
      interval: 30s
      timeout: 5s
      retries: 3
```

这里没有把 `8080` 直接暴露到公网，而是只绑定 `127.0.0.1`。这样即使云防火墙误放行 8080，外部也无法绕过反代直接访问 Headscale。

启动前先准备配置文件。

---

## 3. Headscale 配置文件

创建 `/opt/headscale/config/config.yaml`：

```yaml
server_url: https://hs.example.com
listen_addr: 0.0.0.0:8080
metrics_listen_addr: 127.0.0.1:9090

# Headscale 的持久化数据库。小规模 Homelab 用 SQLite 足够，备份也简单。
database:
  type: sqlite
  sqlite:
    path: /var/lib/headscale/db.sqlite

# 私有密钥路径，首次启动自动生成。这个文件和数据库都必须备份。
private_key_path: /var/lib/headscale/private.key
noise:
  private_key_path: /var/lib/headscale/noise_private.key

# IP 池：给 Tailnet 设备分配的虚拟地址，避免和家庭内网冲突。
prefixes:
  v4: 100.64.0.0/10
  v6: fd7a:115c:a1e0::/48
  allocation: sequential

# MagicDNS，方便用主机名访问设备。
dns:
  magic_dns: true
  base_domain: tailnet.example.com
  nameservers:
    global:
      - 223.5.5.5
      - 119.29.29.29
  search_domains: []
  extra_records: []

# ACL 策略文件。生产环境建议显式限制，不要长期全放通。
policy:
  mode: file
  path: /etc/headscale/acl.hujson

# 默认使用 Tailscale 公共 DERP，国内网络可后续自建 DERP 或优选区域。
derp:
  server:
    enabled: false
  urls:
    - https://controlplane.tailscale.com/derpmap/default
  auto_update_enabled: true
  update_frequency: 24h

log:
  level: info
```

几个容易踩坑的点：

1. `server_url` 必须是客户端能访问的最终 HTTPS 地址，不要写容器内地址；
2. `prefixes.v4` 推荐保留 `100.64.0.0/10`，不要和家里 LAN、Docker 网段、K8s Pod CIDR 冲突；
3. `private.key`、`noise_private.key` 和 `db.sqlite` 是核心资产，迁移时必须一起带走；
4. `policy.path` 开启后，ACL 写错会影响客户端访问，改完要看日志确认加载成功。

---

## 4. Caddy 反向代理与 HTTPS

如果你已经有 Caddy 作为 Homelab 入口，可以加一段：

```caddyfile
hs.example.com {
    encode zstd gzip

    reverse_proxy 127.0.0.1:8080
}
```

然后重载：

```bash
sudo caddy fmt --overwrite /etc/caddy/Caddyfile
sudo systemctl reload caddy
```

如果 Caddy 也用 Docker，可以放到同一台主机上，核心原则仍然是：公网只开放 `80/443`，Headscale 的 `8080` 只允许本机或 Docker 内网访问。

用 curl 验证：

```bash
curl -I https://hs.example.com/health
```

正常应返回 `200 OK`。如果返回 502，先看 Caddy 是否能连到 `127.0.0.1:8080`；如果证书错误，检查域名解析和 80/443 端口是否放行。

---

## 5. 初始化与创建用户

启动 Headscale：

```bash
cd /opt/headscale
docker compose up -d
docker logs -f headscale
```

查看版本：

```bash
docker exec headscale headscale version
```

创建一个用户：

```bash
docker exec headscale headscale users create homelab
```

生成一次性预授权 Key，给 Linux 服务器批量接入很方便：

```bash
docker exec headscale headscale preauthkeys create \
  --user homelab \
  --expiration 24h \
  --reusable=false
```

会输出类似：

```text
4c3f0d8a0bxxxxxxxxxxxxxxxxxxxxxxxx
```

这个 key 只用于注册设备，不要写进公开仓库。

---

## 6. 客户端接入

### 6.1 Linux 节点

在 PVE、NAS、Docker 主机或普通 Linux 服务器上安装 Tailscale 客户端：

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

使用 Headscale 登录：

```bash
sudo tailscale up \
  --login-server=https://hs.example.com \
  --authkey=你的预授权KEY \
  --hostname=pve1
```

查看状态：

```bash
tailscale status
tailscale ip -4
```

回到 Headscale 服务端查看节点：

```bash
docker exec headscale headscale nodes list
```

### 6.2 Windows/macOS/手机

桌面和移动端仍然安装官方 Tailscale 客户端，但登录方式略有不同：

```bash
tailscale up --login-server=https://hs.example.com
```

客户端会打开浏览器，提示你到 Headscale 注册。服务端日志或命令行会看到注册命令，执行后即可加入。例如：

```bash
docker exec headscale headscale nodes register \
  --user homelab \
  --key nodekey:xxxxxxxxxxxxxxxx
```

如果移动端 App 不方便指定 login server，可以先用桌面端验证链路，再考虑使用 Headscale 文档中对应平台的配置方式。生产环境建议优先把长期在线的服务器通过 `authkey` 接入，个人终端再手动注册。

---

## 7. 配置 Subnet Router：带出整个内网

假设家里主 LAN 是 `192.168.10.0/24`，PVE 节点 `pve1` 同时接入了 Headscale 和家庭内网。我们让 `pve1` 做 Subnet Router：

```bash
sudo tailscale up \
  --login-server=https://hs.example.com \
  --advertise-routes=192.168.10.0/24 \
  --accept-dns=false \
  --hostname=pve1-router
```

打开 Linux 转发：

```bash
cat <<'EOF' | sudo tee /etc/sysctl.d/99-tailscale-router.conf
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
EOF
sudo sysctl --system
```

在 Headscale 服务端查看待批准路由：

```bash
docker exec headscale headscale nodes list
docker exec headscale headscale routes list
```

启用对应路由。不同版本命令输出略有差异，核心是找到 route ID 后 enable：

```bash
docker exec headscale headscale routes enable -r <ROUTE_ID>
```

其他客户端需要接受路由：

```bash
sudo tailscale set --accept-routes=true
```

验证：

```bash
# 在外部笔记本上测试访问内网 PVE 或 NAS
ping 192.168.10.2
curl -k https://192.168.10.2:8006
```

如果 ping 不通，按这个顺序查：

```bash
# 1. 客户端是否拿到了路由
tailscale status
tailscale netcheck
ip route | grep 192.168.10

# 2. Subnet Router 是否开启转发
sysctl net.ipv4.ip_forward

# 3. Headscale 是否启用了路由
docker exec headscale headscale routes list

# 4. 内网目标机器的默认网关/防火墙是否允许来自 100.64.0.0/10 的访问
sudo iptables -S
sudo nft list ruleset
```

> 实战经验：很多「Subnet Router 不通」不是 Headscale 的锅，而是内网目标机器的防火墙拒绝了 `100.64.0.0/10`，或者路由器不知道如何回包。Tailscale 默认会在 Subnet Router 上做 SNAT，通常能避免回程路由问题；如果你禁用了 SNAT，就必须在家庭网关上加静态路由。

---

## 8. ACL：不要让所有设备访问所有端口

默认全放通很方便，但不适合长期使用。下面是一份适合 Homelab 的 `acl.hujson` 示例：

```json
{
  "groups": {
    "group:admins": ["homelab"]
  },

  "tagOwners": {
    "tag:server": ["group:admins"],
    "tag:router": ["group:admins"],
    "tag:mobile": ["group:admins"]
  },

  "acls": [
    // 管理员设备可以访问所有 Tailnet 节点
    {
      "action": "accept",
      "src": ["group:admins"],
      "dst": ["*:*"]
    },

    // 普通移动设备只允许访问内网 Web 服务和 SSH 跳板
    {
      "action": "accept",
      "src": ["tag:mobile"],
      "dst": [
        "192.168.10.0/24:80,443,8006,8123,9000",
        "tag:server:22"
      ]
    }
  ],

  "ssh": [
    {
      "action": "accept",
      "src": ["group:admins"],
      "dst": ["tag:server"],
      "users": ["root", "ubuntu"]
    }
  ],

  "autoApprovers": {
    "routes": {
      "192.168.10.0/24": ["tag:router"]
    }
  }
}
```

保存到 `/opt/headscale/config/acl.hujson` 后重启：

```bash
docker compose restart headscale
docker logs --tail=100 headscale
```

给节点打标签通常在注册或重新登录时指定，例如：

```bash
sudo tailscale up \
  --login-server=https://hs.example.com \
  --advertise-tags=tag:server \
  --hostname=docker01
```

ACL 的维护建议：

| 策略 | 建议 |
|---|---|
| 管理员设备 | 可以全访问，但设备数量越少越好 |
| 手机/平板 | 只放行 Web 服务端口，不给 SSH |
| Subnet Router | 单独打 `tag:router`，配合 `autoApprovers` |
| 临时设备 | 用短过期 `preauthkey`，不用后删除节点 |

---

## 9. MagicDNS：用名字访问 Homelab 服务

开启 `magic_dns` 后，Tailnet 内可以通过节点名访问设备，例如：

```bash
ssh root@pve1
curl http://docker01:9000
```

如果想给某些内网服务配置固定记录，可以使用 `extra_records`：

```yaml
dns:
  magic_dns: true
  base_domain: tailnet.example.com
  nameservers:
    global:
      - 223.5.5.5
  extra_records:
    - name: pve.tailnet.example.com
      type: A
      value: 192.168.10.2
    - name: nas.tailnet.example.com
      type: A
      value: 192.168.10.10
```

但我更推荐两种方式：

1. **节点用 MagicDNS 名称**，比如 `pve1`、`docker01`；
2. **业务服务继续走内网反代域名**，比如 `https://grafana.home.example.com`，DNS 指向内网反代 IP，再由 Subnet Router 打通访问。

这样不会把 Headscale 的 DNS 配置写得过重，后续迁移也简单。

---

## 10. 备份、升级与迁移

Headscale 的关键数据都在 `/opt/headscale/data` 和 `/opt/headscale/config`。最小备份脚本如下：

```bash
#!/usr/bin/env bash
set -euo pipefail

BACKUP_DIR="/backup/headscale"
TS="$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"

cd /opt
tar -czf "$BACKUP_DIR/headscale-$TS.tar.gz" headscale/config headscale/data

# 保留最近 14 份
ls -1t "$BACKUP_DIR"/headscale-*.tar.gz | tail -n +15 | xargs -r rm -f
```

配合 crontab：

```bash
sudo crontab -e
```

写入：

```cron
15 3 * * * /usr/local/sbin/backup-headscale.sh >/var/log/backup-headscale.log 2>&1
```

升级前先备份，再拉镜像：

```bash
cd /opt/headscale
./backup-headscale.sh

docker compose pull
docker compose up -d
docker logs --tail=100 headscale
```

迁移到新机器时：

```bash
# 新机器
sudo mkdir -p /opt/headscale
sudo tar -xzf headscale-20260601-030000.tar.gz -C /opt
cd /opt/headscale
docker compose up -d
```

迁移后要确认：

```bash
docker exec headscale headscale users list
docker exec headscale headscale nodes list
curl -I https://hs.example.com/health
```

只要域名、数据库和私钥没变，客户端通常不需要重新注册。

---

## 11. 常见踩坑记录

### ❌ 1. `server_url` 写成了 `http://127.0.0.1:8080`

客户端注册时会拿这个地址作为控制面入口。如果写成本机地址，只有容器自己能访问，外部客户端全部失败。必须写公网可访问的 HTTPS 地址：

```yaml
server_url: https://hs.example.com
```

### ❌ 2. 反代正常，但客户端一直连不上

先看基础健康检查：

```bash
curl -v https://hs.example.com/health
docker logs --tail=200 headscale
```

再检查 Caddy 是否把请求转发到 Headscale：

```bash
sudo journalctl -u caddy -n 100 --no-pager
ss -lntp | grep 8080
```

如果 `8080` 没监听，多半是容器没起来或配置文件解析失败。

### ❌ 3. Subnet Router 已批准，但访问内网服务超时

重点查三件事：

```bash
# 客户端是否接受路由
sudo tailscale set --accept-routes=true
ip route | grep 192.168.10

# Router 是否开启转发
sysctl net.ipv4.ip_forward

# 目标服务是否只监听 localhost
ss -lntp | grep ':8006\|:8123\|:443'
```

有些服务只绑定 `127.0.0.1`，你从 VPN 当然访问不到。需要改成监听 `0.0.0.0` 或内网 IP。

### ❌ 4. Docker 网段和家里 LAN 冲突

如果 Docker 默认 `172.17.0.0/16`、K8s Pod CIDR、家里路由器网段、Tailnet 路由互相重叠，排障会非常痛苦。建议统一规划：

```text
家庭 LAN:        192.168.10.0/24
服务器 VLAN:     192.168.20.0/24
Docker Compose:  172.20.0.0/16
K8s Pod CIDR:    10.42.0.0/16
K8s SVC CIDR:    10.43.0.0/16
Tailnet:         100.64.0.0/10
```

Docker 默认地址池可以在 `/etc/docker/daemon.json` 中调整：

```json
{
  "default-address-pools": [
    {
      "base": "172.20.0.0/14",
      "size": 24
    }
  ]
}
```

重启 Docker：

```bash
sudo systemctl restart docker
```

---

## 12. 安全加固清单

| 项目 | 命令/配置 | 建议 |
|---|---|---|
| 公网端口 | `ss -lntp`、云防火墙 | 只开放 80/443/SSH |
| Headscale 端口 | Compose 绑定 `127.0.0.1:8080` | 不直接暴露公网 |
| 预授权 Key | `--expiration 24h --reusable=false` | 临时使用，过期失效 |
| ACL | `policy.path` | 默认拒绝，按需放行 |
| 备份 | tar config + data | 至少每日一次 |
| 日志 | `docker logs headscale` | 注册失败先看日志 |
| 升级 | 固定镜像 tag | 不建议长期用 `latest` |

如果你还把 Headscale 暴露在公网 VPS 上，SSH 也要做基础加固：禁用密码登录、改用密钥、限制 sudo 用户，并配合防火墙只放行必要端口。

---

## 总结：Headscale 适合谁？

Headscale 不是 WireGuard 的替代品，而是把 WireGuard 的设备管理、NAT 穿透、ACL 和 DNS 做成了一个自托管控制面。对于 Homelab，它最有价值的地方是：**不用把 PVE、NAS、Home Assistant、Grafana 全部暴露到公网，只需要维护一个 HTTPS 控制面，就能在外面安全访问内网资源**。

我的建议：

| 场景 | 推荐方案 |
|---|---|
| 只有 1-2 台设备远程访问 | WireGuard 更简单 |
| 多设备、多系统、经常换网络 | Headscale 更省心 🏆 |
| 需要访问整个家庭网段 | Headscale + Subnet Router |
| 多用户权限隔离 | Headscale + ACL |
| 极致可控和审计 | 自建 Headscale + 自建 DERP |

最后一句话：先把 Headscale 控制面跑稳，再逐步加 Subnet Router、ACL 和 DNS。不要一上来就把所有内网网段全放通，否则你只是把「内网裸奔」换成了「VPN 裸奔」。
