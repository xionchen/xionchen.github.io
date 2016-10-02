---
layout:     post
title:      "Diskimage-builder简介"
subtitle:   " \"diskimage-builder 是openstack社区用于制作镜像的工具,这里对基本的安装和使用进行了简单的介绍\""
date:       2016-10-01 12:00:00
author:     "Xion"
header-img: "img/post-bg-building.jpg"
catalog: true
tags:
    - opentack-infra
    - diskimage-builder
---

# Diskimage-builder简介
> diskimage-builder是openstack社区用于制作镜像的一个工具.
> 它的源码地址位于: https://github.com/openstack/diskimage-builder
> 官方的文档位于: http://docs.openstack.org/developer/diskimage-builder/

这里对这个工具的安装和使用进行简单的介绍

# 1 安装
dib的安装方式有两种:**pip安装**和**源码安装**.
如果不需要对代码进行改变和定制的话可以直接进行源码安装,否则的话推荐使用源码安装

### 1.1 pip安装
pip的安装非常简单,使用如下命令安装:

    pip install diskimage-builder

### 1.2 源码安装
克隆源码的仓库:

    git clone https://git.openstack.org/openstack/diskimage-builder
    git clone https://git.openstack.org/openstack/dib-utils
将下面的bin目录加到路劲中:

    export PATH=$PATH:$(pwd)/diskimage-builder/bin:$(pwd)/dib-utils/bin

### 1.3 虚环境安装
> 虚拟环境可以为python提供一个独立的环境,非常适合用于调试和隔离,在tox工具中尤其用到
> 关于python虚环境的详细文档: https://virtualenv.pypa.io/en/stable/
> 关于tox工具的详细文档: https://tox.readthedocs.io/en/latest/

这里简单说明如何在需环境中安装dib(diskimage-builder)

克隆源码的仓库:

    $ git clone https://git.openstack.org/openstack/diskimage-builder
    $ git clone https://git.openstack.org/openstack/dib-utils
建立虚环境:

    $ virtualenv dib-env

启用虚环境

    $ souce dib-env/bin/activate

安装:

    $ cd diskimage-builder
    $ pip install .

# 2 制作第一个镜像
>安装完成后,用如下的命令就可以制作一个简单的ubuntu的镜像:

    $ disk-image-create vm ubuntu
如果想记录dib运行过程中的所有输出,可以用如下命令

    $ disk-image-create vm ubuntu 2>&1 | tee FILENAME

把上面的FILENAME换成你要存放的文件的路劲

# 3 使用dib

>dib使用element的方式来组织制作镜像的元素.
>如果现有的元素已经能够覆盖当前制作镜像的需求,那么还需要了解下面相关的知识

### 3.1 dib参数列表
>下面这个方法是dib的参数的说明,这里就不详细说明,如果有疑问可以留言讨论

``` shell
function show_options () {
    echo "Usage: ${SCRIPTNAME} [OPTION]... [ELEMENT]..."
    echo
    echo "Options:"
    echo "    -a i386|amd64|armhf -- set the architecture of the image(default amd64)"
    echo "    -o imagename -- set the imagename of the output image file(default image)"
    echo "    -t qcow2,tar,vhd,docker,aci,raw -- set the image types of the output image files (default qcow2)"
    echo "       File types should be comma separated. VHD outputting requires the vhd-util"
    echo "       executable be in your PATH. ACI outputting requires the ACI_MANIFEST "
    echo "       environment variable be a path to a manifest file."
    echo "    -x -- turn on tracing (use -x -x for very detailed tracing)"
    echo "    -u -- uncompressed; do not compress the image - larger but faster"
    echo "    -c -- clear environment before starting work"
    echo "    --image-size size -- image size in GB for the created image"
    echo "    --image-cache directory -- location for cached images(default ~/.cache/image-create)"
    echo "    --max-online-resize size -- max number of filesystem blocks to support when resizing."
    echo "       Useful if you want a really large root partition when the image is deployed."
    echo "       Using a very large value may run into a known bug in resize2fs."
    echo "       Setting the value to 274877906944 will get you a 1PB root file system."
    echo "       Making this value unnecessarily large will consume extra disk space "
    echo "       on the root partition with extra file system inodes."
    echo "    --min-tmpfs size -- minimum size in GB needed in tmpfs to build the image"
    echo "    --mkfs-options -- option flags to be passed directly to mkfs."
    echo "       Options should be passed as a single string value."
    echo "    --no-tmpfs -- do not use tmpfs to speed image build"
    echo "    --offline -- do not update cached resources"
    echo "    --qemu-img-options -- option flags to be passed directly to qemu-img."
    echo "       Options need to be comma separated, and follow the key=value pattern."
    echo "    --root-label label -- label for the root filesystem.  Defaults to 'cloudimg-rootfs'."
    echo "    --ramdisk-element -- specify the main element to be used for building ramdisks."
    echo "       Defaults to 'ramdisk'.  Should be set to 'dracut-ramdisk' for platforms such"
    echo "       as RHEL and CentOS that do not package busybox."
    echo "    --install-type -- specify the default installation type. Defaults to 'source'. Set to 'package' to use package based installations by default."
    echo "    --docker-target -- specify the repo and tag to use if the output type is docker. Defaults to the value of output imagename"
    if [ "$IS_RAMDISK" == "0" ]; then
        echo "    -n skip the default inclusion of the 'base' element"
        echo "    -p package[,package,package] -- list of packages to install in the image"
        fi
   echo "    -h|--help -- display this help and exit"
   echo "    --version -- display version and exit"
   echo
   echo "ELEMENTS_PATH will allow you to specify multiple locations for the elements."
   echo
   echo "NOTE: At least one distribution root element must be specified."
   echo
   echo "NOTE: If using the VHD output format you need to have a patched version of vhd-util installed for the image"
   echo "      to be bootable. The patch is available here: https://github.com/emonty/vhd-util/blob/master/debian/patches/citrix"
   echo "      and a PPA with the patched tool is available here: https://launchpad.net/~openstack-ci-core/+archive/ubuntu/vhd-util"
   echo
   echo "Examples:"
   if [ "$IS_RAMDISK" == "0" ]; then
       echo "    ${SCRIPTNAME} -a amd64 -o ubuntu-amd64 vm ubuntu"
       echo "    export ELEMENTS_PATH=~/source/tripleo-image-elements/elements"
       echo "    ${SCRIPTNAME} -a amd64 -o fedora-amd64-heat-cfntools vm fedora heat-cfntools"
   else
       echo "    ${SCRIPTNAME} -a amd64 -o fedora-deploy deploy fedora"
       echo "    ${SCRIPTNAME} -a amd64 -o ubuntu-ramdisk ramdisk ubuntu"
   fi
}
```

