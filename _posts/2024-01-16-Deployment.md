---
layout:       post
title:        "Deployment"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - Kubernetes
    - Deployment
---


### 配置文件

Deployment 是 Kubernetes 中最常用的控制器之一，用于管理无状态的应用程序。它提供了应用程序的副本管理、自动扩展、滚动升级等功能。  
Deployment 通过 ReplicaSet 实现副本管理，可以确保指定数量的 Pod 副本正在运行，并处理 Pod 的创建、删除和更新。  
Deployment 适用于无状态应用程序，如 Web 服务、API 服务等。

```shell
[root@k8s-master1 test]# kubectl create deploy vito-deploy-nginx --image=nginx:1.7.9
deployment.apps/vito-deploy-nginx created

[root@k8s-master1 test]# kubectl get deploy --show-labels
NAME                READY   UP-TO-DATE   AVAILABLE   AGE    LABELS
vito-deploy-nginx   1/1     1            1           7m9s   app=vito-deploy-nginx

# rsName = deployName-podTemplateHash
[root@k8s-master1 test]# kubectl get replicaset --show-labels
NAME                           DESIRED   CURRENT   READY   AGE     LABELS
vito-deploy-nginx-7899b58cf9   1         1         1       7m48s   app=vito-deploy-nginx,pod-template-hash=7899b58cf9
[root@k8s-master1 test]# kubectl get rs --show-labels
NAME                           DESIRED   CURRENT   READY   AGE    LABELS
vito-deploy-nginx-7899b58cf9   1         1         1       8m7s   app=vito-deploy-nginx,pod-template-hash=7899b58cf9

# poName = rsName-随机值
[root@k8s-master1 test]# kubectl get po --show-labels
NAME                                 READY   STATUS    RESTARTS   AGE     LABELS
vito-deploy-nginx-7899b58cf9-tmvbq   1/1     Running   0          8m35s   app=vito-deploy-nginx,pod-template-hash=7899b58cf9

# [root@k8s-master1 test]# kubectl edit deploy vito-deploy-nginx

[root@k8s-master1 test]# kubectl get deploy vito-deploy-nginx -o yaml
apiVersion: apps/v1 # Deployment api 版本
kind: Deployment # 资源类型为 Deployment
metadata: # 元信息
  annotations:
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: "2023-12-15T08:50:23Z"
  generation: 1
  labels: # 标签
    app: vito-deploy-nginx
  name: vito-deploy-nginx # Deployment名字
  namespace: default # 所在的命名空间
  resourceVersion: "96331"
  uid: 64b840e6-5827-482e-babe-17481e506c48
spec:
  progressDeadlineSeconds: 600
  replicas: 1 # 期望副本数
  revisionHistoryLimit: 10 # 进行滚动更新后，保留的历史版本数
  selector:  # 选择器。 spec.selector.matchLabels 的值必须和 spec.template.metadata.lables 值完全匹配
    matchLabels: # 按照标签匹配
      app: vito-deploy-nginx
  strategy: # 更新策略
    rollingUpdate: # 滚动更新配置
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate # 采用滚动更新策略
  template: # pod 模板
    metadata:
      creationTimestamp: null
      labels:
        app: vito-deploy-nginx
    spec:
      containers:
      - image: nginx:1.7.9
        imagePullPolicy: IfNotPresent
        name: nginx
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status: # Deployment状态信息，手动创建 deploy.yaml 时不需要这些信息
  availableReplicas: 1
  conditions:
  - lastTransitionTime: "2023-12-15T08:50:25Z"
    lastUpdateTime: "2023-12-15T08:50:25Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: "2023-12-15T08:50:23Z"
    lastUpdateTime: "2023-12-15T08:50:25Z"
    message: ReplicaSet "vito-deploy-nginx-7899b58cf9" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  observedGeneration: 1
  readyReplicas: 1
  replicas: 1
  updatedReplicas: 1

```

### 创建

* 用镜像创建一个 Deployment  
  `kubectl create deploy vito-deploy-nginx --image=nginx:1.7.9`  

* 用 yaml 文件创建 Deployment  
  `kubectl create -f xxx.yaml --record`  
  * \-\-record 会在 annotation 中记录当前命令创建或升级的资源，后续可以查看做过哪些变动操作  

### 滚动更新

* 只有修改了 deployment 配置文件中的 template 中的属性后，才会触发更新操作

* 修改 template  属性
  * `kubectl set 属性名 deployment/deploy名字 属性值`
  * 或者 `kubectl edit deployment/deploy名字`

