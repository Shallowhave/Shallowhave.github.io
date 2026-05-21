---
title: "别把 K8s 想太重：Homelab 用 K3s + MetalLB + NFS 搭一套真正可用的小集群"
description: "在 Proxmox VE 的 Homelab 环境中，用 K3s 搭建轻量 Kubernetes 集群，配合 MetalLB 提供内网 LoadBalancer，使用 NFS Subdir External Provisioner 实现动态持久化存储，并记录部署、验证、备份与踩坑经验。"
date: 2026-05-21T01:01:49+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
slug: "k3s-metallb-nfs-homelab"
---

## 前言：Homelab 里要不要上 Kubernetes？

很多 Homelab 玩家都会经历一个阶段：一开始所有服务都跑在单机 Docker Compose 里，后来服务越来越多，开始出现这些问题：

- 哪些服务该放在同一台机器上，靠手工记忆；
- 某台 Docker 主机维护或重装时，一堆服务要手动迁移；
- 反向代理、证书、数据库、定时任务散落在不同目录；
- 服务之间的启动顺序、健康检查、持久化目录没有统一规范；
- 想练 Kubernetes，但又不想上来就维护一套又重又复杂的生产集群。

这时候 **K3s** 很适合作为 Homelab 的 Kubernetes 入门方案。它不是“缩水版玩具”，而是把 Kubernetes 的控制面、组件打包得更轻，默认带 CoreDNS、Traefik、ServiceLB 等组件，适合边缘节点、实验环境和小规模集群。

不过在 Homelab 裸金属/虚拟化环境里，K3s 默认配置并不一定最适合：

- 默认 ServiceLB 会用宿主机端口暴露 `LoadBalancer`，和已有反向代理容易冲突；
- 默认 Traefik 版本和配置由 K3s 管理，后期想精细控制反而麻烦；
- Kubernetes 在公有云里有云厂商 LoadBalancer，在家里没有；
- 持久化存储不能只靠 `hostPath`，否则 Pod 一调度就找不到数据。

本文就搭一套更适合 Homelab 的小集群：

> **K3s 负责 Kubernetes 控制面，MetalLB 负责内网 LoadBalancer，NFS Subdir External Provisioner 负责动态 PVC。**

官方文档参考：K3s 网络组件说明见 `https://docs.k3s.io/networking/networking-services`，MetalLB 安装方式见 `https://metallb.universe.tf/installation/`，NFS Subdir External Provisioner 项目说明见 `https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner`。

---

## 1. 目标架构

假设 Homelab 已经有一台 Proxmox VE，另外有 NAS 或一台 Linux 机器可以提供 NFS 共享。本文用 3 台 Debian/Ubuntu VM 搭一个轻量集群：

| 节点 | IP | 角色 | 推荐配置 |
|------|----|------|----------|
| `k3s-master-01` | `192.168.10.41` | K3s server / 控制面 | 2C / 4G / 40G |
| `k3s-worker-01` | `192.168.10.42` | K3s agent / 工作节点 | 2C / 4G / 40G |
| `k3s-worker-02` | `192.168.10.43` | K3s agent / 工作节点 | 2C / 4G / 40G |
| `nas01` | `192.168.10.30` | NFS Server | 已有 NAS / Linux |
| MetalLB 地址池 | `192.168.10.240-192.168.10.250` | Service LoadBalancer IP | 路由器 DHCP 之外 |

整体链路如下：

```text
                      ┌────────────────────────────┐
                      │        Home Router          │
                      │  LAN: 192.168.10.0/24       │
                      └─────────────┬──────────────┘
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          │                         │                         │
┌─────────▼─────────┐     ┌─────────▼─────────┐     ┌─────────▼─────────┐
│ k3s-master-01     │     │ k3s-worker-01     │     │ k3s-worker-02     │
│ 192.168.10.41     │     │ 192.168.10.42     │     │ 192.168.10.43     │
│ server            │     │ agent             │     │ agent             │
└─────────┬─────────┘     └─────────┬─────────┘     └─────────┬─────────┘
          │                         │                         │
          └────────── Kubernetes Pod / Service Network ───────┘
                                    │
                          ┌─────────▼─────────┐
                          │ MetalLB L2 VIP    │
                          │ 192.168.10.240-250│
                          └─────────┬─────────┘
                                    │
                          ┌─────────▼─────────┐
                          │ NFS / NAS         │
                          │ 192.168.10.30     │
                          └───────────────────┘
```

