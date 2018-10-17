---
title: Clean Architecture读书笔记（一）
date: 2018-09-05 08:20:38
tags: architecture
mathjax: true
---

![](http://p1sz5a5h3.bkt.clouddn.com/20180915150403.png)

> The primary purpose of architecture is to support the life cycle of the system. Good architecture makes the system easy to understand, easy to develop, easy to maintain, and easy to deploy. The ultimate goal is to minimize the lifetime cost of the system and to maximize programmer productivity.

## 什么是架构？

架构关注的是高层抽象，脱离于底层细节。架构和设计有何不同？

以设计房屋为例（IT业常常将软件工程与建筑工程类比），整体风格（欧式、日式）、房屋的形状（三室两厅还是四室三厅）、外观（方形还是举行）、立面以及房间的布局是为架构。深入局部会有大量细节，家具的摆放、灯具的位置、餐厅配色。所有细节又支撑了架构所做的决定。架构和细节就是整个房屋的完整设计。

讲到软件开发，底层的细节和高层的架构才是整个软件开发的全部，它们共同决定了整个系统的模样。

## 编程范式

![](http://p1sz5a5h3.bkt.clouddn.com/20180916082851.png)

软件架构都是基于代码来实现的，所以谈到软件架构先从编程开始讨论，编程语言有很多，但常用的可以总结为几种[范式](https://zh.wikipedia.org/zh-hans/%E7%BC%96%E7%A8%8B%E8%8C%83%E5%9E%8B)，关于编程范式的讨论和发展历史不在本文讨论范围之内，具体可以参考《冒号课堂》，也是非常推荐的一本书。

### 结构化编程

结构化编程让我们递归的将一个程序分解成一系列的小的可以测试的子程序。然后我们对每个子程序进行测试，如果子程序测试成功，程序正常执行。结构化编程引入了for、while等循环结构，来取代传统的goto，并且希望借此来改善程序的可读性、质量以及开发时间。

### 面向对象编程

作者说，好架构的基础就是对面向对象编程的深刻理解，那么怎么理解面向对象编程呢？

一种说法

> The combination of data and function

另一种说法

> A way to model the real world

前人对OO特性的总结

>* 封装
>* 继承
>* 多态

但是作者认为OO最终要的特性是多态，理由如下

对比c语言，可以发现

1. 结构化语言也有封装

2. 结构化语言也有“继承”

3. 结构化语言也能实现多态

作者认为多态是OO最重要的特性，在结构化语言中也有多态，例如 C语言，C通过指针指向不同的function实现多态，但是这种方式需要强制类型转换非常危险，但是OO有了多态机制并且通过interface的方式很容易实现接口的复用，有了interface作者认为OO最重要的特性就是实现了依赖的反转（Dependency Inverse），_golang中的interface_

> OO allows the plugin architecture to be used anywhere, for anything.
>
> This is called dependency inversion, and its implications for the software architect are profund.
>
> The fact that OO languages provide safe and convenient polymorphism means that any source code dependency, no matter where it is, can be inverted.
>
> With this approacdh, software architects working in systems written in OO languages have absolute control over the direction of all source code dependencies in the system.They are not constrained to align those dependencies with the flow of control.No matter which module does the calling and which module is called, the software architect can point the source code dependency in either direction.

## Design Principles

![](http://p1sz5a5h3.bkt.clouddn.com/20180915150135.png)

_S.O.L.I.D应该算是《Clean Architecture》这本书前半部分的重头戏_

> The goal of the principles is the creation of mid-level software structures that:
>
> * Tolerate change
> * Are easy to understand
> * Are the basis of components that can be used in many software systems.

如果将编程比作武侠小说中的武功，那么SOLID可以看作“面向对象”这一门派中最核心的内功心法，就好比少林寺有《易筋经》，精通《易筋经》心法，再学少林长拳轻而易举。此书中作者对SOLID每个原理都做了详细解释，每一章都值得仔细阅读反复推敲，直到了然于心。

**如何理解SOLID？**

单一职责是所有设计原则的基础，开闭原则是终极目标，里式替换是面向对象中帮助实现开闭原则的方法，接口隔离指导如何实现具有里式替换行为的类，并且也体现了单一职责，依赖反转指导如何实现接口。[参见](https://zhuanlan.zhihu.com/p/44344256)

**SOLID在Kubernetes中的体现**

最近阅读Kubernetes存储相关代码，又阅读了《Clean Architecture》此书，顿时甚觉豁然开朗，原来kubernetes在代码设计上几乎全部遵循SOLID设计原则，且看

* volume manager中几乎所有的控制流依赖，都会基于interface实现依赖反转

  ![](http://p1sz5a5h3.bkt.clouddn.com/20180929213659.png)

* VolumePlugin定义Mount/Umount/...方法，ceph/ebs/gcd/glusterfs众多plugin只需依据里氏替换原则实现volumePlugin接口即可实现存储类型扩展

  ![](http://p1sz5a5h3.bkt.clouddn.com/20180929222513.png)

* 遵循接口隔离原则，如无必要勿增实体（奥卡姆剃刀理论），不设计大而全的接口，volumePlugin提供最小化mount/umount等基础方法，ExpandableVolumePlugin/BlockVolumePlugin除了具备volumePlugin接口的方法还各自包含自身业务需要方法

SOLID告诉我们如何设计类，如何构建类与类之间的关系，如何设计组件内层次结构。SOLID结合我们对业务的掌握，一个容易维护、容易扩展的组件即可实现。

## Package Principles

### 模块的内聚

用来指导如何讲类聚合成为package

![](http://p1sz5a5h3.bkt.clouddn.com/20181004113403.png)

#### REP：The Reuse/Release Equivalence Principle

> The granule of reuse is the granule of release.

对于任何使用的组件只要是会发生修改，一定要有版本追踪，从反面来讲，**没有版本追踪的包是不能使用的**。

* 便于追踪问题
* 便于确认每个版本更新什么功能
* 便于确认和其他组件的兼容性

#### CCP: The Common Closure Principle

>Gather into components those classes that change for the same reasons and at the same times.Separate into different components those classes that change at different times and for different reasons.

CCP原理与SRP相通，组件职责单一，更利于发布、更新、部署。

#### CRP: The Common Reuse Principle

>Don't force users of a component to depend on things they don't need.

此原则是ISP原则在模块层面的抽象，模块中不应该有调用者用不到的类。

### 模块的耦合

用来指导如何处理package和package之间的关系

#### ADP: The acyclic dependencies principle

A对于B有依赖，即是B对于A施加影响，环形依赖

* 环形依赖，导致隐式依赖
* 测试困难，trouble
* 非常难以确认按照什么顺序来build，以Java为例

如何解决环形依赖？DIP，增加独立组件等等

#### SDP: The stable dependencies principle

>One sure way to make a software component difficult to change, is to make lots of other software components dependon on it. A component with lots of incoming dependencies is very stable because it requireds a great deal of work to reconcile any changes with all the dependent components.

什么样的组件稳定？

作者认为，如果一个组件被其他组件依赖得越多，则组件越稳定，并且我们可以通过一个公式定义组件的稳定程度

* Stability Metrics，`I=FanOut/(FanIn+FanOut)`
  * FanOut组件中类被其他组件依赖的数量
  * FanIn组件中类依赖外部其他组件的数量

* 其中，`I=0`则组件最稳定，`I=1`则组件最不稳定

* 组件的依赖关系应该是，I值低的组件依赖I值高的组件

#### SAP: The stable abstractions principle

作者认为，如果组件中用于抽象的类越多则组件的可扩展性越高，抽象的类越少则组件的可扩展性越低，并且可以通过一个公式定义

* Abstraction Metrics，`A=Na/Nc`
  * Na为组件中抽象的类和接口的数量
  * Nc为组件中所有类和接口的数量
* 其中，A=0则组件全部为具体的类，组件可扩展性很低，A=1则组件全部为抽象的类

#### 平衡点

`SDP`告诉我们组件越是受到依赖越稳定，`ADP`告诉我们组件中抽象类越多越容易扩展，这两者如何结合使用呢？

以(I, A)为(x,y)轴

1. 接近(0,0)说明组件，抽象程度低，组件稳定性高，太低则组件可扩展性差并且不易修改，可以想想SQL
2. 接近(1,1)说明组件，抽象程序高，组件稳定性低，大部分都是抽象且不稳定的组件，说明这个组件没什么用处

![](http://p1sz5a5h3.bkt.clouddn.com/20181005192831.png)

##### The Main Sequence

​    作者认为组件的A和I值分布在这样一条线上是最好，即连接(0,1)和(1,0)的直线，作者称为`The Main Sequence`组件的(A,I)，越靠近`The Main Sequence`架构越健康。

##### Distance

`D=A+I-1`是到`The Main Sequence`的距离，D值在0-1之间，D越小说明越接近`The Main Sequence`，D越大说明越远离`The Main Sequence`，这样，我们可以通过每次包发布时，D值确认组件的健康程度（是否朝着`The Main Sequence`）发展。

![](http://p1sz5a5h3.bkt.clouddn.com/20181005194455.png)