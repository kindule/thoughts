---
title: Pod挂载Volume失败问题分析
tags: kubernetes
date: 2018-09-25 10:01:56
---

Kubernetes环境偶尔出现Statefulset中的Pod被删除新创建的Pod挂载volume失败的问题，如下图，经过一番定位分析，也让我们对于Kubernetes系统复杂程度有了新的认知。

![image-20180919164047762](http://p1sz5a5h3.bkt.clouddn.com/2018-09-20-090324.png)

在分析此问题之前，作为相关背景知识，先简单介绍对于Kubernetes存储系统的理解。

### 存储简析

存储也是Kubernetes中比较重要而复杂的系统，功能比较庞大，涉及到不同应用中，不同控制器的协作，如下图。

![](http://p1sz5a5h3.bkt.clouddn.com/2018-10-08-035245.png)

从三个维度分析
* 从卷的生命周期来讲，卷被Pod使用或者卷被回收，会依赖顺序严格的几个阶段
  * 卷被Pod使用
    * provision，卷分配成功
    * attach，卷挂载在对应worker node
    * mount，卷挂载为文件系统并且映射给对应Pod

    卷要成功被Pod使用，需要遵循以上顺序
  * 卷被回收
    * umount，卷已经和对应worker node解除映射，且已经从文件系统umount
    * detach，卷已经从worker node卸载
    * recycle，卷被回收

    卷要成功回收，需要遵循以上顺序

* 从Kubernetes存储系统来讲，卷生命周期管理的职责，又分散于不同的控制器中
  * pv controller，负责创建和回收卷
  * attach detach controller，负责挂载和卸载卷
  * volume manager，负责mount和umount卷

* controller模式，每个controller负责特定功能，并且不断进行reconcile（对比期望状态与实际状态），然后进行调整

  比如attach detach controller和volume manager中，各自都会有desiredStateOfWorld（保存期望状态）和actualStateOfWorld（保存实际状态）缓存，并且由各自的syncLoopFunc不断对比两个缓存的差异，并进行调整，下文中会介绍。

  ```go
  for {
  	desired := getDesiredState();
  	current := getCurrentState();
  	makeChanges(desired, current);
  }
  ```


结合以上三个维度，**Kubernetes需要保证卷的管理功能分布在不同控制器的前提下保证卷生命周期顺序的正确性**。以Pod使用卷为例，看Kubernetes是如何做到这一点？

### Pod启动流程

假设scheduler已经完成worker node选择，确定调度的节点，此时启动Pod前，需要先完成卷映射到Pod路径中，结合前面的分析，整个过程如下

1. 卷分配，pvc绑定pv，由pv controller负责完成

   ![](http://p1sz5a5h3.bkt.clouddn.com/20181014140720.png)

   此时pvc绑定pv

   ![](http://p1sz5a5h3.bkt.clouddn.com/20181014142538.png)

2. attach detach controlle，如果pod分配到worker node，并且对应卷已经创建，则将卷挂载到对应worker node，并且给worker node资源增加volume已经挂载对应卷的信息

   ![](http://p1sz5a5h3.bkt.clouddn.com/20181014142016.png)

   ![](http://p1sz5a5h3.bkt.clouddn.com/20181014141914.png)

   此时对应node资源状态中增加volume信息

   ```shell
   [root@10-10-88-152 ~]# kubectl get nodes 10-10-88-113 -o yaml 
   apiVersion: v1
   kind: Node
   ....
     volumesAttached:
     - devicePath: csi-add9fc778d9593d01818d65ccde7013e87327d9f675b47df42a34b860c581711
       name: kubernetes.io/csi/csi-qcfsplugin^csi-qcfs-volume-4faa18f5bbbd11e8-1365
     - devicePath: csi-5dd249387138238e8e2209eb471450a072dd6543adde7a6769c8461943c789ca
       name: kubernetes.io/csi/csi-qcfsplugin^csi-qcfs-volume-4fa9b764bbbd11e8-1366
     - devicePath: csi-bc9b81e32d84e8890d17568964c1e01af97b0c175e0b73d4bf30bba54e3f1a1e
       name: kubernetes.io/csi/csi-qcfsplugin^csi-qcfs-volume-4fa94533bbbd11e8-1364
     volumesInUse:
     - kubernetes.io/csi/csi-qcfsplugin^csi-qcfs-volume-4fa94533bbbd11e8-1364
     - kubernetes.io/csi/csi-qcfsplugin^csi-qcfs-volume-4fa9b764bbbd11e8-1366
     - kubernetes.io/csi/csi-qcfsplugin^csi-qcfs-volume-4faa18f5bbbd11e8-1365
   ```

3. volume manager在worker node中负责将卷挂载到对应路径

   * pod分配到本worker node后，获取Pod需要的volume，通过对比node状态中的volumesAttached，确认volume是否已经attach到node节点，如果attach到node节点则将自身actualStateOfWorld中的volume状态设置成attached

     ![](http://p1sz5a5h3.bkt.clouddn.com/20181014165852.png)

   * 如果已经attached成功，则进入到文件系统挂载流程
     * 第一步，先挂载到node中全局路径，比如`/var/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-3ecd68c7b7d211e8/globalmount`

     * 第二步，映射到Pod对应路径，比如`/var/lib/kubelet/pods/49a5fede-b811-11e8-844f-fa7378845e00/volumes/kubernetes.io~csi/pvc-3ecd68c7b7d211e8/mount`

     * 第三步，actualStateOfWorld中设置volume为挂载成功状态

       ![](http://p1sz5a5h3.bkt.clouddn.com/20181014165935.png)

4. pod controller确认卷已经映射成功，启动Pod，此处不详细展开

### Pod被删除的过程

1. pod controller watch到pod处于被删除状态，执行killPod操作，删除Pod，此处不详细展开
2. volume manager获取被删除的Pod，会执行如下几步
   * 第一步，通过DeletePodFromVolume将Pod从desiredStateOfWorld中删除，此处，如果volume关联的所有Pod都被删除，则此volume也从desiredStateOfWorld的缓存中清除
   * 第二步，将volume到pod的映射解除
   * 第三步，如果volume中没有pod将说明此worker node中volume不需要挂载，此时执行umountDevice，将global path umount
3. attach detach controller发现挂载到node节点的volume没有被使用，执行detach操作，将卷从node节点detach，此时卷完全处于集群中未被使用的状态。

Controller基于watch机制通过etcd实现状态交互，整个过程似乎完美，那么结合我们遇到的问题，statefulset中Pod被删除会发生什么？

场景一 Pod被删除，volume manager umount，statefulset立即创建Pod，scheduler立即指定Pod到原节点

1. volume manager发现卷没有绑定pod，需要被umount，在deviceUmount，devicePath为空
2. attach detach controller发现卷需要detach，detach volume
3. volume manager发现卷没有绑定pod，先确认是否又

场景二 Pod被删除，volume manager umount/deviceUmount，attach detach controller detach，statefulset创建Pod，scheduler指定Pod到原节点

场景三 Pod被删除，volume manager umount/deviceUmount，statefulset创建Pod，scheduler指定Pod到原节点，问题出现

结合出现问题的日志，

总结，Kubernetes之所以在当前成为容器编排领域的事实标准，原因很多，但是对我们来讲基于声明式API的编程范式是我们依赖Kubernetes的重要原因，当然在其解决的问题规模下复杂程度也不言而喻，总之，一句话，没有银弹。
