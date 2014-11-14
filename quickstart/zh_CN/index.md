---
layout: docs
title: CoreOS Quick Start
#redirect handled in alias_generator.rb
---

# Quick Start

如果你没有一台运行CoreOS的机器，请阅读如何在云提供商，虚拟化平台，物理服务器上
运行CoreOS的相关指南，依据这些指南你可以在几分钟内搭建一个运行CoreOS的服务器。

If you don't have a CoreOS machine running, check out the guides on [running CoreOS]({{site.url}}/docs/#running-coreos) on most cloud providers ([EC2]({{site.url}}/docs/running-coreos/cloud-providers/ec2), [Rackspace]({{site.url}}/docs/running-coreos/cloud-providers/rackspace), [GCE]({{site.url}}/docs/running-coreos/cloud-providers/google-compute-engine)), virtualization platforms ([Vagrant]({{site.url}}/docs/running-coreos/platforms/vagrant), [VMware]({{site.url}}/docs/running-coreos/platforms/vmware), [OpenStack]({{site.url}}/docs/running-coreos/platforms/openstack), [QEMU/KVM]({{site.url}}/docs/running-coreos/platforms/qemu)) and bare metal servers ([PXE]({{site.url}}/docs/running-coreos/bare-metal/booting-with-pxe), [iPXE]({{site.url}}/docs/running-coreos/bare-metal/booting-with-ipxe), [ISO]({{site.url}}/docs/running-coreos/platforms/iso), [Installer]({{site.url}}/docs/running-coreos/bare-metal/installing-to-disk)). With any of these guides you will have machines up and running in a few minutes.

强烈推荐你运行一个至少包含三台服务器的集群——在单台服务器上运行有很多功能无法尝
试。如果你手上资料有限，[Vagrant][vagrant-guide] 可以助你在你的笔记本上搭建一
个集群。为了让集群正常运行起来，你必须通过user-data提供cloud-conifg，关于这方
面的详细内容在相应平台的安装指南中有说明。

CoreOS提供了三个必需的工具：服务发现，窗口管理和进程管理。下面我们将逐一介绍。

首先，使用用户core，通过SSH连接到CoreOS服务器。例如在Amazon上，通过下面的命令
：

```sh
$ ssh -A core@an.ip.compute-1.amazonaws.com
CoreOS (beta)
```

The `-A` forwards your ssh-agent to the machine, which is needed for the fleet section of this guide. If you haven't already done so, you will need to add your private key to the SSH agent running on your client machine - for example:

```sh
$ ssh-add
Identity added: .../.ssh/id_rsa (.../.ssh/id_rsa)
```

如果你使用的是Vagrant，操作有点不同：

```sh
$ ssh-add ~/.vagrant.d/insecure_private_key
Identity added: /Users/core/.vagrant.d/insecure_private_key (/Users/core/.vagrant.d/insecure_private_key)
$ vagrant ssh core-01 -- -A
CoreOS (beta)
```

## Service Discovery with etcd

CoreOS的首要基石是etcd提供的服务发现功能。etcd存储的数据分布在集群中所有运行
CoreOS的服务器上。例如，每个app窗口都向代理容器告知自己，代理容器将自动知道哪
个容器将接收流量。将服务发现功能添加到你的应用中，可以使你（方便的）添加大量的
服务器，无缝的扩展应用。

如果你依照第一部分中提到的指南中使用过cloud-config的例子，etcd服务将开机自启动
。
If you used an example [cloud-config]({{site.url}}/docs/cluster-management/setup/cloudinit-cloud-config) from a guide linked in the first paragraph, etcd is automatically started on boot.

一个好的开始会是这样的：

```
#cloud-config

hostname: coreos0
ssh_authorized_keys:
  - ssh-rsa AAAA...
coreos:
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
  etcd:
    name: coreos0
    discovery: https://discovery.etcd.io/<token>
```

