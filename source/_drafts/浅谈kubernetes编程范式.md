---
title: 浅谈kubernetes编程范式
date: 2018-09-25 10:01:56
tags: kubernetes
---

# 学习Kubernetes编程范式

## 前言

Kubernetes之所以在当前成为容器编排领域的事实标准，原因有很多，但是从我们的角度来讲，其声明式的资源管理方式及基于资源的编程范式，是我们重度依赖Kubernetes的重要原因。本文尝试结合我们在Kubernetes存储领域的实践以及遇到的问题给大家分享一下kubernetes编程范式及其特点

## 声明式API

kubernetes最显著的特点是为我们提供了一套声明式的API

**声明式API**

以创建3个Nginx实例为例，大致需要配置段如下，Nginx实例数量、Nginx镜像版本，提交给kubernetes后，kubernetes会自动帮助我们达到期望的状态。

```yaml
  replicas: 3
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

声明式API在kubernetes资源管理的使用好处很多，比如对于用户友好、配置文件更简单明了、更加精确等，但是其中很重要的一点是实现了资源定义的接口与具体实现分离，下面做详细介绍。

### 接口与实现分离

还是以创建3个Nginx为例，通过声明式API，客户端给定的是对于资源的期望---创建3副本Nginx，系统获取这份资源请求后的实现可以有任意不同方案

比如，单个应用程序下实现所有业务逻辑，如下图

![](http://p1sz5a5h3.bkt.clouddn.com/2018-10-10-011642.png)

当然也可以采用分布式的方式，由多个服务共同协作实现业务逻辑，甚至多个服务分布于不同服务器都可以，只要能完成期望

![](http://p1sz5a5h3.bkt.clouddn.com/2018-10-10-033546.png)

总之，声明式API提供了在不改变Kubernetes提供接口的情况下，不断优化系统的可能性。那么Kubernetes基于声明式API，如何实现自身功能呢？Kubernetes的核心代码中大量使用一种特定模式实现功能，下面给大家做介绍。

## Kubernetes编程范式

#### 重度依赖etcd

kubernetes集群管理，重度依赖于etcd，主要有以下几个作用

1. 实现数据持久化，kubernetes中创建的资源全部都存储在etcd中，并且etcd是kubernetes集群系统唯一的持久化存储。
2. 利用etcd实现接耦组件业务逻辑，控制器只需要拿到etcd中资源当前状态，根据当前情况执行自身处理逻辑，然后再更新资源在etcd中的状态。
3. 利用etcd的watch机制实现事件触发，控制器watch资源变化，再结合回调机制，实现控制器业务逻辑。

核心逻辑如下图所示

![](http://p1sz5a5h3.bkt.clouddn.com/2018-10-10-033258.png)

主要工作介绍如下

1. 客户端创建的对象会持久化到etcd
2. 其他控制器，通过watch etcd中自身关注的对象，执行具体任务，并更新etcd中对象状态。

具体到单个控制器，会以如下方式工作

![](http://p1sz5a5h3.bkt.clouddn.com/2018-10-08-034242.png)

控制器

#### 调和(reconcile)

Kubernetes编程范式中另一个重要特点是，控制器端大量reconcile模式

```go
for {
	desired := getDesiredState();
	current := getCurrentState();
	makeChanges(desired, current);
}
```

获取资源期望的状态，并且获取资源当前的状态，然后根据期望状态和当前状态的差异做出调整，直到达到期望状态。
