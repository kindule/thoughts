---
title: access-internal-cluster
date: 2018-08-27 11:32:05
tags: kubernetes
---

## 设置insecure方式访问api-server

很多时候kubernetes集群控制面服务都监听在集群内部网络中，想通过公共网络访问api-server时会比较麻烦，比如在如下网络结构中，如果是在开发环境下可以设置api-server启用insecrue接口，这样方便本地程序连接到api-server进行开发测试。

![image-20180901151431483](http://p1sz5a5h3.bkt.clouddn.com/2018-09-01-071433.png)

## 具体方案如下

### api-server

修改api-server配置文件，我们假设master节点有业务网络IP 192.168.10.2，增加如下两个参数，记得将insecure-port=0注释掉

```shell
    - --insecure-bind-address=192.168.10.2
    - --insecure-port=8181
    # - --insecure-port=0
```

### config

kubeconfig文件按照以下模板配置，记得修改<public-ip>为api-server监听的地址

```yaml
apiVersion: v1
clusters:
- cluster:
    server: http://<public-ip>:8181
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
```

