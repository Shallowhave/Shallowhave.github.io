---
title: "代码即基础设施：用 Terraform 管理 Proxmox VE 虚拟机，告别手动创建"
description: "在 Homelab 中引入 Terraform 管理 Proxmox VE：从安装配置、Provider 认证、虚拟机声明式定义到 Cloud-Init 注入、批量部署与 State 管理，一套 HCL 模板跑通 VM 全生命周期，附踩坑实录与最佳实践。"
date: 2026-05-30T09:20:46+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
---

## 为什么 Homelab 也需要 IaC？

用 PVE Web UI 创建几台虚拟机不算麻烦——但是：

- 重装宿主机后要重新点十多遍鼠标，每台 VM 的资源规格、网络配置全得再来一次
- 想复制一套「测试环境」到另一台 PVE，靠手记配置，漏一个 VLAN 标签就要排查半天
- 团队成员（或多节点 Homelab 合伙人）各自创建 VM，磁盘格式、桥接网卡、CPU 类型从来对不上
- 没有变更历史——谁什么时候改了哪台 VM 的配置，无迹可寻

这些问题在 PVE Cluster 场景下尤其致命。**Terraform 的答案是：把基础设施当作代码来管理。** 一份 `.tf` 文件就是你的虚拟机「菜单」——可版本控制、可复用、可审计。

本文会用一套完整的 Homelab 案例，带你从零上手 Terraform + Proxmox VE。

---

## 1. 环境准备

### 1.1 安装 Terraform

PVE 宿主机或 Homelab 里的任意 Linux 机器都可以作为 Terraform 控制节点。推荐放在管理 VM 或 Docker 宿主机上。

```bash
# 以 Linux AMD64 为例，安装 Terraform 1.10.x
wget https://releases.hashicorp.com/terraform/1.10.5/terraform_1.10.5_linux_amd64.zip
unzip terraform_1.10.5_linux_amd64.zip
sudo mv terraform /usr/local/bin/
terraform --version
# 预期输出：Terraform v1.10.5
```

如果 Homelab 网络不便直连 HashiCorp 仓库，也可以从清华镜像站获取：

```bash
wget https://mirrors.tuna.tsinghua.edu.cn/hashicorp/terraform/1.10.5/terraform_1.10.5_linux_amd64.zip
```

### 1.2 PVE API Token 创建

Terraform 通过 PVE API 通信，需要创建一个专用的 API Token（比直接使用 root 密码更安全）：

```bash
# 在 PVE 宿主机上执行
pveum user add terraform@pve
pveum role add TerraformProv -privs "VM.Allocate VM.Clone VM.Config.Disk VM.Config.CPU VM.Config.Memory VM.Config.Network VM.Config.Options VM.Monitor VM.Audit VM.PowerMgmt Datastore.AllocateSpace Datastore.Audit"
pveum acl modify / -user terraform@pve -role TerraformProv
pveum user token add terraform@pve token1 --privsep=false
```

执行后会输出类似：

```
┌──────────────┬──────────────────────────────────────┐
│ key          │ value                                │
╞══════════════╪══════════════════════════════════════╡
│ full-tokenid │ terraform@pve!token1                 │
├──────────────┼──────────────────────────────────────┤
│ value        │ a1b2c3d4-e5f6-7890-abcd-ef1234567890 │
└──────────────┴──────────────────────────────────────┘
```

**务必保存 token value**，Terraform Provider 配置需要它，且 PVE Web UI 不会二次显示。

> **踩坑**：`--privsep=false` 如果不加，Token 默认是权限分离模式，在 Provider 中无法正常认证，会报 `401 Authentication failure`。如果已经创建了带权限分离的 Token，可以在 PVE Web UI → Permissions → API Tokens 中修改。

---

## 2. Provider 配置与项目初始化

创建项目目录并编写 Terraform 配置：

```bash
mkdir ~/terraform-pve && cd ~/terraform-pve
```

### main.tf

```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = ">= 0.66.0"
    }
  }
}

provider "proxmox" {
  endpoint  = var.pve_endpoint      # 如 https://192.168.1.10:8006/
  api_token = var.pve_api_token

  # PVE 默认使用自签名证书，需要跳过校验
  insecure  = true

  ssh {
    agent    = true
    username = "root"
  }
}
```

### variables.tf

```hcl
variable "pve_endpoint" {
  description = "Proxmox VE API endpoint"
  type        = string
}

variable "pve_api_token" {
  description = "Proxmox VE API token"
  type        = string
  sensitive   = true
}
```

### terraform.tfvars

