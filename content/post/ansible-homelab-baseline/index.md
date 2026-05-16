---
title: "别再裸奔 SSH：用 Ansible 把 Homelab 基线安全和自动化巡检一次做扎实"
description: "从控制机到多台 Linux 节点，用 Ansible 建立 Homelab 运维基线：SSH 加固、自动更新、日志限额、防火墙、巡检脚本与常见踩坑，适合 PVE 虚拟机、NAS、Docker 主机统一管理。"
date: 2026-05-16T09:00:56+08:00
image:
math:
license:
hidden: false
comments: true
draft: false
slug: "ansible-homelab-baseline"
---

## 前言：Homelab 机器越多，手工 SSH 越不靠谱

Homelab 刚开始通常只有一台 PVE：登录后台、开几台虚拟机、跑几个 Docker 服务，一切都还可控。等服务慢慢变多后，问题就会暴露出来：

- 每台 Linux VM 的用户、SSH 配置、防火墙策略不一致；
- 有的机器开了密码登录，有的机器还允许 root 直接登录；
- Docker 日志没有限额，某个容器异常刷日志把系统盘写满；
- 安全更新靠想起来才手动执行，半年不重启也没人知道；
- 想确认所有节点磁盘、内存、内核版本，只能一台台 SSH 上去敲命令。

这篇文章不讲 PVE 核显 SR-IOV，也不重复 vGPU 直通、反向代理或监控栈，而是补上 Homelab 里很容易被忽略的一层：**用 Ansible 给所有 Linux 节点建立可重复执行的运维基线**。

目标很明确：不用引入复杂平台，只用 SSH + YAML，就能完成 SSH 加固、基础软件安装、自动安全更新、日志限额、UFW 防火墙和批量巡检。

---

## 1. 方案设计：把“手工经验”变成可重复执行的 Playbook

Ansible 的优势是无 Agent：被管节点只要能 SSH 登录，并且有 Python，就能执行任务。对于 Homelab 来说，这比上来就部署 K8s、SaltStack 或 Puppet 轻得多。

| 方案 | 优点 | 缺点 | 适合场景 |
|------|------|------|----------|
| 手工 SSH | 上手最快 | 容易漏步骤、不可追踪、不可复现 | 1-2 台临时机器 |
| Shell 脚本 | 简单直接 | 幂等性差，错误处理麻烦 | 单机初始化 |
| Ansible | 🏆 幂等、可分组、可复用 | 需要维护 inventory 和变量 | 3 台以上 Homelab 节点 |
| Terraform | 资源编排强 | 更偏基础设施创建，不擅长系统内配置 | 云资源 / VM 生命周期管理 |

本文示例拓扑如下：

| 主机名 | IP | 角色 |
|--------|----|------|
| `pve` | `192.168.10.2` | Proxmox VE 宿主机，只做巡检，不随意改配置 |
| `docker01` | `192.168.10.21` | Docker 应用主机 |
| `nas01` | `192.168.10.31` | NAS / 文件服务 |
| `ha01` | `192.168.10.41` | Home Assistant 或其他 IoT 服务 |

> 建议：PVE 宿主机不要一上来就套用通用安全加固角色。PVE 自己有防火墙、集群、存储和网络管理逻辑，错误改 SSH 或网络可能导致管理面不可用。本文会把 PVE 放进单独分组，只跑只读巡检。

---

## 2. 控制机准备：安装 Ansible 与 SSH Key

控制机可以是你的笔记本，也可以是一台专门的运维 VM。Debian / Ubuntu 上安装：

```bash
sudo apt update
sudo apt install -y ansible sshpass python3-pip
ansible-galaxy collection install community.general
ansible --version
```

生成专用 SSH Key，不建议复用个人主力密钥：

```bash
ssh-keygen -t ed25519 -C "ansible-homelab" -f ~/.ssh/ansible_homelab
```

把公钥分发到被管节点。首次初始化时如果还只能密码登录，可以临时用 `ssh-copy-id`：

```bash
ssh-copy-id -i ~/.ssh/ansible_homelab.pub admin@192.168.10.21
ssh-copy-id -i ~/.ssh/ansible_homelab.pub admin@192.168.10.31
ssh-copy-id -i ~/.ssh/ansible_homelab.pub admin@192.168.10.41
```

如果节点是最小化安装，先确认 Python 存在：

```bash
ssh -i ~/.ssh/ansible_homelab admin@192.168.10.21 'python3 --version || sudo apt install -y python3'
```

---

## 3. 项目结构与 Inventory

创建一个独立目录，后续所有变更都放进 Git：

