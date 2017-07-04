---
layout:     post
title:      "Python设计模式(二) ——Borg模式"
subtitle:   " \"或许你不需要单例，而是Borg模式\""
date:       2017-03-1 12:00:00
author:     "Xion"
header-img: "img/python-design-background02.jpg"
catalog: true
tags:
    - python
    - 设计模式
---

---

<!-- toc orderedList: -->

- [故事](#故事)
- [Borg模式](#borg模式)
- [python的一个特性](#python的一个特性)
- [python代码实现](#python代码实现)
	- [定义Borg模式](#定义borg模式)
	- [使用例子](#使用例子)
- [使用注意事项](#使用注意事项)
	- [与单例模式的区别](#与单例模式的区别)
	- [GC](#gc)
	- [多线程](#多线程)

<!-- tocstop -->

---




## 故事

[Borg](http://code.activestate.com/recipes/66531/)模式是有Alex Martelii(google 雇员　python天才)利用python实现单例的一种方式。

Borg这个名字来源于Startrek剧集，背后的故事还是非常有意思的,[这里](https://blog.youxu.info/2010/04/29/borg/)有详细的故事。简而言之，Borg是一个邪恶种族，所有的 Borg 成员，都随时听从一个中央命令机构，叫做 Collective (集体). 这个集体, 随时能够和任何一个 Borg 成员通信， 在 Borg 成员里共享一切信息.

对于其他语言而言，实现单例只需要构造一个私有的构造函数，然而在python里有更加所谓*pythonic*的方法，也是Borg模式

## Borg模式

所谓的单例模式，就是确保这个类只有一个对象。虽然单例听起来感觉好像很专业的样子，这可能并非是个好主意。有时候我们可能确实想要很多对象，但是这些对象有共享的状态。我们在乎的是共享的状态，毕竟谁会在乎无限的可能

## python的一个特性

对于python的一个类而言，`self.__dict__`是可以重新绑定的。如果我们在`__init__`方法中重新绑定一个新的字典，那么所有的对象就拥有"we are one"的性质。


## python代码实现

### 定义Borg模式
```
class Borg(object):
    __shared_state = {}

    def __init__(self):
        self.__dict__ = self.__shared_state
        self.state = 'Init'

    def __str__(self):
        return self.state

```
### 使用例子
```
if __name__ == '__main__':
    rm1 = Borg()
    rm2 = Borg()

    rm1.state = 'Idle'
    rm2.state = 'Running'

    print('rm1: {0}'.format(rm1))
    print('rm2: {0}'.format(rm2))

    rm2.state = 'Zombie'

    print('rm1: {0}'.format(rm1))
    print('rm2: {0}'.format(rm2))

    print('rm1 id: {0}'.format(id(rm1)))
    print('rm2 id: {0}'.format(id(rm2)))

    rm3 = YourBorg()

    print('rm1: {0}'.format(rm1))
    print('rm2: {0}'.format(rm2))
    print('rm3: {0}'.format(rm3))

### OUTPUT ###
# rm1: Running
# rm2: Running
# rm1: Zombie
# rm2: Zombie
# rm1 id: 140732837899224
# rm2 id: 140732837899296
# rm1: Init
# rm2: Init
# rm3: Init
```
---

## 使用注意事项

### 与单例模式的区别

**Brog模式与单例模式是不同的**单例模式关注的是对象的一致，然而这可能未必是我们使用单例的原因。有可能我们关注的是状态的一致，这种情况下就可以使用Brog模式

### GC

真正的单例，在GC方面的效果还是优于Brog模式，使用python实现单例模式的例子如下：

```
class Singleton(object):
    def __new__(cls, *p, **k):
        if not '_the_instance' in cls.__dict__:
            cls._the_instance = object.__new__(cls)
        return cls._the_instance
```

### 多线程

无论是单例模式还是Brog模式，如果要支持多线程，在有效性和效率方面都需要考量一个平衡

*注*：
>所有的设计模式相关的代码都在[这里](https://github.com/xionchen/python-patterns)这也是从别的地方看到的，关于Borg的部分写的十分准确,可以直接参考
