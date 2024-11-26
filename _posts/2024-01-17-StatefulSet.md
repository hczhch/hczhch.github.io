---
layout:       post
title:        "StatefulSet"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - Kubernetes
    - StatefulSet
---




### 配置文件

StatefulSet 用于管理有状态的应用程序，这些应用程序通常需要持久性存储和稳定的网络标识。  
与 Deployment 不同，StatefulSet 为每个 Pod 提供唯一的标识符和稳定的网络标识，以便在 Pod 重新启动或迁移时保持状态的稳定性。  
它还支持有序部署和扩展，并提供有状态应用程序所需的有序启动和终止策略。  
StatefulSet 适用于数据库、消息队列、存储节点等有状态应用程序。

```shell
[root@k8s-master1 test]# vim sts-fdr-web.yaml
# YAML文件可以由单个或多个文档组成（文档之间相互独立）。三个横线表示文档开始。
# 本例中包含两个文档：Service yaml 和 StatefulSet yaml
---
apiVersion: v1
kind: Service
metadata:
  name: svc-fdr-web # Service 的名字
  #labels:
    #app: fdr-web
spec:
  selector: # 将服务流量路由到具有与此选择器匹配的 Pod
    app: fdr-web
  type: NodePort
  ports:
  - port: 80 # service 暴露在 cluster ip 上的端口，<clusterIP>:port 是提供给集群内部访问 service 的入口
    targetPort: 80 # pod 上的端口，本例对应 fdr-web-nginx 中的 80 端口 
    #nodePort: 30080 # <nodeIP>:nodePort 是 Kubernetes 提供给集群外部客户访问 service 入口的一种方式（另一种方式是LoadBalancer）
    protocol: TCP
    # 总的来说，port 和 nodePort 都是 service 的端口，前者暴露给集群内客户访问服务，后者暴露给集群外客户访问服务。
    # 从这两个端口到来的数据都需要经过反向代理 kube-proxy 流入后端 Pod 的 targetPort ，从而到达 Pod 中的容器内
    # 注意：nodePort 的取值范围是 30000-32767，通常不手动指定该端口的值，而是让 Kubernetes 自动生成一个随机值
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sts-fdr-web # StatefulSet 的名字
  #labels:
    #app: fdr-web
spec:
  serviceName: "svc-fdr-web" # 使用哪个 service 来管理 dns
  replicas: 5
  selector: # spec.selector.matchLabels 的值必须和 spec.template.metadata.lables 值完全匹配
    matchLabels:
      app: fdr-web
  template:
    metadata:
      labels:
        app: fdr-web
    spec:
      containers:
      - name: fdr-web-nginx
        image: nginx:1.7.9
        ports: # 容器暴露的端口
        - containerPort: 80 # 具体的端口号
          #name: http # 该端口的名字
```

* 补充知识点：参看配置项的含义 `kubectl explain`

```shell
[root@k8s-node1 ~]# kubectl explain StatefulSet.spec.selector
KIND:     StatefulSet
VERSION:  apps/v1

RESOURCE: selector <Object>

DESCRIPTION:
     selector is a label query over pods that should match the replica count. It
     must match the pod template's labels. More info:
     https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors

     A label selector is a label query over a set of resources. The result of
     matchLabels and matchExpressions are ANDed. An empty label selector matches
     all objects. A null label selector matches no objects.

FIELDS:
   matchExpressions	<[]Object>
     matchExpressions is a list of label selector requirements. The requirements
     are ANDed.

   matchLabels	<map[string]string>
     matchLabels is a map of {key,value} pairs. A single {key,value} in the
     matchLabels map is equivalent to an element of matchExpressions, whose key
     field is "key", the operator is "In", and the values array contains only
     "value". The requirements are ANDed.
```

### 创建

```shell
[root@k8s-master1 test]# kubectl create -f sts-fdr-web.yaml 
service/svc-fdr-web created
statefulset.apps/sts-fdr-web created
```

