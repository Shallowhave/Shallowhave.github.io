---
title: "把 DNS 掌握在自己手里：AdGuard Home + Unbound 打造 Homelab 去广告与本地域名解析"
description: "在 Homelab 中用 Docker Compose 部署 AdGuard Home 与 Unbound，构建内网 DNS 去广告、DNSSEC 递归解析、本地域名与反向代理联动方案，包含端口冲突、DHCP 下发、缓存调优和故障排查。"
date: 2026-05-18T01:03:26+08:00
image:
math:
license:
hidden: false
comments: true
draft: false
slug: "adguard-unbound-homelab-dns"
---

## 前言：Homelab 的第一块基石不是虚拟化，而是 DNS

很多人搭 Homelab 会先折腾 Proxmox VE、NAS、Docker、GPU 直通，最后才发现一个更基础的问题：服务越多，入口越乱。今天要记 `192.168.10.20` 的 `3000` 端口，明天要记 `pve.lan` 的 `8006` 端口，后天又多一个 `grafana.home.arpa`。如果 DNS 还完全依赖路由器和运营商，内网服务发现、广告拦截、故障排查都会变得很被动。

这篇文章讲一套 Homelab 很值得落地的 DNS 方案：**AdGuard Home 做内网 DNS 网关和广告拦截，Unbound 做本地递归解析器**。它不是单纯“屏蔽广告”的小工具，而是可以把本地域名、上游解析、缓存、DNSSEC、客户端统计、反向代理入口统一管理起来。

目标效果：

- 全家设备自动使用内网 DNS；
- 广告、追踪域名在 DNS 层直接拦截；
- `*.home.arpa` 指向内网服务，不再背 IP 和端口；
- 外部域名由 Unbound 递归解析，减少对运营商 DNS 的依赖；
- 出问题时能用命令快速定位，而不是盲猜路由器。

> 参考：AdGuard Home 官方 Docker 镜像文档说明需要持久化 `/opt/adguardhome/work` 和 `/opt/adguardhome/conf`；NLnet Labs 的 Unbound 文档将 Unbound 定义为 caching resolver，并使用 root hints 递归查询根域名服务器。

## 架构设计

推荐把 DNS 服务放在一台常开的 Docker 主机或轻量 VM 上。不要放在经常关机的测试机里，否则全家网络都会被你“实验”带崩。

| 组件 | 地址示例 | 作用 | 备注 |
| --- | --- | --- | --- |
| AdGuard Home | `192.168.10.53:53` | 内网 DNS 入口、广告拦截、客户端统计 | DHCP 下发给所有设备 |
| Unbound | `172.30.53.3:5335` | 递归解析、缓存、DNSSEC | 只给 AdGuard 调用 |
| 反向代理 | `192.168.10.9` | Caddy / Traefik / Nginx | `*.home.arpa` 统一指向它 |
| 路由器 DHCP | `192.168.10.1` | 分配 IP 和 DNS | DNS 指向 AdGuard |

数据流很简单：

```text
Client -> AdGuard Home:53 -> 命中拦截/重写/缓存？
                           -> 是：直接返回
                           -> 否：转发给 Unbound:5335 -> 递归查询公网权威 DNS
```

为什么不让 AdGuard 直接用 `223.5.5.5`、`1.1.1.1`？可以，但 Homelab 更推荐 Unbound：第一，它有本地缓存；第二，它能做 DNSSEC 验证；第三，排障时你清楚自己依赖的是哪一层。

## 一、准备目录与 Docker Compose

先在 Docker 主机准备目录：

```bash
sudo mkdir -p /opt/dns-stack/{adguard-work,adguard-conf,unbound}
cd /opt/dns-stack
```

如果宿主机已经运行 `systemd-resolved`，它可能占用 `127.0.0.53:53`。先检查端口：

```bash
sudo ss -lntup | grep ':53 ' || true
resolvectl status | sed -n '1,80p'
```

如果你要让 AdGuard 直接绑定宿主机 `53/tcp` 和 `53/udp`，宿主机上就不能再有其它 DNS 服务抢占 `0.0.0.0:53`。Ubuntu/Debian 常见处理方式是把 `systemd-resolved` 的 stub listener 关掉：

```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
cat <<'EOF' | sudo tee /etc/systemd/resolved.conf.d/no-stub.conf
[Resolve]
DNSStubListener=no
EOF
sudo systemctl restart systemd-resolved
sudo ss -lntup | grep ':53 ' || true
```

> 注意：这是宿主机级别改动。生产环境建议先开一个 SSH 会话保持在线，避免 DNS 改错后自己把远程连接搞断。

创建 `docker-compose.yml`：

```yaml
services:
  adguard:
    image: adguard/adguardhome:latest
    container_name: adguardhome
    restart: unless-stopped
    depends_on:
      - unbound
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "3000:3000/tcp"   # 首次初始化向导
      - "8080:80/tcp"     # 初始化后 Web UI，可按需改成反代
    volumes:
      - ./adguard-work:/opt/adguardhome/work
      - ./adguard-conf:/opt/adguardhome/conf
    networks:
      dns_net:
        ipv4_address: 172.30.53.2

  unbound:
    image: mvance/unbound:latest
    container_name: unbound
    restart: unless-stopped
    volumes:
      - ./unbound/unbound.conf:/opt/unbound/etc/unbound/unbound.conf:ro
    networks:
      dns_net:
        ipv4_address: 172.30.53.3

networks:
  dns_net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.30.53.0/24
```