为什么不用默认 ServiceLB？因为 Homelab 里通常已经有 Traefik、Nginx Proxy Manager、Caddy 或软路由端口转发。K3s 默认 ServiceLB 会在节点上占用宿主机端口，多个服务暴露 80/443 时容易打架。MetalLB 的 L2 模式更像“给 Service 分配一个真实内网 IP”，清晰很多。

---

## 2. PVE 虚拟机准备

K3s 对资源要求不高，但 Kubernetes 对网络稳定性比较敏感。建议在 PVE 里按下面原则建 VM：

| 项目 | 建议 | 原因 |
|------|------|------|
| CPU Type | `host` | 减少虚拟化指令损耗 |
| 网卡 | VirtIO / bridge 到业务 VLAN | 性能好，网络路径清晰 |
| 磁盘控制器 | VirtIO SCSI single | 性能和兼容性较好 |
| QEMU Guest Agent | 启用 | 方便 PVE 获取 IP、优雅关机 |
| 系统 | Debian 12 / Ubuntu 22.04+ | 软件源稳定，K3s 支持好 |
| IP | 静态 IP 或 DHCP 保留 | 节点 IP 变化会影响集群 |

在每台 VM 内先做基础初始化：

```bash
sudo apt update
sudo apt install -y curl ca-certificates gnupg nfs-common open-iscsi qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```

关闭 swap。Kubernetes 不喜欢节点内存被 swap 偷偷换出，轻则性能抖动，重则 kubelet 报警：

```bash
sudo swapoff -a
sudo sed -i.bak '/ swap / s/^/#/' /etc/fstab
free -h
```

加载常用内核模块并写入 sysctl：

```bash
cat <<'EOF' | sudo tee /etc/modules-load.d/k3s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<'EOF' | sudo tee /etc/sysctl.d/99-k3s-homelab.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

如果你使用 UFW 或上游防火墙，至少要确保节点之间这些端口互通：

| 端口 | 协议 | 用途 |
|------|------|------|
| `6443` | TCP | Kubernetes API Server |
| `8472` | UDP | Flannel VXLAN 默认后端 |
| `10250` | TCP | kubelet metrics / exec / logs |
| `2379-2380` | TCP | 嵌入式 etcd，仅多 server HA 需要 |
| `2049` | TCP/UDP | NFS |

单网段 Homelab 里可以先保证 K3s 三台节点互通：

```bash
ping -c 3 192.168.10.41
ping -c 3 192.168.10.42
ping -c 3 192.168.10.43
```

---

## 3. 安装 K3s：先关掉默认 Traefik 和 ServiceLB

K3s 默认会安装 Traefik Ingress Controller 和 ServiceLB。对于纯测试这很方便，但 Homelab 长期使用建议禁用，后续自己部署 Ingress 和 MetalLB。

在 `k3s-master-01` 执行：

```bash
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_EXEC="server \
    --disable traefik \
    --disable servicelb \
    --write-kubeconfig-mode 644 \
    --node-ip 192.168.10.41 \
    --advertise-address 192.168.10.41" \
  sh -
```

确认服务状态：

```bash
sudo systemctl status k3s --no-pager
kubectl get nodes -o wide
kubectl get pods -A
```

拿到 worker 加入集群需要的 token：

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

在 `k3s-worker-01` 执行，替换 `K3S_TOKEN`：

```bash
export K3S_TOKEN='K10xxxx::server:xxxx'

curl -sfL https://get.k3s.io | \
  K3S_URL="https://192.168.10.41:6443" \
  K3S_TOKEN="$K3S_TOKEN" \
  INSTALL_K3S_EXEC="agent --node-ip 192.168.10.42" \
  sh -
```

在 `k3s-worker-02` 执行：

```bash
export K3S_TOKEN='K10xxxx::server:xxxx'

curl -sfL https://get.k3s.io | \
  K3S_URL="https://192.168.10.41:6443" \
  K3S_TOKEN="$K3S_TOKEN" \
  INSTALL_K3S_EXEC="agent --node-ip 192.168.10.43" \
  sh -