```hcl
pve_endpoint  = "https://192.168.1.10:8006/"
pve_api_token = "terraform@pve!token1=a1b2c3d4-e5f6-7890-abcd-ef1234567890"
```

> **安全提示**：`terraform.tfvars` 包含 API Token——**不要提交到 Git**。将 `terraform.tfvars` 加入 `.gitignore`，改为创建 `terraform.tfvars.example` 给团队参考。

### terraform init

```bash
terraform init
```

正常输出会显示下载 bpg/proxmox Provider，并初始化后端。

---

## 3. 声明式定义虚拟机：从模板克隆 Ubuntu Server

以下配置基于已准备好的 Cloud-Init Ubuntu 24.04 模板（假设模板 ID 为 9000），Terraform 会自动克隆并注入配置：

```hcl
# ubuntu-vm.tf
resource "proxmox_virtual_environment_vm" "ubuntu_dev" {
  node_name = "pve1"
  vm_id     = 1001

  clone {
    vm_id = 9000
    full  = true
  }

  agent {
    enabled = true
  }

  cpu {
    cores   = 2
    sockets = 1
    type    = "host"
  }

  memory {
    dedicated = 4096
  }

  disk {
    datastore_id = "local-zfs"
    size         = 40
    interface    = "virtio0"
    discard      = "on"
  }

  network_device {
    bridge = "vmbr0"
    model  = "virtio"
  }

  # Cloud-Init 配置
  initialization {
    ip_config {
      ipv4 {
        address = "192.168.1.101/24"
        gateway = "192.168.1.1"
      }
    }
    dns {
      servers = ["192.168.1.2", "1.1.1.1"]
    }
    user_account {
      username = "deployer"
      password = random_password.vm_password.result
      keys     = [
        file("~/.ssh/id_ed25519.pub")
      ]
    }
  }

  operating_system {
    type = "l26"  # Linux 2.6+
  }

  tags = ["ubuntu", "dev", "homelab"]
}
```

> 用 `random_password` 资源生成随机密码，避免硬编码：
> ```hcl
> resource "random_password" "vm_password" {
>   length  = 24
>   special = true
> }
> ```

执行一下看看效果：

```bash
terraform plan      # 预览变更——「纸上谈兵」确认无误
terraform apply     # 执行创建，输入 yes 确认
```

大约 30–60 秒后，Terraform 会显示 Apply 完成：

```
Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

登录 PVE Web UI，虚拟机 1001 已创建并开机，IP 为 `192.168.1.101`，SSH 密钥已注入——全程无需人工干预。

---

## 4. 进阶：多台虚拟机批量部署

有了 Terraform，批量部署变得极其简单。用 `count` 循环即可：

```hcl
# k3s-vms.tf
resource "proxmox_virtual_environment_vm" "k3s_workers" {
  count     = 3
  node_name = "pve1"
  vm_id     = 1010 + count.index

  clone {
    vm_id = 9000
    full  = true
  }

  name = "k3s-worker-${count.index + 1}"

  agent { enabled = true }

  cpu {
    cores = 2
    type  = "host"
  }

  memory {
    dedicated = 4096
  }

  disk {
    datastore_id = "local-zfs"
    size         = 30
    interface    = "virtio0"
  }

  network_device {
    bridge = "vmbr0"
  }

  initialization {
    ip_config {
      ipv4 {
        address = "192.168.1.12${count.index + 1}/24"
        gateway = "192.168.1.1"
      }
    }
    user_account {
      username = "deployer"
      keys     = [file("~/.ssh/id_ed25519.pub")]
    }
  }

  tags = ["k3s", "worker"]
}
```

执行 `terraform plan` 看看效果：

```
  # proxmox_virtual_environment_vm.k3s_workers[0] will be created
  # proxmox_virtual_environment_vm.k3s_workers[1] will be created
  # proxmox_virtual_environment_vm.k3s_workers[2] will be created