### 3.2 DEBUG DIB
> 很多使用制作镜像可能会遇到问题,dib也提供了debug的方法.

export break 环境变量可以在特定的情况下,启动一个shell, 让你查看镜像的情况.
Break 点的设置方法一般是: “break=[before|after]-hook-name”.
如果有多个break的点,可以用逗号隔开.


一些例子:

    break=before-block-device-size 在块设备钩子前break
    break=before-pre-install pre-install钩子前break
    break=after-error 出错之后会break


### 3.3 DIB element
>dib的element很多,这里有详细的信息,但是有相当一部分的文档写的不是很好.需要阅读源码才知道到底是干什么的.如果有疑问也可以在这里讨论
> DIB element: http://docs.openstack.org/developer/diskimage-builder/elements.html
### 3.4 DIB运行的阶段
>dib通过element和阶段来组织脚本,一个element可能在不同的阶段做不同的事情.

所有的阶段如下:

1. root.d
2. extra-data.d
3. pre-install.d
4. install.d
5. post-install.d
6. block-device.d
7. finalise.d
8. cleanup.d

#### root.d
Create or adapt the initial root filesystem content. This is where alternative distribution support is added, or customisations such as building on an existing image.

Only one element can use this at a time unless particular care is taken not to blindly overwrite but instead to adapt the context extracted by other elements.

runs: outside chroot
inputs:

    $ARCH=i386|amd64|armhf
    $TARGET_ROOT=/path/to/target/workarea

#### extra-data.d
Pull in extra data from the host environment that hooks may need during image creation. This should copy any data (such as SSH keys, http proxy settings and the like) somewhere under $TMP_HOOKS_PATH.

runs: outside chroot
inputs:

    $TMP_HOOKS_PATH
outputs: None

#### pre-install.d
Run code in the chroot before customisation or packages are installed. A good place to add apt repositories.

runs: in chroot

#### install.d
Runs after pre-install.d in the chroot. This is a good place to install packages, chain into configuration management tools or do other image specific operations.

runs: in chroot

#### post-install.d
Run code in the chroot. This is a good place to perform tasks you want to handle after the OS/application install but before the first boot of the image. Some examples of use would be:

Run chkconfig to disable unneeded services

Clean the cache left by the package manager to reduce the size of the image.

runs: in chroot

#### block-device.d
Customise the block device that the image will be made on (for example to make partitions). Runs after the target tree has been fully populated but before the cleanup.d phase runs.

runs: outside chroot
inputs:

    $IMAGE_BLOCK_DEVICE={path}
    $TARGET_ROOT={path}
outputs:

    $IMAGE_BLOCK_DEVICE={path}

#### finalise.d
Perform final tuning of the root filesystem. Runs in a chroot after the root filesystem content has been copied into the mounted filesystem: this is an appropriate place to reset SELinux metadata, install grub bootloaders and so on.

Because this happens inside the final image, it is important to limit operations here to only those necessary to affect the filesystem metadata and image itself. For most operations, post-install.d is preferred.

runs: in chroot

#### cleanup.d
Perform cleanup of the root filesystem content. For instance, temporary settings to use the image build environment HTTP proxy are removed here in the dpkg element.

runs: outside chroot
inputs:

    $ARCH=i386|amd64|armhf
    $TARGET_ROOT=/path/to/target/workarea


# 4 总结
DIB这个工具目前而言,制作镜像的能力主要覆盖社区的基础设施.如果需要定制自己的镜像也可以按照这个框架来.但是进行复杂的定制需要对DIB本身,shell和操作系统三个方面比较都比较了解.所以在之后会细致的解读DIB的源码.
