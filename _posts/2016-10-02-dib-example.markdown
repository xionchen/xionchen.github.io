---
layout:     post
title:      "Diskimage-builder 流程分析"
subtitle:   " \"diskimage-builder 是openstack社区用于制作镜像的工具.为了深入了解dib制作镜像的全过程,对一个简单的例子进行贯通的分析.\""
date:       2016-10-02 12:00:00
author:     "Xion"
header-img: "img/post-bg-tech.jpg"
catalog: true
tags:
    - opentack-infra
    - diskimage-builder
---

# 从日志分析DIB流程
>dib这几篇博客是干货,基于下面三个客观条件:
> 1. 社区较为优秀的代码质量
> 2. 因为工作需求,需要深入理解,所以花费在这上面的时间精力较多.
> 3. DIB项目主要以shell为主,并且涉及到linux的很多方面.非常适合学习linux和shell.

这几篇博客的思路是,先看看"猪是怎么跑的","再吃猪肉".
[Diskimage-builder简介](2016-10-01-dib-introduction.markdown) 介绍了使用dib的安装和使用需要知道的一些知识,可以理解为"怎么让猪跑".

这里先由一个简单的例子,来看看"猪是怎么跑的":

      $ disk-image-create vm ubuntu-minimal

这里有日志文件被我放在这里了:  [DIB_日志](https://github.com/xionchen/note/blob/master/dib/Dib-log.markdown)

working...
