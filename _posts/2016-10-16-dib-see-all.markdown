---
layout:     post
title:      "Diskimage-builder总览"
subtitle:   " \"diskimage-builder 是openstack社区用于制作镜像的工具,这里对DIB进行了一个总览\""
date:       2016-10-16 12:00:00
author:     "Xion"
header-img: "img/post-bg-building.jpg"
catalog: true
tags:
    - opentack-infra
    - diskimage-builder
---

# DIB总览

## 概览

### 语言

这个项目主要以shell和python为主,尤其是shell.但是目录结构还是遵循了python的目录结构

### 测试

#### Testr与setuptool集成

基本上openstack项目中,含有tox测试都是利用这个框架来运行的.可以有如下的功能

- 并行测试
- 覆盖率测试

[FQA](https://wiki.openstack.org/wiki/Testr)
[Openstack大概测试用到的东西](http://www.infoq.com/cn/articles/the-development-of-openstack-unit-test)

#### 功能测试
由于大部分代码是shell,dib的功能测试也是针对shell的.

功能测试的入口是/test/run_funcetests.sh
代码流程如下:
![](/img/post/dib-test.png)

### 文档
保持文档和代码一致比较重要dib的做法是有element中的readme来保持和维持文档

doc/source 规定了所有的文档的数据源头

### releasenote的管理

releasenote的管理和doc的管理类似,通过配置来生成文档.

# DIB逻辑

>这个主要是针对disk image,但是dib同时也能制作docker image和内存image.

## disk-image-create

disk-image-create是DIB的入口,整个disk-image-create的流程如下;

![](/img/post/dib-all.png)

有几点值得注意:
#### 阶段

各种阶段,(root,extra-data,pre_install,install,post_install,block-device,fianlise)除了chroot中和chroot外没有任何区别,只是用他们来规范操作(当然意义是不同的,但是运行的流程是相同的).

#### 启动流程:
  - BIOS,硬件自检,然后交给第一个存储设备(硬盘,网络...)


  - MBR,MBR主要的作用是描述磁盘和找到boot([MBR代码详解](http://blog.csdn.net/sallay/article/details/3668614))(GRUB 不是通过文件系统来找内核文件的，因为这时候内核还没有启动所以也不存在什么文件系统，而是直接访问硬盘的第1个硬盘第1个分区（MBR里面存在分区表）的来找到内核文件)


  - BOOT loader,现在的boot loader主要有两种grub和extlinux,boot loader的作用是把磁盘中的内核文件加载到内存汇中.在DIB中的bootloader元素,中bootloader安装脚本是一个非常好的bootloader安装脚本,几乎在所有的linux发行版上都可以用这个脚本进行bootloader的安装


  - 内核加载程序,这里也有多种,以前用的是init,现在很多linux系统使用systemd,但是同时也保留init的兼容.
   探测硬件
   加载驱动
   挂载根文件系统
   执行第一个程序/sbin/init

#### 镜像是什么

镜像就是一块存储,里面保留了从分区表到文件系统到文件内容的所有数据.大致的组成是这样的:
MBR,文件系统的格式,比如inode

#### 分区表

:分区表保存在MBR中,它占据了磁盘的前512个字节;
  - 001-440 bytes	由 BIOS 启动的 MBR 启动代码
  - 441-446 bytes	MBR 硬盘签名
  - 447-510 bytes	分区表 (主分区和扩展分区，而非逻辑分区)
  - 511-512 bytes	MBR 启动签名 0xAA55.
# DIB element
>dib的element组织的比较杂,没有分类,而且文档写的也一般.

这里说一些比较有特色的:

## package-installs
这个元素很有意思,这是一个用于安装的元素,优点在于为所有安装方式提供了一层抽象层.
类似与saltstack中的安装包的功能,为所有的发型版提供了一个统一安装的入口.

用它主要有两种用法:

### pkg-map

因为不同的发行版的包的名字是不同的,有些公用的元素,例如bootloade,需要安装一个gurb的时候可能会面对根据不同的发行版现则不同的包名的问题,如果这些携程逻辑,就会很麻烦.
所以这里提供了一个map的能力,例如还是bootloader中定义了如下的pkg-map;

```
{
  "family": {
    "gentoo": {
      "dkms_package": "",
      "extlinux": "syslinux",
      "grub-pc": "grub"
    },
    "suse": {
      "dkms_package": "",
      "grub-pc": "grub2"
    },
    "redhat": {
      "extlinux": "syslinux-extlinux",
      "grub-pc": "grub2-tools grub2"
    }
  },
  "default": {
    "dkms_package": "dkms",
    "extlinux": "extlinux",
    "grub-pc": "grub-pc"
  }
}

```

具体使用的时候,只需要用grub-pc这个名字,package-install就会读取这个配置文件,安装对应的包.

### 利用配置文件装包
目前package-installs这个元素支持利用json和yaml来定以所需进行的操作,阶段和包名.
例子如下:

```
# Install any packages in this file that may not be in the base cloud
# image but could reasonably be expected
lsof:
tcpdump:
traceroute:
which:
gettext:
    phase: pre-install.d

# these are being installed to satisfy the dependencies of grub2.  See
# 15-remove-grub for more details
grub2-tools:
    phase: pre-install.d
os-prober:
    phase: pre-install.d
redhat-lsb-core:
    phase: pre-install.d
system-logos:
    phase: pre-install.d
```

# 总结

- DIB这个项目遵循了社区python项目的基本结构,但是非常简单,展示了一个最简单的标准带tox的项目的例子
- 这个项目质量还是比较可靠的,而且兼容很多linux发行版,抽象了一些常用的功能.
- 缺点也是有的,文档不是很好,element找起来比较困难.甚至有的自己的element有功能重复实现的地方.说明有一部分贡献代码的人都还没有完整的看一边所有element

## dib能力

- 基本能力:在已经有镜像文件的基础上,对镜像进行定制,包括软件的安装,系统的设置等.
- 可以制作镜像的类型,disk,docker(应该也属于disk),内存镜像(给裸机使用的)
- 可以制作的镜像格式:qcow2,tar,vhd,docker,aci,raw除此之外根据它使用的格式工具,应该还支持vhdk.在element中iso也支持了iso的安装镜像.

## 社区用途
在[TripleO Preoject](https://wiki.openstack.org/wiki/TripleO),大致用来建立一个[部署云](https://github.com/rbrady/tripleo/blob/master/docs/architecture_overview.rst)提供端到端的裸机之上的规划,部署和操作

[Openstack Infrastrucure](http://docs.openstack.org/infra/system-config/)主要是nodepool用于定制[特定功能](https://github.com/openstack-infra/project-config/tree/master/nodepool/elements)的机器
