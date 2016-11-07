---
layout:     post
title:      "Vim Days"
subtitle:   " \"这个博客的目的是介绍我自己每天学习vim的内容,每天的内容大概会在10分钟左右,也不是每天都会写.积累vim的用法\""
date:       2016-11-1 12:00:00
author:     "Xion"
header-img: "img/post-bg-vim.jpg"
catalog: true
tags:
    - vim

---

# Vim Days
> 本来这个博客叫vim everyday的,但是我发现一个问题: 每天都坚持,是一件十分困难的事.同时,中间间隔了一些时间也没关系,所以改名叫vim days.可以看做是一些vim的小练习.

- 首先,这里不会介绍插件.因为不是所有环境都有插件.
- 其次,如果想一起学,除了day0以外,每天(不用严格的每天)大概就花10分钟左右,因为学习vim是个过程,慢慢学就会了,所以每天的内容不多,但是没事休闲的时候要记得玩玩.
- 然后,开始的时候不要在工作中使用vim.
- 再然后,把键盘的大写锁定映射为Esc键.
- 最后,总有一天你会发现,vim会极大的提升你的效率.

# day0 基础

>如果之前没有使用过vim,下面有一些很好的交互式的网站,拿出一个晚上把这些联系都敲一遍,基本上能够入门(或者你也可以看文档).

- [Learn vim in a week](https://www.youtube.com/watch?v=_NUO4JEtkDw)我就是看这个视频找到了学习方法

- [OpenVim](http://www.openvim.com/tutorial.html)交互式的vim学习网址,能够学到最基础的vim操作

- [Vim Adventures](http://vim-adventures.com/)vim小游戏

- [vim genius](http://vimgenius.com/)交互式的vim教程


## day1 用visual模式练习移动
有时候,visual模式可以用来帮助我们很快的查看选择的范围.举个例子,比如我们要复制一个方法体:

```
function specialAdd(a, b) {
  if (!a || !b) {
    return 0;
  }
  return a + b;
}
```
如果光标在 **function** 上,然后输入 **v/}** ,就会把整个if语句选中.然后按 **n** 会选中整个方法.

接下来可以试试 **}** 移动(选中整个段).如果输入 **v}** 就会选中整个方法.因为是在视觉模式下,查看移动就很容易.然后我们就可以结合 **y** 利用 **y}** 来复制(yank) 整个方法.

但是如果方法里面有空行,这个就不靠谱了

```
function specialAdd(a, b) {
  if (!a || !b) {
    return 0;
  }

  return a + b;
}
```

在这里,整个方法就不是"paragraph".所以 **y}** 就用不了,但是我们可以用'sections'使用 **v][** 来选择整个方法,用 **y][** 就可以复制这个方法.

使用 **vaB** 可以选择整个方法的body或者是整个if语句,具体看光标在哪.

使用 **v%** 也能选中整个方法提,因为 **%** 会选择到下一个括号的位置.所以使用 **y%** 就会赋值到下一个括号的内容.

现在你就学会了怎么用visual模式来查看选择区域的方法啦.



## day2 粘贴内容到命令行下

在使用vim的时候,使用后会使用 **:** 来输入一些命令,如果希望在输入命令的时候也能够粘贴,就需要用到 **寄存器**

**CTRL-R** 指令会把寄存器中的内容粘贴懂啊当前输入的空间中. **:help ins-special-keys** 可以查看它的用法细节.

假设现在我们要搜索一个词.
- 首先,使用**"ayw** 来复制这个词,然后按esc.  
- 然后输入 **/** 来搜索.  
- 然后按 **CTRL-R** 和 **a** 来放只寄存器a中的内容

还有一种更快捷的方式,直接用 **yw** 来把内容复制到默认寄存器中,然后按 **CTRL-R_"** 来输入默认寄存器的内容.

寄存器还是个很爽的功能的,值得一试.

## day3 搜索作为动作

vim里面有一种的玩法--动作+操作,例如复制一个词, **yaw**(yank a word) ,删除一个句子, **das**(delete a sentence) , **yap** (复制一段)

这里的 'aw' 'as' 'ap' 被成为motion,除了这些常见的motion以外, 搜索也能作为motion.
距离而言,使用 **d/,<CR>** (CR表示回车) 可以删除到逗号. 更加快捷的方式是使用 **dt,**.

(扩展阅读 vim中的motion)[http://vimdoc.sourceforge.net/htmldoc/motion.html#operator]
