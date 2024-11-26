---
layout:       post
title:        "DaemonSet"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - Kubernetes
    - DaemonSet
---




### 配置文件

```shell
[root@k8s-master1 test]# vim ds-fdr-fluentd.yaml
apiVersion: apps/v1
kind: DaemonSet # 创建 DaemonSet 资源
metadata:
  name: ds-fdr-fluentd  # DaemonSet 的名字
spec:
  selector:
    matchLabels:
      app: fdr-logs
  template:
    metadata:
      labels:
        app: fdr-logs
        id: fdr-fluentd
      name: ds-fdr-fluentd-po-template
    spec:
      nodeSelector: # DaemonSet 要部署的 Node
        type: fdr-microservices
      volumes: # 定义数据卷
      - name: ds-fdr-fluentd-volume-containers # 定义的数据卷的名称
        hostPath: # 数据卷类型，主机路径的模式，也就是与 Node 共享目录
          path: /var/lib/docker/containers # Node 主机中的目录
      - name: ds-fdr-fluentd-volume-varlog
        hostPath:
          path: /var/log
      containers:
      - name: ds-fdr-fluentd-c-es
        image: agilestacks/fluentd-elasticsearch:v1.3.0
        env: # 环境变量配置
        - name: FLUENTD_ARGS # 环境变量的key
          value: -qq # 环境变量的value
        volumeMounts: # 加载数据卷，避免数据丢失
        - name: ds-fdr-fluentd-volume-containers # 数据卷的名字
          mountPath: /var/lib/docker/containers # 将数据卷挂载到的容器内的目录
        - name: ds-fdr-fluentd-volume-varlog
          mountPath: /var/log
```

### 创建

```shell
[root@k8s-master1 test]# kubectl create -f ds-fdr-fluentd.yaml 
daemonset.apps/ds-fdr-fluentd created

[root@k8s-master1 test]# kubectl get daemonsets
NAME             DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
ds-fdr-fluentd   0         0         0       0            0           type=fdr-microservices   40s
[root@k8s-master1 test]# kubectl get ds
NAME             DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
ds-fdr-fluentd   0         0         0       0            0           type=fdr-microservices   42s
[root@k8s-master1 test]# kubectl get po
No resources found in default namespace.

```

```shell
[root@k8s-master1 test]# kubectl label no k8s-node1.zhch.lan type=fdr-microservices
node/k8s-node1.zhch.lan labeled
[root@k8s-master1 test]# kubectl get po -o wide
NAME                   READY   STATUS    RESTARTS   AGE    IP              NODE                 NOMINATED NODE   READINESS GATES
ds-fdr-fluentd-tdqlh   1/1     Running   0          2m4s   10.244.37.234   k8s-node1.zhch.lan   <none>           <none>

[root@k8s-master1 test]# kubectl label no k8s-node2.zhch.lan type=fdr-microservices
node/k8s-node2.zhch.lan labeled
[root@k8s-master1 test]# kubectl get po -o wide
NAME                   READY   STATUS    RESTARTS   AGE     IP              NODE                 NOMINATED NODE   READINESS GATES
ds-fdr-fluentd-8bcl8   1/1     Running   0          49s     10.244.8.240    k8s-node2.zhch.lan   <none>           <none>
ds-fdr-fluentd-tdqlh   1/1     Running   0          3m31s   10.244.37.234   k8s-node1.zhch.lan   <none>           <none>
```

* 删除标签

```shell
[root@k8s-master1 ~]# kubectl get no --show-labels
NAME                   STATUS   ROLES           AGE   VERSION    LABELS
k8s-master1.zhch.lan   Ready    control-plane   12d   v1.25.14   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master1.zhch.lan,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
k8s-node1.zhch.lan     Ready    <none>          12d   v1.25.14   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node1.zhch.lan,kubernetes.io/os=linux,type=fdr-microservices
k8s-node2.zhch.lan     Ready    <none>          12d   v1.25.14   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node2.zhch.lan,kubernetes.io/os=linux,type=fdr-microservices

[root@k8s-master1 ~]# kubectl label no k8s-node1.zhch.lan type-
node/k8s-node1.zhch.lan unlabeled
[root@k8s-master1 ~]# kubectl label no k8s-node2.zhch.lan type-
node/k8s-node2.zhch.lan unlabeled
[root@k8s-master1 ~]# kubectl get no --show-labels
NAME                   STATUS   ROLES           AGE   VERSION    LABELS
k8s-master1.zhch.lan   Ready    control-plane   12d   v1.25.14   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master1.zhch.lan,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
k8s-node1.zhch.lan     Ready    <none>          12d   v1.25.14   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node1.zhch.lan,kubernetes.io/os=linux
k8s-node2.zhch.lan     Ready    <none>          12d   v1.25.14   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node2.zhch.lan,kubernetes.io/os=linux
```