```bash
mkdir -p ~/homelab-ansible/{group_vars,roles,playbooks,scripts}
cd ~/homelab-ansible
```

`inventory.yml` 示例：

```yaml
all:
  vars:
    ansible_user: admin
    ansible_ssh_private_key_file: ~/.ssh/ansible_homelab
    ansible_become: true
    ansible_become_method: sudo
  children:
    linux_nodes:
      hosts:
        docker01:
          ansible_host: 192.168.10.21
        nas01:
          ansible_host: 192.168.10.31
        ha01:
          ansible_host: 192.168.10.41
    pve_nodes:
      hosts:
        pve:
          ansible_host: 192.168.10.2
          ansible_user: root
```

先做连通性测试：

```bash
ansible -i inventory.yml linux_nodes -m ping
ansible -i inventory.yml all -m setup -a 'filter=ansible_distribution*'
```

如果 `ping` 成功，输出类似：

```text
docker01 | SUCCESS => {"changed": false, "ping": "pong"}
nas01    | SUCCESS => {"changed": false, "ping": "pong"}
```

---

## 4. 基线变量：把策略写清楚

创建 `group_vars/linux_nodes.yml`：

```yaml
baseline_packages:
  - curl
  - wget
  - vim
  - htop
  - iftop
  - iotop
  - net-tools
  - ca-certificates
  - gnupg
  - unattended-upgrades
  - ufw
  - fail2ban

ssh_port: 22
allowed_tcp_ports:
  - 22
  - 80
  - 443

admin_users:
  - name: admin
    groups: "sudo,docker"
    shell: /bin/bash
```

这里有两个原则：

1. **变量集中管理**：端口、软件包、用户策略不要散落在多个 playbook；
2. **最小可用**：先做安全基线，不要一开始就把 Docker、数据库、反代、监控全部混在一个角色里。

---

## 5. 核心 Playbook：系统更新、SSH 加固、UFW、日志限额

创建 `playbooks/baseline.yml`：

```yaml
---
- name: Apply Homelab Linux baseline
  hosts: linux_nodes
  gather_facts: true

  tasks:
    - name: Install baseline packages
      ansible.builtin.apt:
        name: "{{ baseline_packages }}"
        state: present
        update_cache: true
        cache_valid_time: 3600
      when: ansible_os_family == "Debian"

    - name: Ensure sudo group can sudo without password for automation user
      ansible.builtin.copy:
        dest: /etc/sudoers.d/90-homelab-admin
        content: "%sudo ALL=(ALL:ALL) NOPASSWD:ALL\n"
        owner: root
        group: root
        mode: "0440"
        validate: "visudo -cf %s"

    - name: Harden sshd config
      ansible.builtin.copy:
        dest: /etc/ssh/sshd_config.d/99-homelab-baseline.conf
        owner: root
        group: root
        mode: "0644"
        content: |
          PermitRootLogin no
          PasswordAuthentication no
          PubkeyAuthentication yes
          X11Forwarding no
          ClientAliveInterval 300
          ClientAliveCountMax 2
      notify: Restart ssh

    - name: Configure unattended upgrades
      ansible.builtin.copy:
        dest: /etc/apt/apt.conf.d/20auto-upgrades
        owner: root
        group: root
        mode: "0644"
        content: |
          APT::Periodic::Update-Package-Lists "1";
          APT::Periodic::Unattended-Upgrade "1";
          APT::Periodic::AutocleanInterval "7";

    - name: Set journald disk limit
      ansible.builtin.copy:
        dest: /etc/systemd/journald.conf.d/99-homelab-limit.conf
        owner: root
        group: root
        mode: "0644"
        content: |
          [Journal]
          SystemMaxUse=500M
          RuntimeMaxUse=100M
          MaxRetentionSec=14day
      notify: Restart journald

    - name: Check whether Docker is installed
      ansible.builtin.stat:
        path: /usr/bin/docker
      register: docker_binary

    - name: Ensure Docker config directory exists
      ansible.builtin.file:
        path: /etc/docker
        state: directory
        owner: root
        group: root
        mode: "0755"
      when: docker_binary.stat.exists

    - name: Configure Docker daemon log rotation if Docker exists
      ansible.builtin.copy:
        dest: /etc/docker/daemon.json
        owner: root
        group: root
        mode: "0644"
        content: |
          {
            "log-driver": "json-file",
            "log-opts": {
              "max-size": "50m",
              "max-file": "3"
            },
            "live-restore": true
          }
      when: docker_binary.stat.exists
      notify: Restart docker

    - name: Allow required TCP ports in UFW
      community.general.ufw:
        rule: allow
        port: "{{ item }}"
        proto: tcp
      loop: "{{ allowed_tcp_ports }}"

    - name: Enable UFW with deny incoming policy
      community.general.ufw:
        state: enabled
        policy: deny
        direction: incoming

  handlers:
    - name: Restart ssh
      ansible.builtin.service:
        name: ssh
        state: restarted

    - name: Restart journald
      ansible.builtin.service:
        name: systemd-journald
        state: restarted

    - name: Restart docker
      ansible.builtin.service:
        name: docker
        state: restarted
      failed_when: false
```