这里把 Unbound 固定在 Docker 内网 `172.30.53.3`，后面 AdGuard 的上游 DNS 就填 `172.30.53.3:5335`。不把 Unbound 暴露到宿主机端口，是为了减少攻击面，也避免家庭网络里其它设备绕过 AdGuard 直接查询。

## 二、配置 Unbound：递归、缓存、内网保护

创建 `/opt/dns-stack/unbound/unbound.conf`：

```ini
server:
    interface: 0.0.0.0
    port: 5335
    do-ip4: yes
    do-ip6: no
    do-udp: yes
    do-tcp: yes

    access-control: 172.30.53.0/24 allow

    num-threads: 2
    msg-cache-size: 64m
    rrset-cache-size: 128m
    cache-min-ttl: 60
    cache-max-ttl: 86400
    prefetch: yes
    serve-expired: yes

    hide-identity: yes
    hide-version: yes
    harden-glue: yes
    harden-dnssec-stripped: yes
    qname-minimisation: yes

    private-address: 10.0.0.0/8
    private-address: 172.16.0.0/12
    private-address: 192.168.0.0/16
    private-address: fd00::/8

    verbosity: 1
```

启动服务：

```bash
cd /opt/dns-stack
docker compose up -d
docker compose logs -f --tail=80 unbound
```

验证 Unbound：

```bash
docker exec -it unbound drill @127.0.0.1 -p 5335 example.com
# 如果镜像里没有 drill，可改用 nslookup：
docker exec -it unbound nslookup example.com 127.0.0.1
```

如果这里都解析失败，不要急着改 AdGuard，先看 Unbound 日志和 Docker 网络。DNS 链路排障一定要从下游往上游一层层验证。

## 三、初始化 AdGuard Home

浏览器打开 `192.168.10.53` 的 `3000` 端口进行初始化。初始化时注意：

1. Web 管理界面监听可以选 `0.0.0.0:80`，因为 Compose 映射到了宿主机 `8080:80`，外部访问就是 `192.168.10.53` 的 `8080` 端口；
2. DNS 服务监听 `0.0.0.0:53`；
3. 设置强密码，别用默认弱口令；
4. 初始化完成后，首次向导端口 `3000` 基本不再需要，可以从 Compose 里删掉或用防火墙限制。

进入后台后，在 **Settings -> DNS settings -> Upstream DNS servers** 填：

```text
172.30.53.3:5335
```

然后开启几个关键选项：

| 选项 | 建议 | 原因 |
| --- | --- | --- |
| Cache size | `4194304` 或更高 | 家庭网络足够用，减少重复查询 |
| Optimistic caching | 开启 | TTL 过期时先返回旧记录再刷新，体感更稳 |
| DNSSEC | 视情况开启 | 如果由 Unbound 验证，可在 AdGuard 侧保持默认 |
| Query logs | 保留 24h 到 7d | 太久会占磁盘，也会留下隐私数据 |
| Statistics | 保留 7d 到 30d | 方便观察客户端和拦截率 |

## 四、本地域名：让服务入口变成人能记住的名字

建议使用 `home.arpa` 作为家庭内网域名后缀。它是 RFC 8375 为家庭网络保留的域名，比随手写 `.lan`、`.local` 更不容易和 mDNS 或真实顶级域冲突。

在 AdGuard Home 的 **Filters -> DNS rewrites** 添加：

| 域名 | 指向 | 用途 |
| --- | --- | --- |
| `pve.home.arpa` | `192.168.10.10` | Proxmox VE |
| `nas.home.arpa` | `192.168.10.20` | NAS |
| `grafana.home.arpa` | `192.168.10.30` | Grafana |
| `*.home.arpa` | `192.168.10.9` | 统一指向反向代理 |

如果你用 Caddy，可以这样接：

```caddyfile
grafana.home.arpa {
    reverse_proxy 192.168.10.30:3000
    tls internal
}

ha.home.arpa {
    reverse_proxy 192.168.10.40:8123
    tls internal
}
```

如果你暂时不想折腾内网 CA，也可以先用 HTTP：

```caddyfile
grafana.home.arpa:80 {
    reverse_proxy 192.168.10.30:3000
}
```

客户端验证：

```bash
nslookup grafana.home.arpa 192.168.10.53
curl -I grafana.home.arpa
```

这一步做完，Homelab 的体验会明显提升：服务迁移时只改 DNS rewrite 或反代配置，不再挨个改收藏夹、脚本和监控目标。

## 五、让全网设备自动使用 AdGuard

最推荐在路由器 DHCP 里下发 DNS：

```text
DHCP DNS Server: 192.168.10.53
DHCP Domain: home.arpa
```

