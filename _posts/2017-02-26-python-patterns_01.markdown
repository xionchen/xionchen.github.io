---
layout:     post
title:      "Python设计模式(一) ——抽象工厂"
subtitle:   " \"抽象工厂模式是常见设计模式之一\""
date:       2017-02-25 12:00:00
author:     "Xion"
header-img: "img/python-design-background.jpg"
catalog: true
tags:
    - python
    - 设计模式
---


<!-- toc orderedList: -->

- [定义](#定义)
- [模式例子](#模式例子)
- [场景](#场景)
- [UML](#uml)
- [python 代码实现](#python-代码实现)
	- [用户类](#用户类)
	- [抽象工厂](#抽象工厂)
	- [抽象产品](#抽象产品)
	- [实际产品](#实际产品)
	- [实际工厂](#实际工厂)
	- [调用](#调用)
- [抽象工厂模式与工厂模式与简单工厂模式的关系](#抽象工厂模式与工厂模式与简单工厂模式的关系)
- [抽象工厂的优缺点](#抽象工厂的优缺点)
	- [优点](#优点)
	- [缺点](#缺点)

<!-- tocstop -->


最近写了一些python的代码之后发现一个问题，之前学过的设计模式，包括面向对象的一下思想都没用用上，正好在网上找到了一个包含很多python设计模式的项目(在结尾引用中有)通过python把这些设计模式都实现一遍，也可以作为使用的模板


<!-- untoc -->
# 抽象工厂

## 定义

抽象工厂模式(Abstract Factory Pattern)：提供一个创建一系列相关或相互依赖对象的接口，而无须指定它们具体的类。

## 模式例子

抽象工厂模式中有工厂和产品簇的概念。而一簇的产品都是成套出现的。比如现在要给每个士兵发一套武器，包括枪和子弹。步枪和步枪子弹，手枪和手枪子弹。生产步枪的工厂就是步枪工厂，而生产手枪的工厂就是手枪工厂。步枪工厂和手枪工厂都是工厂，这就是一种抽象工厂的例子

## 场景

例如windows的bottons和unix的bottons

## UML
以士兵的例子为例uml如下

![](/img/python_abs_fac_uml.png)

下面是plantuml的编码
```
@startuml
skinparam monochrome true
"AbstractFactory" <|-- "RifleFactory"
"AbstractFactory" <|-- "HandgunFactory"
"AbstractGun" <|-- Rifle
"AbstractGun" <|-- Handgun
"AbstractBullet" <|--  "RfileBullet"
"AbstractBullet" <|-- "HandgunBullet"
"RifleFactory" ..>  Rifle
"RifleFactory" ..> "RfileBullet"
"HandgunFactory" ..> Handgun
"HandgunFactory" ..> "HandgunBullet"
@enduml
```
## python 代码实现

### 用户类
```
#!/usr/bin/env python
# -*- coding: utf-8 -*-

# http://ginstrom.com/scribbles/2007/10/08/design-patterns-python-style/

"""Implementation of the abstract factory pattern"""
import abc
import random


class Solider(object):

    def __init__(self, gun, buttle):
        self.gun = gun
        self.buttle = buttle

    def fire(self):
        self.gun.pong()
        self.buttle.pa()
```
### 抽象工厂
```
class Gunfactory(object):
    __metaclass__ = abc.ABCMeta

    @abc.abstractmethod
    def get_gun(self):
        pass

    @abc.abstractmethod
    def get_bullet(self):
        pass
```
### 抽象产品
```
class Gun(object):
    __metaclass__ = abc.ABCMeta

    @abc.abstractmethod
    def pong(self):
        pass


class Bullet(object):
    __metaclass__ = abc.ABCMeta

    @abc.abstractmethod
    def pa(self):
        pass

```
### 实际产品
```
class Rifle(Gun):

    def pong(self):
        print "Rifle fire,pong!"


class Handgun(Gun):

def pong(self):
print "Handgun fire,pong,pong,pong"


class RifleBullet(Bullet):

def pa(self):
print "Rifle buttle,pa!"


class HandgunBullet(Bullet):

def pa(self):
print "Handgun buttle,pa,pa,pa"
```
### 实际工厂
```
class RifleFactory(Gunfactory):

def get_gun(self):
return Rifle()

def get_bullet(self):
return RifleBullet()


class HandgunFactory(object):

def get_gun(self):
return Handgun()

def get_bullet(self):
return HandgunBullet()
```
### 调用
```
if __name__ == "__main__":
rifle_factory = RifleFactory()
handgun_factory = HandgunFactory()
factories = [rifle_factory, handgun_factory]
for i in range(4):
factory = random.choice(factories)
gun = factory.get_gun()
bullet = factory.get_bullet()
solider = Solider(gun, bullet)
solider.fire()
print("=" * 20)

### OUTPUT ###
# Rifle fire,pong!
# Rifle buttle,pa!
# ====================
# Handgun fire,pong,pong,pong
# Handgun buttle,pa,pa,pa
# ====================
# Handgun fire,pong,pong,pong
# Handgun buttle,pa,pa,pa
# ====================
# Rifle fire,pong!
# Rifle buttle,pa!
# ====================
```


## 抽象工厂模式与工厂模式与简单工厂模式的关系
当抽象工厂模式中每一个具体工厂类只创建一个产品对象（工厂只造枪），也就是只存在一个产品等级结构时，抽象工厂模式退化成工厂方法模式；当工厂方法模式中抽象工厂与具体工厂合并，提供一个统一的工厂（只有步枪工厂）来创建产品对象，并将创建对象的工厂方法设计为静态方法时，工厂方法模式退化成简单工厂模式。

## 抽象工厂的优缺点
### 优点
- 客户并不需要知道什么被创建。由于这种隔离，更换一个具体工厂就变得相对容易。
所有的具体工厂都实现了抽象工厂中定义的那些公共接口，因此只需改变具体工厂的实例，就可以在某种程度上改变整个软件系统的行为。

- 另外，应用抽象工厂模式可以实现高内聚低耦合的设计目的，因此抽象工厂模式得到了广泛的应用。
当一个产品族中的多个对象被设计成一起工作时，它能够保证客户端始终只使用同一个产品族中的对象。这对一些需要根据当前环境来决定其行为的软件系统来说，是一种非常实用的设计模式。
增加新的具体工厂和产品族很方便，无须修改已有系统，符合“开闭原则”。

### 缺点
- 在添加新的产品对象比较困难，因为要对所有的工厂都添加类似产品
开闭原则的倾斜性（增加新的工厂和产品族容易，增加新的产品等级结构麻烦）。

---

*注*：
>所有的设计模式相关的代码都在[这里](https://github.com/xionchen/python-patterns)这也是从别的地方看到的，我自己fork了一份，但是里面有些地方不是很准确，所有我自己修改了一份