Plan: 3 to add, 0 to change, 0 to destroy.
```

一条 `terraform apply`，三台 K3s Worker 节点规格一致、IP 连贯、SSH 密钥统一——交给自动化，比手动创建省 90% 的时间。

> **踩坑**：VM ID 不能重复。如果 PVE 上已有手动创建的 ID 1001，Terraform 会报 `vm ID already exists`。建议手动管理 ID 段：Terraform 管理的 VM 统一使用 1000–1999 段，手工创建的 VM 用 2000+ 段。

---

## 5. Terraform State：怎么管？

Terraform 使用 State 文件记录当前资源状态。默认是本地文件 `terraform.tfstate`。

**本地 State 的风险**：如果两台机器分别执行 `apply`，State 不一致会导致变更混乱。对于 Homelab，有两个常用方案：

### 方案 A：Git 管理（最简单）

```bash
# 将 State 加入 Git 跟踪
echo "terraform.tfvars" >> .gitignore
git add .
git commit -m "init: terraform pve infrastructure"
```

缺点是多人协作时容易冲突，但单人 Homelab 完全够用。

### 方案 B：S3 兼容存储（推荐）

如果有 MinIO 或 NAS 提供的 S3 服务：

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket                      = "terraform-state"
    key                         = "pve/terraform.tfstate"
    region                      = "us-east-1"
    endpoint                    = "https://minio.homelab.local:9000"
    skip_region_validation      = true
    skip_credentials_validation = true
    skip_requesting_account_id  = true
    skip_s3_checksum            = true
    use_path_style              = true
  }
}
```

切换后端后执行迁移：

```bash
terraform init -migrate-state
```

---

## 6. 生命周期操作

### 6.1 变配

想从 2 核 4G 升到 4 核 8G？改一行 HCL，执行 `terraform apply`：

```hcl
cpu {
  cores = 4  # 原来是 2
}
memory {
  dedicated = 8192  # 原来是 4096
}
```

CPU 在 `type = "host"` 时支持热添加，内存建议关机后再变配。

### 6.2 防止误删除

核心 VM（如 DNS 服务器、监控主机）可以加保护锁：

```hcl
resource "proxmox_virtual_environment_vm" "infra_dns" {
  lifecycle {
    prevent_destroy = true
  }
}
```

这样即便执行 `terraform destroy` 也不会删除这台 VM。

### 6.3 强制重建

```bash
terraform taint proxmox_virtual_environment_vm.ubuntu_dev
terraform apply
```

某些属性（如 `datastore_id`）修改后需要重建 VM，Terraform 会在 `plan` 中标注 `forces replacement`。

---

## 7. 完整项目结构

```
terraform-pve/
├── main.tf              # Provider + Terraform 配置
├── variables.tf         # 变量声明
├── terraform.tfvars     # 实际变量值（.gitignore）
├── outputs.tf           # 输出 VM IP、ID 等信息
├── ubuntu-vm.tf         # Ubuntu 开发机
├── k3s-vms.tf           # K3s 集群节点
├── .gitignore
└── README.md
```

建议进一步用 Git 分支管理不同环境（main → prod，dev → staging），配合 Git 标签做发布管理。

---

## 8. 常见踩坑与排障

| 问题 | 现象 | 解决方案 |
|------|------|---------|
| API 认证失败 | `401 Authentication failure` | Token 默认开启权限分离，重建时加 `--privsep=false` |
| VM ID 冲突 | `vm ID 1001 already exists` | 手动规划 ID 段，Terraform 用 1000–1999 段 |
| Cloud-Init 未生效 | VM 启动后 IP 不对或 SSH 连不上 | 确认模板已安装 `qemu-guest-agent` 和 `cloud-init` |
| State 锁定 | `Error acquiring state lock` | S3 后端时删除 `terraform-state` bucket 中的 lock 文件 |
| 网络不通 | 克隆后 VM 无法访问 | 检查模板的 `/etc/network/interfaces` 和 Cloud-Init ip_config 是否冲突 |
| Provider 版本 | `Invalid provider registry host` | Terraform ≥ 1.6，bpg/proxmox ≥ 0.66 |
| SSH Agent 失效 | `ssh: handshake failed` | 确认 `ssh-add -l` 有密钥，且已添加到 PVE 主机的 `~/.ssh/authorized_keys` |

---

## 9. 总结

把 Terraform 引入 Homelab 带来的核心收益：

1. **可复现**——重装 PVE 后一条 `terraform apply` 恢复所有 VM
2. **可审计**——`git diff` 一眼看清谁改了什么配置
3. **可批量**——`count` 循环一次建 N 台机器，规格、配置完全一致
4. **可销毁**——测试环境用完 `terraform destroy` 一键清理

但这只是 IaC 的起点。进阶方向：

- **Packer + Cloud-Init**：自动化制作模板镜像，彻底告别手动制作模板
- **Terraform Workspace**：多个 Homelab 环境（dev / staging / prod）的 State 隔离
- **GitOps**：Git Push → CI pipeline → `terraform apply` 全自动化
- **Terragrunt**：减少代码重复，管理多模块依赖

如果你的 PVE 上已经有 5 台以上的 VM，或者规划过重装宿主机后不想再手动配置一遍——是时候让 Terraform 接管了。
