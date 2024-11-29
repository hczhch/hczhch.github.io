---
layout:       post
title:        "Kubernetes 微服务实战"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - Kubernetes
    - 微服务
    - MySQL

---

## MySQL

### NameSpace

```shell
[root@k8s-master1 ~]# kubectl create namespace dev-mysql
namespace/dev-mysql created
```

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-x-conf
  namespace: dev-mysql
data: 
  my.cnf: |
    [client]
    default_character_set=utf8
    [mysqld]
    collation_server = utf8_general_ci
    character_set_server = utf8
```

### Install

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-x
  namespace: dev-mysql
spec:
  selector: 
    app: mysql-x
  type: ClusterIP
  ports:
  - port: 3306
    targetPort: 3306
    protocol: TCP
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-x
  namespace: dev-mysql
  labels:
    app: mysql-x
spec:
  serviceName: "mysql-x"
  replicas: 1
  selector:
    matchLabels:
      app: mysql-x
  template:
    metadata:
      labels:
        app: mysql-x
    spec:
      containers:
      - name: mysql-x
        image: mysql:5.7.44
        imagePullPolicy: IfNotPresent
        command : [ "/bin/sh", "-c", "--" ]
        args: [ "ulimit -n 262144; docker-entrypoint.sh mysqld;" ] # 覆盖镜像的默认命令，解决 OOMKilled 问题
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "xhBQ9QN7d+sF"
        volumeMounts:
          - name: mysql-x
            mountPath: /var/lib/mysql
          - name: conf
            mountPath: /etc/mysql/conf.d/my.cnf
            subPath: my.cnf
          - name: timezone
            mountPath: /etc/localtime
            readOnly: true
      volumes:
      - name: conf
        configMap:
          name: mysql-x-conf
          items: 
          - key: my.cnf
            path: my.cnf
      - name: timezone
        hostPath:
          path: /usr/share/zoneinfo/Asia/Shanghai
  volumeClaimTemplates:
  - metadata:
      name: mysql-x
      labels:
        app: mysql-x
    spec:
      accessModes: [ "ReadWriteOnce" ]
      #storageClassName: "managed-nfs-storage"
      resources:
        requests:
          storage: 2Gi
```

### Ingress TCP

* ConfigMap

```yaml
[root@k8s-master1 ~]# kubectl edit cm ingress-tcp -n ingress-nginx
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingress-tcp
  namespace: ingress-nginx
data:
  22: "devops/gitlab:22" # namespace/service:port ，key 与 port 可以不一样
  3306: "dev-mysql/mysql-x:3306"
```

* service/ingress-nginx-controller


```shell
[root@k8s-master1 ~]# kubectl edit service/ingress-nginx-controller -n ingress-nginx
  ports:
  - appProtocol: http
    name: http
    nodePort: 31734
    port: 80
    protocol: TCP
    targetPort: http
  - appProtocol: https
    name: https
    nodePort: 32740
    port: 443
    protocol: TCP
    targetPort: https
  - name: gitlab-ssh
    port: 22
    protocol: TCP
    targetPort: 22
  - name: dev-mysql-x # 新增
    port: 31437 # 外网访问的端口
    protocol: TCP
    targetPort: 3306 # ConfigMap 中的 key
```

* deploy/ingress-nginx-controller

```shell
[root@k8s-master1 ~]# kubectl edit deploy/ingress-nginx-controller -n ingress-nginx
  template:
    metadata:
      creationTimestamp: null
      labels:
        app.kubernetes.io/component: controller
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
        app.kubernetes.io/version: 1.9.4
    spec:
      containers:
      - args:
        - /nginx-ingress-controller
        - --tcp-services-configmap=ingress-nginx/ingress-tcp # 新增
        - --publish-service=$(POD_NAMESPACE)/ingress-nginx-controller
        - --election-id=ingress-nginx-leader
        - --controller-class=k8s.io/ingress-nginx
        - --ingress-class=nginx
        - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
        - --validating-webhook=:8443
        - --validating-webhook-certificate=/usr/local/certificates/cert
        - --validating-webhook-key=/usr/local/certificates/key

[root@k8s-master1 ~]# kubectl rollout restart deploy/ingress-nginx-controller -n ingress-nginx
deployment.apps/ingress-nginx-controller restarted
```

### 访问

```shell
[root@k8s-master1 kubernetes]# kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                                   AGE
ingress-nginx-controller             LoadBalancer   10.108.29.171   192.168.99.240   80:31734/TCP,443:32740/TCP,22:31256/TCP   13d
ingress-nginx-controller-admission   ClusterIP      10.103.114.33   <none>           443/TCP                                   13d
```

* kubernetes 集群内部：
    * mysql-x.dev-mysql:3306
* 外网：
    * 192.168.99.240:31437

