---
title: Pod挂载Volume失败问题分析
tags: kubernetes
date: 2018-09-25 10:01:56
---

Kubernetes环境偶尔出现Statefulset中的Pod被删除，新启动的Pod（还是调度到原有节点）挂载volume失败的问题，如下图，经过一番定位分析，也让我们对于Kubernetes系统复杂程度有了新的认知。

![image-20180919164047762](http://p1sz5a5h3.bkt.clouddn.com/2018-09-20-090324.png)

在分析此问题之前，作为相关背景知识，先简单介绍对于Kubernetes存储系统的理解。

### 存储系统简析

存储也是Kubernetes中比较重要而复杂的系统，功能比较庞大，涉及到不同组件中，不同控制器的协作，如下图。

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

  比如attach detach controller和volume manager中，各自都会有desiredStateOfWorld（缓存期望状态）和actualStateOfWorld（缓存实际状态）缓存，并且由各自的syncLoopFunc不断对比两个缓存的差异，并进行调整，下文中会介绍。

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

1. 卷分配，pvc绑定pv，由pv controller负责完成，结合[相关代码](https://github.com/kubernetes/kubernetes/blob/release-1.10/pkg/controller/volume/persistentvolume/pv_controller.go#L301)

   ![](http://p1sz5a5h3.bkt.clouddn.com/20181018002900.png)

   此时pvc绑定pv

   ![](http://p1sz5a5h3.bkt.clouddn.com/20181014142538.png)

2. attach detach controlle，如果pod分配到worker node，并且对应卷已经创建，则将卷挂载到对应worker node，并且给worker node资源增加volume已经挂载对应卷的状态信息，结合相关[代码1](https://github.com/kubernetes/kubernetes/blob/release-1.10/pkg/controller/volume/attachdetach/populator/desired_state_of_world_populator.go#L88)和[代码2](https://github.com/kubernetes/kubernetes/blob/release-1.10/pkg/controller/volume/attachdetach/reconciler/reconciler.go#L251)

   ![](http://p1sz5a5h3.bkt.clouddn.com/20181018004733.png)

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

   * pod分配到本worker node后，获取Pod需要的volume，通过对比node状态中的volumesAttached，确认volume是否已经attach到node节点，如果attach到node节点则将自身actualStateOfWorld中的volume状态设置成attached，对应[代码1](https://github.com/kubernetes/kubernetes/blob/release-1.10/pkg/kubelet/volumemanager/populator/desired_state_of_world_populator.go#L152)、[代码2](https://github.com/kubernetes/kubernetes/blob/release-1.10/pkg/kubelet/volumemanager/reconciler/reconciler.go#L160)

     ![](http://p1sz5a5h3.bkt.clouddn.com/20181018011611.png)

   * 如果已经attached成功，则进入到文件系统挂载流程，相关[代码](https://github.com/kubernetes/kubernetes/blob/release-1.10/pkg/kubelet/volumemanager/reconciler/reconciler.go#L238)
     * 先挂载到node中全局路径，比如`/var/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-3ecd68c7b7d211e8/globalmount`

     * 映射到Pod对应路径，比如`/var/lib/kubelet/pods/49a5fede-b811-11e8-844f-fa7378845e00/volumes/kubernetes.io~csi/pvc-3ecd68c7b7d211e8/mount`

     * actualStateOfWorld中设置volume为挂载成功状态

       ![](http://p1sz5a5h3.bkt.clouddn.com/20181018015029.png)

4. pod controller确认卷已经映射成功，启动Pod，此处不详细展开

### Pod被删除的过程

1. pod controller watch到pod处于被删除状态，执行killPod操作，删除Pod，此处不详细展开

2. volume manager获取到Pod被删除的信息，会执行如下几步，相关[代码](https://github.com/kubernetes/kubernetes/blob/release-1.10/pkg/kubelet/volumemanager/reconciler/reconciler.go#L160)
   * 将Pod从desiredStateOfWorld的缓存信息中清除

     ![](http://p1sz5a5h3.bkt.clouddn.com/20181018024146.png)

   * actualStateOfWorld中已经挂载的卷和desiredStateOfWorld发现Pod不应该挂载，执行UmountVolume操作，将Pod和卷映射关系解除，并将Pod从actualStateOfWorld的卷信息中剔除

     ![](http://p1sz5a5h3.bkt.clouddn.com/20181018024451.png)

   * 此时如果实际状态中卷没有关联任何Pod，则说明卷需要可以完全与节点分离，则先执行UnmountDevice将卷的globalpath umount掉，等到下次reconcile时执行MarkVolumeAsDetached将卷完全从实际状态中删除掉。

     ![](http://p1sz5a5h3.bkt.clouddn.com/20181018024518.png)

3. attach detach controller发现挂载到node节点的volume没有被Pod使用，执行detach操作，将卷从node节点detach，此时卷完全处于集群中未被使用的状态，此处不详细展开。

总结为Kubernetes存储系统的特点

* 不同组件通过资源状态协作，attach detach controller需要PVC绑定PV的状态，volume manager需要node status中volume attached状态。
* 组件通过reconcile方式达到期望状态，并且状态可能需要多次reconcile中完成，如Pod清除掉后，volume最终和node分离

### 问题

理解了存储系统的整体过程之后，回到问题，statefulset中Pod被删除会发生什么？

首先，对于statefulset的了解，Pod被删除，statefulset controller应该会很快创建Pod，在我们的场景中，Pod还是调度到先前节点中启动。结合对存储的理解，可能的场景

场景一 delete Pod，感知顺序为volume manager(umount)->statefulset->scheduler->volume manager(mount)

1. volume manager发现Pod被删除，执行umount
2. statefulset发现Pod被删除，马上创建Pod
3. scheduler发现Pod进行调度
4. volume manager发现原有volume需要绑定Pod，执行mount

场景二 delete Pod，感知顺序为volume manager(umount)->volume manager(unmountDevice)->volume manager(MarkVolumeAsDelete)->attach detach controller(Detach)->statefulset->scheduler->attach detach controller(Attach)->volume manager

1. volume manager发现Pod被删除，执行umount/unmountDevice/MarkVolumeAsDelete（通过几次reconcile）
2. attach detach controller发现volume在node节点未被使用，执行detach
3. scheduler发现Pod进行调度
4. attach detach controller发现volume需要attach，执行attach
5. volume manager挂载

场景三 delete Pod，感知顺序为statefulset->volume manager(umount/deviceUmount)->scheduler->volume manager(mount)

1. statefulset发现Pod被删除，马上创建Pod
2. volume manager发现Pod被删除，执行umount/deviceUmount（通过几次reconcile），注意此时devicePath和deviceMountPath都为空
3. scheduler发现Pod进行调度
4. volume manager发现原有volume需要绑定Pod，执行mount而此时devicePath和deviceMountPath都为空，**问题出现**

再结合问题出现日志分析

```verilog
Sep 14 19:28:33 10-10-40-16 kubelet: I0914 19:28:33.174310    1953 operation_generator.go:1168] Controller attach succeeded for volume "pvc-3ecd68c7b7d211e8" (UniqueName: "kubernetes.io/csi/csi-qcfsplugin^csi-qcfs-volume-3ecd68c7b7d211e8-338") pod "yoooo-416ea0-0" (UID: "49a5fede-b811-11e8-844f-fa7378845e00") device path: "csi-eb93736e654600786d95eaffa7cd5d616f11a90bdc109e0df575e8646c250eb2"
Sep 14 19:28:33 10-10-40-16 kubelet: I0914 19:28:33.273344    1953 operation_generator.go:486] MountVolume.WaitForAttach entering for volume "pvc-3ecd68c7b7d211e8" (UniqueName: "kubernetes.io/csi/csi-qcfsplugin^csi-qcfs-volume-3ecd68c7b7d211e8-338") pod "yoooo-416ea0-0" (UID: "49a5fede-b811-11e8-844f-fa7378845e00") DevicePath "csi-eb93736e654600786d95eaffa7cd5d616f11a90bdc109e0df575e8646c250eb2"
Sep 14 19:28:33 10-10-40-16 kubelet: I0914 19:28:33.318275    1953 operation_generator.go:495] MountVolume.WaitForAttach succeeded for volume "pvc-3ecd68c7b7d211e8" (UniqueName: "kubernetes.io/csi/csi-qcfsplugin^csi-qcfs-volume-3ecd68c7b7d211e8-338") pod "yoooo-416ea0-0" (UID: "49a5fede-b811-11e8-844f-fa7378845e00") DevicePath "csi-eb93736e654600786d95eaffa7cd5d616f11a90bdc109e0df575e8646c250eb2"
Sep 14 19:28:33 10-10-40-16 kubelet: I0914 19:28:33.319345    1953 operation_generator.go:514] MountVolume.MountDevice succeeded for volume "pvc-3ecd68c7b7d211e8" (UniqueName: "kubernetes.io/csi/csi-qcfsplugin^csi-qcfs-volume-3ecd68c7b7d211e8-338") pod "yoooo-416ea0-0" (UID: "49a5fede-b811-11e8-844f-fa7378845e00") device mount path "/var/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-3ecd68c7b7d211e8/globalmount"
Sep 14 19:29:12 10-10-40-16 kubelet: I0914 19:29:12.826916    1953 operation_generator.go:486] MountVolume.WaitForAttach entering for volume "pvc-3ecd68c7b7d211e8" (UniqueName: "kubernetes.io/csi/csi-qcfsplugin^csi-qcfs-volume-3ecd68c7b7d211e8-338") pod "yoooo-416ea0-0" (UID: "67f223dc-b811-11e8-844f-fa7378845e00") DevicePath "csi-eb93736e654600786d95eaffa7cd5d616f11a90bdc109e0df575e8646c250eb2"
Sep 14 19:29:14 10-10-40-16 kubelet: I0914 19:29:14.465225    1953 operation_generator.go:495] MountVolume.WaitForAttach succeeded for volume "pvc-3ecd68c7b7d211e8" (UniqueName: "kubernetes.io/csi/csi-qcfsplugin^csi-qcfs-volume-3ecd68c7b7d211e8-338") pod "yoooo-416ea0-0" (UID: "67f223dc-b811-11e8-844f-fa7378845e00") DevicePath "csi-eb93736e654600786d95eaffa7cd5d616f11a90bdc109e0df575e8646c250eb2"
Sep 14 19:29:14 10-10-40-16 kubelet: I0914 19:29:14.466483    1953 operation_generator.go:514] MountVolume.MountDevice succeeded for volume "pvc-3ecd68c7b7d211e8" (UniqueName: "kubernetes.io/csi/csi-qcfsplugin^csi-qcfs-volume-3ecd68c7b7d211e8-338") pod "yoooo-416ea0-0" (UID: "67f223dc-b811-11e8-844f-fa7378845e00") device mount path "/var/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-3ecd68c7b7d211e8/globalmount"
Sep 14 19:29:15 10-10-40-16 kubelet: W0914 19:29:15.491424    1953 csi_mounter.go:354] kubernetes.io/csi: skipping mount dir removal, path does not exist [/var/lib/kubelet/pods/49a5fede-b811-11e8-844f-fa7378845e00/volumes/kubernetes.io~csi/pvc-3ecd68c7b7d211e8/mount]
Sep 14 19:29:15 10-10-40-16 kubelet: I0914 19:29:15.491450    1953 operation_generator.go:686] UnmountVolume.TearDown succeeded for volume "kubernetes.io/csi/csi-qcfsplugin^csi-qcfs-volume-3ecd68c7b7d211e8-338" (OuterVolumeSpecName: "data") pod "49a5fede-b811-11e8-844f-fa7378845e00" (UID: "49a5fede-b811-11e8-844f-fa7378845e00"). InnerVolumeSpecName "pvc-3ecd68c7b7d211e8". PluginName "kubernetes.io/csi", VolumeGidValue ""
Sep 14 19:29:44 10-10-40-16 kubelet: W0914 19:29:44.896387    1953 csi_mounter.go:354] kubernetes.io/csi: skipping mount dir removal, path does not exist [/var/lib/kubelet/pods/67f223dc-b811-11e8-844f-fa7378845e00/volumes/kubernetes.io~csi/pvc-3ecd68c7b7d211e8/mount]
Sep 14 19:29:44 10-10-40-16 kubelet: I0914 19:29:44.896403    1953 operation_generator.go:686] UnmountVolume.TearDown succeeded for volume "kubernetes.io/csi/csi-qcfsplugin^csi-qcfs-volume-3ecd68c7b7d211e8-338" (OuterVolumeSpecName: "data") pod "67f223dc-b811-11e8-844f-fa7378845e00" (UID: "67f223dc-b811-11e8-844f-fa7378845e00"). InnerVolumeSpecName "pvc-3ecd68c7b7d211e8". PluginName "kubernetes.io/csi", VolumeGidValue ""
Sep 14 19:29:44 10-10-40-16 kubelet: I0914 19:29:44.917540    1953 reconciler.go:278] operationExecutor.UnmountDevice started for volume "pvc-3ecd68c7b7d211e8" (UniqueName: "kubernetes.io/csi/csi-qcfsplugin^csi-qcfs-volume-3ecd68c7b7d211e8-338") on node "10-10-40-16"
Sep 14 19:29:44 10-10-40-16 kubelet: W0914 19:29:44.919231    1953 mount_linux.go:179] could not determine device for path: "/var/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-3ecd68c7b7d211e8/globalmount"
Sep 14 19:29:45 10-10-40-16 kubelet: I0914 19:29:45.609605    1953 operation_generator.go:760] UnmountDevice succeeded for volume "pvc-3ecd68c7b7d211e8" %!(EXTRA string=UnmountDevice succeeded for volume "pvc-3ecd68c7b7d211e8" (UniqueName: "kubernetes.io/csi/csi-qcfsplugin^csi-qcfs-volume-3ecd68c7b7d211e8-338") on node "10-10-40-16" )
Sep 14 19:29:45 10-10-40-16 kubelet: I0914 19:29:45.624963    1953 operation_generator.go:486] MountVolume.WaitForAttach entering for volume "pvc-3ecd68c7b7d211e8" (UniqueName: "kubernetes.io/csi/csi-qcfsplugin^csi-qcfs-volume-3ecd68c7b7d211e8-338") pod "yoooo-416ea0-0" (UID: "77b8caf7-b811-11e8-844f-fa7378845e00") DevicePath ""
Sep 14 19:29:46 10-10-40-16 kubelet: E0914 19:29:46.006612    1953 nestedpendingoperations.go:267] Operation for "\"kubernetes.io/csi/csi-qcfsplugin^csi-qcfs-volume-3ecd68c7b7d211e8-338\"" failed. No retries permitted until 2018-09-14 19:29:46.506583596 +0800 CST m=+105572.978439381 (durationBeforeRetry 500ms). Error: "MountVolume.WaitForAttach failed for volume \"pvc-3ecd68c7b7d211e8\" (UniqueName: \"kubernetes.io/csi/csi-qcfsplugin^csi-qcfs-volume-3ecd68c7b7d211e8-338\") pod \"yoooo-416ea0-0\" (UID: \"77b8caf7-b811-11e8-844f-fa7378845e00\") : resource name may not be empty"
Sep 14 19:29:46 10-10-40-16 kubelet: I0914 19:29:46.533962    1953 operation_generator.go:486] MountVolume.WaitForAttach entering for volume "pvc-3ecd68c7b7d211e8" (UniqueName: "kubernetes.io/csi/csi-qcfsplugin^csi-qcfs-volume-3ecd68c7b7d211e8-338") pod "yoooo-416ea0-0" (UID: "77b8caf7-b811-11e8-844f-fa7378845e00") DevicePath ""
```

WaitForAttach有两个阶段

- `Sep 14 19:29:14`以及之前DevicePath非空
- `Sep 14 19:29:45`以及之后DevicePath为空

那么在这两个时间点之间发生了什么，怀疑这个时间点时间发生的问题造成卷无法挂载

通过日志发现`Sep 14 19:29:14`到`Sep 14 19:29:45`有一段日志信息比较关键，分析如下

1. Sep 14 19:29:14 ...... MountVolume.MountDevice ......
2. Sep 14 19:29:15 ..... UnmountVolume.TearDown ......
3. Sep 14 19:29:44 ...... UnmountVolume.TearDown ......
4. Sep 14 19:29:44 ...... operationExecutor.UnmountDevice ......
5. Sep 14 19:29:44 ...... could not determine device for path ....

在步骤4中，有设置相关函数的

> UnmountDevice->GenerateUnmountDeviceFunc->actualStateOfWorld.MarkDeviceAsUnmounted->asw.SetVolumeGloballyMounted

其中比较关键的函数`SetVolumeGloballyMounted`

```go
asw.SetVolumeGloballyMounted(volumeName, false /* globallyMounted */, /* devicePath */"", /*  deviceMountPath */"")
```

### 总结

Kubernetes之所以在当前成为容器编排领域的事实标准，原因很多，但是对我们来讲基于声明式API的编程范式是我们依赖Kubernetes的重要原因，当然在其解决的问题规模下复杂程度也不言而喻，总之，一句话，没有银弹。