```shell
[root@k8s-master1 test]# kubectl get statefulsets --show-labels
NAME          READY   AGE   LABELS
sts-fdr-web   5/5     91s   <none>
[root@k8s-master1 test]# kubectl get sts --show-labels
NAME          READY   AGE   LABELS
sts-fdr-web   5/5     98s   <none>

[root@k8s-master1 test]# kubectl get services --show-labels
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE   LABELS
kubernetes    ClusterIP   10.96.0.1      <none>        443/TCP        5d    component=apiserver,provider=kubernetes
svc-fdr-web   NodePort    10.100.88.85   <none>        80:31510/TCP   29s   <none>
[root@k8s-master1 test]# kubectl get svc --show-labels
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE   LABELS
kubernetes    ClusterIP   10.96.0.1      <none>        443/TCP        5d    component=apiserver,provider=kubernetes
svc-fdr-web   NodePort    10.100.88.85   <none>        80:31510/TCP   53s   <none>
# curl 10.100.88.85:80  # <clusterIP>:port
# curl k8s-node2:31510  # <nodeIP>:nodePort

[root@k8s-master1 test]# kubectl get po --show-labels
NAME            READY   STATUS    RESTARTS   AGE   LABELS
sts-fdr-web-0   1/1     Running   0          70s   app=fdr-web,controller-revision-hash=sts-fdr-web-854ccfcf74,statefulset.kubernetes.io/pod-name=sts-fdr-web-0
sts-fdr-web-1   1/1     Running   0          68s   app=fdr-web,controller-revision-hash=sts-fdr-web-854ccfcf74,statefulset.kubernetes.io/pod-name=sts-fdr-web-1
sts-fdr-web-2   1/1     Running   0          67s   app=fdr-web,controller-revision-hash=sts-fdr-web-854ccfcf74,statefulset.kubernetes.io/pod-name=sts-fdr-web-2
sts-fdr-web-3   1/1     Running   0          66s   app=fdr-web,controller-revision-hash=sts-fdr-web-854ccfcf74,statefulset.kubernetes.io/pod-name=sts-fdr-web-3
sts-fdr-web-4   1/1     Running   0          65s   app=fdr-web,controller-revision-hash=sts-fdr-web-854ccfcf74,statefulset.kubernetes.io/pod-name=sts-fdr-web-4
```

```shell
# busybox 是一个包含很多工具的镜像
# --rm 退出时自动删除
[root@k8s-master1 ~]# kubectl run --image busybox:1.28.4 dns-test --restart=Never --rm -it /bin/sh
/ # nslookup 10.100.88.85
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      10.100.88.85
Address 1: 10.100.88.85 svc-fdr-web.default.svc.cluster.local
/ # 
/ # nslookup svc-fdr-web
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
# 集群内部可以通过 serviceName.NameSpace:port 访问与 Service 匹配的 Pod
# 同一个 NameSpace 内访问，可以省略 `.NameSpace`

Name:      svc-fdr-web
Address 1: 10.100.88.85 svc-fdr-web.default.svc.cluster.local
/ # 
/ # nslookup sts-fdr-web-0.svc-fdr-web
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      sts-fdr-web-0.svc-fdr-web
Address 1: 10.244.8.220 sts-fdr-web-0.svc-fdr-web.default.svc.cluster.local
/ # 
/ # nslookup sts-fdr-web-3.svc-fdr-web.default.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      sts-fdr-web-3.svc-fdr-web.default.svc.cluster.local
Address 1: 10.244.37.219 sts-fdr-web-3.svc-fdr-web.default.svc.cluster.local
# StatefulSet 中每个 Pod 的 Domain 的格式为：StatefulSetName-{0.1...N}.serviceName.NameSpace.svc.cluster.local
# svc.cluster.local 可以省略
# 同一个 NameSpace 内，NameSpace 也可以省略，直接 StatefulSetName-{0.1...N}.serviceName 即可访问对应的 Pod
# 本例中 5个 Pod 的 Domain 分别为： sts-fdr-web-0/1/2/3/4.svc-fdr-web.default.svc.cluster.local
```

### 扩缩容

* `kubectl scale statefulset stsName --replicas=5`
* `kubectl patch statefulset stsName -p '{"spec":{"replicas":5}}'`

```shell
[root@k8s-master1 ~]# kubectl scale sts sts-fdr-web --replicas=2
statefulset.apps/sts-fdr-web scaled
[root@k8s-master1 ~]# kubectl get po
NAME            READY   STATUS    RESTARTS   AGE
sts-fdr-web-0   1/1     Running   0          10h
sts-fdr-web-1   1/1     Running   0          10h

[root@k8s-master1 ~]# kubectl patch sts sts-fdr-web -p '{"spec":{"replicas":5}}'
statefulset.apps/sts-fdr-web patched
[root@k8s-master1 ~]# kubectl get po
NAME            READY   STATUS    RESTARTS   AGE
sts-fdr-web-0   1/1     Running   0          10h
sts-fdr-web-1   1/1     Running   0          10h
sts-fdr-web-2   1/1     Running   0          14s
sts-fdr-web-3   1/1     Running   0          13s
sts-fdr-web-4   1/1     Running   0          11s
```

### 更新

* StatefulSet 也可以采用滚动更新策略 `RollingUpdate`，同样是修改 pod template 属性后会触发更新
  * StatefulSet 的 pod 是有序的，在 StatefulSet 中更新时是基于 pod 的顺序倒序更新的
* 滚动更新时，利用 `partition` 属性，可以实现简易的灰度发布的效果

  * 例如我们有 5 个 pod，如果当前 partition 设置为 3，那么此时滚动更新时，只会更新那些 序号 >= 3 的 pod
  * 利用该机制，我们可以通过控制 partition 的值，来决定只更新其中一部分 pod，确认没有问题后再**逐步增大**更新的 pod 数量，最终实现全部 pod 更新

