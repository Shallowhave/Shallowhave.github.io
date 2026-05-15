---
title: "别只会快照：用 Restic + MinIO 给 Homelab 做一套可恢复的文件级备份"
description: "在 Homelab 中用 MinIO 搭建 S3 兼容备份仓库，并用 Restic 实现 Linux、Docker 数据目录和 NAS 共享的加密增量备份，覆盖自动化脚本、保留策略、校验恢复与踩坑。"
date: 2026-05-15T01:01:00+08:00
image:
math:
license:
hidden: false
comments: true
draft: false
slug: "restic-minio-homelab-backup"
---

## 前言：快照不是备份，能恢复才算闭环

很多 Homelab 一开始都会依赖 PVE 快照、ZFS snapshot、NAS 回收站来“防误删”。这些能力很好，但它们本质上更偏向**本机时间点回滚**，并不天然解决几个问题：

1. 宿主机或存储池整体损坏时，快照也跟着没了；
2. Docker 应用的数据散落在 `/opt`、`/var/lib/docker/volumes`、NFS 挂载点里，PVE VM 级备份粒度太粗；
3. 想恢复单个配置文件、某个数据库 dump、某张照片时，整机恢复成本太高；
4. 没有定期校验和恢复演练，备份文件存在不等于可用。

所以这篇文章不讲 PVE 核显 SR-IOV、vGPU，也不重复 PBS 整机备份，而是做一套**文件级、加密、增量、可校验、可自动化**的 Homelab 备份方案：

- MinIO：在内网提供 S3 兼容对象存储，作为 Restic 仓库后端；
- Restic：负责客户端加密、去重、增量备份、保留策略和恢复；
- systemd timer：替代脆弱的 crontab，做可观测的定时任务；
- `check` + 恢复演练：确认备份真的能用。

> MinIO 官方文档强调容器部署时磁盘、控制器和网络会直接影响性能；Restic 官方文档也支持用环境变量配置仓库地址和密码，适合无人值守备份脚本。

## 方案架构

推荐拓扑如下：

| 角色 | 示例 IP | 作用 | 建议 |
| --- | --- | --- | --- |
| MinIO 备份端 | `192.168.10.20` | S3 兼容对象存储 | 独立磁盘 / NAS / 另一台机器 ✅ |
| Docker 主机 | `192.168.10.30` | 运行应用容器 | 备份 `/opt/app`、compose、数据库 dump |
| PVE / Linux 主机 | `192.168.10.10` | 宿主机配置 | 备份 `/etc`、脚本、关键配置 |
| 异地副本 | 可选 | 防火灾/误删/勒索 | rclone 同步到云端或另一地点 🏆 |

核心原则：**备份端不要和生产数据在同一块盘上**。如果 MinIO 和 Docker 服务都跑在同一台机器同一块 SSD 上，那只能防误删，不能防硬盘损坏。

## 一、部署 MinIO 作为 S3 仓库

先在备份服务器准备目录：

```bash
sudo mkdir -p /opt/minio/{data,config}
sudo useradd -r -s /usr/sbin/nologin minio || true
sudo chown -R 1000:1000 /opt/minio
```

`docker-compose.yml` 示例：

```yaml
services:
  minio:
    image: quay.io/minio/minio:latest
    container_name: minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: "minioadmin"
      MINIO_ROOT_PASSWORD: "请换成至少32位随机密码"
      TZ: "Asia/Shanghai"
    volumes:
      - /opt/minio/data:/data
      - /opt/minio/config:/root/.minio
    ports:
      - "9000:9000"
      - "9001:9001"
    restart: unless-stopped
```

启动：

```bash
cd /opt/minio
docker compose up -d
docker logs -f minio
```

访问控制台：`http://192.168.10.20:9001`，创建 bucket：`homelab-restic`。

生产环境建议不要直接使用 root key 给 Restic。用 MinIO Client 创建专用用户：

```bash
docker run --rm -it --network host quay.io/minio/mc:latest sh
mc alias set local http://192.168.10.20:9000 minioadmin '你的Root密码'
mc admin user add local restic-backup '再生成一个强密码'
mc admin policy attach local readwrite --user restic-backup
mc ls local
```

如果只想限制到单个 bucket，可以写更细的 JSON policy，不过 Homelab 初期 `readwrite` + 独立用户已经比直接暴露 root 凭据好很多。

## 二、客户端安装 Restic 并初始化仓库