为了取得发现的令牌，访问[https://discovery.etcd.io/new]，你会收到一个包含令牌
的URL，将它粘贴到你的cloud-config文件中。

这个API的使用非常方便，在CoreOS上，你可以利用curl与etcd进行交互，设定或查询key
。

设定key，`message` 的值为`Hello world`：

```sh
curl -L http://127.0.0.1:4001/v1/keys/message -d value="Hello world"
```

取得`message`的值：

```sh
curl -L http://127.0.0.1:4001/v1/keys/message
```

如果你依照指南中的介绍配置了多个CoreOS服务器，你可以通过ssh连接上另外一台服务
器，也能够查询到相同的值。

#### More Detailed Information
<a class="btn btn-primary" href="{{ site.url }}/docs/distributed-configuration/getting-started-with-etcd/" data-category="More Information" data-event="Docs: Getting Started etcd">View Complete Guide</a>
<a class="btn btn-default" href="{{site.url}}/docs/distributed-configuration/etcd-api/">Read etcd API Docs</a>

## Container Management with docker

CoreOS的第二个基石是：docker，你的应用，代码运行于其上。docker被安装在每个
CoreOS上。你可以将你的每个服务（Web服务，缓存，数据库）运行于容器中，通过读写
etcd将它们连接在一起。你可以用下面两种方式尝试运行Ubuntu容器：

The second building block, **docker** ([docs][docker-docs]), is where your applications and code run. It is installed on each CoreOS machine. You should make each of your services (web server, caching, database) into a container and connect them together by reading and writing to etcd. You can quickly try out a Ubuntu container in two different ways:

在容器中运行一个命令，然后停止：

```sh
docker run busybox /bin/echo hello world
```

打开一个容器内的shell：

```sh
docker run -i -t busybox /bin/sh
```

#### More Detailed Information
<a class="btn btn-primary" href="{{ site.url }}/docs/launching-containers/building/getting-started-with-docker" data-category="More Information" data-event="Docs: Getting Started docker">View Complete Guide</a>
<a class="btn btn-default" href="http://docs.docker.io/">Read docker Docs</a>

## Process Management with fleet

CoreOS的基石是 **fleet**，一个用于集群的分布init系统。你可以使用fleet来管理你
的docker容器的生命周期。

Fleet通过systemd unit文件来工作，并且根据unit文件中定义的一些互斥和其它定义将
它们分发到集群中服务器上运行。

首先，我们来写启动docker容器的一个简单的systemd unit文件。保存在home目录的hello.service

#### hello.service

```ini
[Unit]
Description=My Service
After=docker.service

[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker kill hello
ExecStartPre=-/usr/bin/docker rm hello
ExecStartPre=/usr/bin/docker pull busybox
ExecStart=/usr/bin/docker run --name hello busybox /bin/sh -c "while true; do echo Hello World; sleep 1; done"
ExecStop=/usr/bin/docker stop hello
```

指南中详细的介绍unit文件的格式。

然后加载和运行unit：

Then load and start the unit:

```sh
$ fleetctl load hello.service
Job hello.service loaded on 8145ebb7.../172.17.8.105
$ fleetctl start hello.service
Job hello.service launched on 8145ebb7.../172.17.8.105
```

你的容器将运行于集群中的某台服务器上，为了验证状态，请运行：

```sh
$ fleetctl status hello.service
● hello.service - My Service
   Loaded: loaded (/run/fleet/units/hello.service; linked-runtime)
   Active: active (running) since Wed 2014-06-04 19:04:13 UTC; 44s ago
 Main PID: 27503 (bash)
   CGroup: /system.slice/hello.service
           ├─27503 /bin/bash -c /usr/bin/docker start -a hello || /usr/bin/docker run --name hello busybox /bin/sh -c "while true; do echo Hello World; sleep 1; done"
           └─27509 /usr/bin/docker run --name hello busybox /bin/sh -c while true; do echo Hello World; sleep 1; done

Jun 04 19:04:57 core-01 bash[27503]: Hello World
..snip...
Jun 04 19:05:06 core-01 bash[27503]: Hello World
```

为了停止容器，请运行：

```sh
fleetctl destroy hello.service
```

Fleet有很多特性你可以继续阅读下面的文档。

#### More Detailed Information
<a class="btn btn-primary" href="{{ site.url }}/docs/launching-containers/launching/launching-containers-fleet/" data-category="More Information" data-event="Docs: Launching Containers Fleet">View Complete Guide</a>
<a class="btn btn-default" href="{{ site.url }}/docs/launching-containers/launching/getting-started-with-systemd/" data-category="More Information" data-event="Docs: Getting Started with systemd">View Getting Started with systemd Guide</a>
