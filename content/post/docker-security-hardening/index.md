---
title: "Homelab Docker 安全加固实战：Rootless、Seccomp、AppArmor 与最佳实践"
description: "Docker 默认以 root 运行所有容器，容器逃逸就能拿下宿主机。从 Rootless Docker、Capability 裁剪、Seccomp 过滤到 AppArmor 强制访问控制，手把手给你的容器加上十道安全锁。"
date: 2026-05-28T01:21:02+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
slug: "docker-security-hardening"
---

## 前言

容器的本质是进程隔离。但在默认配置下，Docker 做的隔离远没有你想象的那么安全。

```
# 在容器里看到的是 root
$ docker run --rm -it alpine id
uid=0(root) gid=0(root) groups=0(root)

# 在宿主机上这个容器进程同样是 root
$ ps aux | grep <container-pid>
root  12345  ...  /bin/sh
```

更糟的是——默认容器拥有一大堆 Linux capabilities（14 个）、没有 Seccomp 过滤（虽然 Docker 有默认配置，但很多人不知道它的存在）、没有 AppArmor 限制，甚至 `/proc`、`/sys` 等内核文件系统也是可写的。

对 Homelab 来说，虽然威胁没那么大（一般不暴露公网），但**内网不等于安全**——你装过的每个第三方容器镜像，都可能有未知的安全风险：

- 恶意镜像在运行后向宿主机写入定时任务
- 容器内未修复的 CVE 导致攻击者利用漏洞逃逸
- 容器进程读取宿主机的 `/proc/1/environ` 获取敏感信息

本文从轻到重，逐步加固你的 Docker 环境。**每一条都是可落地、可验证的实战操作。**

## 1. Linux Capability 裁剪：扔掉你用不到的权限

### 什么是 Capability？

传统 UNIX 把进程分为 root (UID 0) 和普通用户，二分法太粗糙。Linux Capability 把 root 权限拆成 40+ 个细粒度能力，比如：

| Capability | 权限 | 风险 |
|---|---|---|
| `CAP_NET_RAW` | 创建 RAW socket (ping, ARP) | 可构造恶意网络包 |
| `CAP_SYS_ADMIN` | 挂载文件系统、命名空间操作 | **逃逸的核心能力** |
| `CAP_SYS_PTRACE` | 跟踪任何进程 | 可读写宿主机进程内存 |
| `CAP_NET_BIND_SERVICE` | 绑定低于 1024 的端口 | 低风险，但说明容器有网络特权 |
| `CAP_SYS_MODULE` | 加载内核模块 | 加载恶意内核模块即逃逸 |

Docker 容器默认会带着一堆你根本不需要的 capabilities。来看怎么收窄。

### 实战：最小权限启动容器

```bash
# ❌ 默认：14 个 capabilities 全给
docker run --rm alpine capsh --print

# ✅ 最小权限：丢弃全部，只加你需要的
docker run --rm --cap-drop ALL --cap-add NET_BIND_SERVICE alpine capsh --print
```

用 `--cap-drop ALL` 丢弃所有权限，再用 `--cap-add` 逐一添加你明确需要的。这是**安全容器的第一守则**。

### Docker Compose 中的配置

```yaml
services:
  nginx:
    image: nginx:alpine
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE   # 绑定 80/443 端口需要
      - NET_RAW            # 如果需要健康检查 ping
      - CHOWN              # 日志文件属主修改
```

**踩坑经验**：PostgreSQL 容器至少需要 `CHOWN`、`DAC_OVERRIDE`、`SETUID`、`SETGID`，`cap_drop` 太多会导致容器启动失败。建议先用默认跑起来，再用 `capsh --print` 查看实际需要的 capabilities，逐步裁剪。

## 2. Docker 默认 Seccomp：你的第一条防线

Seccomp（Secure Computing Mode）是 Linux 内核的安全机制，限制进程可以调用的**系统调用**。Docker 默认带了一份 Seccomp 配置，**但大多数人不知道它在工作**。