```shell
[root@k8s-master1 test]# kubectl set image deploy/vito-deploy-nginx nginx=nginx:1.8.1
deployment.apps/vito-deploy-nginx image updated

[root@k8s-master1 test]# kubectl rollout status deploy vito-deploy-nginx
deployment "vito-deploy-nginx" successfully rolled out

[root@k8s-master1 test]# kubectl get rs
NAME                           DESIRED   CURRENT   READY   AGE
vito-deploy-nginx-55bd44cbc6   1         1         1       2m42s
vito-deploy-nginx-7899b58cf9   0         0         0       21m
[root@k8s-master1 test]# kubectl get po
NAME                                 READY   STATUS    RESTARTS   AGE
vito-deploy-nginx-55bd44cbc6-w7485   1/1     Running   0          2m50s
```

```shell
[root@k8s-master1 test]# kubectl edit deploy vito-deploy-nginx
# 把希望副本数 replicas 改为 2
deployment.apps/vito-deploy-nginx edited

# 非 template 属性，没有触发更新操作，仅仅是增加了副本数量（注意观察资源名称中的 podTemplateHash ）
[root@k8s-master1 test]# kubectl get rs
NAME                           DESIRED   CURRENT   READY   AGE
vito-deploy-nginx-55bd44cbc6   2         2         2       4m42s
vito-deploy-nginx-7899b58cf9   0         0         0       23m
[root@k8s-master1 test]# kubectl get po
NAME                                 READY   STATUS    RESTARTS   AGE
vito-deploy-nginx-55bd44cbc6-w7485   1/1     Running   0          4m49s
vito-deploy-nginx-55bd44cbc6-xs6xc   1/1     Running   0          43s
```

```shell
[root@k8s-master1 test]# kubectl edit deploy/vito-deploy-nginx
# 把 nginx 版本改为 1.8.2
deployment.apps/vito-deploy-nginx edited

# 触发了更新，但是 image pull 失败
[root@k8s-master1 test]# kubectl get rs
NAME                           DESIRED   CURRENT   READY   AGE
vito-deploy-nginx-55bd44cbc6   2         2         2       9m55s
vito-deploy-nginx-5db66f7b55   1         1         0       101s
vito-deploy-nginx-7899b58cf9   0         0         0       28m
[root@k8s-master1 test]# kubectl get po
NAME                                 READY   STATUS             RESTARTS   AGE
vito-deploy-nginx-55bd44cbc6-w7485   1/1     Running            0          9m58s
vito-deploy-nginx-55bd44cbc6-xs6xc   1/1     Running            0          5m52s
vito-deploy-nginx-5db66f7b55-hnw44   0/1     ImagePullBackOff   0          104s

[root@k8s-master1 test]# kubectl rollout status deployments vito-deploy-nginx
Waiting for deployment "vito-deploy-nginx" rollout to finish: 1 out of 2 new replicas have been updated...
```

### 回滚

```shell
# 获取 revision 的列表
[root@k8s-master1 test]# kubectl rollout history deploy vito-deploy-nginx
deployment.apps/vito-deploy-nginx 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>

# 查看 revision 详细信息
[root@k8s-master1 test]# kubectl rollout history deploy/vito-deploy-nginx --revision=3
deployment.apps/vito-deploy-nginx with revision #3
Pod Template:
  Labels:	app=vito-deploy-nginx
	pod-template-hash=5db66f7b55
  Containers:
   nginx:
    Image:	nginx:1.8.2
    Port:	<none>
    Host Port:	<none>
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>

[root@k8s-master1 test]# kubectl rollout history deploy/vito-deploy-nginx --revision=2
deployment.apps/vito-deploy-nginx with revision #2
Pod Template:
  Labels:	app=vito-deploy-nginx
	pod-template-hash=55bd44cbc6
  Containers:
   nginx:
    Image:	nginx:1.8.1
    Port:	<none>
    Host Port:	<none>
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>

# 回退到上一个版本: kubectl rollout undo deployment/deployName 
# 回退到指定的 revision : kubectl rollout undo deployment/deployName --to-revision=revisionValue
[root@k8s-master1 test]# kubectl rollout undo deploy/vito-deploy-nginx --to-revision=2
deployment.apps/vito-deploy-nginx rolled back

[root@k8s-master1 test]# kubectl get rs
NAME                           DESIRED   CURRENT   READY   AGE
vito-deploy-nginx-55bd44cbc6   2         2         2       20m
vito-deploy-nginx-5db66f7b55   0         0         0       12m
vito-deploy-nginx-7899b58cf9   0         0         0       38m
[root@k8s-master1 test]# kubectl get po
NAME                                 READY   STATUS    RESTARTS   AGE
vito-deploy-nginx-55bd44cbc6-w7485   1/1     Running   0          20m
vito-deploy-nginx-55bd44cbc6-xs6xc   1/1     Running   0          16m
```

### 扩缩容

* 通过 `kube scale` 命令可以进行自动扩容/缩容，或者通过 `kube edit` 编辑 `replicas` 也可以实现扩容/缩容

* 扩容与缩容只是直接创建副本数，没有更新 pod template 因此不会创建新的 rs

