---
title: 不断删除Pod，报错resource name may not be empty
date: 2018-09-19 12:08:34
tags:
---

## 问题现象

```
	// Example of global map path:
	//   globalMapPath/linkName: plugins/kubernetes.io/{PluginName}/{DefaultKubeletVolumeDevicesDirName}/{volumePluginDependentPath}/{podUid}
	//   linkName: {podUid}
	//
	// Example of pod device map path:
	//   podDeviceMapPath/linkName: pods/{podUid}/{DefaultKubeletVolumeDevicesDirName}/{escapeQualifiedPluginName}/{volumeName}
	//   linkName: {volumeName}
```



不断删除Pod，小概率会出现Pod运行失败并且报错`resource name may not be empty`

![image-20180919164047762](http://p1sz5a5h3.bkt.clouddn.com/2018-09-20-090324.png)

报错代码如下`c.k8s.StorageV1beta1().VolumeAttachments().Get(attachID, meta.GetOptions{})`执行Get(attachID,...) attachID为空

```go
func (c *csiAttacher) waitForVolumeAttachmentInternal(volumeHandle, attachID string, timer *time.Timer, timeout time.Duration) (string, error) {
	glog.V(4).Info(log("probing VolumeAttachment [id=%v]", attachID))
	attach, err := c.k8s.StorageV1beta1().VolumeAttachments().Get(attachID, meta.GetOptions{})
	if err != nil {
		glog.Error(log("attacher.WaitForAttach failed for volume [%s] (will continue to try): %v", volumeHandle, err))
		return "", err
	}
    ......
}
```

## 思路

1. 找到报错代码具体位置，具体哪个函数报错，发现volumeId为空，分析为什么volumeId，分析整个函数调用栈实际上是volumeId就是volumeToMount.DevicePath，问题就归结到了**为什么volumeToMount中的DevicePath为空，可能有哪个逻辑讲DevicePath设置为空了**，volumeToMount是对象，思路是找到volumeToMount在哪些地方做了修改，设置里DevicePath。
2. 看代码，熟悉处理流程，初步熟悉代码，发现逻辑似乎没有问题
3. 尝试通过自己模拟复现，一直无法复现
4. 结合日志，找到是从什么时候开始出现报错

### 关键日志

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

这个函数比较关键所以看到函数实现，结合到上面使用函数的姿势，将volumeobj.devicePath设置为空.

```go
func (asw *actualStateOfWorld) SetVolumeGloballyMounted(
	volumeName v1.UniqueVolumeName, globallyMounted bool, devicePath, deviceMountPath string) error {
	asw.Lock()
	defer asw.Unlock()

	volumeObj, volumeExists := asw.attachedVolumes[volumeName]
	if !volumeExists {
		return fmt.Errorf(
			"no volume with the name %q exists in the list of attached volumes",
			volumeName)
	}

	volumeObj.globallyMounted = globallyMounted
	volumeObj.deviceMountPath = deviceMountPath
	volumeObj.devicePath = devicePath
	asw.attachedVolumes[volumeName] = volumeObj
	return nil
}
```

### 结合`pkg/kubelet/volumemanager/reconciler/reconciler.go`分析整个过程发生报错的原因

![](http://p1sz5a5h3.bkt.clouddn.com/2018-09-20-085202.jpg)

![](http://p1sz5a5h3.bkt.clouddn.com/2018-09-20-085214.jpg)

![](http://p1sz5a5h3.bkt.clouddn.com/2018-09-20-085225.jpg)

![](http://p1sz5a5h3.bkt.clouddn.com/20180920220713.png)

### 总结

1. Worker Node中Pod-0（statefulset生产的Pod）正常启动后，不断删除Pod-0，在reconcile的[第三部分](https://github.com/kubernetes/kubernetes/blob/v1.10.5/pkg/kubelet/volumemanager/reconciler/reconciler.go#L261)，如果volume未被Pod绑定进入UnmountDevice流程会将volumeToAttach.devicePath设置未空
2. 此时如果attachDetachController未来得及触发detach操作，Pod-0又被创建出来，则会进入reconcile的[第二部分](https://github.com/kubernetes/kubernetes/blob/v1.10.5/pkg/kubelet/volumemanager/reconciler/reconciler.go#L187)，此时会走到[MountVolume](https://github.com/kubernetes/kubernetes/blob/v1.10.5/pkg/kubelet/volumemanager/reconciler/reconciler.go#L238)，进入waitForAttach，最终带入空devicePath调用`c.k8s.StorageV1beta1().VolumeAttachments().Get(attachID, meta.GetOptions{})`，从而产生报错

------

### 以下为具体报错信息及相关信息整理

```shell
Name:           yoooo-416ea0-0
Namespace:      default
Node:           10-10-40-16/172.16.130.188
Start Time:     Fri, 14 Sep 2018 19:29:38 +0800
Labels:         AppName=yoooo-416ea
                CreatedBy=orain.com
                DBRole=Master
                DBType=mysql
                Name=yoooo-416ea0
                Type=Database
                controller-revision-hash=yoooo-416ea0-98b88859d
                statefulset.kubernetes.io/pod-name=yoooo-416ea0-0
Annotations:    pod.beta.kubernetes.io/initialized=true
Status:         Pending
IP:
Controlled By:  StatefulSet/yoooo-416ea0
Init Containers:
  restorer:
    Container ID:
    Image:         registry.woqutech.com/woqutech/s3cmd-xtracbackup-sshd:v1.0.0
    Image ID:
    Port:          <none>
    Host Port:     <none>
    Command:
      /usr/local/bin/restore.sh
    State:          Waiting
      Reason:       PodInitializing
    Ready:          False
    Restart Count:  0
    Environment:
      DBROLE:                 Master
      DATADIR:                /var/lib/mysql/data
      ARCHIVEDIR:             /var/lib/mysql/archive
      REDODIR:                /var/lib/mysql/redo
      MYSQL_DATABASE:         mysql
      MYSQL_MASTER_USER:      <set to the key 'username' in secret 'yoooo-416ea0-master-suffix'>  Optional: false
      MYSQL_MASTER_PASSWORD:  <set to the key 'password' in secret 'yoooo-416ea0-master-suffix'>  Optional: false
      MYSQL_ROOT_USER:        <set to the key 'username' in secret 'yoooo-416ea0-root-suffix'>    Optional: false
      MYSQL_ROOT_PASSWORD:    <set to the key 'password' in secret 'yoooo-416ea0-root-suffix'>    Optional: false
      MY_POD_IP:               (v1:status.podIP)
      CLUSTER_REPLICA:        1
    Mounts:
      /etc/mysql/ from database-config (rw)
      /etc/s3cfg from s3cfg (rw)
      /var/lib/mysql/archive from archive (rw)
      /var/lib/mysql/data from data (rw)
      /var/lib/mysql/redo from redo (rw)
      /var/log/mysql from log-data (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-nhg9s (ro)
Containers:
  mysql:
    Container ID:
    Image:          registry.woqutech.com/woqutech/mysql:5.7.21
    Image ID:
    Port:           <none>
    Host Port:      <none>
    State:          Waiting
      Reason:       PodInitializing
    Ready:          False
    Restart Count:  0
    Limits:
      cpu:     2
      memory:  4Gi
    Requests:
      cpu:      2
      memory:   4Gi
    Readiness:  exec [mysqlprobe] delay=0s timeout=1s period=3s #success=1 #failure=3
    Environment:
      DBROLE:                 Master
      DATADIR:                /var/lib/mysql/data
      ARCHIVEDIR:             /var/lib/mysql/archive
      REDODIR:                /var/lib/mysql/redo
      MYSQL_DATABASE:         mysql
      MYSQL_MASTER_USER:      <set to the key 'username' in secret 'yoooo-416ea0-master-suffix'>  Optional: false
      MYSQL_MASTER_PASSWORD:  <set to the key 'password' in secret 'yoooo-416ea0-master-suffix'>  Optional: false
      MYSQL_ROOT_USER:        <set to the key 'username' in secret 'yoooo-416ea0-root-suffix'>    Optional: false
      MYSQL_ROOT_PASSWORD:    <set to the key 'password' in secret 'yoooo-416ea0-root-suffix'>    Optional: false
      MY_POD_IP:               (v1:status.podIP)
      CLUSTER_REPLICA:        1
    Mounts:
      /etc/localtime from localtime (rw)
      /etc/my.cnf.d/global.conf from yoooo-416ea0-slave-delay-threshold (rw)
      /etc/mysql/ from database-config (rw)
      /etc/timezone from timezone (rw)
      /var/lib/mysql/archive from archive (rw)
      /var/lib/mysql/data from data (rw)
      /var/lib/mysql/redo from redo (rw)
      /var/log/mysql from log-data (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-nhg9s (ro)
  fluentbit:
    Container ID:
    Image:          registry.woqutech.com/google_containers/fluent-bit:0.12.9
    Image ID:
    Port:           <none>
    Host Port:      <none>
    State:          Waiting
      Reason:       PodInitializing
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /etc/localtime from localtime (rw)
      /etc/my.cnf.d/global.conf from yoooo-416ea0-slave-delay-threshold (rw)
      /etc/timezone from timezone (rw)
      /fluent-bit/etc from fluentbit-config (ro)
      /tmp/log from log-data (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-nhg9s (ro)
  pmm-client:
    Container ID:
    Image:         registry.woqutech.com/library/pmm-client:v1.10.4
    Image ID:
    Port:          <none>
    Host Port:     <none>
    Command:
      /bin/bash
      /pmm-config/pmm-conn-config.sh
    State:          Waiting
      Reason:       PodInitializing
    Ready:          False
    Restart Count:  0
    Environment:
      USER:        <set to the key 'username' in secret 'yoooo-416ea0-root-suffix'>  Optional: false
      PASSWORD:    <set to the key 'password' in secret 'yoooo-416ea0-root-suffix'>  Optional: false
      PMM_SERVER:  pmm-server.kube-system.svc
    Mounts:
      /etc/localtime from localtime (rw)
      /etc/my.cnf.d/global.conf from yoooo-416ea0-slave-delay-threshold (rw)
      /etc/timezone from timezone (rw)
      /var/log/mysql from log-data (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-nhg9s (ro)
  exporter:
    Container ID:
    Image:          registry.woqutech.com/woqutech/mysql-exporter
    Image ID:
    Port:           <none>
    Host Port:      <none>
    State:          Waiting
      Reason:       PodInitializing
    Ready:          False
    Restart Count:  0
    Environment:
      USER:              <set to the key 'username' in secret 'yoooo-416ea0-root-suffix'>  Optional: false
      PASSWORD:          <set to the key 'password' in secret 'yoooo-416ea0-root-suffix'>  Optional: false
      DATA_SOURCE_NAME:  $(USER):$(PASSWORD)@(localhost:3306)/
    Mounts:
      /etc/localtime from localtime (rw)
      /etc/my.cnf.d/global.conf from yoooo-416ea0-slave-delay-threshold (rw)
      /etc/timezone from timezone (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-nhg9s (ro)
  s3backup:
    Container ID:
    Image:          registry.woqutech.com/woqutech/s3cmd-xtracbackup-sshd:v1.0.0
    Image ID:
    Port:           <none>
    Host Port:      <none>
    State:          Waiting
      Reason:       PodInitializing
    Ready:          False
    Restart Count:  0
    Environment:
      DBROLE:                 Master
      DATADIR:                /var/lib/mysql/data
      ARCHIVEDIR:             /var/lib/mysql/archive
      REDODIR:                /var/lib/mysql/redo
      MYSQL_DATABASE:         mysql
      MYSQL_MASTER_USER:      <set to the key 'username' in secret 'yoooo-416ea0-master-suffix'>  Optional: false
      MYSQL_MASTER_PASSWORD:  <set to the key 'password' in secret 'yoooo-416ea0-master-suffix'>  Optional: false
      MYSQL_ROOT_USER:        <set to the key 'username' in secret 'yoooo-416ea0-root-suffix'>    Optional: false
      MYSQL_ROOT_PASSWORD:    <set to the key 'password' in secret 'yoooo-416ea0-root-suffix'>    Optional: false
      MY_POD_IP:               (v1:status.podIP)
      CLUSTER_REPLICA:        1
    Mounts:
      /etc/localtime from localtime (rw)
      /etc/my.cnf.d/global.conf from yoooo-416ea0-slave-delay-threshold (rw)
      /etc/mysql/ from database-config (rw)
      /etc/s3cfg from s3cfg (rw)
      /etc/timezone from timezone (rw)
      /var/lib/mysql/archive from archive (rw)
      /var/lib/mysql/data from data (rw)
      /var/lib/mysql/redo from redo (rw)
      /var/log/mysql from log-data (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-nhg9s (ro)
Conditions:
  Type           Status
  Initialized    False
  Ready          False
  PodScheduled   True
Volumes:
  data:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  data-yoooo-416ea0-0
    ReadOnly:   false
  archive:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  archive-yoooo-416ea0-0
    ReadOnly:   false
  redo:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  redo-yoooo-416ea0-0
    ReadOnly:   false
  log-data:
    Type:    EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
  fluentbit-config:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      yoooo-416ea0-fluentbit
    Optional:  false
  timezone:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      timezone
    Optional:  false
  localtime:
    Type:          HostPath (bare host directory volume)
    Path:          /usr/share/zoneinfo/Asia/Shanghai
    HostPathType:
  yoooo-416ea0-slave-delay-threshold:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      yoooo-416ea0-slave-delay-threshold
    Optional:  false
  s3cfg:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  s3cfg
    Optional:    false
  database-config:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      yoooo-416ea0
    Optional:  false
  default-token-nhg9s:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-nhg9s
    Optional:    false
QoS Class:       Burstable
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason       Age                 From                  Message
  ----     ------       ----                ----                  -------
  Warning  FailedMount  8m (x3445 over 4d)  kubelet, 10-10-40-16  MountVolume.WaitForAttach failed for volume "pvc-3ecd68c7b7d211e8" : resource name may not be empty
  Warning  FailedMount  3m (x3097 over 4d)  kubelet, 10-10-40-16  Unable to mount volumes for pod "yoooo-416ea0-0_default(77b8caf7-b811-11e8-844f-fa7378845e00)": timeout expired waiting for volumes to attach or mount for pod "default"/"yoooo-416ea0-0". list of unmounted volumes=[data]. list of unattached volumes=[data archive redo log-data fluentbit-config timezone localtime yoooo-416ea0-slave-delay-threshold s3cfg database-config default-token-nhg9s]
```

