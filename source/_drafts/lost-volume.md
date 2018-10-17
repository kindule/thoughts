---
title: lost-volume
date: 2018-09-26 11:00:41
tags: kubernetes
---

### 卷无法正常挂载，但是删除Pod，Pod被重新拉起能正常启动，使用本地存储

#### 思路

* 环境

  * nfs
  * csi

* 过程思考

  * attach成功

  * global mount成功

  * mount -v 应该不成功--检查mount是否成功

    * NFS未mount成功，原因**执行mount nfs如果remote target没有也不会报错**

    * ```shell
      [root@10-10-40-98 ~]# mount -t nfs -o nolock,async,rw,bg,hard,nointr,rsize=32768,wsize=32768,tcp,actimeo=0,vers=3,timeo=600 172.16.130.1:pvc-3eef6820c13e11e8/root /mnt
      mount.nfs: backgrounding "172.16.130.1:pvc-3eef6820c13e11e8/root"
      mount.nfs: mount options: "rw,nolock,bg,hard,nointr,rsize=32768,wsize=32768,tcp,actimeo=0,vers=3,timeo=600"
      [root@10-10-40-98 ~]# echo $?
      0
      ```

最终发现nfs mount参数配置不正确，参数`bg`释义，简单来讲mount时立即返回0

> bg / fg
>
> Determines  how  the  mount(8)  command  behaves if an attempt to mount an export fails.  The fg option causes mount(8) to exit with an error status if any part of the mount request times out or fails outright.  This is called a "foreground" mount, and is the default behavior if neither the fg nor bg mount option is specified.
> If the bg option is specified, a timeout or failure causes the mount(8) command to fork a child which continues to attempt to mount the export.  The parent immediately returns with a zero exit code.  This is known as a "background" mount.
> If the local mount point directory is missing, the mount(8) command acts as if the mount request timed out.  This permits nested NFS mounts specified in /etc/fstab to proceed in any order  during  system  initialization,  even  if  some  NFS servers are not yet available.  Alternatively these issues can be addressed using an automounter (refer to automount(8) for details).