在需要备份的 Linux / Docker 主机安装：

```bash
sudo apt update
sudo apt install -y restic
restic version
```

创建环境文件，避免把密钥写进脚本：

```bash
sudo install -d -m 700 /etc/restic
sudo tee /etc/restic/homelab.env >/dev/null <<'EOF'
export AWS_ACCESS_KEY_ID="restic-backup"
export AWS_SECRET_ACCESS_KEY="你的MinIO用户密码"
export RESTIC_REPOSITORY="s3:http://192.168.10.20:9000/homelab-restic/docker-host-01"
export RESTIC_PASSWORD="Restic仓库加密密码，务必离线保存"
EOF
sudo chmod 600 /etc/restic/homelab.env
```

初始化仓库：

```bash
sudo bash -c 'source /etc/restic/homelab.env && restic init'
```

注意：`RESTIC_PASSWORD` 是客户端加密密码，MinIO 管理员也不能解密你的备份。这个密码一旦丢失，备份基本等于废数据，建议放进密码管理器，并离线保存一份。

## 三、备份什么：不要无脑备份整个 `/`

Homelab 里最值得备份的是**配置、编排文件、应用数据和数据库导出**，而不是系统缓存、镜像层、临时文件。

一个 Docker 主机可以这样组织：

```bash
/opt/stacks/                 # docker compose 项目
/opt/appdata/                # 应用持久化目录
/opt/backup-dump/            # 数据库导出临时目录
/etc/                         # 关键系统配置
/usr/local/bin/               # 自写脚本
```

排除文件 `/etc/restic/excludes.txt`：

```text
/var/lib/docker/overlay2
/var/lib/docker/image
/var/lib/docker/containers/*/*-json.log
/tmp
/var/tmp
/cache
node_modules
*.tmp
*.cache
```

备份脚本 `/usr/local/sbin/restic-homelab-backup.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

source /etc/restic/homelab.env
HOST_TAG="docker-host-01"
DUMP_DIR="/opt/backup-dump"
mkdir -p "$DUMP_DIR"

# 示例：备份 PostgreSQL 容器数据库，按实际容器名修改
if docker ps --format '{{.Names}}' | grep -q '^postgres$'; then
  docker exec postgres pg_dumpall -U postgres > "$DUMP_DIR/postgres-all.sql"
fi

# 示例：备份 MariaDB / MySQL
if docker ps --format '{{.Names}}' | grep -q '^mariadb$'; then
  docker exec mariadb mariadb-dump --all-databases -uroot -p"$MYSQL_ROOT_PASSWORD" > "$DUMP_DIR/mariadb-all.sql"
fi

restic backup \
  /etc \
  /usr/local/bin \
  /opt/stacks \
  /opt/appdata \
  "$DUMP_DIR" \
  --exclude-file /etc/restic/excludes.txt \
  --one-file-system \
  --tag "$HOST_TAG"

# 保留策略：近 7 天每天、近 4 周每周、近 12 个月每月
restic forget \
  --tag "$HOST_TAG" \
  --keep-daily 7 \
  --keep-weekly 4 \
  --keep-monthly 12 \
  --prune

# 快速校验仓库结构；深度校验可以放到周任务
restic check
```

授权：

```bash
sudo chmod 700 /usr/local/sbin/restic-homelab-backup.sh
sudo /usr/local/sbin/restic-homelab-backup.sh
```

这里有一个关键点：数据库不要只备份裸 volume。PostgreSQL、MySQL 这类数据库在运行时直接复制数据目录，可能得到不一致状态。更稳妥的方式是先 dump，再让 Restic 备份 dump 文件。大数据库可以用物理备份工具，但 Homelab 多数场景 dump 足够。

## 四、用 systemd timer 自动化

创建 service：

```ini
# /etc/systemd/system/restic-homelab-backup.service
[Unit]
Description=Restic Homelab Backup
Wants=network-online.target
After=network-online.target docker.service

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/restic-homelab-backup.sh
Nice=10
IOSchedulingClass=best-effort
IOSchedulingPriority=7
```

创建 timer：

```ini
# /etc/systemd/system/restic-homelab-backup.timer
[Unit]
Description=Run Restic Homelab Backup Daily

[Timer]
OnCalendar=*-*-* 03:30:00
RandomizedDelaySec=20m
Persistent=true

[Install]
WantedBy=timers.target
```

