---
title: "把相册搬回家：Immich + Docker Compose 搭建私有照片云与备份闭环"
description: "在 Homelab 中用 Docker Compose 部署 Immich 私有相册，接入 NAS 存储、反向代理、PostgreSQL 数据库备份与恢复演练，解决照片上云后的隐私和可控性问题。"
date: 2026-05-14T09:01:23+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
slug: "immich-private-photo-cloud"
---

## 前言

Homelab 最容易被家人感知价值的服务，不是 PVE、不是 Grafana，也不是各种炫酷的 Dashboard，而是**照片备份**。

手机相册长期放在 iCloud、Google Photos 或各种国产云里，确实省心，但也带来几个问题：容量付费、隐私不可控、原图下载麻烦、服务策略随时变化。如果家里已经有一台 NAS 或 Proxmox VE 主机，再自建一套照片云就很自然了。

这篇文章选择 **Immich**。它的体验接近 Google Photos，支持手机 App 自动备份、时间线浏览、人物识别、地图、相册共享和机器学习搜索。更关键的是：它可以完整运行在 Docker Compose 中，数据落在自己的磁盘上，适合 Homelab 长期维护。

本文不是简单贴一份 `docker-compose.yml` 就结束，而是从目录规划、部署、反向代理、备份、恢复演练、踩坑处理几个角度，搭一套能长期运行的私有照片云。

---

## 1. 架构设计

这套部署面向普通 Homelab 环境：一台 Debian/Ubuntu Docker 主机，后端挂载 NAS 或本地大容量磁盘，前端通过反向代理提供 HTTPS。

```text
手机 App / 浏览器
        │
        ▼
  https://photos.example.com
        │
        ▼
  Traefik / Nginx Proxy Manager
        │
        ▼
  Immich Server + Web + ML
        │
        ├── PostgreSQL：元数据、相册、用户、任务状态
        ├── Redis：队列缓存
        └── Upload Library：照片和视频原文件
```

Immich 里最重要的两类数据：

| 数据 | 路径/组件 | 重要程度 | 备份方式 |
|---|---|---:|---|
| 原图、视频、缩略图 | `UPLOAD_LOCATION` | ⭐⭐⭐⭐⭐ | 文件级备份 / ZFS 快照 / NAS 同步 |
| PostgreSQL 数据库 | `database` 容器 | ⭐⭐⭐⭐⭐ | `pg_dumpall` 定时导出 |
| Redis 队列 | `redis` 容器 | ⭐⭐ | 可不备份 |
| 机器学习模型缓存 | `model-cache` | ⭐ | 可重建 |

> 重点：**只备份照片目录是不够的**。相册关系、用户、分享链接、人脸识别结果等都在 PostgreSQL 里，数据库必须一起备份。

---

## 2. 准备目录

假设 Docker 服务跑在 `/opt/immich`，照片数据放到 `/mnt/tank/photos/immich`。如果你用的是 NAS NFS 挂载，也可以把 `UPLOAD_LOCATION` 指向 NFS 目录。

```bash
sudo mkdir -p /opt/immich
sudo mkdir -p /mnt/tank/photos/immich
sudo mkdir -p /mnt/tank/backups/immich-db
sudo chown -R $USER:$USER /opt/immich /mnt/tank/photos/immich /mnt/tank/backups/immich-db
cd /opt/immich
```

建议把 Immich 的 Compose 文件和 `.env` 放在 SSD 上，照片库放在 HDD/ZFS 池上。这样数据库和容器启动速度更好，照片容量也更好扩展。

如果后端是 NFS，挂载建议加上 `_netdev`，避免开机 Docker 比网络先启动导致目录为空：

```ini
# /etc/fstab
192.168.1.10:/volume1/photos /mnt/tank/photos nfs4 defaults,_netdev,noatime 0 0
```

修改后验证：

```bash
sudo mount -a
findmnt /mnt/tank/photos
```

---

## 3. 编写 .env

先创建环境变量文件：

```bash
cd /opt/immich
openssl rand -base64 32
```

`.env` 示例：

```dotenv
# Immich Web 访问端口，只暴露给反向代理或内网
IMMICH_VERSION=release

# 照片上传目录，务必放在大容量磁盘或 NAS 上
UPLOAD_LOCATION=/mnt/tank/photos/immich

# PostgreSQL
DB_USERNAME=immich
DB_PASSWORD=请替换成上面生成的强密码
DB_DATABASE_NAME=immich

# 时区
TZ=Asia/Shanghai
```

注意：`UPLOAD_LOCATION` 不建议放在 `/opt/immich` 下面。应用配置和用户数据分离，后续迁移、备份、重建都会简单很多。

---

## 4. Docker Compose 部署 Immich

下面是一份适合 Homelab 的 `docker-compose.yml`。它没有直接暴露 2283 到公网，而是只绑定到本机，由反向代理转发。

