---
title: container-conception
date: 2018-09-26 16:14:37
tags: kubernetes
---

## container conception

## docker

docker有两种解释

1. docker是一家公司
2. docker也是一项基于容器的技术

## moby

moby是docker容器技术的开源版本

## dockerd

docker技术的管理进程，管理镜像/容器等

## containerd

dockerd实际上调用了containerd实现对于镜像/容器等的管理

## containerd-shim

containerd调用containerd-shim

## runc

容器运行时，封装cgroup/namespace调用

## runv

虚拟化容器，运行时

## cri

kubernetes定义调用容器的接口

## oci

开放容器标准

## katacontainers

一个致力于构建与容器一样轻量级的虚拟机，并且保有虚拟机本身基于内核层面安全隔离属性的开源项目

## pouch

阿里集团内部的容器技术与docker相对应

## gvisor

谷歌发布的一个用于提供安全隔离的轻量级容器运行时沙箱

## csi

容器存储接口

## cni

容器网络接口

## cri

容器运行时接口

## cri-o

容器运行时接口，兼容OCI接口

## libcontainer

用于容器管理的包，管理namespaces、cgroups、capabilities以及文件系统来对容器控制

![](http://p1sz5a5h3.bkt.clouddn.com/20181006165323.png)