启用：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now restic-homelab-backup.timer
systemctl list-timers | grep restic
journalctl -u restic-homelab-backup.service -n 100 --no-pager
```

`Persistent=true` 的好处是：如果凌晨 3:30 主机没开机，下一次开机后会补跑一次，比普通 crontab 更适合偶尔关机的 Homelab。

## 五、恢复演练：每个月至少跑一次

只会备份不会恢复，等于没备份。常用命令：

```bash
source /etc/restic/homelab.env

# 查看快照
restic snapshots

# 查看某个快照里的文件
restic ls latest | less

# 恢复单个目录到临时位置
sudo mkdir -p /tmp/restore-test
restic restore latest --target /tmp/restore-test --include /opt/stacks

# 挂载仓库浏览，适合找文件
sudo mkdir -p /mnt/restic
restic mount /mnt/restic
```

恢复 Docker 应用时，建议流程是：

```bash
cd /opt/stacks/你的应用
docker compose down
rsync -aH --delete /tmp/restore-test/opt/appdata/你的应用/ /opt/appdata/你的应用/
docker compose up -d
docker compose logs -f
```

数据库恢复示例：

```bash
# PostgreSQL
cat /tmp/restore-test/opt/backup-dump/postgres-all.sql | docker exec -i postgres psql -U postgres

# MariaDB
cat /tmp/restore-test/opt/backup-dump/mariadb-all.sql | docker exec -i mariadb mariadb -uroot -p
```

## 踩坑记录

### ❌ 1. MinIO 和源数据放同一块盘

这只能防误删，不防硬盘坏。最低要求是不同物理盘；更推荐另一台机器、NAS 或异地对象存储。

### ❌ 2. 只做 `forget` 不做 `prune`

`restic forget` 只是标记旧快照不再保留，真正释放空间需要 `--prune`。如果担心每日任务太慢，可以每日 `forget`，每周单独跑：

```bash
source /etc/restic/homelab.env
restic prune
restic check --read-data-subset=10%
```

### ❌ 3. 备份 Docker overlay2

`/var/lib/docker/overlay2` 体积巨大且恢复价值低，Restic 会浪费大量时间扫描。真正要备份的是 compose 文件、环境变量模板、应用持久化 volume 和数据库 dump。

### ❌ 4. 忘记保存 Restic 密码

MinIO access key 可以重置，但 Restic 仓库密码丢了无法解密。建议把 `/etc/restic/homelab.env` 中的 `RESTIC_PASSWORD` 放进密码管理器，并打印一份离线保存。

### ❌ 5. 从不做恢复测试

建议每月创建一个临时目录恢复 `latest`，至少验证：

```bash
restic check
restic restore latest --target /tmp/restore-test --include /etc/hostname
diff /etc/hostname /tmp/restore-test/etc/hostname
```

## 进阶：异地副本与不可变备份

如果预算允许，可以再加一层：

| 方案 | 成本 | 防护能力 | 适合场景 |
| --- | --- | --- | --- |
| 本机 MinIO | 低 | 防误删 ✅ | 入门测试 |
| 独立 NAS / 另一台机器 | 中 | 防主机损坏 ✅ | 常规 Homelab 🏆 |
| 云端 S3 / B2 / R2 | 中 | 防本地灾难 ✅ | 重要数据 |
| 对象锁 / WORM | 较高 | 防勒索 ✅ | 家庭照片、文档 |

用 `rclone` 同步 MinIO bucket 到云端也可以，但要注意：Restic 仓库是大量小对象和索引文件，复制时必须保证完整性，复制后最好在目标端跑一次：

```bash
RESTIC_REPOSITORY="s3:https://你的云端S3/homelab-restic/docker-host-01" restic check
```

## 总结

这套 Restic + MinIO 方案适合补齐 Homelab 的“文件级备份”短板：

| 需求 | 推荐做法 |
| --- | --- |
| PVE VM 整机恢复 | PBS / vzdump |
| 配置文件、Compose、脚本 | Restic 文件级备份 🏆 |
| 数据库 | 先 dump，再 Restic |
| 快速误删恢复 | ZFS snapshot / Restic restore |
| 防硬盘损坏 | 备份到独立物理设备 |
| 防本地灾难 | 再同步一份异地 |

一句话：**快照负责回滚，Restic 负责可移植恢复，异地副本负责兜底。** Homelab 数据量可以不大，但恢复链路一定要完整；否则备份只是心理安慰。