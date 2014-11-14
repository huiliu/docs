---
layout: docs
slug: pxe
title: 通过PXE来启动coreos
category: running_coreos
sub_category: bare_metal
supported: true
weight: 5
---

# 通过PXE来启动coreos

这份文档将指导你在物理机或虚拟机上通过来PXE来启动CoreOS。此文档中默认CoreOS是
完全运行于内存中。CoreOS也能被[安装至硬盘][disk]。

[disk]: {{site.url}}/docs/running-coreos/bare-metal/installing-to-disk

## Configuring pxelinux
## 配置pxelinux

这里假定你已经有一台运行着[pxelinux][pxelinux]的PXE服务器。如果你需要了解如何
配置PXE服务器，请参照[Debian][debian-pxe], [Fedora][fedora-pxe]或
[Ubuntu][ubuntu-pxe]的相应指南。

[debian-pxe]: http://www.debian-administration.org/articles/478
[ubuntu-pxe]: https://help.ubuntu.com/community/DisklessUbuntuHowto
[fedora-pxe]: http://docs.fedoraproject.org/en-US/Fedora/7/html/Installation_Guide/ap-pxe-server.html
[pxelinux]: http://www.syslinux.org/wiki/index.php/PXELINUX

### Setting up pxelinux.cfg
### 配置pxelinux.cfg

配置pxelinux.cfg中关于CoreOS的菜单栏时，这里有一些有用的可选内核参数用于满足你
的定制需求。

如果你计划运行docker，\ `/var/lib/docker`\ 的文件系统必须是btrfs，通过指定内核
参数\ `rootfstype=btrfs`\ 可以将整个根目录指定为btrfs格式，轻松满足要求。虽然
此选项一直处于开发测试中。

- **rootfstype=tmpfs**: Use tmpfs for the writable root filesystem. This is the default behavior.
- **rootfstype=btrfs**: Use btrfs in ram for the writable root filesystem. *Experimental*
- **root**: Use a local filesystem for root instead of one of two in-ram options above. The filesystem must be formatted in advance but may be completely blank, it will be initialized on boot. The filesystem may be specified by any of the usual ways including device, label, or UUID; e.g: `root=/dev/sda1`, `root=LABEL=ROOT` or `root=UUID=2c618316-d17a-4688-b43b-aa19d97ea821`.
- **sshkey**: Add the given SSH public key to the `core` user's authorized_keys file. Replace the example key below with your own (it is usually in `~/.ssh/id_rsa.pub`)
- **console**: Enable kernel output and a login prompt on a given tty. The default, `tty0`, generally maps to VGA. Can be used multiple times, e.g. `console=tty0 console=ttyS0`
- **coreos.autologin**: Drop directly to a shell on a given console without prompting for a password. Useful for troubleshooting but use with caution. For any console that doesn't normally get a login prompt by default be sure to combine with the `console` option, e.g. `console=tty0 console=ttyS0 coreos.autologin=tty1 coreos.autologin=ttyS0`. Without any argument it enables access on all consoles. Note that for the VGA console the login prompts are on virtual terminals (`tty1`, `tty2`, etc), not the VGA console itself (`tty0`).
- **cloud-config-url**: CoreOS will attempt to download a cloud-config document and use it to provision your booted system. See the [coreos-cloudinit-project][cloudinit] for more information.

[cloudinit]: https://github.com/coreos/coreos-cloudinit

下面一个仅包含CoreOS选项的pxelinux.cfg文件。指定好一个cloud-config URL后，你可
以将它拷贝到\ `/var/lib/tftpboot/pxelinux.cfg/default`\ 中

```sh
default coreos
prompt 1
timeout 15

display boot.msg

label coreos
  menu default
  kernel coreos_production_pxe.vmlinuz
  append initrd=coreos_production_pxe_image.cpio.gz cloud-config-url=http://example.com/pxe-cloud-config.yml
```

下面是一个通用的cloud-config文件，将它存放在上面的cloud-config-url指向的URL。

