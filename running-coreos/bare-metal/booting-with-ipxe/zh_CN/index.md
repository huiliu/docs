---
layout: docs
slug: ipxe
title: 通过iPXE启动coreos
category: running_coreos
sub_category: bare_metal
supported: true
weight: 5
---

# 通过iPXE启动coreos

CoreOS当前正在快速开发和不断测试中。这部分说明将指导你在物理硬件或虚拟硬件上通过
iPXE启动CoreOS。默认情况下，本例中的coreos完全运行于内存中。当然CoreOS也可以安装
在[磁盘]({{site.url}}/docs/running-coreos/bare-metal/installing-to-disk)上。

CoreOS is currently in heavy development and actively being tested. These instructions will walk you through booting CoreOS via iPXE on real or virtual hardware. By default, this will run CoreOS completely out of RAM. CoreOS can also be [installed to disk]({{site.url}}/docs/running-coreos/bare-metal/installing-to-disk).

## 配置pxelinux

任何可以从ISO镜像启动的平台都可以使用iPXE，包括很多云提供商和物理硬件。

在当前文档中，我们使用qemu-kvm来说明如何使用iPXE。

### 选择一个发布通道（版本）

CoreOS有alpha和beta版发布通道，每一个发布通道的版本都是下一通道的发布候选版本。
例如，一个没有Bug的alpha版本通过一点点的改进进入到beta通道。


### 配置启动脚本

<div id="ipxe-create">
  <ul class="nav nav-tabs">
    <li class="active"><a href="#stable-create" data-toggle="tab">Stable Channel</a></li>
    <li><a href="#beta-create" data-toggle="tab">Beta Channel</a></li>
    <li><a href="#alpha-create" data-toggle="tab">Alpha Channel</a></li>
  </ul>
  <div class="tab-content coreos-docs-image-table">
    <div class="tab-pane" id="alpha-create">
      <p>iPXE downloads a boot script from a publicly available URL. You will need to host this URL somewhere public and replace the example SSH key with your own. You can also run a <a href="https://github.com/kelseyhightower/coreos-ipxe-server">custom iPXE server</a>.</p>
      <pre>
#!ipxe

set base-url http://alpha.release.core-os.net/amd64-usr/current
kernel ${base-url}/coreos_production_pxe.vmlinuz sshkey="ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAAYQC2PxAKTLdczK9+RNsGGPsz0eC2pBlydBEcrbI7LSfiN7Bo5hQQVjki+Xpnp8EEYKpzu6eakL8MJj3E28wT/vNklT1KyMZrXnVhtsmOtBKKG/++odpaavdW2/AU0l7RZiE= coreos pxe demo"
initrd ${base-url}/coreos_production_pxe_image.cpio.gz
boot</pre>
    </div>
    <div class="tab-pane" id="beta-create">
      <p>iPXE downloads a boot script from a publicly available URL. You will need to host this URL somewhere public and replace the example SSH key with your own. You can also run a <a href="https://github.com/kelseyhightower/coreos-ipxe-server">custom iPXE server</a>.</p>
      <pre>
#!ipxe

set base-url http://beta.release.core-os.net/amd64-usr/current
kernel ${base-url}/coreos_production_pxe.vmlinuz sshkey="ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAAYQC2PxAKTLdczK9+RNsGGPsz0eC2pBlydBEcrbI7LSfiN7Bo5hQQVjki+Xpnp8EEYKpzu6eakL8MJj3E28wT/vNklT1KyMZrXnVhtsmOtBKKG/++odpaavdW2/AU0l7RZiE= coreos pxe demo"
initrd ${base-url}/coreos_production_pxe_image.cpio.gz
boot</pre>
    </div>
    <div class="tab-pane active" id="stable-create">
      <p>iPXE downloads a boot script from a publicly available URL. You will need to host this URL somewhere public and replace the example SSH key with your own. You can also run a <a href="https://github.com/kelseyhightower/coreos-ipxe-server">custom iPXE server</a>.</p>
      <pre>
#!ipxe

set base-url http://stable.release.core-os.net/amd64-usr/current
kernel ${base-url}/coreos_production_pxe.vmlinuz sshkey="ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAAYQC2PxAKTLdczK9+RNsGGPsz0eC2pBlydBEcrbI7LSfiN7Bo5hQQVjki+Xpnp8EEYKpzu6eakL8MJj3E28wT/vNklT1KyMZrXnVhtsmOtBKKG/++odpaavdW2/AU0l7RZiE= coreos pxe demo"
initrd ${base-url}/coreos_production_pxe_image.cpio.gz
boot</pre>
    </div>
  </div>
</div>

可以方便的将这个启动脚本存放在[http://pastie.org](http://pastie.org)。请务必使
用"raw"版本的脚本，通过点击右上角的剪贴板访问。

注意：iPXE不支持https协议，这意味着你不能使用
[https://gist.github.com](https://gist.github.com)来存放你的脚本。Bummer,
right?


### 启动iPXE

首先，下载和启动iPXE镜像。在本指南中我们使用\ `qemu-kvm`\ ，但是你可以根据你的平
台选择合适的方法。

```sh
wget http://boot.ipxe.org/ipxe.iso
qemu-kvm -m 1024 ipxe.iso --curses
```

接下来按Ctrl+B进入到iPXE的shell界面，输入下面的命令：

```sh
iPXE> dhcp
iPXE> chain http://${YOUR_BOOT_URL}
```

iPXE将从指定人URL下载你的启动脚本，并且从CoreOS的站点开始下载镜像文件。

```sh
${YOUR_BOOT_URL}... ok
http://alpha.release.core-os.net/amd64-usr/current/coreos_production_pxe.vmlinuz... 98%
```

下载完成后，CoreOS将正常启动。

## 更新过程

CoreOS的升级过程需要一个磁盘，这个镜像不具备自我更新的选项，而是需要通过一次简
单的重启来运行最新的版本，在此我们假定PXE服务器提供的镜像文件会定期更新。

## 安装

CoreOS可以被安装到磁盘上，或者运行于内存中而将用户数据存放在磁盘上。详细说明请
参考[CoreOS安装指南]({{site.url}}/docs/running-coreos/bare-metal/booting-with-
pxe/#installation)。

## 添加一个OEM定制

与CoreOS的磁盘镜像的[OEM分区][oem]类似，iPXE镜像也可以通过向initramfs中添加一
个[cloud config][cloud-config]文件来定制。详细说明请参考[PXE文档中的说明][pxe]。

[oem]: {{site.url}}/docs/sdk-distributors/distributors/notes-for-distributors/#image-customization
[cloud-config]: {{site.url}}/docs/cluster-management/setup/cloudinit-cloud-config/
[pxe]: {{site.url}}/docs/running-coreos/bare-metal/booting-with-pxe/#adding-a-custom-oem

## 使用CoreOS

现在你有一台运行CoreOS的机器了，尽情的玩耍吧。查看
[CoreOS快速指南]({{site.url}}/docs/quickstart)，或者研究一下其它
[特定的主题]({{site.url}}/docs)。
