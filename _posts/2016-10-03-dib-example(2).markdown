---
layout:     post
title:      "从日志分析DIB流程(2) root阶段"
subtitle:   " \"diskimage-builder 是openstack社区用于制作镜像的工具.为了深入了解dib制作镜像的全过程,对一个简单的例子进行贯通的分析.\""
date:       2016-10-03 12:00:00
author:     "Xion"
header-img: "img/post-bg-tech.jpg"
catalog: true
tags:
    - opentack-infra
    - diskimage-builder
    - 从日志分析DIB
---

# 从日志分析DIB流程(2) root阶段
---

>dib这几篇博客是干货,基于下面三个客观条件:    
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



# 该例子中的root阶段的脚本
> 之前了解了run_d方法的流程,但是实际上完成root阶段工作的还是root阶段的脚本,所以接下来看看在这里例子中每个脚本做了什么.  

> 这里对应了日志中的757:1154line

root阶段的脚本从日志757line开始,之前导入创建了钩子文件夹,导入了一些环境变量.

运行如下命令,就可以看到生产的钩子文件夹中的内容:

    $ break=before-root disk-image-create  vm ubuntu
    $ cd /tmp/dib_build.xxxx/hooks

查看所有脚本:

```
-rwxrwxr-x  1 xion xion  611 10月  1 11:22 01-ccache
-rwxrwxr-x  1 xion xion 2523 10月  1 11:22 10-cache-ubuntu-tarball
-rwxrwxr-x  1 xion xion  308 10月  1 11:22 50-build-with-http-cache
-rwxrwxr-x  1 xion xion  486 10月  1 11:22 60-block-apt-translations
-rwxrwxr-x  1 xion xion  365 10月  1 11:22 90-base-dib-run-parts
-rwxrwxr-x  1 xion xion  983 10月  1 11:22 99-block-daemons
-rwxrwxr-x  1 xion xion  376 10月  1 11:22 99-shared_apt_cache
-rwxrwxr-x  1 xion xion  403 10月  1 11:22 99-trim-dpkg

```

下面按照顺序说明每个脚本的作用
    
## 01-ccache
```shell
  1 #!/bin/bash                                                                                    
  2                                                                                
  3 if [ ${DIB_DEBUG_TRACE:-0} -gt 0 ]; then                                       
  4     set -x                                                                     
  5 fi                                                                             
  6 set -eu                                                                        
  7 set -o pipefail                                                                
  8                                                                                
  9 # Don't do anything if already mounted (if disk-image-create is invoked with   
 10 # no elements specified, this hook actually fires twice, once during           
 11 # `run_d root` for the base element, then again when `run_d root` is called    
 12 # after automatically pulling in the Ubuntu element)                           
 13 grep " $TMP_MOUNT_PATH/tmp/ccache" /proc/mounts && exit                        
 14                                                                                
 15 DIB_CCACHE_DIR=${DIB_CCACHE_DIR:-$DIB_IMAGE_CACHE/ccache}                      
 16 mkdir -p $DIB_CCACHE_DIR                                                       
 17                                                                                
 18 sudo mkdir -p $TMP_MOUNT_PATH/tmp/ccache                                       
 19 sudo mount --bind $DIB_CCACHE_DIR $TMP_MOUNT_PATH/tmp/ccache
 ```
 
这个脚本属于 base element.  
此处该做脚本做了下面事情
>检查cache文件夹(一个用于存放临时文件的文件夹)是否已经挂载  
> 如果没有挂载,创建cache文件夹和挂载点,并且将其挂载到镜像build的目录下(使用bind的mount方式)

## 10-cache-ubuntu-tarball

