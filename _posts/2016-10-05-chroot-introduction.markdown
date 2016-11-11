---
layout:     post
title:      "chroot技术简介"
subtitle:   " \"chroot 在 Linux 系统中发挥了根目录的切换工作，同时带来了系统的安全性等好处。\""
date:       2016-10-05 12:00:00
author:     "Xion"
header-img: "img/post-bg-chroot.jpg"
catalog: true
tags:
    - linux
    - chroot
---

# chroot技术简介
---

> chroot是一种隔离技术,开始主要是为了测试安装和构建系统(做镜像也算).
> 到2008年的时候,基于cgroups开发除了LXC,以及Docker.可以说chroot技术是容器技术的前身.

## chroot命令
[官方的man手册](https://www.freebsd.org/cgi/man.cgi?query=chroot&apropos=0&sektion=8&manpath=FreeBSD+10.3-RELEASE+and+Ports&arch=default&format=html)
一般的用法为:

    chroot -u 用户 -g 组 -G 组,组... 新的根目录 命令

通常这里的命令很可能是运行一个bash.

关于chroot,这里也有一些资料[理解chroot](https://www.ibm.com/developerworks/cn/linux/l-cn-chroot/)

## 使用chroot
>并不是任何目录都可以作为chroot的目标,chroot有一下条件

首先chroot的目标目录需要满足[linxu文件系统标准](chrome-extension://ikhdkkncnoglghljlkmcimlnlhkeamad/pdf-viewer/web/viewer.html?file=https%3A%2F%2Frefspecs.linuxfoundation.org%2FFHS_3.0%2Ffhs-3.0.pdf)
其次还需要挂载一些特殊的目录:
```shell
 cd /mnt/arch
 mount -t proc proc proc/
 mount -t sysfs sys sys/
 mount -o bind /dev dev/
 mount -t devpts pts dev/pts/
```


这个文档详细的介绍了如何使用chroot[如何使用chroot](https://wiki.archlinux.org/index.php/change_root)
这个文档说明了chroot的一个基本概念[BasciChroot](https://help.ubuntu.com/community/BasicChroot)

## 例子

```shell
 cd /home/user
 mkdir myroot   #创建一个用于存放系统的根目录
 pacman -S arch-install-scripts #这是arch linux的例子,这里相当于安装了一个arch linux系统
 # pacstrap must see myroot as mounted:
 mount --bind myroot myroot      #挂载chroot的目录
 pacstrap -i myroot base base-devel
 mount -t proc proc myroot/proc/   #挂载一些需要的目录
 mount -t sysfs sys myroot/sys/
 mount -o bind /dev myroot/dev/
 mount -t devpts pts myroot/dev/pts/
 cp -i /etc/resolv.conf myroot/etc/  #配置dns
 chroot myroot                        #chroot
 # inside chroot:

```