```yaml
#cloud-config
coreos:
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
ssh_authorized_keys:
  - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAAYQC2PxAKTLdczK9+RNsGGPsz0eC2pBlydBEcrbI7LSfiN7Bo5hQQVjki+Xpnp8EEYKpzu6eakL8MJj3E28wT/vNklT1KyMZrXnVhtsmOtBKKG/++odpaavdW2/AU0l7RZiE=
```

在[这里][cloud-config]你可以查看cloud-config的全部选项。
[cloud-config]: {{site.url}}/docs/cluster-management/setup/cloudinit-cloud-config/

注意：其它文档中提到的\ `$private_ipv4`\ 和\ `$public_ipv4`\ 两个变量不被PXE系
统支持。

### Choose a Channel
### 选择一个版本（通道）

CoreOS有alpha和beta版两个发布通道，每一个发布通道的版本都是下一通道的发布候选版
本。例如，一个没有Bug的alpha版本通过一点点的改进进入到beta通道。

当新版本发布后，通过PXE启动的机器不能更新。为了更新到最新版本的CoreOS，请下载
并校验下面的文件，然后重启。

<div id="pxe-create">
  <ul class="nav nav-tabs">
    <li class="active"><a href="#stable-create" data-toggle="tab">Stable Channel</a></li>
    <li><a href="#beta-create" data-toggle="tab">Beta Channel</a></li>
    <li><a href="#alpha-create" data-toggle="tab">Alpha Channel</a></li>
  </ul>
  <div class="tab-content coreos-docs-image-table">
    <div class="tab-pane" id="alpha-create">
      <p>In the config above you can see that a Kernel image and a initramfs file is needed. Download these two files into your tftp root.</p>
      <p>The <code>coreos_production_pxe.vmlinuz.sig</code> and <code>coreos_production_pxe_image.cpio.gz.sig</code> files can be used to <a href="{{site.url}}/docs/sdk-distributors/distributors/notes-for-distributors/#importing-images">verify the downloaded files</a>.</p>
      <pre>
cd /var/lib/tftpboot
wget http://alpha.release.core-os.net/amd64-usr/current/coreos_production_pxe.vmlinuz
wget http://alpha.release.core-os.net/amd64-usr/current/coreos_production_pxe.vmlinuz.sig
wget http://alpha.release.core-os.net/amd64-usr/current/coreos_production_pxe_image.cpio.gz
wget http://alpha.release.core-os.net/amd64-usr/current/coreos_production_pxe_image.cpio.gz.sig
gpg --verify coreos_production_pxe.vmlinuz.sig
gpg --verify coreos_production_pxe_image.cpio.gz.sig
      </pre>
    </div>
    <div class="tab-pane" id="beta-create">
      <p>In the config above you can see that a Kernel image and a initramfs file is needed. Download these two files into your tftp root.</p>
      <p>The <code>coreos_production_pxe.vmlinuz.sig</code> and <code>coreos_production_pxe_image.cpio.gz.sig</code> files can be used to <a href="{{site.url}}/docs/sdk-distributors/distributors/notes-for-distributors/#importing-images">verify the downloaded files</a>.</p>
      <pre>
cd /var/lib/tftpboot
wget http://beta.release.core-os.net/amd64-usr/current/coreos_production_pxe.vmlinuz
wget http://beta.release.core-os.net/amd64-usr/current/coreos_production_pxe.vmlinuz.sig
wget http://beta.release.core-os.net/amd64-usr/current/coreos_production_pxe_image.cpio.gz
wget http://beta.release.core-os.net/amd64-usr/current/coreos_production_pxe_image.cpio.gz.sig
gpg --verify coreos_production_pxe.vmlinuz.sig
gpg --verify coreos_production_pxe_image.cpio.gz.sig
      </pre>
    </div>
    <div class="tab-pane active" id="stable-create">
      <p>In the config above you can see that a Kernel image and a initramfs file is needed. Download these two files into your tftp root.</p>
      <p>The <code>coreos_production_pxe.vmlinuz.sig</code> and <code>coreos_production_pxe_image.cpio.gz.sig</code> files can be used to <a href="{{site.url}}/docs/sdk-distributors/distributors/notes-for-distributors/#importing-images">verify the downloaded files</a>.</p>
      <pre>