如果路由器不支持自定义 DNS，退而求其次：

1. 关闭路由器 DHCP；
2. 让 AdGuard Home 接管 DHCP；
3. 或者用 OpenWrt / OPNsense 作为 DHCP 服务器。

AdGuard DHCP 示例配置思路：

```text
Range: 192.168.10.100 - 192.168.10.200
Gateway: 192.168.10.1
Subnet mask: 255.255.255.0
DNS server: 192.168.10.53
```

注意一个坑：**不要同时开两个 DHCP**。两个 DHCP 同网段同时发租约，会出现一部分设备走 AdGuard，另一部分设备走路由器 DNS，表现为“有的手机能屏蔽广告，有的不能”。

## 六、常用运维命令

查看容器状态：

```bash
cd /opt/dns-stack
docker compose ps
docker compose logs --tail=100 adguard
docker compose logs --tail=100 unbound
```

测试 DNS 链路：

```bash
# 直接问 AdGuard
nslookup example.com 192.168.10.53

# 测试本地域名
nslookup pve.home.arpa 192.168.10.53

# 测试是否真的走了拦截规则，返回 0.0.0.0 或 NXDOMAIN 都可能是正常拦截表现
nslookup doubleclick.net 192.168.10.53

# 测试 TCP DNS，排除 UDP 被防火墙拦截的问题
dig @192.168.10.53 example.com +tcp
```

备份配置：

```bash
tar -C /opt -czf /root/dns-stack-$(date +%F).tar.gz dns-stack/adguard-conf dns-stack/adguard-work dns-stack/unbound docker-compose.yml
```

升级：

```bash
cd /opt/dns-stack
docker compose pull
docker compose up -d
docker image prune -f
```

恢复时只要把 `adguard-conf`、`adguard-work`、`unbound.conf` 和 Compose 文件放回原路径，再 `docker compose up -d` 即可。

## 七、踩坑记录

### 1. 端口 53 被占用

症状：AdGuard 容器启动失败，日志里有 `bind: address already in use`。

排查：

```bash
sudo ss -lntup | grep ':53 '
sudo ss -lnuap | grep ':53 '
```

修复方向：关闭 `systemd-resolved` stub、停止 dnsmasq，或者把 AdGuard 放到 macvlan 独立 IP。不要把容器端口随便改成 `5353` 然后指望全家设备自动使用，客户端默认只会问 53。

### 2. AdGuard 能打开后台，但客户端解析不了

常见原因是只映射了 TCP 53，没有映射 UDP 53。DNS 大部分查询走 UDP，所以 Compose 必须同时有：

```yaml
- "53:53/tcp"
- "53:53/udp"
```

### 3. 本地域名偶尔失效

如果你用了 `.local`，很可能和 mDNS 冲突。建议迁移到 `home.arpa`。另外检查客户端是否真的拿到了 AdGuard DNS：

```bash
# Linux
resolvectl status

# Windows PowerShell
Get-DnsClientServerAddress

# macOS
scutil --dns | grep nameserver
```

### 4. DNS 服务一挂，全家断网

DNS 是基础设施，别只部署一个实例。进阶方案是准备两个 AdGuard：

| 节点 | IP | 角色 |
| --- | --- | --- |
| DNS-1 | `192.168.10.53` | 主 DNS |
| DNS-2 | `192.168.10.54` | 备用 DNS |

DHCP 同时下发两个 DNS。注意主备配置要同步，可以定期备份 `AdGuardHome.yaml`，或者用 Ansible / Git 管理配置。

## 八、推荐规则与隐私边界

广告过滤不是规则越多越好。规则过多会增加内存占用和误杀概率。家庭网络建议从这几类开始：

| 类型 | 建议 | 说明 |
| --- | --- | --- |
| 广告追踪 | AdGuard DNS filter | 默认规则，稳定性较好 |
| 恶意域名 | 开启安全浏览 | 防钓鱼、恶意软件 |
| 家长控制 | 按需 | 容易误伤，别默认全开 |
| 自定义黑名单 | 少量维护 | 针对电视广告、IoT 上报域名 |

隐私上也要有边界：AdGuard 能看到全网 DNS 查询，管理员可以知道每台设备访问过哪些域名。家庭环境建议减少日志保留时间，不要把查询日志长期上传到第三方分析系统。

## 总结

这套 AdGuard Home + Unbound 的方案，核心价值不是“屏蔽几个广告”，而是把 Homelab 的服务发现和 DNS 控制权拿回来。

| 场景 | 推荐配置 |
| --- | --- |
| 小型 Homelab | 单 AdGuard + 单 Unbound，Docker Compose 部署 |
| 服务较多 | `*.home.arpa` DNS rewrite + Caddy/Traefik 反向代理 |
| 对稳定性敏感 | 双 AdGuard 主备，DHCP 下发两个 DNS |
| 对隐私敏感 | 缩短 Query log 保留时间，Unbound 本地递归 |

一句话：**先把 DNS 做稳，再谈反代、证书、监控和自动化。Homelab 里很多“玄学网络问题”，最后都是 DNS 没设计好。**
