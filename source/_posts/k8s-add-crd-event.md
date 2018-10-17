---
title: k8s-add-crd-event
date: 2018-08-21 09:08:27
tags: kubernetes
---

对于kubernetes primitive resource增加event非常简单，需要做如下几步

### 注册crd的scheme

```go
utilruntime.Must(samplescheme.AddToScheme(scheme.Scheme))
```

### 创建recorder

```go
eventBroadcaster := record.NewBroadcaster()
eventBroadcaster.StartLogging(glog.Infof)
eventBroadcaster.StartRecordingToSink(&typedcorev1.EventSinkImpl{Interface: kubeclientset.CoreV1().Events("")})
recorder := eventBroadcaster.NewRecorder(scheme.Scheme, corev1.EventSource{Component: controllerAgentName})
```

### 通过recorder来记录事件

```go
recorder.Event(foo, corev1.EventTypeNormal, SuccessSynced, MessageResourceSynced)
```