* `OnDelete` 更新策略：与滚动更新策略不同，该更新策略只在 Pod 被删除时才会触发更新操作

```shell
[root@k8s-master1 ~]# kubectl get po
NAME            READY   STATUS    RESTARTS   AGE
sts-fdr-web-0   1/1     Running   0          10h
sts-fdr-web-1   1/1     Running   0          10h
sts-fdr-web-2   1/1     Running   0          8m9s
sts-fdr-web-3   1/1     Running   0          8m8s
sts-fdr-web-4   1/1     Running   0          8m6s
[root@k8s-master1 ~]# kubectl edit sts sts-fdr-web
spec:
  replicas: 5
  serviceName: svc-fdr-web
  template:
    spec:
      containers:
      - image: nginx:1.9.1 # 把 image 由 nginx:1.7.9 改为 nginx:1.9.1 
  updateStrategy:
    rollingUpdate:
      partition: 3 # 把 partition 由 0 改为 3 
    type: RollingUpdate

# 更新 yaml 后，通过 kubectl describe po sts-fdr-web-0 逐个查看每个 Pod ， 发现：
# sts-fdr-web-3 和 sts-fdr-web-4 的 image 变为 nginx:1.9.1
# sts-fdr-web-0 sts-fdr-web-1 sts-fdr-web-2 的 image 仍然为 nginx:1.7.9

# 再次更新 yaml ，把 partition 由 3 改为 1 ，发现：
# sts-fdr-web-1 sts-fdr-web-2 的 image 也变为 nginx:1.9.1
# 仅有 sts-fdr-web-0 的 image 仍然为 nginx:1.7.9

# 第三次更新 yaml ，把 partition 由 1 改为 0 ，发现：
# 所有 Pod 的 image 都变为 nginx:1.9.1
```

```shell
[root@k8s-master1 ~]# kubectl edit sts sts-fdr-web
spec:
  replicas: 5
  serviceName: svc-fdr-web
  template:
    spec:
      containers:
      - image: nginx:1.7.9 # 把 image 由 nginx:1.9.1 改回为 nginx:1.7.9 
  updateStrategy:
    type: OnDelete # updateStrategy.type 由 RollingUpdate 改为 OnDelete

# 更新 yaml 后，通过 kubectl describe po sts-fdr-web-0 逐个查看每个 Pod ， 发现：
# 所有 Pod 的 image 仍然为 nginx:1.9.1

[root@k8s-master1 ~]# kubectl delete po sts-fdr-web-2
pod "sts-fdr-web-2" deleted
# 再次通过 kubectl describe po sts-fdr-web-0 逐个查看每个 Pod ， 发现：
# sts-fdr-web-2 的 image 变为 nginx:1.7.9 ，其他 Pod 的 image 仍然为 nginx:1.9.1
```

### 删除

* 删除 `StatefulSet` 时默认会级联删除 `Pod`
* 添加参数`--cascade=orphan`则不会级联删除 `Pod`

```shell
[root@k8s-master1 ~]# kubectl delete sts sts-fdr-web
statefulset.apps "sts-fdr-web" deleted
[root@k8s-master1 ~]# kubectl get sts
No resources found in default namespace.
[root@k8s-master1 ~]# kubectl get po
No resources found in default namespace.

[root@k8s-master1 ~]# kubectl delete svc svc-fdr-web
service "svc-fdr-web" deleted
[root@k8s-master1 ~]# kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   5d11h
```

```shell
[root@k8s-master1 ~]# kubectl create -f test/sts-fdr-web.yaml 
service/svc-fdr-web created
statefulset.apps/sts-fdr-web created

[root@k8s-master1 ~]# kubectl delete sts sts-fdr-web --cascade=orphan
statefulset.apps "sts-fdr-web" deleted
[root@k8s-master1 ~]# kubectl get sts
No resources found in default namespace.
[root@k8s-master1 ~]# kubectl get po
NAME            READY   STATUS    RESTARTS   AGE
sts-fdr-web-0   1/1     Running   0          39s
sts-fdr-web-1   1/1     Running   0          38s
sts-fdr-web-2   1/1     Running   0          36s
sts-fdr-web-3   1/1     Running   0          35s
sts-fdr-web-4   1/1     Running   0          33s
[root@k8s-master1 ~]# kubectl delete po sts-fdr-web-0 sts-fdr-web-1 sts-fdr-web-2 sts-fdr-web-3 sts-fdr-web-4
pod "sts-fdr-web-0" deleted
pod "sts-fdr-web-1" deleted
pod "sts-fdr-web-2" deleted
pod "sts-fdr-web-3" deleted
pod "sts-fdr-web-4" deleted
[root@k8s-master1 ~]# kubectl get po
No resources found in default namespace.

[root@k8s-master1 ~]# kubectl delete svc svc-fdr-web
service "svc-fdr-web" deleted
[root@k8s-master1 ~]# kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   5d11h
```