```yaml
services:
  immich-server:
    image: ghcr.io/immich-app/immich-server:${IMMICH_VERSION:-release}
    container_name: immich_server
    volumes:
      - ${UPLOAD_LOCATION}:/usr/src/app/upload
      - /etc/localtime:/etc/localtime:ro
    env_file:
      - .env
    ports:
      - "127.0.0.1:2283:2283"
    depends_on:
      - redis
      - database
    restart: unless-stopped

  immich-machine-learning:
    image: ghcr.io/immich-app/immich-machine-learning:${IMMICH_VERSION:-release}
    container_name: immich_machine_learning
    volumes:
      - model-cache:/cache
    env_file:
      - .env
    restart: unless-stopped

  redis:
    image: docker.io/valkey/valkey:8-bookworm@sha256:fea8b3e67b15729d4bb70589eb03367bab9ad1ee89c876f54327fc7c6e618571
    container_name: immich_redis
    restart: unless-stopped

  database:
    image: ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0
    container_name: immich_postgres
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_DB: ${DB_DATABASE_NAME}
      POSTGRES_INITDB_ARGS: '--data-checksums'
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  pgdata:
  model-cache:
```

启动：

```bash
cd /opt/immich
docker compose pull
docker compose up -d
docker compose ps
```

查看日志：

```bash
docker logs -f immich_server
docker logs -f immich_machine_learning
```

确认本机端口可用：

```bash
curl -I http://127.0.0.1:2283
```

浏览器访问 `http://宿主机IP:2283`，创建管理员账号。确认能上传一张测试照片后，再接反向代理。

---

## 5. 接入反向代理

如果你已经有 Traefik，可以把 Immich 放到外部 Docker 网络里，通过 Label 暴露服务。示例：

```bash
docker network create traefik-net
```

Compose 中给 `immich-server` 增加：

```yaml
services:
  immich-server:
    networks:
      - default
      - traefik-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.immich.rule=Host(`photos.example.com`)"
      - "traefik.http.routers.immich.entrypoints=websecure"
      - "traefik.http.routers.immich.tls.certresolver=letsencrypt"
      - "traefik.http.services.immich.loadbalancer.server.port=2283"

networks:
  traefik-net:
    external: true
```

如果用 Nginx Proxy Manager，后端填写：

```text
Scheme: http
Forward Hostname/IP: Docker宿主机IP
Forward Port: 2283
Websockets Support: Enabled
Block Common Exploits: Enabled
```

反向代理层建议设置上传体积限制，否则大视频可能上传失败。Nginx 示例：

```nginx
client_max_body_size 0;
proxy_read_timeout 600s;
proxy_send_timeout 600s;
proxy_request_buffering off;
```

---

## 6. 数据库备份：不要只备份照片目录

官方建议对数据库做逻辑备份。最简单可靠的方式是在宿主机定时执行 `pg_dumpall`。

创建脚本 `/opt/immich/backup-db.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

BACKUP_DIR="/mnt/tank/backups/immich-db"
DATE="$(date +%F_%H-%M-%S)"
KEEP_DAYS=14

mkdir -p "$BACKUP_DIR"

docker exec -t immich_postgres pg_dumpall \
  --clean \
  --if-exists \
  --username="${DB_USERNAME:-immich}" \
  | gzip > "$BACKUP_DIR/immich-db-$DATE.sql.gz"

find "$BACKUP_DIR" -type f -name 'immich-db-*.sql.gz' -mtime +"$KEEP_DAYS" -delete

ls -lh "$BACKUP_DIR" | tail
```

由于脚本里需要 `.env` 的变量，更稳的写法是用 `docker compose exec` 读取当前项目环境：

```bash
#!/usr/bin/env bash
set -euo pipefail
cd /opt/immich
set -a
source .env
set +a

BACKUP_DIR="/mnt/tank/backups/immich-db"
DATE="$(date +%F_%H-%M-%S)"
mkdir -p "$BACKUP_DIR"

docker exec -t immich_postgres pg_dumpall \
  --clean --if-exists \
  --username="$DB_USERNAME" \
  | gzip > "$BACKUP_DIR/immich-db-$DATE.sql.gz"

find "$BACKUP_DIR" -type f -name 'immich-db-*.sql.gz' -mtime +14 -delete
```

授权并测试：

```bash
chmod +x /opt/immich/backup-db.sh
/opt/immich/backup-db.sh
ls -lh /mnt/tank/backups/immich-db/
```

加入定时任务：

```bash
crontab -e
```

```cron
# 每天凌晨 03:30 备份 Immich PostgreSQL
30 3 * * * /opt/immich/backup-db.sh >> /var/log/immich-db-backup.log 2>&1
```

照片目录可以用 ZFS 快照、restic、rsync 或 NAS 自带快照。ZFS 示例：

```bash
# 每天快照
zfs snapshot tank/photos@immich-$(date +%F)

# 查看快照
zfs list -t snapshot | grep immich
```

如果你没有 ZFS，用 `rsync` 同步到另一块盘也比没有强：

```bash
rsync -aHAX --delete --info=progress2 \
  /mnt/tank/photos/immich/ \
  /mnt/backup/photos/immich/
```

