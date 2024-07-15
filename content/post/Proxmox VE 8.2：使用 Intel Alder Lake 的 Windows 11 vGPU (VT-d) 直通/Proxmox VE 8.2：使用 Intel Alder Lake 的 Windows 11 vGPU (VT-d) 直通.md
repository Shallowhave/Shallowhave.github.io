---
title: Proxmox VE 8.2：使用 Intel Alder Lake 的 Windows 11 vGPU (VT-d) 直通
description: Proxmox VE 8.2：使用 Intel Alder Lake 的 Windows 11 vGPU (VT-d) 直通
date: 2024-02-21T13:07:05+08:00
image: ./img/pve.png
---

## GPU 直通与虚拟化

典型的 GPU 直通将整个 PCIe 图形设备传递给 VM。这意味着只有一台虚拟机可以使用 GPU 资源。这样做的优点是能够通过计算机的 HDMI 或 DisplayPort 端口将视频显示输出到外部显示器。但是，您只能使用一台虚拟机使用 GPU。如果您希望在台式计算机上使用 Proxmox，但又希望通过外部显示器充分利用主桌面操作系统的 GPU 资源，这会很有用。
## Proxmox 8.1/8.2 内核要求

默认情况下，Proxmox 8.2 安装 Linux 内核 6.8.x，提供 vGPU 功能的 DKMS 模块不支持该内核。要使用 Proxmox 8.2，您需要固定内核 6.5.13-3，因为该版本已知与 DKMS/vGPU 兼容。按照以下过程降级到并使用 Proxmox 8.2 固定内核 6.5.13-3。我还建议 Proxmox 8.1 用户也使用并固定 6.5.13-3 内核。

对于 Proxmox 8.1 和 8.2 用户运行以下命令：
```bash
apt update
apt install proxmox-headers-6.5.13-3-pve
apt install proxmox-kernel-6.5.13-3-pve-signed
proxmox-boot-tool kernel pin 6.5.13-3-pve
proxmox-boot-tool refresh
reboot
```

重新启动后，验证 Proxmox 是否使用内核 6.5.13-3。

## Proxmox 内核配置

>注意：这些命令会自动检测您正在运行的内核版本并相应地调整命令。正如前面提到的，对于 Proxmox 8.1 和 8.2，您应该使用并固定内核 6.5.13-5。

1. 在 Proxmox 主机上打开 shell 并运行以下命令。首先我们需要安装 Git、内核头文件，进行一些清理，然后使用正确的版本设置内核变量。

```bash
apt update && apt install git sysfsutils pve-headers mokutil -y
rm -rf /var/lib/dkms/i915-sriov-dkms*
rm -rf /usr/src/i915-sriov-dkms*
rm -rf ~/i915-sriov-dkms
KERNEL=$(uname -r); KERNEL=${KERNEL%-pve}
```

2. 现在我们需要克隆 DKMS 存储库并修改配置文件以设置内核版本。检查软件包名称是否为 i915-sriov-dkms 并且软件包版本与您的内核版本匹配。
```bash
cd ~
git clone https://github.com/strongtz/i915-sriov-dkms.git
cd ~/i915-sriov-dkms
cp -a ~/i915-sriov-dkms/dkms.conf{,.bak}
sed -i 's/"@_PKGBASE@"/"i915-sriov-dkms"/g' ~/i915-sriov-dkms/dkms.conf
sed -i 's/"@PKGVER@"/"'"$KERNEL"'"/g' ~/i915-sriov-dkms/dkms.conf
sed -i 's/ -j$(nproc)//g' ~/i915-sriov-dkms/dkms.conf
cat ~/i915-sriov-dkms/dkms.conf
```

3. 这里我们安装DKMS，链接内核源，并检查状态。验证内核是否显示为已添加。

```bash
apt install --reinstall dkms -y
dkms add .
cd /usr/src/i915-sriov-dkms-$KERNEL
dkms status
```

4. 现在让我们构建新内核并检查状态。验证它是否显示已安装
```bash
dkms install -m i915-sriov-dkms -v $KERNEL -k $(uname -r) --force -j 1
dkms status
```

5. 对于全新的 Proxmox 8.1 及更高版本安装，可能会启用安全启动。为了以防万一，我们需要加载 DKMS 密钥，以便内核加载该模块。运行以下命令，然后输入密码。该密码仅用于 MOK 设置，重启主机时将再次使用。之后就不需要密码了。它不需要与您用于 root 帐户的密码相同。

```bash
mokutil --import /var/lib/dkms/mok.pub
```

## Proxmox GRUB 配置

1. 如果您的 Proxmox 主机中没有 Google Coral PCIe TPU，请返回 Proxmox shell 运行以下命令。如果您这样做了，您就会知道，所以如果您不确定，请运行第一组命令。如果您的 Google Coral 是 USB，也请使用第一组命令。如果您的 Google Coral 是 PCIe 模块，请运行第二个命令块。
```bash
cp -a /etc/default/grub{,.bak}
sudo sed -i '/^GRUB_CMDLINE_LINUX_DEFAULT/c\GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt i915.enable_guc=3 i915.max_vfs=7"' /etc/default/grub
update-grub
update-initramfs -u -k all
apt install sysfsutils -y
```

>如果您的 Proxmox 主机确实有 Google Coral PCIe TPU，并且您使用 PCIe 直通 LXC 或 VM，请改用此命令。这将在 Proxmox 主机级别将 Coral 设备列入黑名单，以便您的 LXC/VM 可以获得独占访问权限。
>```bash
>cp -a /etc/default/grub{,.bak}
sudo sed -i '/^GRUB_CMDLINE_LINUX_DEFAULT/c\GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt i915.enable_guc=3 i915.max_vfs=7 initcall_blacklist=sysfb_init pcie_aspm=off"' /etc/default/grub
update-grub
update-initramfs -u -k all
apt install sysfsutils -y

2. 运行以下命令并根据需要修改 PCIe 总线编号。在本例中，我使用 00:02.0。要验证文件是否已修改，请 cat 该文件并确保它已被修改。
```bash
echo "devices/pci0000:00/0000:00:02.0/sriov_numvfs = 7" > /etc/sysfs.conf
cat /etc/sysfs.conf
```

