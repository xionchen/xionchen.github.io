---
layout:     post
title:      "从日志分析DIB流程(3) extra-data和pre-install阶段"
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

# 从日志分析DIB流程(3) extra-data和pre-install阶段
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

---



# 该例子中的阶段的脚本
working



