---
layout:     post
title:      "从日志分析DIB流程(4) pre-install阶段和install阶段"
subtitle:   " \"diskimage-builder 是openstack社区用于制作镜像的工具.为了深入了解dib制作镜像的全过程,对一个简单的例子进行贯通的分析.\""
date:       2016-10-04 12:00:00
author:     "Xion"
header-img: "img/post-bg-tech.jpg"
catalog: true
tags:
    - opentack-infra
    - diskimage-builder
    - 从日志分析DIB
---

# 从日志分析DIB流程(4) pre-install阶段和install阶段
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




# run_d_in_target
只有有的阶段利用了chroot,也之前略有不同.
 
> 区别于之前的root阶段和extra-data阶段,pre-install阶段不是在宿主系统中运行的,而是chroot到镜像中运行的.    
> chroot是一种隔离技术,开始主要是为了测试安装和构建系统(做镜像也算).  
> 到2008年的时候,基于cgroups开发除了LXC,以及Docker.可以说chroot技术是容器技术的前身.  
>[这里](http://127.0.0.1:4000/2016/10/05/chroot-introduction/)对chroot进行了简单的介绍

这里先介绍一下在chroot下调用钩子的入口函数run_d_in_target  
run_d_in_target方法的流程图如下:

![](/img/post/run_d_in_target.png)

其中run_in_target方法如下:

```shell
56 function run_in_target () {                                                                
57     cmd="$@"                                                                   
58     # -E to preserve http_proxy                                                
59     ORIG_HOME=$HOME                                                            
60     export HOME=/root                                                          
61     # Force an empty TMPDIR inside the chroot. There is no need to use an user 
62     # defined tmp dir which may not exist in the chroot.                       
63     # Bug: #1330290                                                            
64     # Force the inclusion of a typical set of dirs in PATH, this is needed for guest
65     # distros that have path elements not in the host PATH.                    
66     sudo -E chroot $TMP_MOUNT_PATH env -u TMPDIR PATH="\$PATH:/usr/local/sbin:/usr/local/bi    n:/usr/sbin:/usr/bin:/sbin:/bin" sh -c "$cmd"
67     export HOME=$ORIG_HOME                                                     
68 }  
```

run_d_in_target与run_d方法很相似,但是有两处不同:
1. 需要挂载钩子目录到镜像中
2. 用chroot运行钩子

# pre-install阶段
> pre-install是在chroot下运行的,该阶段主要做一些安装前的准备工作  
> 下面介绍本例子用的脚本
> 对应脚本中的1439:1943line

本阶段的脚本如下:
```
-rwxrwxr-x  1 xion xion 264 10月  1 11:22 00-disable-apt-recommends
-rwxrwxr-x  1 xion xion 269 10月  1 11:22 00-remove-apt-xapian-index
-rwxrwxr-x  1 xion xion 393 10月  1 11:22 00-remove-grub
-rwxrwxr-x  1 xion xion 298 10月  1 11:22 01-dib-python
-rwxrwxr-x  1 xion xion 163 10月  1 11:22 01-install-bin
-rwxrwxr-x  1 xion xion 310 10月  1 11:22 01-set-ubuntu-mirror
-rwxrwxr-x  1 xion xion 991 10月  1 11:22 02-add-apt-keys
-rwxrwxr-x  1 xion xion 196 10月  1 11:22 02-package-installs
-rwxrwxr-x  1 xion xion 567 10月  1 11:22 03-baseline-tools
-rwxrwxr-x  1 xion xion 168 10月  1 11:22 04-dib-init-system
-rwxrwxr-x  1 xion xion 170 10月  1 11:22 99-apt-get-update
-rwxrwxr-x  1 xion xion 210 10月  1 11:22 99-package-uninstalls

```

## 00-disable-apt-recommends
这个脚本对apt进行了配置:

    APT::Install-Recommends "0";                                                   
    Apt::Install-Suggests "0"; 

## 00-remove-apt-xapian-index
这个脚本卸载了 apt-xapian-index  
按照脚本中的说法,原因是这个包有问题,在更新的时候会导致出错

## 00-remove-grub
这个脚本暂时卸载了grub  
因为在chroot的时候,没有块设备的存在,所以grub的安装钩子会报错.    
所以暂时移除grub,来避免冲突

## 01-dib-python
这个脚本建立了dib-python的软连接到系统中的python

## 01-install-bin
这个脚本中执行的命令如下:

    install -m 0755 -o root -g root $(dirname $0)/../bin/* /usr/local/bin

将diskimage-builder的bin目录下的内容拷贝到了镜像内

## 01-set-ubuntu-mirror
这个脚本配置了ubuntu的apt源

## 02-add-apt-keys
这个脚本将之前配置的apt的key用apt-key add xxx命令配置

## 02-package-installs
这个脚本从/tmp/in_target.d/pre-install.d和package-installs.json获取了安装的信息,来判断现在这个阶段是否要进行安装.

## 03-baseline-tools
这个脚本安装了一些python的基本包

## 04-dib-init-system
这个脚本将dib-init-system这个脚本拷贝到了镜像系统的/usr/bin目录下  
dib-init-system脚本用于判断系统的init的类型:  
- systemd
- upstart
- openrc
- sysv

##  99-apt-get-update
这个脚本中apt-get进行更新

## 99-package-uninstalls
这个脚本和之前的package-install是同一个套路,只不过反过来了.

## 其他
package-installs是一个用来统一调度安装的元素,但是由于历史原因,现在有两个版本同时在使用  
[这里](https://review.openstack.org/#/c/140795/)的代码引入了第二个版本的package-install

# install 阶段
> 在install之前会有一步do_extra_package_install  
> 它会安装在变量$INSTALL_PACKAGES中的包,但是在这个例子用没有用到,所有就略过

> install阶段应该是最主要的一个解决,在这里会对镜像进行具体的软件的安装,按照需求安装一些包.

在这个例子中install阶段的脚本如下:
```
-rwxrwxr-x  1 xion xion 202 10月  1 11:22 00-baseline-environment
-rwxrwxr-x  1 xion xion 201 10月  1 11:22 00-up-to-date
-rwxrwxr-x  1 xion xion 192 10月  1 11:22 01-package-installs
-rwxrwxr-x  1 xion xion 860 10月  1 11:22 05-set-cloud-init-sources
-rwxrwxr-x  1 xion xion 368 10月  1 11:22 10-cloud-init
-rwxrwxr-x  1 xion xion 781 10月  1 11:22 20-install-init-scripts
-rwxrwxr-x  1 xion xion 348 10月  1 11:22 50-store-build-settings
-rwxrwxr-x  1 xion xion 970 10月  1 11:22 80-disable-rfc3041
-rwxrwxr-x  1 xion xion 115 10月  1 11:22 99-autoremove
-rwxrwxr-x  1 xion xion 206 10月  1 11:22 99-package-uninstalls

```

## 00-baseline-environment
>这个脚本的内容如下

```shell
1 +--  2 lines: !/bin/bash------------------------------------------------------------------------
3                                                                                
4 if [ ${DIB_DEBUG_TRACE:-0} -gt 0 ]; then                                       
5     set -x                                                                     
6 fi                                                                             
7 set -eu                                                                        
8 set -o pipefail                                                                
9                                                                                
10 install-packages -m base iscsi_package      
```

虽然脚本很简单,但是有几点值得注意:  
1. 这里的install-packages -m 的-m是mapper.这里通过mapper的方式来对系统和包进行了解耦.
    - 例如这里的iscsi_package,具体的mapper如下:
    
```json
    {                                                                                                   
  "family": {                                                                  
    "redhat": {                                                                
      "iscsi_package": "iscsi-initiator-utils"                                 
    },                                                                         
    "gentoo": {                                                                
                                
      "iscsi_package": "sys-block/open-iscsi",                                                                                
    }                                                                          
  },                                                                           
  "default": {                                                                                                              
    "iscsi_package": "open-iscsi"                                              
  }   
```    

install-packages -m 通过iscsi_package和现在对应的操作系统,就能找到具体要安装的包的名字
2. iscsi是用来干嘛的?为什么要安装它?
    - 使用iscsi可以通过网络来访问磁盘

## 00-up-to-date
这个脚本有用的也只有一条:

    install-packages -u  

但是这个命令也很有意思,因为对应不同的发行版其实内容是不同的
可是使用如下命令来查看
```shell
$ find -name 'install-packages'
./dpkg/bin/install-packages
./gentoo/bin/install-packages
./opensuse/bin/install-packages
./yum/bin/install-packages
```
但是他们的接口,或者是调用的方法是相同的,这里也是一层解耦的封装

## 01-package-installs 99-package-uninstalls
这个脚本与之前的"02-package-installs"的内容是完全一样的.  
事实上,他们都属于package-installs 这个元素,这个元素中的内容如下:

```
.
├── bin
│   ├── package-installs
│   ├── package-installs-squash
│   ├── package-installs-v2
│   └── package-uninstalls
├── element-deps
├── extra-data.d
│   └── 99-squash-package-install
├── install.d
│   ├── 01-package-installs
│   └── 99-package-uninstalls
├── post-install.d
│   ├── 00-package-installs
│   └── 95-package-uninstalls
├── pre-install.d
│   ├── 02-package-installs
│   └── 99-package-uninstalls
└── README.rst

```

这个元素会在extra-data install post-install和pre-install根据配置文件对包进行安装和删除.

在这个例子中的配置文件如下:

```json
{
 "install.d": {
  "install": [
   [
    "ccache_package",
    "base"
   ],
   [
    "linux-image-generic",
    "ubuntu"
   ],
   [
    "dkms",
    "dkms"
   ]
  ]
 }

```

## 05-set-cloud-init-sources
脚本如下:

```shell
1 #!/bin/bash                                                                                      
 2                                                                                
 3 if [ ${DIB_DEBUG_TRACE:-0} -gt 0 ]; then                                       
 4     set -x                                                                     
 5 fi                                                                             
 6 set -eu                                                                        
 7 set -o pipefail                                                                
 8                                                                                
 9 DIB_CLOUD_INIT_DATASOURCES=${DIB_CLOUD_INIT_DATASOURCES:-""}                   
10                                                                                
11 if [ -z "$DIB_CLOUD_INIT_DATASOURCES" ] ; then                                 
12     echo "DIB_CLOUD_INIT_DATASOURCES must be set to a comma-separated list "   
13     echo "of cloud-init data sources you wish to use, ie 'Ec2, NoCloud, ConfigDrive'"
14     exit 1                                                                     
15 fi                                                                             
16                                                                                
17 if [ -d /etc/cloud/cloud.cfg.d ]; then                                         
18     # DatasourceNone doesn't exist in Ubuntu 12.04 (Precise)                   
19     # which uses cloud-init version 0.6.3                                      
20     if [ "$(lsb_release -cs)" = 'precise' ] ; then                             
21         cat > /etc/cloud/cloud.cfg.d/91-dib-cloud-init-datasources.cfg <<EOF   
22 datasource_list: [  $DIB_CLOUD_INIT_DATASOURCES ]                              
23 EOF                                                                            
24     else                                                                       
25         cat > /etc/cloud/cloud.cfg.d/91-dib-cloud-init-datasources.cfg <<EOF   
26 datasource_list: [  $DIB_CLOUD_INIT_DATASOURCES, None ]                        
27 EOF                                                                            
28     fi                                                                         
29 fi 
```

在本例中,DIB_CLOUD_INIT_DATASOURCES的值是Ec2,这里把这个参数写到了[cloud-init](https://wiki.archlinux.org/index.php/Cloud-init)的配置文件中

## 10-cloud-init
和上面的内容类似,这里配置了manage_etc_host这个选项

## 20-install-init-scripts
脚本如下:

```shell
1 +--  4 lines: !/bin/bash-------------------------------------------------------------------------
5                                                                                
6 if [ ${DIB_DEBUG_TRACE:-1} -gt 0 ]; then                                       
7     set -x                                                                     
8 fi                                                                             
9 set -eu                                                                        
10 set -o pipefail                                                                
11                                                                                
12 scripts_dir="$(dirname $0)/../init-scripts/$DIB_INIT_SYSTEM/"                  
13 if [ -d "$scripts_dir" ]; then                                                 
14     dest=                                                                      
15     case $DIB_INIT_SYSTEM in                                                   
16         upstart) dest=/etc/init/ ;;                                            
17         openrc) dest=/etc/init.d/ ;;                                           
18         systemd) dest=/usr/lib/systemd/system/ ;;                              
19         sysv) dest=/etc/init.d/ ;;                                             
20     esac                                                                       
21                                                                                
22     if [ -z "$dest" ]; then                                                    
23         echo "ERROR: DIB_INIT_SYSTEM ($DIB_INIT_SYSTEM) is not a known type"   
24         exit 1                                                                 
25     fi                                                                         
26     cp -RP $scripts_dir. $dest || true                                         
27 fi  
```

这里把希望开机启动的脚本拷贝到了对应的位置让他们发挥作用

## 50-store-build-settings

```shell
1 +--  2 lines: !/bin/bash-------------------------------------------------------------------------
3                                                                                
4 if [ ${DIB_DEBUG_TRACE:-0} -gt 0 ]; then                                       
5     set -x                                                                     
6 fi                                                                             
7 set -eu                                                                        
8 set -o pipefail                                                                
9                                                                                
10 if [ -e "/tmp/in_target.d/dib_environment" ]; then                             
11     cp /tmp/in_target.d/dib_environment /etc/                                  
12 fi                                                                             
13                                                                                
14 if [ -e "/tmp/in_target.d/dib_arguments" ]; then                               
15     cp /tmp/in_target.d/dib_arguments /etc/                                    
16 fi  
```

这里保存了创建dib的参数

## 80-disable-rfc3041

这个脚本里面，有用的就下面几句话

```shell
24 # This will disable the disable Privacy extensions for IPv6 (RFC3041)          
25 cat > /etc/sysctl.d/99-cloudimg-ipv6.conf <<EOF                                
26 # See https://bugs.launchpad.net/ubuntu/+source/procps/+bug/1068756            
27 net.ipv6.conf.all.use_tempaddr=0                                               
28 net.ipv6.conf.default.use_tempaddr=0                                           
29 EOF 
```

作用是禁止ipv6

## 99-autoremove
这个脚本就执行了一句话:

    apt-get -y autoremove
    
> 这一片主要讲了pre-install和install阶段的脚本的内容.  
> dib中的package-installs方法和   
> 下一篇会继续说之后的内容.