上面这段配置做了几件关键事情：

- 禁止 root SSH 和密码登录，只允许密钥；
- 自动安装常用排障工具；
- 开启安全更新；
- 限制 journald 日志占用；
- 给 Docker 日志加 `max-size` 和 `max-file`，避免容器刷爆系统盘；
- UFW 默认拒绝入站，只开放明确端口。

执行前先 dry-run：

```bash
ansible-playbook -i inventory.yml playbooks/baseline.yml --check --diff
```

确认无误后再真正执行：

```bash
ansible-playbook -i inventory.yml playbooks/baseline.yml --diff
```

---

## 6. 更安全的执行顺序：先保留一个“逃生窗口”

SSH 加固最容易把自己锁在门外。我的建议是第一次执行时分两步：

1. 先创建用户、公钥、sudoers，不改 `PasswordAuthentication`；
2. 新开一个终端，用密钥确认能登录；
3. 再禁用密码登录和 root 登录；
4. 不要关闭旧 SSH 会话，直到新会话验证成功。

可以临时用 ad-hoc 命令验证：

```bash
ansible -i inventory.yml linux_nodes -m command -a 'whoami'
ansible -i inventory.yml linux_nodes -m command -a 'sudo -n true'
ansible -i inventory.yml linux_nodes -m command -a 'ss -tlnp | grep :22'
```

如果你改了 SSH 端口，比如 `2222`，Inventory 也要同步：

```yaml
docker01:
  ansible_host: 192.168.10.21
  ansible_port: 2222
```

---

## 7. 批量巡检：不要等告警响了才看机器状态

基础加固完成后，可以再加一个只读巡检 Playbook。创建 `playbooks/audit.yml`：

```yaml
---
- name: Homelab quick audit
  hosts: all
  gather_facts: true
  become: false

  tasks:
    - name: Show system summary
      ansible.builtin.debug:
        msg:
          host: "{{ inventory_hostname }}"
          ip: "{{ ansible_default_ipv4.address | default('unknown') }}"
          distro: "{{ ansible_distribution }} {{ ansible_distribution_version }}"
          kernel: "{{ ansible_kernel }}"
          uptime_days: "{{ (ansible_uptime_seconds / 86400) | round(1) }}"
          mem_total_mb: "{{ ansible_memtotal_mb }}"

    - name: Disk usage
      ansible.builtin.command: df -h /
      changed_when: false
      register: root_disk

    - name: Print root disk
      ansible.builtin.debug:
        var: root_disk.stdout_lines

    - name: Check failed systemd units
      ansible.builtin.command: systemctl --failed --no-pager
      changed_when: false
      failed_when: false
      register: failed_units

    - name: Print failed units
      ansible.builtin.debug:
        var: failed_units.stdout_lines
```

执行：

```bash
ansible-playbook -i inventory.yml playbooks/audit.yml
```

你也可以用 cron 每天跑一次，把结果写入日志：

```bash
mkdir -p ~/homelab-ansible/logs
cat > ~/homelab-ansible/scripts/nightly-audit.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
cd "$HOME/homelab-ansible"
mkdir -p logs
ansible-playbook -i inventory.yml playbooks/audit.yml \
  > "logs/audit-$(date +%F).log" 2>&1
find logs -name 'audit-*.log' -mtime +30 -delete
EOF
chmod +x ~/homelab-ansible/scripts/nightly-audit.sh

(crontab -l 2>/dev/null; echo '30 2 * * * /home/admin/homelab-ansible/scripts/nightly-audit.sh') | crontab -
```

如果你已经部署了 Prometheus + Grafana，可以把这个巡检当作补充：监控负责持续指标，Ansible 负责配置一致性和批量状态确认。

---

## 8. Docker 主机额外基线：Compose 目录、备份标签和网络命名

Docker 主机最怕“随手起容器”。建议统一约定：

```text
/opt/compose/<stack>/docker-compose.yml
/opt/compose/<stack>/.env
/opt/data/<stack>/
```