```shell
  1 #!/bin/bash                                                                    
  2 # These are useful, or at worst not harmful, for all images we build.          
  3                                                                                
  4 if [ ${DIB_DEBUG_TRACE:-1} -gt 0 ]; then                                       
  5     set -x                                                                                     
  6 fi                                                                             
  7 set -eu                                                                        
  8 set -o pipefail                                                                
  9                                                                                
 10 [ -n "$ARCH" ]                                                                 
 11 [ -n "$TARGET_ROOT" ]                                                          
 12                                                                                
 13 shopt -s extglob                                                               
 14                                                                                
 15 DIB_CLOUD_IMAGES=${DIB_CLOUD_IMAGES:-http://cloud-images.ubuntu.com}           
 16 DIB_RELEASE=${DIB_RELEASE:-trusty}                                             
 17 BASE_IMAGE_FILE=${BASE_IMAGE_FILE:-$DIB_RELEASE-server-cloudimg-$ARCH-root.tar.gz}
 18 SHA256SUMS=${SHA256SUMS:-https://${DIB_CLOUD_IMAGES##http?(s)://}/$DIB_RELEASE/current/SHA256SU    MS}
 19 CACHED_FILE=$DIB_IMAGE_CACHE/$BASE_IMAGE_FILE                                  
 20 CACHED_FILE_LOCK=$DIB_IMAGE_CACHE/$BASE_IMAGE_FILE.lock                        
 21 CACHED_SUMS=$DIB_IMAGE_CACHE/SHA256SUMS.ubuntu.$DIB_RELEASE.$ARCH              
 22                                                                                
 23 function get_ubuntu_tarball() {                                                
 24     if [ -n "$DIB_OFFLINE" -a -f "$CACHED_FILE" ] ; then                       
 25         echo "Not checking freshness of cached $CACHED_FILE."                  
 26     else                                                                       
 27         echo "Fetching Base Image"                                             
 28         $TMP_HOOKS_PATH/bin/cache-url $SHA256SUMS $CACHED_SUMS                 
 29         $TMP_HOOKS_PATH/bin/cache-url \                                        
 30             $DIB_CLOUD_IMAGES/$DIB_RELEASE/current/$BASE_IMAGE_FILE $CACHED_FILE
 31         pushd $DIB_IMAGE_CACHE                                                 
 32         if ! grep "$BASE_IMAGE_FILE" $CACHED_SUMS | sha256sum --check - ; then 
 33             # It is likely that an upstream http(s) proxy has given us a skewed
 34             # result - either a cached SHA file or a cached image. Use cache-busting
 35             # to get (as long as caches are compliant...) fresh files.         
 36             # Try the sha256sum first, just in case that is the stale one (avoiding
 37             # downloading the larger image), and then if the sums still fail retry
 38             # the image.                                                       
 39             $TMP_HOOKS_PATH/bin/cache-url -f $SHA256SUMS $CACHED_SUMS          
 40             if ! grep "$BASE_IMAGE_FILE" $CACHED_SUMS | sha256sum --check - ; then
 41                 $TMP_HOOKS_PATH/bin/cache-url -f \                             
 42                     $DIB_CLOUD_IMAGES/$DIB_RELEASE/current/$BASE_IMAGE_FILE $CACHED_FILE
 43                 grep "$BASE_IMAGE_FILE" $CACHED_SUMS | sha256sum --check -     
 44             fi                                                                 
 45         fi                                                                     
 46         popd                                                                   
 47     fi                                                                         
 48     # Extract the base image (use --numeric-owner to avoid UID/GID mismatch between
 49     # image tarball and host OS e.g. when building Ubuntu image on an openSUSE host)
 50     sudo tar -C $TARGET_ROOT --numeric-owner -xzf $DIB_IMAGE_CACHE/$BASE_IMAGE_FILE
 51 }                                                                              
 52                                                                                
 53 (   
 54     echo "Getting $CACHED_FILE_LOCK: $(date)"                                  
 55     # Wait up to 20 minutes for another process to download                    
 56     if ! flock -w 1200 9 ; then                                                
 57         echo "Did not get $CACHED_FILE_LOCK: $(date)"                          
 58         exit 1                                                                 
 59     fi                                                                         
 60     get_ubuntu_tarball                                                         
 61 ) 9> $CACHED_FILE_LOCK                  
```

这个脚本属于ubuntu element  
它做了以下事情:  
> 等待锁  
> 下载镜像:  

![](/img/post/download_img.png)


### NOTE  

---

