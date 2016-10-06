---
layout:     post
title:      "从日志分析DIB流程(3) extra-data阶段"
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

# 从日志分析DIB流程(3) extra-data阶段
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



# extra-data阶段
> 1185:1417line  
> extra-data阶段的工作是将一些数据拷贝到镜像中备用

首先一览该阶段的脚本

```shell
-rwxrwxr-x  1 xion xion 1125 10月  1 11:22 01-copy-apt-keys
-rwxrwxr-x  1 xion xion  381 10月  1 11:22 10-create-pkg-map-dir
-rwxrwxr-x  1 xion xion  754 10月  1 11:22 20-manifest-dir
-rwxrwxr-x  1 xion xion  312 10月  1 11:22 50-store-build-settings
-rwxrwxr-x  1 xion xion 2387 10月  1 11:22 99-enable-install-types
-rwxrwxr-x  1 xion xion  423 10月  1 11:22 99-squash-package-install
xion@xion-Aspire-7750G:/tmp/dib_build.oRGH38zL/hooks/extra-data.d$ 

```

这些脚本的作用和写法都比较简单,所以这里只说每个脚本的作用是什么

## 01-copy-apt-keys 
> 这个脚本属于dpkg element
它的作用是拷贝apt-keys到 tmp/apt_keys目录下
 
## 10-create-pkg-map-dir     
> 这个脚本属于pkg-map element
它的作用是把每个元素中的pkg-map文件拷贝到 /user/share/pkg-map/$元素目录下

## 20-manifest-dir
> 这个脚本属于manifests element
它的作用是创建了一个用于存放manifest文件的文件夹

## 50-store-build-settings*
> 这个脚本属于base element
它作用是把环境变量写到了钩子文件夹下的文件中

## 99-enable-install-types
> 这个脚本属于install-types element
它的作用是让安装的软件可以有不同的类型,例如git,pip等  
在脚本中,建立了正确的安装类型的软链接  
如果没有指定安装的类型,就用默认的方式安装

## 99-squash-package-install
> 这个脚本属于package-install element

脚本的内容如下,十分简单,但是package-install-squash这个命令值得细看一下
```shell
1 #!/bin/bash                                                                                      
2 if [ ${DIB_DEBUG_TRACE:-0} -gt 0 ]; then                                       
3     set -x                                                                     
4 fi                                                                             
5 set -eu                                                                        
6 set -o pipefail                                                                
7                                                                                
8 # Search for python first in case we are in a venv with python3 which          
9 # should take precedence                                                       
10 python_path=$(command -v python || command -v python2 || command -v python3)   
11                                                                                
12 sudo -E $python_path $(dirname $0)/../bin/package-installs-squash --elements="$IMAGE_ELEMENT" --p    ath=$ELEMENTS_PATH $TMP_MOUNT_PATH/tmp/package-installs.json
~                                                            
```

在**elements/package-installs/bin**下的package-installs-squash中说这个脚本的作用是把所有的安装包的文件汇总到一个文件中.

查看一下目前镜像中的文件/tmp/package-install.json
```json
1 {                                                                              
2  "install.d": {                                                                
3   "install": [                                                                 
4    [                                                                           
5     "ccache_package",                                                          
6     "base"                                                                                       
7    ],                                                                          
8    [                                                                           
9     "linux-image-generic",                                                     
10     "ubuntu"                                                                   
11    ],                                                                          
12    [                                                                           
13     "dkms",                                                                    
14     "dkms"                                                                     
15    ]                                                                           
16   ]                                                                            
17  }                                                                             
18 }                                                                              
~     
```
它的格式是
```
阶段:{
  操作:
  [
  包名,
  元素名
  ]  
}
 
```

之后有的包是按照这个文件中的描述进行安装的.

---

extra-data 阶段的内容大概就是这些,下一篇介绍pre-install阶段.

