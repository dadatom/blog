---
title: Linux 快速安装之 Fedora CoreOS
date: 2023-05-04 14:06:39
tags: [Linux, 教程, Fedora, Technology]
---

Fedora CoreOS是一个基于容器化技术的轻量级操作系统，专门为运行容器工作负载而设计。它采用了自动化更新、不可变性和安全性等最佳实践，以提供更高效、更安全的容器环境。

<!-- more -->

# 概述

使用 Fedora CoreOS 的 ISO 镜像安装文件，将系统安装到物理机或虚拟机上，本文以 PVE 虚拟化系统举例，讲述如何快速安装并配置 Fedora CoreOS 系统。

> 注意：此文章默认已经创建好 PVE 虚拟机，默认网络、硬盘、cpu、内存等选项已经配置完成，可以正常连通网络

# 使用场景

我主要使用 Fedora CoreOS 部署 Kubernetes 容器集群，其原因是 Fedora CoreOS 自带针对 Kubernetes 集群的软件、配置支持，且省去了很多不必要的软件包，极大节省不必要的性能开支。

> 建议创建 PVE 虚拟机时，CPU设置4核, 内存设置 `8GB` 或 `8192MB`，此配置是创建 Kubernetes 集群的单节点最低配置，其余用途可以根据实际情况做出调整