##### [shopt -s extglob](https://my.oschina.net/maxio/blog/672645)
```
shopt -s extglob 用于开启支持更多的通配符  
?(pattern)        匹配0次或1次；  
*(pattern)         匹配0次以上包括0次；  
+(pattern)        匹配1次以上包括1次；  
@(pattern)       匹配1次；  
!(pattern)          不匹配。 
```


##### [BASH Locking](http://www.kfirlavi.com/blog/2012/11/06/elegant-locking-of-bash-program/)
```
(
    # Wait for lock on /var/lock/.myscript.exclusivelock (fd 200) for 10 seconds
    flock -n 200

    # Do stuff

) 200>/var/lock/.myscript.exclusivelock
200>/var/lock/.myscript.exclusivelock 的意思是将该程序的文件描述符号200指向"/var/lock/.myscript.exclusivelock"文件.
flock -w 1200 9的意思是等待1200s,等待文件描述符为9的文件.

```

##### pushd popd
```
pushd的全称是push dir
栈顶的目录就会作为当前的工作目录
popd的全称是pop dir
popd会弹出目前栈顶的目录
```

## 50-build-with-http-cache

```shell
1 #!/bin/bash                                                                                    
2                                                                                
3 if [ ${DIB_DEBUG_TRACE:-0} -gt 0 ]; then                                       
4     set -x                                                                     
5 fi                                                                             
6 set -eu                                                                        
7 set -o pipefail                                                                
8                                                                                
9 [ -n "$TARGET_ROOT" ]                                                          
10                                                                                
11 # If we have a network proxy, use it.                                          
12 if [ -n "${http_proxy:-}" ] ; then                                             
13     sudo dd of=$TARGET_ROOT/etc/apt/apt.conf.d/60img-build-proxy << _EOF_      
14 Acquire::http::Proxy "$http_proxy";                                            
15 _EOF_                                                                          
16 fi 
```

这个脚本属于dpkg  
它的作用就是如果配置了代理,就将Acquire::http::Proxy "$http_proxy";写入配置文件中

## 60-block-apt-translations
```shell
1 #!/bin/bash                                                                                    
2                                                                                
3 if [ ${DIB_DEBUG_TRACE:-0} -gt 0 ]; then                                       
4     set -x                                                                     
5 fi                                                                             
6 set -eu                                                                        
7 set -o pipefail                                                                
8                                                                                
9 [ -n "$TARGET_ROOT" ]                                                          
10                                                                                
11 # Configure APT not to fetch translations files                                
12 sudo dd of=$TARGET_ROOT/etc/apt/apt.conf.d/95no-translations <<EOF             
13 Acquire::Languages "none";                                                     
14 EOF                                                                            
15                                                                                
16 # And now make sure that we don't fall foul of Debian bug 641967               
17 find $TARGET_ROOT/var/lib/apt/lists/ -path $TARGET_ROOT/var/lib/apt/lists/partial -prune -o -ty    pe f -name '*_i18n_Translation-*' -print -exec sudo rm -f {} +

```

这个脚本属于dpkg元素
这个脚本做了下面这些事:
> 配置apt不获取 translations files




## 90-base-dib-run-parts
```shell
1 #!/bin/bash                                                                                    
2                                                                                
3 if [ ${DIB_DEBUG_TRACE:-0} -gt 0 ]; then                                       
4     set -x                                                                     
5 fi                                                                             
6 set -eu                                                                        
7 set -o pipefail                                                                
8                                                                                
9 # Abort early if dib-run-parts is not found to prevent a meaningless           
10 # error message from the subsequent install command                            
11 DIB_RUN_PARTS=$(which dib-run-parts)                                           
12                                                                                
13 exec sudo install -m 0755 -o root -g root -D \                                 
14     $DIB_RUN_PARTS \                                                           
15     $TARGET_ROOT/usr/local/bin/dib-run-parts   
```

这个脚本的作用是将dib-run-parts放到镜像的bin目录下

