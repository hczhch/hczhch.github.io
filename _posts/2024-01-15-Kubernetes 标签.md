---
layout:       post
title:        "Kubernetes 标签"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - Kubernetes
    - Label
---

```shell
[root@k8s-master1 test]# kubectl get po --show-labels
NAME                 READY   STATUS    RESTARTS   AGE   LABELS
vito-demo-po-nginx   1/1     Running   0          31s   type=app,version=1.0.0

# 添加临时标签 kubectl label 资源类型 资源名称 标签名=标签值
[root@k8s-master1 test]# kubectl label po vito-demo-po-nginx author=ztx
pod/vito-demo-po-nginx labeled
[root@k8s-master1 test]# kubectl get po --show-labels
NAME                 READY   STATUS    RESTARTS   AGE   LABELS
vito-demo-po-nginx   1/1     Running   0          66s   author=ztx,type=app,version=1.0.0

# 修改已存在的标签 --overwrite
[root@k8s-master1 test]# kubectl label po vito-demo-po-nginx author=fdr --overwrite
pod/vito-demo-po-nginx labeled
[root@k8s-master1 test]# kubectl get po --show-labels
NAME                 READY   STATUS    RESTARTS   AGE   LABELS
vito-demo-po-nginx   1/1     Running   0          91s   author=fdr,type=app,version=1.0.0

# 根据标签查找 -l 
[root@k8s-master1 test]# kubectl get po -l type=app
NAME                 READY   STATUS    RESTARTS   AGE
vito-demo-po-nginx   1/1     Running   0          2m9s
[root@k8s-master1 test]# kubectl get po -l type=app,'version in (1.0.0,1.2.0)'
NAME                 READY   STATUS    RESTARTS   AGE
vito-demo-po-nginx   1/1     Running   0          2m50s

```

