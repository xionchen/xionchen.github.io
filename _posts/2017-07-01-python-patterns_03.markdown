---
layout:     post
title:      "Python设计模式(三) ——builder模式"
subtitle:   " \"建造者模式与工厂模式比较类似,但是略有不同\""
date:       2017-07-1 12:00:00
author:     "Xion"
header-img: "img/python-design-background03.jpg"
catalog: true
tags:
    - python
    - 设计模式
---




<!-- toc orderedList:0 -->

- [建造者模式](#建造者模式)
- [类图](#类图)
- [例子](#例子)
	- [代码](#代码)
	- [类图](#类图-1)
- [优点&缺点](#优点缺点)
	- [优点](#优点)
	- [缺点](#缺点)
- [规则](#规则)

<!-- tocstop -->







## 建造者模式


- Builder模式的关键在于，将“要做什么”与“做出来”分离，将“how to compose”与“construction”分离：
- Director知道“how to compose”， Builder负责“how to construction”；
- Director负责指挥， Builder负责实施；
- Director负责制定步骤，Builder负责执行具体步骤；


**例如我们要建造一个房子，在建造者模式中。有专门负责建造房子的人(Director)，用来建造房子的步骤(Builder)，和定实际建造房房子的细节(xxxBuilder)以及建造的房子(product)**

乍一看似乎和工厂模式或者抽象工厂模式十分类似,区别在于抽象工厂和工厂模式是为了实现多态,而建造者模式是为了屏蔽复杂对象的创建,将可能存在的多个对象的构建通过"建造者"统一起来,提供固定的创建者.

## 类图

构建者模式的类图如下所示:

plantuml代码如下
![](/img/builder_pattern.png)

使用的puml代码如下
```
skinparam monochrome true
"Director" o-- "Builder"
"Builder" <|-- "ConcreteBuilder"
"ConcreteBuilder" ..> Product : << create >>

class Director{
  builder: Builder
  construct()
}
class Builder{
  builderPart()
}
class ConcreteBuilder{
  builderPart()
  getResult() :Product
}
```
上面的类图中各个部分的作用如下:
- Director类提供了构造的函数
- Builder提供了创造project的接口
- ConcreteBuilder是实际的Builder

## 例子

### 代码

Director类

```
import ABC

# Director
class Director(object):

    def __init__(self):
        self.builder = None

    def construct_building(self):
        self.builder.new_building()
        self.builder.build_floor()
        self.builder.build_size()

    def get_building(self):
        return self.builder.building
```

Builder类
```
# Abstract Builder
class Builder(object):
    __metaclass__ = ABCMeta
    def __init__(self):
        self.building = None

    def new_building(self):
        self.building = Building()

    @ABC.abstractmethod
    def build_floor(self):
        raise NotImplementedError

    @ABC.abstractmethod
    def build_size(self):
        raise NotImplementedError
```

两个ConcreteBuilder类
```
# Concrete Builder


class BuilderHouse(Builder):

    def build_floor(self):
        self.building.floor = 'One'

    def build_size(self):
        self.building.size = 'Big'


class BuilderFlat(Builder):

    def build_floor(self):
        self.building.floor = 'More than One'

    def build_size(self):
        self.building.size = 'Small'
```

Product类
```
# Product
class Building(object):

    def __init__(self):
        self.floor = None
        self.size = None

    def __repr__(self):
        return 'Floor: {0.floor} | Size: {0.size}'.format(self)
```

使用例子
```
if __name__ == "__main__":
    director = Director()
    director.builder = BuilderHouse()
    director.construct_building()
    building = director.get_building()
    print(building)
    director.builder = BuilderFlat()
    director.construct_building()
    building = director.get_building()
    print(building)
```

### 类图
![](\img\python-design-builder-pattern-example.png)

puml代码
```
skinparam monochrome true
"Director" o-- "Builder"
"Builder" <|-- "BuilderHouse"
"Builder" <|-- "BuilderFlat"
"BuilderHouse" ..> Building : << create >>
"BuilderFlat" ..> Building : << create >>

class Director{
  builder: Builder
  construct()
}

class Builder{
  builderPart()
}

```
## 优点&缺点

### 优点

- 同一个Product类实际上可以是不同的东西(取决于ConcreteBuilder)
- 封装了构造和表示的代码
- 控制了构建的过程

### 缺点

- 需要为每个project提供一个单独的ConcreteBuilder类
- 需要builder类是可变的

## 规则

- creational的模式都是可以互补的,在使用一种模式的时候,你也可以使用别的,例如把builder模式和单例模式结合使用
- Builder模式最重要的意义是针对复杂的初始化过程.抽象工厂强调的是产品的系列.Builder最后一步才返回对象,但是一旦抽象工厂实现了,对象马上就能返回
- 通常,设计的开始会使用工厂模式(简单,个性,易于子类的扩展)但是随着面对需求的复杂会专向抽象工厂,Prototype或者是builder模式,这两种模式更加灵活同事也更加复杂.


*注*：
>所有的设计模式相关的代码都在[这里](https://github.com/xionchen/python-patterns)这也是从别的地方看到的，关于builder模式的部分写与文中的内容基本相同,可以直接参考