```shell
[root@k8s-master1 test]# kubectl scale --replicas=5 deploy vito-deploy-nginx
deployment.apps/vito-deploy-nginx scaled

[root@k8s-master1 test]# kubectl get rs
NAME                           DESIRED   CURRENT   READY   AGE
vito-deploy-nginx-55bd44cbc6   5         5         5       26m
vito-deploy-nginx-5db66f7b55   0         0         0       17m
vito-deploy-nginx-7899b58cf9   0         0         0       44m
[root@k8s-master1 test]# kubectl get po
NAME                                 READY   STATUS    RESTARTS   AGE
vito-deploy-nginx-55bd44cbc6-bmj2f   1/1     Running   0          6s
vito-deploy-nginx-55bd44cbc6-gfj4t   1/1     Running   0          6s
vito-deploy-nginx-55bd44cbc6-pf2vv   1/1     Running   0          6s
vito-deploy-nginx-55bd44cbc6-w7485   1/1     Running   0          26m
vito-deploy-nginx-55bd44cbc6-xs6xc   1/1     Running   0          22m
```

### 暂停与恢复更新

* 短时间内要多次 edit yaml ，最后一次更新完成后才希望更新 deploy
* 编辑前暂停更新，最终编辑完后恢复更新即可

```shell
# 暂停更新
[root@k8s-master1 test]# kubectl rollout pause deploy vito-deploy-nginx
deployment.apps/vito-deploy-nginx paused

[root@k8s-master1 test]# kubectl edit deploy vito-deploy-nginx
deployment.apps/vito-deploy-nginx edited
[root@k8s-master1 test]# kubectl edit deploy vito-deploy-nginx
deployment.apps/vito-deploy-nginx edited

# 恢复更新
[root@k8s-master1 test]# kubectl rollout resume deploy vito-deploy-nginx
deployment.apps/vito-deploy-nginx resumed

[root@k8s-master1 test]# kubectl get rs
NAME                           DESIRED   CURRENT   READY   AGE
vito-deploy-nginx-55bd44cbc6   0         0         0       32m
vito-deploy-nginx-5db66f7b55   0         0         0       23m
vito-deploy-nginx-64bbc7c864   5         5         5       53s
vito-deploy-nginx-7899b58cf9   0         0         0       50m
[root@k8s-master1 test]# kubectl get po
NAME                                 READY   STATUS    RESTARTS   AGE
vito-deploy-nginx-64bbc7c864-22fcx   1/1     Running   0          44s
vito-deploy-nginx-64bbc7c864-8txlq   1/1     Running   0          55s
vito-deploy-nginx-64bbc7c864-ckbnc   1/1     Running   0          55s
vito-deploy-nginx-64bbc7c864-kmc9v   1/1     Running   0          55s
vito-deploy-nginx-64bbc7c864-nqtlc   1/1     Running   0          45s
[root@k8s-master1 test]# kubectl rollout history deploy/vito-deploy-nginx
deployment.apps/vito-deploy-nginx 
REVISION  CHANGE-CAUSE
1         <none>
3         <none>
4         <none>
5         <none>
# 总共4个版本，回滚一次后： 2 变成了 4
```

### 删除

* 删除 deploy 的同时，会级联删除 replicaset 和 pod

```shell
[root@k8s-master1 ~]# kubectl get deploy
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
vito-deploy-nginx   5/5     5            5           2d5h
[root@k8s-master1 ~]# kubectl get rs
NAME                           DESIRED   CURRENT   READY   AGE
vito-deploy-nginx-55bd44cbc6   0         0         0       2d5h
vito-deploy-nginx-5db66f7b55   0         0         0       2d5h
vito-deploy-nginx-64bbc7c864   5         5         5       2d4h
vito-deploy-nginx-7899b58cf9   0         0         0       2d5h
[root@k8s-master1 ~]# kubectl get po
NAME                                 READY   STATUS    RESTARTS       AGE
vito-deploy-nginx-64bbc7c864-22fcx   1/1     Running   1 (2d2h ago)   2d4h
vito-deploy-nginx-64bbc7c864-8txlq   1/1     Running   1 (2d2h ago)   2d4h
vito-deploy-nginx-64bbc7c864-ckbnc   1/1     Running   1 (2d2h ago)   2d4h
vito-deploy-nginx-64bbc7c864-kmc9v   1/1     Running   1 (2d2h ago)   2d4h
vito-deploy-nginx-64bbc7c864-nqtlc   1/1     Running   1 (2d2h ago)   2d4h

[root@k8s-master1 ~]# kubectl delete deploy vito-deploy-nginx
deployment.apps "vito-deploy-nginx" deleted

[root@k8s-master1 ~]# kubectl get deploy
No resources found in default namespace.
[root@k8s-master1 ~]# kubectl get rs
No resources found in default namespace.
[root@k8s-master1 ~]# kubectl get po
No resources found in default namespace.
```