### 验证 Seccomp 是否生效

```bash
# 检查容器是否在 Seccomp 下运行
docker run --rm alpine grep Seccomp /proc/1/status
# 输出: Seccomp:	2  (2 = filter mode)

# 尝试被禁用的系统调用——比如 unshare（逃逸常用）
docker run --rm alpine unshare --help
# 默认配置下 unshare 已经被 Seccomp 拦截！
```

### 查看 Docker 默认 Seccomp 配置文件

Docker 的默认 seccomp 配置在 `/etc/docker/seccomp-profiles/default.json`（如果存在）或内嵌在 Docker 守护进程里。

```bash
# 提取 Docker 内置的默认 seccomp 配置
container=$(docker run -d --rm alpine sleep 300)
docker inspect "$container" --format '{{ .HostConfig.SecurityOpt }}'
# 输出类似 [seccomp=built-in-default]
```

### 当容器需要额外的系统调用时

某些应用（比如 Chrome/Selenium 需要 `clone` 和 `unshare`，或者某些数据库需要 `personality`）会被默认 seccomp 拦截。你可以创建自定义 seccomp 配置文件：

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64", "SCMP_ARCH_AARCH64"],
  "syscalls": [
    {
      "names": ["personality", "clone", "unshare"],
      "action": "SCMP_ACT_ALLOW",
      "args": [],
      "comment": "Chrome浏览器所需的系统调用"
    }
  ]
}
```

保存为 `chrome-seccomp.json`，然后：

```bash
docker run --security-opt seccomp=chrome-seccomp.json selenium/chrome
```

### 完全禁用 Seccomp（不推荐）

```bash
# ⚠️ 只在调试时使用
docker run --security-opt seccomp=unconfined alpine
```

**踩坑经验**：如果你遇到 `operation not permitted` 但容器以 root 运行，且 capabilities 也给够了，十有八九是 Seccomp 拦截。检查 Docker 默认允许列表，或者用 `strace` 找出容器实际需要的系统调用。

## 3. AppArmor：强制访问控制

AppArmor 是 Linux 的 LSM（Linux Security Module），通过附加到程序上的安全策略限制程序的文件访问、网络访问等能力。

### 检查 AppArmor 状态

```bash
# 宿主机是否启用了 AppArmor
sudo aa-status
# 如果没有输出，说明没有启用 AppArmor 或不是 AppArmor（可能是 SELinux）

# Docker 默认使用的 AppArmor 配置
docker run --rm alpine cat /proc/1/attr/current
# 输出: docker-default (enforce)  或  unconfined
```

### Docker 默认 AppArmor 策略

Docker 自带的 `docker-default` AppArmor 策略会限制：

- 写入 `/proc/sys` 和 `/sys`
- 挂载文件系统
- 创建设备节点
- 某些网络操作

### 应用自定义 AppArmor 配置

```bash
# 1. 编写自定义策略文件
cat > /etc/apparmor.d/docker-nginx << 'EOF'
#include <tunables/global>