---

## 7. 恢复演练

备份没有恢复过，就等于没有备份。建议在另一台测试机或临时目录做一次恢复演练。

停止服务：

```bash
cd /opt/immich
docker compose down
```

如果是全新机器，先恢复照片目录：

```bash
rsync -aHAX /mnt/backup/photos/immich/ /mnt/tank/photos/immich/
```

启动数据库容器，但先不要让 Immich 正常对外使用：

```bash
docker compose up -d database redis
```

恢复 SQL：

```bash
gunzip < /mnt/tank/backups/immich-db/immich-db-2026-05-14_03-30-00.sql.gz \
  | docker exec -i immich_postgres psql --username=immich --dbname=postgres
```

再启动全部服务：

```bash
docker compose up -d
docker compose logs -f immich_server
```

恢复后检查三件事：

```bash
# 1. 容器状态
docker compose ps

# 2. 上传目录是否有文件
du -sh /mnt/tank/photos/immich

# 3. Web 是否返回
curl -I http://127.0.0.1:2283
```

最后登录网页确认相册、人物、分享链接是否正常。演练时发现问题，比硬盘真挂时才发现强太多。

---

## 8. 日常升级流程

Immich 更新比较频繁，不建议无脑 Watchtower 自动升级。更稳的流程是：先备份数据库，再拉镜像，再重启。

```bash
cd /opt/immich
./backup-db.sh

docker compose pull
docker compose up -d
docker image prune -f
```

升级后看日志：

```bash
docker compose logs --tail=200 immich-server
docker compose ps
```

如果升级后异常，先不要删除旧镜像和备份。可以回退 `.env` 里的 `IMMICH_VERSION` 到上一个版本，再 `docker compose up -d`。

---

## 9. 踩坑记录

### ❌ 1. NFS 目录没挂载，照片写进了本地空目录

现象：NAS 重启后 Docker 先启动，`/mnt/tank/photos` 变成本地空目录，Immich 继续往里面写数据。

修复：

```bash
findmnt /mnt/tank/photos || systemctl restart remote-fs.target
docker compose restart immich-server
```

长期方案：fstab 加 `_netdev`，并让 Docker 服务依赖远程文件系统：

```bash
sudo systemctl edit docker
```

```ini
[Unit]
After=network-online.target remote-fs.target
Wants=network-online.target
RequiresMountsFor=/mnt/tank/photos
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### ❌ 2. 大视频上传失败

通常是反向代理限制了 body size 或 timeout。Nginx 加：

```nginx
client_max_body_size 0;
proxy_read_timeout 600s;
proxy_send_timeout 600s;
proxy_request_buffering off;
```

Traefik 一般不限制 body size，但如果前面还有 Cloudflare，需要注意上传大小限制。

### ❌ 3. 只备份 upload 目录，恢复后相册全乱

原图还在，但用户、相册、分享链接、人物识别都丢了。原因是数据库没备份。解决方式就是本文第 6 节：照片目录 + PostgreSQL 逻辑备份必须成对存在。

### ❌ 4. 机器学习容器吃 CPU

初次导入大量照片时，缩略图、元数据提取、人脸识别会让 CPU 飙升。这不是异常。建议第一次导入放在夜间，或者临时降低并发任务数量。

可以在后台任务页面观察队列，也可以看容器资源：

```bash
docker stats immich_server immich_machine_learning immich_postgres
```

---

## 10. 安全建议

Immich 不是只给自己看的静态网页，它有登录、上传、分享和移动端同步，所以安全边界要认真处理：

| 项目 | 建议 |
|---|---|
| HTTPS | 必须启用，移动 App 同步也走 HTTPS |
| 管理员密码 | 使用密码管理器生成强密码 |
| 公网暴露 | 优先走 WireGuard / Tailscale；公网开放时必须及时升级 |
| 数据库端口 | 不要映射到公网，只在 Docker 网络内访问 |
| 备份 | 数据库备份 + 原图备份 + 恢复演练三件套 |
| 更新 | 手动升级，不建议无脑自动更新 |

如果只是家庭成员使用，最稳的是：Immich 只监听内网，通过 WireGuard 回家访问。这样攻击面比直接暴露公网小很多。

---

## 总结

Immich 是目前 Homelab 里非常值得部署的服务：用户体验好，家人能直接感受到价值，也能把照片数据重新拿回自己手里。但它不是“起一个容器就完事”的玩具服务，真正长期运行要注意三个点：

1. **存储规划**：照片目录放大容量盘，配置和数据库独立管理；
2. **备份闭环**：原图目录和 PostgreSQL 必须同时备份；
3. **恢复演练**：至少做一次从数据库备份和照片目录恢复的完整测试。

我的建议是：先在内网跑通，再接反向代理；先导入少量照片测试，再迁移全量相册；先把备份脚本和恢复流程写好，再放心让手机自动同步。Homelab 的核心不是“服务跑起来”，而是“服务坏了还能救回来”。