```

回到 master 验证：

```bash
kubectl get nodes -o wide
```

正常应该看到类似：

```text
NAME            STATUS   ROLES                  AGE   VERSION        INTERNAL-IP
k3s-master-01   Ready    control-plane,master   5m    v1.30.x+k3s1   192.168.10.41
k3s-worker-01   Ready    <none>                 2m    v1.30.x+k3s1   192.168.10.42
k3s-worker-02   Ready    <none>                 2m    v1.30.x+k3s1   192.168.10.43
```

如果你在个人电脑上操作集群，可以把 kubeconfig 拷出来：

```bash
mkdir -p ~/.kube
scp root@192.168.10.41:/etc/rancher/k3s/k3s.yaml ~/.kube/k3s-homelab.yaml
sed -i 's/127.0.0.1/192.168.10.41/g' ~/.kube/k3s-homelab.yaml
export KUBECONFIG=~/.kube/k3s-homelab.yaml
kubectl get nodes
```

---

## 4. 安装 Helm

后续安装 NFS Provisioner、Ingress Controller、监控组件都可以用 Helm，先在管理机或 master 上安装：

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

给常用 namespace 建好：

```bash
kubectl create namespace metallb-system
kubectl create namespace storage
kubectl create namespace ingress
kubectl create namespace demo
```

如果 namespace 已存在会报错，可以改用幂等写法：

```bash
kubectl create namespace demo --dry-run=client -o yaml | kubectl apply -f -
```

---

## 5. MetalLB：给 Service 分配真实内网 IP

### 5.1 规划地址池

MetalLB L2 模式本质上是让集群里的 speaker 通过二层网络响应 ARP/NDP，把某个 LoadBalancer IP 宣告出去。因此地址池必须满足：

- 和客户端在同一个二层网络或能被正确路由；
- 不在路由器 DHCP 自动分配范围里；
- 不与现有 NAS、PVE、打印机、摄像头等静态 IP 冲突；
- 最好在路由器里备注或保留。

假设路由器 DHCP 范围是 `192.168.10.100-192.168.10.199`，那我把 MetalLB 放到 `192.168.10.240-192.168.10.250`。

### 5.2 安装 MetalLB

使用官方 manifest 安装：

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
kubectl -n metallb-system get pods -o wide
```

等待 controller 和 speaker 都 Ready：

```bash
kubectl -n metallb-system wait --for=condition=Ready pod --all --timeout=180s
```

创建地址池和 L2Advertisement：

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: homelab-lan-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.10.240-192.168.10.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: homelab-l2
  namespace: metallb-system
spec:
  ipAddressPools:
    - homelab-lan-pool
EOF
```

### 5.3 用 whoami 测试 LoadBalancer

部署一个最小 HTTP 服务：

```bash
cat <<'EOF' | kubectl apply -n demo -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
spec:
  replicas: 2
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
        - name: whoami
          image: traefik/whoami:v1.10
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: whoami-lb
spec:
  type: LoadBalancer
  selector:
    app: whoami
  ports:
    - name: http
      port: 80
      targetPort: 80
EOF
```

查看分配到的 IP：

```bash
kubectl -n demo get svc whoami-lb -o wide
```

输出类似：

```text
NAME        TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)
whoami-lb   LoadBalancer   10.43.100.20    192.168.10.240   80:31234/TCP
```

从同网段任意机器访问：

```bash
curl http://192.168.10.240
```

能看到 `Hostname`、`IP`、`RemoteAddr` 等信息，就说明 MetalLB 工作正常。

---

## 6. NFS 动态存储：别让 Pod 绑定死在某台节点

Kubernetes 里最容易被忽略的是存储。很多教程为了简单直接用 `hostPath`，但它有个致命问题：Pod 一旦被调度到另一台节点，原来的路径和数据就不在了。

Homelab 没必要一上来就 Ceph、Longhorn、Rook 全家桶。已有 NAS 的情况下，NFS 是最轻量、最容易理解的持久化方案。

### 6.1 在 NAS / Linux 上准备 NFS 共享

如果你的 NAS 有 Web UI，可以直接创建共享目录并允许 K3s 节点访问。这里用 Debian 作为 NFS Server 示例：

```bash
sudo apt update
sudo apt install -y nfs-kernel-server
sudo mkdir -p /srv/nfs/k3s
sudo chown nobody:nogroup /srv/nfs/k3s
sudo chmod 0777 /srv/nfs/k3s
```

配置 `/etc/exports`：

```bash
cat <<'EOF' | sudo tee /etc/exports.d/k3s.exports
/srv/nfs/k3s 192.168.10.0/24(rw,sync,no_subtree_check,no_root_squash)
EOF