profile docker-nginx flags=(attach_disconnected) {
  #include <abstractions/base>
  
  network inet tcp,
  network inet udp,
  
  /etc/nginx/** r,
  /var/log/nginx/** w,
  /var/run/nginx.pid w,
  
  deny /proc/** w,
  deny /sys/** w,
  deny /etc/shadow r,
}
EOF

# 2. 加载策略
sudo apparmor_parser -r -W /etc/apparmor.d/docker-nginx

# 3. 使用自定义策略启动容器
docker run --security-opt apparmor=docker-nginx nginx
```

### 在 Compose 中使用 AppArmor

```yaml
services:
  app:
    image: myapp:latest
    security_opt:
      - apparmor=docker-myapp
```

**踩坑经验**：AppArmor 配置错误会导致容器启动失败，且日志信息非常隐晦。先在 `complain` 模式下调试（日志只警告不拦截），确认没问题再切到 `enforce` 模式：

```bash
# 先用 complain 模式加载
apparmor_parser -C -W /etc/apparmor.d/docker-nginx
# 观察 /var/log/syslog 或 audit.log
# 确认无误后切到 enforce
apparmor_parser -r -W /etc/apparmor.d/docker-nginx
```

## 4. Read-only Root Filesystem：让容器只读

容器一旦被攻破，攻击者第一件事就是在 `/tmp` 或 `/var/tmp` 写恶意脚本。把根文件系统设为只读，可以有效阻止持久化攻击。

```bash
# ❌ 默认：容器的根文件系统可写
docker run alpine touch /malicious-file  # ✅ 成功

# ✅ 只读：任何写入根文件系统的操作都会失败
docker run --read-only alpine touch /malicious-file
# touch: /malicious-file: Read-only file system
```

但是容器服务通常需要写某些目录（日志、临时文件、数据）。用 `--tmpfs` 为特定目录创建可写空间：

```bash
docker run --read-only \
  --tmpfs /tmp:noexec,nosuid,size=64M \
  --tmpfs /var/run:noexec,nosuid \
  nginx:alpine
```

对应的 Compose 配置：

```yaml
services:
  nginx:
    image: nginx:alpine
    read_only: true
    tmpfs:
      - /tmp:noexec,nosuid,size=64M
      - /var/run:noexec,nosuid
```

**踩坑经验**：很多基于 Debian/Ubuntu 的镜像在启动时会写 `/var/cache` 或 `/var/lib/apt`。遇到 `Read-only file system` 错误时，用 `docker diff <container>` 找出容器在写哪些路径，然后为这些路径添加 `tmpfs` 或 volume。

## 5. User Namespace Remapping：不让容器 root 等于宿主机 root

这是**最有效、也最容易被忽略**的安全加固手段。

### 原理

默认情况下，容器内的 UID 0 直接映射到宿主机的 UID 0。启用 userns-remap 后，容器内的 root (UID 0) 会被映射到宿主机上的一个**非特权用户**（比如 UID 165536）。

```
# 没有 userns-remap：
容器 root (0) → 宿主机 root (0)  ← 逃逸即 GAME OVER

# 有 userns-remap：
容器 root (0) → 宿主机 nobody (165536)  ← 逃逸也只能访问这个用户的权限
```

### 实战配置

```bash
# 1. 创建映射用户
sudo useradd --system dockremap

# 2. 编辑 /etc/docker/daemon.json
sudo tee /etc/docker/daemon.json << 'EOF'
{
  "userns-remap": "dockremap"
}
EOF

# 3. 重启 Docker
sudo systemctl restart docker
```

重启后验证：

```bash
# 容器里的 root
docker run --rm alpine id
# uid=0(root) gid=0(root)

# 宿主机上的进程
ps aux | grep sleep
# dockremap  12345  0.0  0.0  ...  sleep 300
# 进程属于 dockremap 用户，不是 root！
```

### 限制和注意事项

⚠️ **userns-remap 不是无痛的**：

1. **Volume 权限问题**：容器内 UID 0 映射到宿主机 UID 165536，所以挂载到容器的 volume 必须允许这个 UID 写入：

```bash
# 宿主机创建目录并授权给映射用户
sudo mkdir -p /data/mysql
sudo chown 165536:165536 /data/mysql
```

或者使用：

```bash
# 这行命令在宿主机上找出容器内的 UID 对应到哪个宿主机 UID
# dockremap: 子UID范围从 /etc/subuid 查看
grep dockremap /etc/subuid
# dockremap:165536:65536
```

2. **Host 网络模式不可用**：userns-remap 后 `--network host` 会失败。

3. **privileged 模式不可用**：`--privileged` 被禁止。

4. **某些系统级工具不可用**：比如 `ping` 需要 `NET_RAW` capability。

**踩坑经验**：如果你正在运行大量容器，不要在**生产负载下**启用 userns-remap。它需要重建所有容器的网络命名空间映射，旧容器在重建前会无法启动。建议先在一台实验机器上验证，或者新建一个 Docker 节点逐步迁移。

## 6. 限制容器对 /proc 和 /sys 的访问

即使有了 AppArmor，某些敏感的内核文件系统仍然可以通过 Docker 的掩码（masked paths）来保护。

```bash
# 默认情况下以下路径已被 mask（空卷覆盖）：
# /proc/acpi, /proc/kcore, /proc/keys, /proc/latency_stats
# /proc/timer_list, /proc/timer_stats, /proc/sched_debug
# /sys/firmware, /sys/devices/virtual/powercap

# 检查容器的 masked paths
docker run --rm alpine ls /proc/kcore  # 不存在（已被 mask）
```

自定义 masked paths：

```bash
docker run \
  --security-opt mask=/proc/self/sched:/proc/self/comm \
  alpine
```

## 7. 非 root 用户运行容器

镜像内部不要用 root 运行进程：

```dockerfile
# ❌ 默认：以 root 运行
FROM node:20-alpine
COPY . /app
CMD ["node", "server.js"]
# 进程在容器内是 root

# ✅ 推荐：创建专用用户
FROM node:20-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY . /app
RUN chown -R appuser:appgroup /app
USER appuser
CMD ["node", "server.js"]
```

对于官方镜像也可以指定用户：

```yaml
services:
  nginx:
    image: nginx:alpine
    user: "101:101"  # nginx 用户的 UID/GID
```

**踩坑经验**：注意容器内使用 `USER` 指令后，绑定端口低于 1024 会失败（需要 `NET_BIND_SERVICE` capability）。对于 nginx，镜像内已配置了 libcap 自动降权，不需要额外处理。

## 8. Docker 守护进程安全加固

### 限制 Docker API 访问

默认 Docker 只监听 Unix socket（`/var/run/docker.sock`），这在安全性上反而是最好的（只有 root 和 docker 组的用户能访问）。

**千万不要**直接把 Docker API 暴露到 TCP：

```bash
# ❌ 高危：任何人都可以访问 Docker API
dockerd -H tcp://0.0.0.0:2375

# ✅ 安全的远程访问：使用 TLS 双向认证
dockerd \
  --tlsverify \
  --tlscacert=/etc/docker/ca.pem \
  --tlscert=/etc/docker/server-cert.pem \
  --tlskey=/etc/docker/server-key.pem \
  -H=0.0.0.0:2376
```

### 用 docker-socket-proxy 代替直接挂载 socket

很多 Homelab 服务（Portainer、Traefik、Watchtower 等）需要访问 Docker socket。最安全的做法不是直接挂载 socket，而是用 [docker-socket-proxy](https://github.com/Tecnativa/docker-socket-proxy) 提供**只读的、有权限控制的 HTTP API**：

```yaml
services:
  docker-proxy:
    image: tecnativa/docker-socket-proxy
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      CONTAINERS: 1        # 只允许读取容器信息
      SERVICES: 1
      TASKS: 1
      POST: 0              # 禁止创建操作
      AUTH: 0

  traefik:
    image: traefik:v3
    environment:
      - DOCKER_HOST=tcp://docker-proxy:2375
```

这样 Traefik 只能**读取**容器信息来动态配置路由，但无法创建、删除容器。

### 限制 docker 组访问

`docker` 组即 root 权限。不要轻易把用户加入 docker 组：

```bash
# 审计谁在 docker 组
getent group docker
# docker:x:999:user1,user2

# 考虑使用 Keycloak 或 Authelia 做细粒度授权（企业级需求）
# 或者干脆用 Rootless Docker，让用户用非特权模式运行容器
```

## 9. Docker Bench Security：一键安全审计

Docker 官方推荐的安全审计工具。安装即用：

```bash
# 运行完整安全审计（会检查 200+ 项配置）
docker run --rm \
  --net host \
  --pid host \
  --userns host \
  --cap-add audit_control \
  -e DockerBenchArea=/etc/docker \
  -e DockerBenchArea=/usr/lib/systemd/system \
  -e DockerBenchArea=/var/lib/docker \
  -e DockerBenchArea=/etc/default/docker \
  -v /etc:/etc:ro \
  -v /usr/lib/systemd/system:/usr/lib/systemd/system:ro \
  -v /var/lib/docker:/var/lib/docker:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  docker/docker-bench-security
```

输出示例：

```
[INFO] 1 - Host Configuration
[WARN] 1.1  - Ensure a separate partition for containers has been created
[PASS] 1.2  - Ensure the container host has been Hardened
[INFO] 1.3  - Ensure Docker is up to date
...
[WARN] 4.1  - Ensure a user for the container has been created
[PASS] 4.2  - Ensure containers use only trusted base images
[WARN] 5.4  - Ensure privileged containers are not used
```

把审计结果作为安全基线，逐项修复。

## 10. 终极模板：一份生产级安全的 Docker Compose

把以上加固措施整合到一起，这是你可以直接用于 Homelab 的完整模板：

```yaml
name: secure-service

services:
  webapp:
    image: your-app:latest
    
    # 只读文件系统，只有必要目录可写
    read_only: true
    tmpfs:
      - /tmp:noexec,nosuid,size=64M
      - /var/cache:noexec,nosuid,size=32M
    
    # 最小权限原则：只给需要的 capabilities
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
      - CHOWN
      - SETGID
      - SETUID
      - DAC_OVERRIDE
    
    # 非 root 用户运行
    user: "1001:1001"
    
    # 安全选项
    security_opt:
      - seccomp=/path/to/custom-seccomp.json   # 或直接使用 docker-default
      - apparmor=custom-profile                 # 可选
      - no-new-privileges:true                  # 阻止容器进程获取新权限
    
    # 内核参数安全
    sysctls:
      - net.ipv4.ip_unprivileged_port_start=80 # 非 root 用户绑定 80 端口
    
    # 文件系统掩码
    tmpfs:
      - /tmp
    volumes:
      - app_data:/data/app
```

**一份完整的 Checklist** 供你在新装 Docker 后逐项检查：

| # | 项目 | 命令/配置 |
|---|------|----------|
| 1 | 启用 userns-remap | `/etc/docker/daemon.json` > `userns-remap` |
| 2 | Seccomp 生效 | `grep Seccomp /proc/1/status` 输出 2 |
| 3 | AppArmor 生效 | `cat /proc/1/attr/current` 输出 `docker-default (enforce)` |
| 4 | 容器非 root 运行 | `user` 指令或 USER Dockerfile |
| 5 | Capability 裁剪 | `cap_drop: ALL` + 白名单 `cap_add` |
| 6 | 只读根文件系统 | `read_only: true` + tmpfs |
| 7 | 非特权升级 | `no-new-privileges:true` |
| 8 | Docker socket 代理 | 替代直接挂载 `/var/run/docker.sock` |
| 9 | Docker Bench | 定期运行安全审计 |
| 10 | 镜像签名 | `DOCKER_CONTENT_TRUST=1` |

## 总结

Docker 的安全加固不是非黑即白的事。**每一层加固都有代价**：userns-remap 会增加 volume 管理复杂度，read-only 文件系统需要排查服务写入路径，cap_drop ALL 可能让某些镜像没法启动。

我的建议是从最容易入手的做起：

1. **马上就能做的**：给所有新容器加 `cap_drop ALL` + `cap_add` 白名单、用 `read_only: true`、镜像内用 `USER`
2. **逐步推进的**：自定义 Seccomp 配置、AppArmor 策略、`no-new-privileges`
3. **底大包天的**：`userns-remap`——效果最好，但建议在重建环境时一次性启用

安全是持续的工程，不是一次性配置。你的 Homelab 越多依赖容器，花在安全上的时间就越值得。

---

*延伸阅读：*
- [Docker Security Cheat Sheet (OWASP)](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)
- [Docker Bench Security](https://github.com/docker/docker-bench-security)
- [Docker 官方安全文档](https://docs.docker.com/engine/security/)
