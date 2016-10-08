---
layout:     post
title:      "从日志分析DIB流程(1) "
subtitle:   " \"diskimage-builder 是openstack社区用于制作镜像的工具.为了深入了解dib制作镜像的全过程,对一个简单的例子进行贯通的分析.\""
date:       2016-10-02 12:00:00
author:     "Xion"
header-img: "img/post-bg-tech.jpg"
catalog: true
tags:
    - opentack-infra
    - diskimage-builder
    - 从日志分析DIB
---

# 从日志分析DIB流程(1) 

---

>dib这几篇博客有一些优势,基于下面三个客观条件:    
> 1. 社区较为优秀的代码质量  
> 2. 因为工作需求,需要深入理解,所以花费在这上面的时间精力较多.  
> 3. DIB项目主要以shell为主,并且涉及到linux的很多方面.非常适合学习linux和shell.  

> 看完这几篇博客你将会了解:  
> 1. 如何使用DIB    
> 2. DIB的代码架构  
> 3. 制作一个系统镜像的全过程  
> 4. 一些shell的用法  
> 5. linux下的一些知识和概念  


[ Diskimage-builder简介 ](https://xionchen.github.io/2016/10/01/dib-introduction/)介绍了使用dib的安装和使用需要知道的一些知识  


这里由一个简单的例子,比较细致的了解dib:

      $ disk-image-create vm ubuntu-minimal

日志文件被我放在这里:  [DIB_日志](https://github.com/xionchen/note/blob/master/dib/Dib-log.markdown)  
有任何问题,都希望能够留言一起探讨.

---

这个系列的博客一共五篇:
- 从日志分析DIB流程(1)
  - 介绍了root阶段的顶层方法和开始的工作
- [从日志分析DIB流程(2) root阶段](/2016/10/03/dib-example(2)/)
  - 解读了例子中root阶段的所有脚本的作用
- [从日志分析DIB流程(3) extra-data阶段](/2016/10/03/dib-example(3)/)
  - 解读了extra-data阶段所有脚本的作用
- [从日志分析DIB流程(4) install系列阶段](/2016/10/04/dib-example(4)/)
  - 解读了pre-install,install,post-install阶段的所有脚本的作用
  - 介绍了chroot技术
- [从日志分析DIB流程(5) 从文件到镜像](/2016/10/06/dib-example(5)/)  
  - 介绍了dib如何把一个文件制作成一个镜像,这也是dib的最后阶段
  
---

## 1. 导入方法和Expand element
>日志 1:76 行

首先是导入了一下方法,这些方法再后面都会见到,类似于c语言中的include和python中的import,主要是为了方便组织方法的编写.所以这里不详述

作为参数传入的元素只有vm和ubuntu,但是这些元素可能依赖于其他元素,所有要把所有被依赖的元素找出来.  
如何找到这些依赖的元素不是本篇的重点,本篇只分析在制作进行的过程中发生了什么.

## 2. 创建制作进行目录
>日志 77:105 行
>Building in /tmp/dib_build.Y0FGIa3a

对应这行日志的方法是:

```shell
TMP_BUILD_DIR=$(mktemp -t -d --tmpdir=${TMP_DIR:-/tmp} dib_build.XXXXXXXX)
TMP_IMAGE_DIR=$(mktemp -t -d --tmpdir=${TMP_DIR:-/tmp} dib_image.XXXXXXXX)
[ $? -eq 0 ] || die "Failed to create tmp directory"
export TMP_BUILD_DIR
if tmpfs_check ; then
  sudo mount -t tmpfs tmpfs $TMP_BUILD_DIR
  sudo mount -t tmpfs tmpfs $TMP_IMAGE_DIR
  sudo chown $(id -u):$(id -g) $TMP_BUILD_DIR $TMP_IMAGE_DIR
fi
trap trap_cleanup EXIT
echo Building in $TMP_BUILD_DIR
export TMP_IMAGE_PATH=$TMP_IMAGE_DIR/image.raw
export OUT_IMAGE_PATH=$TMP_IMAGE_PATH
export TMP_HOOKS_PATH=$TMP_BUILD_DIR/hooks
```

mktemp是用于创建临时目录的(可以看做是随机生产文件名)
然后利用mount命令在这里挂载了tmpfs,tmpfs实际上是存在于内存中的(一般情况下)

然后到这里有几个目录:

    TMP_BUILD_DIR :  镜像是在这个目录下面制作的
    TMP_IMAGE_DIR :  最终镜像存放在这里
    OUT_IMAGE_PATH:  和IMAGE_DIR是一样的
    TMP_HOOKS_PATH:  所有的钩子(脚本)被拷贝到的地方.

## 3. root阶段入口方法

>root阶段准备好了制作进行所需的"base"

root阶段的顶层方法如下:

```shell
mkdir $TMP_BUILD_DIR/mnt
export TMP_MOUNT_PATH=$TMP_BUILD_DIR/mnt
# Copy data in to the root.
TARGET_ROOT=$TMP_MOUNT_PATH run_d root
if [ -z "$(ls $TMP_MOUNT_PATH | grep -v '^lost+found\|tmp$')" ] ; then
    # No root element copied in. Note the test above allows
    # root.d elements to put things in /tmp
    echo "Failed to deploy the root element."
    exit 1
fi

# Configure Image
# Setup resolv.conf so we can chroot to install some packages
if [ -L $TMP_MOUNT_PATH/etc/resolv.conf ] || [ -f $TMP_MOUNT_PATH/etc/resolv.conf ] ; then
    sudo mv $TMP_MOUNT_PATH/etc/resolv.conf $TMP_MOUNT_PATH/etc/resolv.conf.ORIG
fi

# Recreate resolv.conf
sudo touch $TMP_MOUNT_PATH/etc/resolv.conf
sudo chmod 777 $TMP_MOUNT_PATH/etc/resolv.conf
# use system configured resolv.conf if available to support internal proxy resolving
if [ -e /etc/resolv.conf ]; then
    cat /etc/resolv.conf > $TMP_MOUNT_PATH/etc/resolv.conf
else
    echo nameserver 8.8.8.8 > $TMP_MOUNT_PATH/etc/resolv.conf
fi
mount_proc_dev_sys
```
这个方法做的事情如下:
![](/img/post/create_base.png)

创建mnt目录和配置DNS都顾名思义,这里的mount_proc_dev中执行了如下命令:
>注意,如果root阶段没有建立好base不会进行到proc和dev的挂载

```shell
sudo mount -t proc none $TMP_MOUNT_PATH/proc
sudo mount --bind /dev $TMP_MOUNT_PATH/dev
sudo mount --bind /dev/pts $TMP_MOUNT_PATH/dev/pts
sudo mount -t sysfs none $TMP_MOUNT_PATH/sys

```
mount --bind的用法可以查看["bind mount 的用法"](https://xionchen.github.io/2016/08/25/linux-bind-mount/)

mount -t proc/sysfs 是指以proc/sysfs 的类型进行挂载,这是chroot中较为常用的用法,可以参考[arch linux chroot](https://wiki.archlinux.org/index.php/change_root)

以上这几步在所有的dib的过程中都是相同的,下面来解析这个例子中root步骤到底干了啥

### 3.1 run_d root
> run_d 是dib中的一个方法,这个方法是所有过程(pharse)的入口  
> run_d root表示运行root阶段的所有脚本,这里仅仅只是一个参数名  

run_d方法的流程图如下:  
![](/img/post/run_d.png)
过程比较简单:提供了两个debug的断点,生产钩子文件夹,运行脚本.

其中generate_hooks流程如下:  
![](/img/post/generate_hooks.png)
generate会一次性将所有需要的脚本按照阶段放到钩子文件夹中.

dib-run-parts流程图如下:  
![](/img/post/dib-run-parts.png)
dib-run-parts首先会寻找 environment.d文件夹,将其中的环境变量导入
然后以此运行该阶段的脚本

> 之前了解了run_d方法的流程,但是实际上完成root阶段工作的还是root阶段的脚本.  
> 在下一篇中会详细分析例子中的root阶段运行的脚本.




