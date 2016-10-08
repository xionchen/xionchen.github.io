---
layout:     post
title:      "从日志分析DIB流程(5) 从文件到镜像"
subtitle:   " \"这里是从日志分析DIB的第五篇,之前都是对文件的操作.如何从文件到一个镜像,都会在这里讲述\""
date:       2016-10-06 12:00:00
author:     "Xion"
header-img: "img/post-bg-tech.jpg"
catalog: true
tags:
    - opentack-infra
    - diskimage-builder
    - 从日志分析DIB
---

# 从日志分析DIB流程(5) 从文件到镜像
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

这几篇博客的思路是,先看看"猪是怎么跑的","再吃猪肉".  
[ Diskimage-builder简介 ](https://xionchen.github.io/2016/10/01/dib-introduction/)介绍了使用dib的安装和使用需要知道的一些知识  
可以理解为"怎么让猪跑".  

这里先由一个简单的例子,来看看"猪是怎么跑的",顺便"吃点猪肉":

      $ disk-image-create vm ubuntu-minimal

日志文件被我放在这里:  [DIB_日志](https://github.com/xionchen/note/blob/master/dib/Dib-log.markdown)  
有任何问题,都希望能够留言一起探讨.

---

# 从文件到镜像

## 创建一个合适的块设备并且建立文件系统

> 这里对应日志的2742:2909

这个过程的流程图如下:

![](/img/post/create_block.png)

#### NOTE
> 删除lost+found是因为,后面创建文件系统的时候也会创建一个lost+found文件夹.如果这里不删除,等下复制文件的时候就会发生冲突

> 使用truncate命令来创建指定大小的文件
> 例子truncat -s 100M testfile 可以创建大小为100M的名为testfile的文件

> 使用losetup命令来在文件上创建块设备
> 例子losetup filename 可以在filename上创建块设备

> run_d block-device 让脚本可以定制分区

> 使用mkfs来建立文件系统

> 使用tune2fs来修改文件系统的属性

## eval_run_d block-device

> eval_run_d 和 run_d 的区别知识eval_run_d过滤了一些输出的内容,但是实际上运行的东西还是一样的.

在本例子中bloack-device阶段运行的脚本只有一个:10-parttion

### 10-parttion
> 这个脚本属于vm元素,它的作用是进行了分区

脚本内容如下;

```shell
11 [ -n "$IMAGE_BLOCK_DEVICE" ] || die "Image block device not set"               
 12                                                                                
 13 # Create 2 partitions for PPC, one for PReP boot and other for root            
 14 if [[ "$ARCH" =~ "ppc" ]] ; then                                               
 15     sudo parted -a optimal -s $IMAGE_BLOCK_DEVICE \                            
 16         mklabel msdos \                                                        
 17         mkpart primary 0 8cyl \                                                
 18         set 1 boot on \                                                        
 19         set 1 prep on \                                                        
 20         mkpart primary 9cyl 100%                                               
 21 else                                                                           
 22     sudo parted -a optimal -s $IMAGE_BLOCK_DEVICE \                            
 23         mklabel msdos \                                                        
 24         mkpart primary 1MiB 100% \                                             
 25         set 1 boot on                                                          
 26 fi                                                                             
 27                                                                                
 28 sudo partprobe $IMAGE_BLOCK_DEVICE                                             
 29                                                                                
 30 # To ensure no race conditions exist from calling partprobe                    
 31 sudo udevadm settle                                                            
 32                                                                                
 33 # If the partition isn't under /dev/loop*p1, create it with kpartx             
 34 DM=                                                                            
 35 if [ ! -e "${IMAGE_BLOCK_DEVICE}p1" ]; then                                    
 36     DM=${IMAGE_BLOCK_DEVICE/#\/dev/\/dev\/mapper}                              
 37     # If running inside Docker, make our nodes manually, because udev will not be working.
 38     if [ -f /.dockerenv ]; then                                                
 39         # kpartx cannot run in sync mode in docker.                            
 40         sudo kpartx -av $TMP_IMAGE_PATH                                        
 41         sudo dmsetup --noudevsync mknodes                                      
 42     else                                                                       
 43         sudo kpartx -asv $TMP_IMAGE_PATH                                       
 44     fi                                                                         
 45 elif [[ "$ARCH" =~ "ppc" ]]; then                                              
 46     sudo kpartx -asv $TMP_IMAGE_PATH                                           
 47 fi                                                                             
 48                                                                                
 49 if [ -n "$DM" ]; then                                                          
 50     echo "IMAGE_BLOCK_DEVICE=${DM}p1"                                          
 51 elif [[ "$ARCH" =~ "ppc" ]]; then                                              
 52     DM=${IMAGE_BLOCK_DEVICE/#\/dev/\/dev\/mapper}                              
 53     echo "IMAGE_BLOCK_DEVICE=${DM}p2"                                          
 54 else                                                                           
 55     echo "IMAGE_BLOCK_DEVICE=${IMAGE_BLOCK_DEVICE}p1"                          
 56 fi                      
```
这个脚本的作用就是对这个快设备进行了分区.
#### NOTE
>[parted](https://linux.die.net/man/8/parted)是分区命令,可以用这个命令来进行分区

> partprobe这个命令是用于提交parted命令的记录的

>[kpartx](https://linux.die.net/man/8/kpartx)这个工具是用于建立块设备的分区映射的.

## 将镜像文件放到快设备中    
这里的过程比较简单:

- 创建mnt目录
- 把块设备挂载到mnt目录上
- 把built中人所有东西cp到mnt中


## run_d_in_target finalise
>这个步骤进行一些最后的收尾工作

因为是在chroot下运行的,所以先挂载了proc,dev和sys文件夹

在本例子中,该阶段的脚本如下:

```
-rwxrwxr-x  1 xion xion 7205 10月  1 11:22 50-bootloader
-rwxrwxr-x  1 xion xion  242 10月  1 11:22 50-remove-bogus-udev-links
-rwxrwxr-x  1 xion xion  216 10月  1 11:22 99-clean-up-cache
-rwxrwxr-x  1 xion xion 1368 10月  1 11:22 99-write-dpkg-manifest
```

### 50-bootloader
>这个脚本的作用是安装bootloader

目前的bootloader主要有两种,[extlinux](http://www.syslinux.org/wiki/index.php?title=EXTLINUX)和[grub2](https://wiki.archlinux.org/index.php/GRUB)
这个脚本的作用是一个通用的bootloader安装的程序,在dib支持的所用linux发行版都可以用这个脚本来安装bootloader

在本例子中使用的是gurb2,具体的安装步骤见日志2964:3146line

### 50-remove-bogus-udev-links

这是为了解决opensuse的一个[bug](https://bugzilla.novell.com/show_bug.cgi?id=859493)

### 99-clean-up-cache

这个脚本只运行了:

    apt-get clean

### 99-write-dpkg-manifest

把安装的包写到了manifest文件中



    
### finalise_base

- 删除resolv.conf啊
- unmount所有镜像中tmp下挂载的目录
- 
### run_d cleanup
这部分的脚本都是清理一些垃圾,不详细讲了

## 格式和清理
经过之前的所以步骤,在这个例子中,一个可启动的镜像已经做好存放在一个文件中了,接下来要根据设定将他转换特定的格式.

具体的流程如下:

```
如果image_type中有 tar或者aci
如果是aci
    利用tar命令压缩文件,然后把aci manifest文件也加入到压缩包中
如果是tar
    直接压缩
如果是docker
    用tar压缩文件 | docker import -

如果元素中有没有 no-final-image 元素
  fstrim-image,这个可以看做是磁盘整理
  
unmount之前挂载的目录
cleanup_build_dir

如果元素中没有no-final-image并且不是ramdisk

在所有的image_type中
    如果不是raw
      compress_and_save_image  xxx.格式
    如果是raw_type
      先记下来
如果有raw
  compress_and_save_image   xxx.raw

cleanUp_image_dir
```

**对于compress_and_save_image**

现在已有的镜像是xxx.raw.compress_and_save_image方法如下:

```shell
115 function compress_and_save_image () {                                          
116     # Recreate our image to throw away unnecessary data                        
117     test $IMAGE_TYPE != qcow2 && COMPRESS_IMAGE=""                             
118     if [ -n "$QEMU_IMG_OPTIONS" ]; then                                        
119         EXTRA_OPTIONS="-o $QEMU_IMG_OPTIONS"                                   
120     else                                                                       
121         EXTRA_OPTIONS=""                                                       
122     fi                                                                         
123     if [ "$IMAGE_TYPE" = "raw" ]; then                                         
124         mv $TMP_IMAGE_PATH $1-new                                              
125     elif [ "$IMAGE_TYPE" == "vhd" ]; then                                      
126         cp $TMP_IMAGE_PATH $1-intermediate                                     
127         vhd-util convert -s 0 -t 1 -i $1-intermediate -o $1-intermediate       
128         vhd-util convert -s 1 -t 2 -i $1-intermediate -o $1-new                
129         # The previous command creates a .bak file                             
130         rm $1-intermediate.bak                                                 
131         OUT_IMAGE_PATH=$1-new                                                  
132     else                                                                       
133         echo "Converting image using qemu-img convert"                         
134         qemu-img convert ${COMPRESS_IMAGE:+-c} -f raw -O $IMAGE_TYPE $EXTRA_OPTIONS $TMP_IMAGE_PA    TH $1-new
135     fi                                                                         
136                                                                                
137     OUT_IMAGE_PATH=$1-new                                                      
138     finish_image $1                                                            
139 }             
```

这个函数的功能是进行镜像格式的转换

进行完这一步,一个镜像就做出来了.