sudo exportfs -rav
sudo systemctl enable --now nfs-server
showmount -e 127.0.0.1
```

在每个 K3s 节点上测试挂载：

```bash
sudo apt install -y nfs-common
sudo mkdir -p /mnt/k3s-test
sudo mount -t nfs 192.168.10.30:/srv/nfs/k3s /mnt/k3s-test
sudo touch /mnt/k3s-test/from-$(hostname)
ls -l /mnt/k3s-test
sudo umount /mnt/k3s-test
```

如果这里都挂不上，不要急着装 Helm Chart，先排查 NFS、防火墙和 NAS 权限。

### 6.2 安装 NFS Subdir External Provisioner

这个 provisioner 会把每个 PVC 自动创建成 NFS 共享目录里的子目录，命名类似 `${namespace}-${pvcName}-${pvName}`，适合 Homelab 的轻量动态存储。

安装：

```bash
helm repo add nfs-subdir-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update

helm upgrade --install nfs-client \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --namespace storage \
  --set nfs.server=192.168.10.30 \
  --set nfs.path=/srv/nfs/k3s \
  --set storageClass.name=nfs-client \
  --set storageClass.defaultClass=true \
  --set storageClass.archiveOnDelete=true
```

检查：

```bash
kubectl -n storage get pods -o wide
kubectl get storageclass
```

应该看到 `nfs-client`，并且标记为默认 StorageClass：

```text
NAME                   PROVISIONER                                   RECLAIMPOLICY
nfs-client (default)   cluster.local/nfs-client-nfs-subdir-external-provisioner Delete
```

### 6.3 用 PVC 验证动态创建目录

创建一个 PVC 和测试 Pod：

```bash
cat <<'EOF' | kubectl apply -n demo -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-html
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: nfs-test
spec:
  containers:
    - name: shell
      image: busybox:1.36
      command: ["sh", "-c", "date > /data/index.html && sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: nginx-html
EOF
```

验证 PVC 绑定：

```bash
kubectl -n demo get pvc,pod
kubectl -n demo exec nfs-test -- cat /data/index.html
```

到 NAS 上看目录：

```bash
ls -lah /srv/nfs/k3s
find /srv/nfs/k3s -maxdepth 2 -type f -name index.html -print -exec cat {} \;
```

能看到文件，说明 Kubernetes PVC -> NFS 子目录 -> Pod 挂载整条链路打通。

---

## 7. 部署一个更接近真实场景的应用

下面部署一个带持久化目录的 `nginx`，用 PVC 存网页文件，用 MetalLB 暴露内网 IP。

```bash
cat <<'EOF' | kubectl apply -n demo -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-data
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-nfs
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-nfs
  template:
    metadata:
      labels:
        app: nginx-nfs
    spec:
      initContainers:
        - name: init-index
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              if [ ! -f /usr/share/nginx/html/index.html ]; then
                echo "Hello from K3s + MetalLB + NFS at $(date)" > /usr/share/nginx/html/index.html
              fi
          volumeMounts:
            - name: web-data
              mountPath: /usr/share/nginx/html
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
          volumeMounts:
            - name: web-data
              mountPath: /usr/share/nginx/html
      volumes:
        - name: web-data
          persistentVolumeClaim:
            claimName: web-data
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-nfs-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx-nfs
  ports:
    - name: http
      port: 80
      targetPort: 80
EOF
```

查看调度和 Service：

```bash
kubectl -n demo get pods -o wide
kubectl -n demo get svc nginx-nfs-lb
```

访问：

```bash
curl http://$(kubectl -n demo get svc nginx-nfs-lb -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

强制删除一个 Pod，验证 Kubernetes 自动拉起后数据仍在：

```bash
kubectl -n demo delete pod -l app=nginx-nfs
kubectl -n demo rollout status deploy/nginx-nfs
kubectl -n demo get pods -o wide
```

只要 PVC 正常绑定，Pod 换节点后仍能读到同一个 NFS 目录。

---

## 8. 运维命令：日常排障先看这些

Kubernetes 最大的问题不是部署，而是出问题时不知道看哪里。下面这组命令建议收藏。

### 8.1 节点状态

```bash
kubectl get nodes -o wide
kubectl describe node k3s-worker-01
kubectl top nodes   # 需要 metrics-server，K3s 通常默认带
```

### 8.2 Pod 为什么起不来

```bash
kubectl -n demo get pods -o wide
kubectl -n demo describe pod <pod-name>
kubectl -n demo logs <pod-name> --all-containers --tail=100
```

重点看 Events：

```bash
kubectl -n demo get events --sort-by='.lastTimestamp' | tail -30
```

### 8.3 PVC 为什么 Pending

```bash
kubectl -n demo get pvc
kubectl -n demo describe pvc nginx-html
kubectl get pv
kubectl -n storage logs deploy/nfs-client-nfs-subdir-external-provisioner --tail=100
```

常见原因：NFS Server 地址写错、节点没装 `nfs-common`、NAS 权限没放行、StorageClass 名字不匹配。

### 8.4 LoadBalancer 为什么没有 EXTERNAL-IP

```bash
kubectl -n metallb-system get pods -o wide
kubectl -n metallb-system logs deploy/controller --tail=100
kubectl -n metallb-system get ipaddresspool,l2advertisement
kubectl -n demo describe svc whoami-lb
```

如果 `EXTERNAL-IP` 一直 Pending，通常是 MetalLB 没有地址池或地址池配置 namespace/name 不对。如果有 IP 但访问不通，更多是二层网络、防火墙或 IP 冲突。

---

## 9. 备份策略：K3s 不等于不用备份

K3s server 默认使用 SQLite 存储集群状态，单 master Homelab 足够简单，但必须备份：

```bash
sudo mkdir -p /backup/k3s
sudo k3s etcd-snapshot save --name homelab-manual-$(date +%F-%H%M) 2>/dev/null || true
sudo tar -czf /backup/k3s/k3s-server-$(date +%F).tar.gz \
  /etc/rancher/k3s \
  /var/lib/rancher/k3s/server/db \
  /var/lib/rancher/k3s/server/manifests
```

注意：如果你是单 server 默认 SQLite，`k3s etcd-snapshot` 不适用，所以这里用 `tar` 备份 server DB 目录。更推荐把关键 Kubernetes YAML 也纳入 Git：

```bash
mkdir -p ~/homelab-k8s/{apps,infra,storage}
kubectl get deploy,svc,pvc,ingress -A -o yaml > ~/homelab-k8s/cluster-export.yaml
```

真正的数据在 NFS 上，所以还要备份 NAS 目录。例如用 `rsync` 每天同步到另一块盘：

```bash
rsync -aH --delete /srv/nfs/k3s/ /mnt/backup/k3s-nfs/
```

或者接入之前文章提到的 Restic/MinIO 这类备份体系。原则很简单：

| 对象 | 备份方式 | 恢复优先级 |
|------|----------|------------|
| K3s 配置 / token | tar + 离线保存 | 高 |
| Kubernetes YAML | Git 管理 | 高 |
| NFS 持久化数据 | NAS 快照 / rsync / restic | 最高 |
| 容器镜像 | 不建议备份，保留版本号即可 | 中 |

---

## 10. 踩坑记录

### ❌ 坑 1：MetalLB 地址池和 DHCP 冲突

现象：Service 已经拿到 `192.168.10.240`，但访问时偶尔通、偶尔不通，路由器 ARP 表里 MAC 地址还会变化。

根因：这个 IP 被 DHCP 分给了其他设备，MetalLB 又在宣告同一个 IP。

解决：

```bash
arp -a | grep 192.168.10.240
kubectl -n metallb-system get ipaddresspool -o yaml
```

把 MetalLB 地址池放到 DHCP 范围之外，并在路由器备注保留。不要偷懒直接用一段“看起来没人用”的 IP。

### ❌ 坑 2：K3s 默认 Traefik 占了 80/443

现象：你明明部署了自己的 Ingress 或反向代理，但 80/443 总是被奇怪的 Pod 接管。

根因：K3s 默认安装 Traefik；如果同时使用 ServiceLB，还可能在节点上创建 svclb Pod 绑定宿主机端口。

解决：安装 K3s 时加：

```bash
--disable traefik --disable servicelb
```

已经安装过的环境，可以查看：

```bash
kubectl -n kube-system get helmchart
kubectl -n kube-system get pods | grep -E 'traefik|svclb'
```

测试环境可以卸载重装；长期环境则要先确认现有 Ingress 资源是否依赖默认 Traefik。

### ❌ 坑 3：PVC 一直 Pending

现象：应用 Pod 起不来，`kubectl describe pod` 看到 `pod has unbound immediate PersistentVolumeClaims`。

排查顺序：

```bash
kubectl get storageclass
kubectl -n demo describe pvc <pvc-name>
kubectl -n storage logs deploy/nfs-client-nfs-subdir-external-provisioner --tail=100
showmount -e 192.168.10.30
```

常见原因有三个：

1. 没有默认 StorageClass，PVC 又没写 `storageClassName`；
2. NFS Server 路径不对；
3. K3s 节点没有安装 `nfs-common`。

### ❌ 坑 4：NFS 权限导致容器写入失败

现象：PVC Bound，Pod 也 Running，但应用日志报 `Permission denied`。

临时验证可以放宽权限：

```bash
sudo chmod 0777 /srv/nfs/k3s
```

但长期建议按应用 UID/GID 管理目录权限。例如很多容器使用 `1000:1000`：

```bash
sudo chown -R 1000:1000 /srv/nfs/k3s/demo-web-data-*/
```

如果 NAS 支持 ACL，也可以用 NAS 自己的权限系统处理。不要在不理解后果的情况下给所有共享都开 `no_root_squash`，本文只是为了 Homelab 快速打通链路。

### ❌ 坑 5：节点 IP 变化导致集群异常

K3s 节点 IP 如果通过 DHCP 随机变化，证书、Node InternalIP、Flannel 路由都可能异常。解决方式：

- 路由器上做 DHCP 静态租约；或
- VM 内配置静态 IP；
- 安装时显式指定 `--node-ip` 和 server 的 `--advertise-address`。

查看当前节点 IP：

```bash
kubectl get nodes -o wide
```

---

## 11. Homelab 场景下的最佳实践

| 场景 | 推荐配置 | 理由 |
|------|----------|------|
| 只想练 K8s | 单 master + 1 worker | 成本低，足够理解对象模型 |
| 长期跑轻量服务 | 1 master + 2 worker | 有基本调度冗余 |
| 需要高可用控制面 | 3 server + embedded etcd | 控制面可容忍单点故障 |
| 已有 NAS | NFS Provisioner | 简单、透明、容易备份 |
| 追求块存储和副本 | Longhorn / Ceph | 功能强，但复杂度明显上升 |
| 暴露内网服务 | MetalLB L2 | 比 NodePort 更直观 |
| 对外 HTTPS | Ingress + 反向代理 / DNS-01 | 统一证书和入口 |

我的建议是：**先不要一口气上 GitOps、Service Mesh、Ceph、三主高可用。** Homelab 的目标是稳定、可恢复、可理解。K3s + MetalLB + NFS 已经足够承载 Dashboard、Wiki、轻量监控、测试服务、内部 API 等场景。

等你真的遇到 NFS 性能瓶颈、Pod 调度隔离、灰度发布、Secret 管理等问题，再逐步引入 Longhorn、Argo CD、External Secrets、cert-manager，会比一开始堆满组件健康得多。

---

## 总结

这套方案的核心不是“为了 Kubernetes 而 Kubernetes”，而是给 Homelab 增加一层可调度、可声明、可迁移的应用运行环境。

| 维度 | 结论 |
|------|------|
| ✅ 集群方案 | K3s 单 server + 多 worker，轻量够用 |
| ✅ 网络暴露 | 禁用默认 ServiceLB，用 MetalLB 分配真实内网 IP |
| ✅ 持久化 | 使用 NFS Subdir External Provisioner 动态创建 PVC |
| ✅ 运维重点 | 固定节点 IP、规划地址池、备份 K3s 配置和 NFS 数据 |
| ⚠️ 不建议 | 一开始就堆 Ceph、Service Mesh、复杂 GitOps |

如果你已经有 Proxmox VE、NAS 和几台 Linux VM，这套 K3s 小集群基本就是 Homelab 进阶的下一块拼图：比 Docker Compose 更规范，比生产级 Kubernetes 更轻，出了问题也能靠普通 Linux、网络和存储知识排查清楚。