## 99-block-daemons
```shell
1 #!/bin/bash                                                                                    
2                                                                                
3 if [ ${DIB_DEBUG_TRACE:-0} -gt 0 ]; then                                       
4     set -x                                                                     
5 fi                                                                             
6 set -eu                                                                        
7 set -o pipefail                                                                
8                                                                                
9 [ -n "$TARGET_ROOT" ]                                                          
10                                                                                
11 # Prevent package installs from starting daemons                               
12 sudo mv $TARGET_ROOT/sbin/start-stop-daemon $TARGET_ROOT/sbin/start-stop-daemon.REAL
13 sudo dd of=$TARGET_ROOT/sbin/start-stop-daemon <<EOF                           
14 #!/bin/sh                                                                      
15 echo                                                                           
16 echo "Warning: Fake start-stop-daemon called, doing nothing"                   
17 EOF                                                                            
18 sudo chmod 755 $TARGET_ROOT/sbin/start-stop-daemon                             
19                                                                                
20 if [ -f $TARGET_ROOT/sbin/initctl ]; then                                      
21     sudo mv $TARGET_ROOT/sbin/initctl $TARGET_ROOT/sbin/initctl.REAL           
22     sudo dd of=$TARGET_ROOT/sbin/initctl <<EOF                                 
23 #!/bin/sh                                                                      
24 echo "initctl (tripleo 1.0)"                                                   
25 echo "Warning: Fake initctl called, doing nothing"                             
26 EOF                                                                            
27     sudo chmod 755 $TARGET_ROOT/sbin/initctl                                   
28 fi                                                                             
29                                                                                
30 sudo dd of=$TARGET_ROOT/usr/sbin/policy-rc.d <<EOF                             
31 #!/bin/sh                                                                      
32 # 101 Action not allowed. The requested action will not be performed because   
33 #     of runlevel or local policy constraints.                                 
34 exit 101                                                                       
35 EOF                                                                            
36 sudo chmod 755 $TARGET_ROOT/usr/sbin/policy-rc.d  
```

这个脚本的作用是防止很多进程和服务自动启动

###NOTE
[policy-rc.d](https://jpetazzo.github.io/2013/10/06/policy-rc-d-do-not-start-services-automatically/)

## 99-shared_apt_cache
```shell
1 #!/bin/bash                                                                                    
2                                                                                
3 if [ ${DIB_DEBUG_TRACE:-0} -gt 0 ]; then                                       
4     set -x                                                                     
5 fi                                                                             
6 set -eu                                                                        
7 set -o pipefail                                                                
8                                                                                
9 DIB_APT_LOCAL_CACHE=${DIB_APT_LOCAL_CACHE:-1}                                  
10                                                                                
11 if [ $DIB_APT_LOCAL_CACHE = "0" ]; then                                        
12     exit 0                                                                     
13 fi                                                                             
14                                                                                
15 apt_cache_dir=$DIB_IMAGE_CACHE/apt/$DISTRO_NAME                                
16 if [ ! -d $apt_cache_dir ]; then                                               
17     mkdir -p $apt_cache_dir                                                    
18 fi                                                                             
19 sudo mount --bind $apt_cache_dir $TARGET_ROOT/var/cache/apt/archives           
~                                                                        
```

这个脚本的作用就是创建了一个apt_cache_dir目录用于cacheapt的包,然后把这个目录挂载到了镜像下的目录

## 99-trim-dpkg
```shell
1 #!/bin/bash                                                                    
2                                                                                
3 if [ ${DIB_DEBUG_TRACE:-0} -gt 0 ]; then                                       
4     set -x                                                                     
5 fi                                                                             
6 set -eu                                                                        
7 set -o pipefail                                                                
8                                                                                
9 [ -n "$TARGET_ROOT" ]                                                          
10                                                                                
11 # During image build, sync calls are expensive overhead                        
12 echo 'force-unsafe-io' | sudo tee $TARGET_ROOT/etc/dpkg/dpkg.cfg.d/02apt-speedup > /dev/null
13                                                                                
14 # and remove the translations, too                                             
15 echo 'Acquire::Languages "none";' | sudo tee $TARGET_ROOT/etc/apt/apt.conf.d/no-languages > /de    v/null   
```

这个脚本配为dpkg配置了force-unsafe-io和无语言,主要是dpkg的配置.

下一篇介绍extra-data.d