cd /var/lib/tftpboot
wget http://stable.release.core-os.net/amd64-usr/current/coreos_production_pxe.vmlinuz
wget http://stable.release.core-os.net/amd64-usr/current/coreos_production_pxe.vmlinuz.sig
wget http://stable.release.core-os.net/amd64-usr/current/coreos_production_pxe_image.cpio.gz
wget http://stable.release.core-os.net/amd64-usr/current/coreos_production_pxe_image.cpio.gz.sig
gpg --verify coreos_production_pxe.vmlinuz.sig
gpg --verify coreos_production_pxe_image.cpio.gz.sig
      </pre>
    </div>
  </div>
</div>

## Booting the Box
## 启动CoreOS

在按上面的说明设置好PXE服务器后，你可以在目标主机上进入PXE启动模式。主机将从PXE
服务器上下载镜像并启动CoreOS。如果出现问题，你可以将问题发布到
[IRC channel][irc]或[mailing list][coreos-dev]。

```sh
This is localhost.unknown_domain (Linux x86_64 3.10.10+) 19:53:36
SSH host key: 24:2e:f1:3f:5f:9c:63:e5:8c:17:47:32:f4:09:5d:78 (RSA)
SSH host key: ed:84:4d:05:e3:7d:e3:d0:b9:58:90:58:3b:99:3a:4c (DSA)
ens0: 10.0.2.15 fe80::5054:ff:fe12:3456
localhost login:
```

## 登入系统

为了方便，系统的IP地址会输出至终端。如果没有立即显示，按几次回车键将会显示。现
在你可以通公钥认证模式使用SSH登入CoreOS系统了。

```sh
ssh core@10.0.2.15
```

## Update Process
## 更新程序

Since our upgrade process requires a disk, this image does not have the option to update itself. Instead, the box simply needs to be rebooted and will be running the latest version, assuming that the image served by the PXE server is regularly updated.

## Installation
## 安装

当CoreOS启动后，你可以选择将CoreOS安装至本地磁盘或继续通过PXE启动CoreOS，使用
本地磁盘作用root分区。如果你计划运行docker，我们推荐使用一个本地brtfs文件系统
而不是ext4，如果不使用docker可以使用ext4。下面的例子将配置一个btrfs文件系统在
`/dev/sda`\ 上。

```sh
cfdisk -z /dev/sda
touch "/usr.squashfs (deleted)"     # work around a bug in mkfs.btrfs 3.12
mkfs.btrfs -L ROOT /dev/sda1
```

然后如前面描述的，加上内核选项\ `root=/dev/sda1`\ 或\ `root=LABEL=ROOT`\ 。

[install-to-disk]: {{site.url}}/docs/running-coreos/bare-metal/installing-to-disk

## Adding a Custom OEM
## 增加OEM定制

与CoreOS的磁盘镜像的[OEM分区][oem]类似，iPXE镜像也可以通过向initramfs中添加一
个[cloud config][cloud-config]文件来定制。只需要创建一个\ `./usr/share/oem/`\
目录包含\ `cloud-config.yml`\ 文件，并使用cpio打包好。

```sh
mkdir -p usr/share/oem
cp cloud-config.yml ./usr/share/oem
gzip -d coreos_production_pxe_image.cpio.gz
find usr | cpio -o -A -H newc -O coreos_production_pxe_image.cpio
gzip coreos_production_pxe_image.cpio
```

确认打包正确：

```sh
gzip -dc coreos_production_pxe_image.cpio.gz | cpio -it
./
usr.squashfs
usr
usr/share
usr/share/oem
usr/share/oem/cloud-config.yml
```

[oem]: {{site.url}}/docs/sdk-distributors/distributors/notes-for-distributors/#image-customization
[cloud-config]: {{site.url}}/docs/cluster-management/setup/cloudinit-cloud-config/

## Using CoreOS
## 使用CoreOS

现在你有一台运行CoreOS的机器了，尽情的玩耍吧。查看
[CoreOS快速指南]({{site.url}}/docs/quickstart)，或者研究一下其它
[特定的主题]({{site.url}}/docs)。