用 Ansible 创建目录：

```yaml
- name: Prepare Docker compose layout
  hosts: docker01
  become: true
  tasks:
    - name: Create compose and data directories
      ansible.builtin.file:
        path: "{{ item }}"
        state: directory
        owner: admin
        group: admin
        mode: "0755"
      loop:
        - /opt/compose
        - /opt/data
```

Compose 文件里给需要备份的数据卷加清晰标签或目录名，例如：

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    restart: unless-stopped
    volumes:
      - /opt/data/uptime-kuma:/app/data
    networks:
      - proxy
    labels:
      homelab.backup: "true"
      homelab.owner: "ops"

networks:
  proxy:
    external: true
```

这样后续不管是用 Restic 做文件级备份，还是接入 Traefik、监控告警，都能通过目录和标签快速定位。

---

## 9. 踩坑记录

### ❌ 坑 1：直接禁用密码登录，结果密钥没生效

现象：Playbook 执行成功后，新 SSH 会话登录失败，旧会话一断就只能接显示器或进 PVE Console。

修复步骤：

```bash
# 在旧会话中检查 authorized_keys 权限
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

# 检查 sshd 实际配置
sudo sshd -T | grep -E 'passwordauthentication|permitrootlogin|pubkeyauthentication'
```

第一次加固时不要同时关闭旧终端，这是最重要的逃生窗口。

### ❌ 坑 2：UFW 开启后 Docker 端口暴露行为不符合预期

Docker 会操作 iptables，某些情况下容器映射端口可能绕过 UFW 的直觉规则。解决思路：

- 对外服务尽量只通过反向代理入口暴露；
- 内部服务绑定到 `127.0.0.1` 或内网地址；
- 对真正需要公网访问的端口，在路由器、防火墙、反代三层都明确限制。

检查当前监听：

```bash
sudo ss -tulpen
sudo iptables -S DOCKER-USER
```

需要更强约束时，可以在 `DOCKER-USER` 链里加白名单策略，而不是只依赖 UFW。

### ❌ 坑 3：Ansible 变量写得太死，换一台机器就报错

例如所有机器都套 `groups: sudo,docker`，但 NAS 节点没安装 Docker，用户组不存在就会失败。解决方法是分组：

```yaml
children:
  docker_hosts:
    hosts:
      docker01:
        ansible_host: 192.168.10.21
  storage_hosts:
    hosts:
      nas01:
        ansible_host: 192.168.10.31
```

然后只对 `docker_hosts` 执行 Docker 相关任务。

### ❌ 坑 4：PVE 宿主机被当普通 Debian 改坏

PVE 虽然基于 Debian，但它不是普通 Debian。不要随意对 PVE 宿主机批量套用：

- 网络配置重写；
- 防火墙默认策略；
- unattended-upgrades 自动重启；
- Docker 安装脚本；
- 内核参数模板。

PVE 更适合单独写 `pve_audit.yml` 做只读巡检，真正改动前先快照配置并确认影响面。

---

## 10. 最佳实践总结

| 场景 | 推荐做法 | 理由 |
|------|----------|------|
| 3 台以内临时环境 | Shell + 文档 | 成本最低 |
| 3 台以上长期 Homelab | 🏆 Ansible Inventory + Playbook | 配置可复现、可审计 |
| PVE 宿主机 | 只读巡检优先 | 避免自动化误改管理面 |
| Docker 主机 | 日志限额 + 目录约定 + 标签 | 避免磁盘爆满，方便备份和迁移 |
| SSH 加固 | 先验证密钥，再禁密码 | 防止把自己锁出去 |
| 防火墙 | 默认拒绝入站，只开放必要端口 | 降低横向移动风险 |

一句话总结：**Homelab 的自动化不是为了炫技，而是为了让每一次初始化、加固、巡检都可重复、可回滚、可解释。**

如果你已经搭好了反向代理、监控栈、备份系统，那么下一步就应该把这些“靠记忆维护的配置”沉淀成 Ansible Playbook。机器坏了可以重装，配置丢了才是真正麻烦。

关联阅读：

- [Homelab 必备：Traefik 反向代理 + SSL 证书自动化部署指南](/p/traefik-reverse-proxy-homelab/)
- [Homelab 监控体系：Prometheus + Grafana + Node Exporter + Alertmanager 实战](/p/homelab-monitoring-stack-guide/)
- [别只会快照：用 Restic + MinIO 给 Homelab 做一套可恢复的文件级备份](/p/restic-minio-homelab-backup/)