# 官方文档

 - [Installing CoreOS on Bare Metal](https://docs.fedoraproject.org/en-US/fedora-coreos/bare-metal/)
 - [Configuration specifications](https://coreos.github.io/butane/specs/)

# 安装教程

## 1. 镜像下载

 - 打开下载页面：[https://fedoraproject.org/coreos/download](https://fedoraproject.org/coreos/download)
 - 选择镜像：`Stable` => `x86_64` => `Bare Metal & Virtualized` => `Live DVD (iso)`
 - 开始下载

## 2. 挂载镜像

 - 物理机安装：使用 [Etcher](https://www.balena.io/etcher) 烧录到U盘开始安装
 - PVE 虚拟机：上传 ISO 镜像，挂载 CD/DVD 进行安装

> 注意：挂载 ISO 镜像后打开系统，所进入的是镜像系统，即便操作模式、软件、服务都可以正常使用，但是这种操作方式是不可持续化的，仅在内存中进行操作，而非存储单元，需要完成安装步骤才可以正常使用。

## 3. 查看系统信息

Fedora CoreOS 的安装方式与其他的 Linux 发行版有些不同，并非使用可视化界面配置的方式，而是使用 fcc 转 ign 配置文件的方式进行安装，所以在编写配置文件之前，我们需要记录一些系统信息，用于后续配置文件的更改。

### 查看网卡信息

> 后续所有的命令默认都是用 `root` 账户进行操作

```bash
# 默认登录用户为 core, 使用此命令切换到 root 用户
[core@localhost ~]$ sudo su -
```

```bash
# 查看网络、网卡信息 (前提是已经配置好系统外部网络拓扑)
[root@localhost ~]$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 01:8b:77:f6:a5:8c brd ff:ff:ff:ff:ff:ff
    altname enp0s18
    inet 192.168.1.109/24 brd 192.168.1.255 scope global noprefixroute ens18
       valid_lft forever preferred_lft forever
    inet6 2408:8207:4983:9985::a62/128 scope global dynamic noprefixroute
       valid_lft 4123sec preferred_lft 4123sec
    inet6 2408:8207:4983:9985:8952:175c:2ecc:b679/64 scope global dynamic noprefixroute
       valid_lft 219592sec preferred_lft 133192sec
    inet6 fe80::461:55c7:dce6:23f3/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

此设备的网卡名称为：`ens18`

### 查看磁盘信息

```bash
[root@localhost ~]$ fdisk -l
Disk /dev/sda: 40 GiB, 42949672960 bytes, 83886080 sectors
Disk model: QEMU HARDDISK
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
...
```

> 注意：此命令仅用于查找要安装的磁盘名称，输出内容为安装后的磁盘信息，数据上会有很大差异，无需关心

要安装系统的磁盘名称为 `/dev/sda`

## 4. 创建 fcos.fcc 配置文件

```bash
[root@localhost ~]$ touch fcos.fcc
[root@localhost ~]$ nano fcos.fcc
```

fcc 配置文件模板，官方文档：[Producing an Ignition Config](https://docs.fedoraproject.org/en-US/fedora-coreos/producing-ign/)

```yaml
variant: fcos
version: 1.0.0
storage:
  files:
    - path: /etc/NetworkManager/system-connections/eth0.nmconnection
      mode: 0600
      overwrite: true
      contents:
        inline: |
          [connection]
          type=ethernet
        # 将 ens18 替换为查找到的网卡名称
          interface-name=ens18
          [ipv4]
          method=manual
        # 将ipv4地址配置适合所处网络环境的ipv4地址
          addresses=192.168.1.100/24
        # 将网关配置为所处网络的网关
          gateway=192.168.1.1
          dns=119.29.29.29;223.5.5.5
passwd:
  users:
    - name: core
      # 生成命令为：openssl passwd <PASSOWRD>
      # 生成对应hash值后进行替换
      password_hash: $1$RsFZy0ZH$Y7KsqVFH623EQ67.ARrvW/
      # 免密登录密钥，可自行配置
      # ssh_authorized_keys:
      #   - "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCsKc8PGzOU/+i7..."
      groups: [ sudo, docker ]
```

> 我的模板中没有使用免密登录密钥的配置方式，是因为组建容器集群，需要配置多个 Fedora CoreOS 系统，为了节省配置时间，以及让后续登录维护更简单，因此使用密码登录的方式。

## 5. 生成 ign 文件

```bash
# 拉取镜像
[root@localhost ~]$ docker pull quay.io/coreos/fcct
# 将 fcc 配置转换为 ign 配置
[root@localhost ~]$ docker run -i --rm quay.io/coreos/fcct --pretty --strict <fcos.fcc > fcos.ign
```

> Fedora CoreOS 自带 docker 管理，因此可以直接执行命令

## 6. 安装到硬盘

```bash
# 将 /dev/sda 替换为查询到的磁盘名称
[root@localhost ~]$ coreos-installer install /dev/sda --ignition-file fcos.ign
```

## 7. 关机并卸载镜像文件

```bash
# 安装过程
Installing Fedora CoreOS 32.20200629.3.0 x86_64 (512-byte sectors)
> Read disk 2.7 GiB/2.7 GiB (100%)
Writing Ignition config
Install complete.

# 当显示 Install complete. 则表示安装完成，执行立即关机命令
[root@localhost ~]$ shutdown -h now
```

关机后卸载挂载的CD/DVD 镜像，防止重启后再次进入镜像系统

## 8. 允许远程密码登录

```bash
# 开机进入系统，使用 core 账户以及设置好的密码进行登录，再执行下方命令
[core@localhost ~]$ sudo su -
[root@localhost ~]$ nano /etc/ssh/sshd_config.d/40-disable-passwords.conf
```

在 40-disable-passwords.conf 文件中，将 no 改为 yes，表示允许远程使用密码登录

```bash
# PasswordAuthentication no
PasswordAuthentication yes
```

```bash
# 重启 sshd 服务
[root@localhost ~]$ systemctl restart sshd
```

## 9. 修改主机名称 (可选)

```bash
[root@localhost ~]$ hostnamectl set-hostname <YOUR_CUSTOM_HOSTNAME>
# eg: hostnamectl set-hostname rke2-server-1
```

> 修改主机名称的原因是因为配置 rke2 服务节点时，需要不同节点的主机名称不同

## 10. 远程登录管理

```bash
# 在其他主机上执行
$ ssh core@xxx.xxx.xxx.xxx
```

> 默认ssh端口为22，未做变更

# 总结

至此，安装过程结束了。此文章仅为快速创建并配置 Fedora CoreOS 系统，如需更详细配置可以到官网查看。

# References

*<small style="vertical-align: top;">[1]</small> vmware安装fedora-coreos: [https://www.cnblogs.com/xiaochina/p/13377930.html](https://www.cnblogs.com/xiaochina/p/13377930.html)*
*<small style="vertical-align: top;">[2]</small> 3种方法更改Linux系统的主机名(hostname): [https://blog.51cto.com/u_15278282/2931915](https://blog.51cto.com/u_15278282/2931915)*