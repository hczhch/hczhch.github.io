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
    - Redis
    - RocketMQ
    - Nacos
    - Seata

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


## Redis
### NameSpace

```shell
[root@k8s-master1 ~]# kubectl create namespace dev-redis
namespace/dev-redis created
```

### Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: redis-x-conf
  namespace: dev-redis
type: Opaque
stringData: 
  redis.conf: |
    port 6379
    bind 0.0.0.0
    appendonly yes
    daemonize no
    protected-mode yes
    requirepass A3c5XpA~3g+8
```

### Install

```yaml
apiVersion: v1
kind: Service 
metadata:
  name: redis-x
  namespace: dev-redis
spec:
  selector: 
    app: redis-x
  type: ClusterIP
  ports:
  - port: 6379
    targetPort: 6379
    protocol: TCP
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-x
  namespace: dev-redis
  labels:
    app: redis-x
spec:
  serviceName: "redis-x"
  replicas: 1
  selector:
    matchLabels:
      app: redis-x
  template:
    metadata:
      labels:
        app: redis-x
    spec:
      containers:
      - name: redis-x
        image: redis:6.2.14
        imagePullPolicy: IfNotPresent
        #command: ["docker-entrypoint.sh"]
        args: ["redis-server", "/etc/redis/redis.conf"]
        ports:
        - containerPort: 6379
        volumeMounts:
          - name: redis-x
            mountPath: /data
          - name: conf
            mountPath: /etc/redis/redis.conf
            subPath: redis.conf
          - name: timezone
            mountPath: /etc/localtime
            readOnly: true
      volumes:
      - name: conf
        secret:
          secretName: redis-x-conf
          items: 
          - key: redis.conf
            path: redis.conf
      - name: timezone
        hostPath:
          path: /usr/share/zoneinfo/Asia/Shanghai
  volumeClaimTemplates:
  - metadata:
      name: redis-x
      labels:
        app: redis-x
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
  6379: "dev-redis/redis-x:6379"
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
  - name: dev-mysql-x
    port: 31437
    protocol: TCP
    targetPort: 3306
  - name: dev-redis-x # 新增
    port: 18623 # 外网访问的端口
    protocol: TCP
    targetPort: 6379 # ConfigMap 中的 key
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
  * redis-x.dev-redis:6379
* 外网：
  * 192.168.99.240:18623


## RocketMQ

### NameSpace

```shell
[root@k8s-master1 ~]# kubectl create namespace dev-mq
namespace/dev-mq created
```

### Secret

```shell
[root@k8s-master1 ~]# kubectl create secret tls zhch.lan -n dev-mq \
                      --cert=/home/zc/cert/zhch.lan.crt \
                      --key=/home/zc/cert/zhch.lan.key
secret/zhch.lan created
```

### Helm

* <https://github.com/itboon/rocketmq-helm>

```shell
[root@k8s-master1 ~]# helm repo add rocketmq-repo https://helm-charts.itboon.top/rocketmq
"rocketmq-repo" has been added to your repositories

[root@k8s-master1 ~]# helm repo update rocketmq-repo

[root@k8s-master1 ~]# helm search repo rocketmq
NAME                  	CHART VERSION	APP VERSION	DESCRIPTION        
rocketmq-repo/rocketmq	3.0.3        	4.9.7      	RocketMQ Helm chart
```

```shell
[root@k8s-master1 ~]# mkdir test/rocketmq
[root@k8s-master1 ~]# cd test/rocketmq
[root@k8s-master1 rocketmq]# helm pull rocketmq-repo/rocketmq
[root@k8s-master1 rocketmq]# ls
rocketmq-3.0.3.tgz
[root@k8s-master1 rocketmq]# tar -zxf rocketmq-3.0.3.tgz
[root@k8s-master1 rocketmq]# ls
rocketmq  rocketmq-3.0.3.tgz
[root@k8s-master1 rocketmq]# cp rocketmq/values.yaml .
[root@k8s-master1 rocketmq]# vim values.yaml
```

```yaml
clusterName: "rocketmq-cluster-a"

image:
  repository: "apache/rocketmq"
  pullPolicy: IfNotPresent
  tag: "4.9.7"

broker:
  size:
    master: 1
    replica: 0
  
  master:
    brokerRole: ASYNC_MASTER
    ## jvmMemory: "-Xms1g -Xmx1g"
    jvm:
      maxHeapSize: 1024M
      # javaOptsOverride: ""
    resources:
      limits:
        cpu: 1
        memory: 1.5Gi
      requests:
        cpu: 20m
        memory: 0.5Gi
  
  replica:
    ## jvmMemory: "-Xms1g -Xmx1g"
    jvm:
      maxHeapSize: 1024M
      # javaOptsOverride: ""
    resources:
      limits:
        cpu: 1
        memory: 1.5Gi
      requests:
        cpu: 20m
        memory: 0.5Gi

  persistence:
    enabled: true
    size: 2Gi
    #storageClass: "gp2"
  
  config:
    ## brokerClusterName brokerName brokerRole brokerId 由内置脚本自动生成
    deleteWhen: "04"
    fileReservedTime: "48"
    flushDiskType: "ASYNC_FLUSH"
    waitTimeMillsInSendQueue: "1000"
    #transientStorePoolEnable: "true"
    #transferMsgByHeap: "false"

  affinityOverride: {}
  tolerations: []
  nodeSelector: {}

  ## broker.readinessProbe
  readinessProbe:
    tcpSocket:
      port: main
    initialDelaySeconds: 20
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 3

nameserver:
  replicaCount: 1

  jvm:
    maxHeapSize: 1024M
    # javaOptsOverride: ""

  resources:
    limits:
      cpu: 1
      memory: 1.5Gi
      ephemeral-storage: 2Gi
    requests:
      cpu: 20m
      memory: 0.5Gi
      ephemeral-storage: 0.2Gi
  
  persistence:
    enabled: true
    size: 2Gi
    #storageClass: "gp2"

  affinityOverride: {}
  tolerations: []
  nodeSelector: {}

  ## nameserver.readinessProbe
  readinessProbe:
    tcpSocket:
      port: main
    initialDelaySeconds: 20
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 3

proxy:
  enabled: false
  replicaCount: 1
  jvm:
    maxHeapSize: 1024M
    # javaOptsOverride: ""

  resources:
    limits:
      cpu: 2
      memory: 1.5Gi
    requests:
      cpu: 10m
      memory: 0.5Gi

  affinityOverride: {}
  tolerations: []
  nodeSelector: {}

  ## proxy.readinessProbe
  readinessProbe:
    tcpSocket:
      port: main
    initialDelaySeconds: 20
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 3

  ## proxy.service
  service:
    annotations: {}
    type: ClusterIP

dashboard:
  enabled: true
  replicaCount: 1
  image:
    repository: "apacherocketmq/rocketmq-dashboard"
    pullPolicy: IfNotPresent
    tag: "1.0.0"

  jvm:
    maxHeapSize: 256M

  resources:
    limits:
      cpu: 1
      memory: 1Gi
    requests:
      cpu: 20m
      memory: 512Mi
  
  service:
    annotations: {}
    type: ClusterIP
    # nodePort: 31007
  
  ingress:
    enabled: true
    className: "nginx"
    annotations: {}
      # nginx.ingress.kubernetes.io/whitelist-source-range: 10.0.0.0/8,124.160.30.50
    hosts:
      - host: rocketmq-dashboard.zhch.lan
    tls: 
    - secretName: zhch.lan
      hosts:
      - rocketmq-dashboard.zhch.lan
```

```shell
[root@k8s-master1 rocketmq]# cd ..
[root@k8s-master1 rocketmq]# helm install dev-rocketmq-x -f ./values.yaml ./rocketmq/ -n dev-mq
W0104 14:15:28.111141  617336 warnings.go:70] spec.template.spec.containers[0].resources.requests[ephemeral-storage]: fractional byte value "214748364800m" is invalid, must be an integer
NAME: dev-rocketmq-x
LAST DEPLOYED: Thu Jan  4 14:15:27 2024
NAMESPACE: dev-mq
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
>>> Nameserver Address:
    dev-rocketmq-x-nameserver.dev-mq:9876

>>> Visit RocketMQ Dashboard:
    https://rocketmq-dashboard.zhch.lan/
```



## Nacos

* <https://github.com/nacos-group/nacos-k8s/blob/master/deploy/nacos/nacos-pvc-nfs.yaml>

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev-nacos
---
apiVersion: v1
kind: Secret
type: kubernetes.io/tls
metadata:
  name: zhch.lan
  namespace: dev-nacos
data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUZuakNDQTRhZ0F3SUJBZ0lVWXhDSDhnWVRrL2hHK0lnSnd5QXhIdEtYN2Jzd0RRWUpLb1pJaHZjTkFRRUwKQlFBd1pqRUxNQWtHQTFVRUJoTUNRMDR4RWpBUUJnTlZCQWdNQ1VkMVlXNW5aRzl1WnpFU01CQUdBMVVFQnd3SgpSM1ZoYm1kNmFHOTFNUTB3Q3dZRFZRUUtEQVI2YUdOb01RMHdDd1lEVlFRTERBUjZhR05vTVJFd0R3WURWUVFECkRBaDZhR05vTG14aGJqQWdGdzB5TXpFeU1URXdOakExTVRsYUdBOHlNVEl6TVRFeE56QTJNRFV4T1Zvd1pqRUwKTUFrR0ExVUVCaE1DUTA0eEVqQVFCZ05WQkFnTUNVZDFZVzVuWkc5dVp6RVNNQkFHQTFVRUJ3d0pSM1ZoYm1kNgphRzkxTVEwd0N3WURWUVFLREFSNmFHTm9NUTB3Q3dZRFZRUUxEQVI2YUdOb01SRXdEd1lEVlFRRERBaDZhR05vCkxteGhiakNDQWlJd0RRWUpLb1pJaHZjTkFRRUJCUUFEZ2dJUEFEQ0NBZ29DZ2dJQkFNMDNHNjV5OUlIR1VEdVMKWXpqNHdsMUpFNE9ZWlUyeXZZRWhTYzVaZngrYXVuM09MM1BRVG00eUY5cFIvMDlCYzRZNmcxU2NtSmhTWnI0dgpDNGFWS1FxcmtyYnhEM0VRREg0dEcxdnprUmZaTmxFNFA2bG5XUWYyS0g4RmVRdEtuWHpGNm0xK3Y3M0grQmlQCmJyMnlYUkRqUUNWWm5QZXM5NEhuUExGRFNhSXNpME1oNmFuc29nM0h4bnoybnZnRWQ4bndEekJhc1Rwd0wvR0QKOFlPOFN5ZmhEcE5tdmozZmx5bmQ5TWJFREpWcHBTN1pXbC9UTG1GNHJEZy9WRUJkSkVSR05lMk4xYkR3K3VQdQo1VXdYV2ZmNzduVnNuTS9VaFV6b3JlMU01U3Vwbll3ZkxQaWZxUmlSRTFOVEd1dHhpQm9LRlQ3blNsUVVRZWNhCkZwV0pNUHBTMU54M1l0eFM0eHFxakhJZEduR3V3eDQ3WFk0MjNOUGROcDlMUnM3THRHQ2lFYlhGeWVOM3FmNEgKTWxKQzdudCtVbmtob2dtQ2Urek5TajlmenpSOTdJcmEwTGVTNm9FaFg3ZXVWYU51VGNjZG1nRU9uc2xzZUZEWgp3bHVwREwvV2hjZmNjUkM1UGUwZFBqR0VBeW1ZcXAxelJ1OXNKSTRmUHFoTkVVRmZSUUpsSTExWVp3S1lmMzcvCnBOV0g4a25lNWlBU01Sd2tJdzN6V290REpyQVl3WVFjYzJIaytlUEJyeUFaaG5BcnVPaVZIUDdYQ3lwUFBpeTAKRXJmYWcrc1N3aUVjSzZsZUw1dTdDdWRQN3Y4b1FEcklENWRXZS8zMFI5MDN4Z3NJSUpYOG9qZlFDdzhwTXZ1dApaWnU1RGxmN0JVSHNuRmE2ZWZEVjVSdEdaQ0dSQWdNQkFBR2pRakJBTUI4R0ExVWRFUVFZTUJhQ0NIcG9ZMmd1CmJHRnVnZ29xTG5wb1kyZ3ViR0Z1TUIwR0ExVWREZ1FXQkJSSklHVjk2bURjNEY5OU9ybDRSWCs1MTMrc2ZEQU4KQmdrcWhraUc5dzBCQVFzRkFBT0NBZ0VBdTNLV0JJVVNWOExVK2VoamdyM2ZnSVdSWGZmekZRSUhXc2tGcXRLSgp3YVZISVdCaG1Eakx0cTdEczFJam01MlZ5ekVXZ0xiUllmbTVlbmFlWDlISm8rTVkyMytjTWlUTEFVZitVRG9DClN1Y0V1NnZaVFhKSVlDWUM1ZFJKWGVMZTlBMWhhZ1VHRUJqZmIwYitRUEh5WHhwdFpVeWloSlluNHJPUmkrd3cKMkZJdU5zdlRIMkcxOXdpeXgxTHQzUUd2ekVoSnRIWWNRNTNSVzQwazE3eHNYMTBvVlBSeFZEVFVjclN4cnljVAoyeTJxcndJNm9zb0JjTWtIYkRhQUtzRXZkZUZhdnZtbFF4Z3Y0QkZaMVR6dCsxWjZVcGVLQXB6YkJJenIzamw4CjVIQS9oVGd5V0hGRlMrY1NYdzN4bVhkbzZkMW9YUFhVQXhaSWpTd29USjJWalNCd2lxOFJocjh3ekl1cGszdk8KcUlhZ0JuczB1TmYrK3lhRlJFbXdsWjhTY21aTVhwRWptanFWTC9mQUdYRGsreTNZSnFIUm9KUjltSzdGdndXWAplMnN3UkVZNFN1TDdqbDhvSVhpSTNQQ2poR1B2VWxPc0lvVUNXQW9yZ24wYW5IQTEyYW1NQ3VFcEIxSWkvNllpClVoQ0ZLOTFlUno0ZzdiWVZpZ2VVd2JnUmtvOVEwVmt3SjJqMjFOdDdPRXorR0Z1YUNKeVZEaHY3WE5YOHR2cVcKRS9LS1VYbTkyeW1rWTIzNjNKUTRDbE5XNkpadHVSYjg5L1RyYjFtdis4dnVpUmh5dlcvU3ZPS3FkbTQvRlVaVwpPNlR5Q0tubVdXZWJZV1ZpU2c3NEQzYmhpdGNvV21kZDdFd0hBSERaeFNoN0U1TWEvbTlrbzc0ZWRFQ2lKd3BNCnE5ST0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
  tls.key: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JSUpRZ0lCQURBTkJna3Foa2lHOXcwQkFRRUZBQVNDQ1N3d2dna29BZ0VBQW9JQ0FRRE5OeHV1Y3ZTQnhsQTcKa21NNCtNSmRTUk9EbUdWTnNyMkJJVW5PV1g4Zm1ycDl6aTl6MEU1dU1oZmFVZjlQUVhPR09vTlVuSmlZVW1hKwpMd3VHbFNrS3E1SzI4UTl4RUF4K0xSdGI4NUVYMlRaUk9EK3BaMWtIOWloL0JYa0xTcDE4eGVwdGZyKzl4L2dZCmoyNjlzbDBRNDBBbFdaejNyUGVCNXp5eFEwbWlMSXRESWVtcDdLSU54OFo4OXA3NEJIZko4QTh3V3JFNmNDL3gKZy9HRHZFc240UTZUWnI0OTM1Y3AzZlRHeEF5VmFhVXUyVnBmMHk1aGVLdzRQMVJBWFNSRVJqWHRqZFd3OFByago3dVZNRjFuMysrNTFiSnpQMUlWTTZLM3RUT1VycVoyTUh5ejRuNmtZa1JOVFV4cnJjWWdhQ2hVKzUwcFVGRUhuCkdoYVZpVEQ2VXRUY2QyTGNVdU1hcW94eUhScHhyc01lTzEyT050elQzVGFmUzBiT3k3UmdvaEcxeGNuamQ2bisKQnpKU1F1NTdmbEo1SWFJSmdudnN6VW8vWDg4MGZleUsydEMza3VxQklWKzNybFdqYmszSEhab0JEcDdKYkhoUQoyY0picVF5LzFvWEgzSEVRdVQzdEhUNHhoQU1wbUtxZGMwYnZiQ1NPSHo2b1RSRkJYMFVDWlNOZFdHY0NtSDkrCi82VFZoL0pKM3VZZ0VqRWNKQ01OODFxTFF5YXdHTUdFSEhOaDVQbmp3YThnR1lad0s3am9sUnorMXdzcVR6NHMKdEJLMzJvUHJFc0loSEN1cFhpK2J1d3JuVCs3L0tFQTZ5QStYVm52OTlFZmROOFlMQ0NDVi9LSTMwQXNQS1RMNwpyV1didVE1WCt3VkI3SnhXdW5udzFlVWJSbVFoa1FJREFRQUJBb0lCLzN6emNRZG5OemxOWnN6ZTlVdGJLLzFnCjRXRGZDYytsWlgyYXB6WGRpR25WN0hkdGM3Y3d2cENhTDZ2ZkFYVmdoTmJXQ2VFYStFN0czWWd2WFBVMUhTaEMKRDdNVVZES2pjdmZndnlmZHhocWZSMU5zekZaNWR0eENKYVl4enVIeExMTXNUdkVjbStNU1B4MjFOOWlKSWVHRwpmU2hBeURLR1BxMzUvaHB3dmdUZzJtcWwyNEI3ZExDdlUwd0RYZ2Zsc0lwa2dOc1FYWmtYZGhtNEhQWDVVRW1YCjN5Z2hCdlRsajBVT3dGdkdRMk0yVUQyV1dsQytaUjgwT3FpRTV1Zkp6cXREbE5KdjZnMHlyWkRiaFFJdnRiZ28KemFqeDJRa3lmWGUydFRBb0FlSDBCTm1zb2RWQVlkVnpnRERjQ1NnU21LeENOMjExcHV4SzZWV3RyTktnRmhFOQpJUXM5Rit6cTVGTkFOS2VZbEUxZFhkY1ArU0l6SUsyRXl0TVhjWTBmd0hha1BpWlRLMW1Ud2JMYkQyb1Jvck80CjRtUEEwak1BWEJ4QkdRUlFxNldTc28rVjZFdHV6ZzhJNnRJKzBCOFIzVmpaZDQ2Mlh4SUNoTTBzNWZNbFpRYm0KNlBWNmdFTWlpMi84Umh2S1Q4eXlkdThDMDRnbExwbHlnMlB5UElWSC9LdHplN2hKWGVZeldjempmTWxCYm9NQgpqQlBwODVCSitZa2dqbmhJK3FCS09GaU5FaDN5OS9QclVMSHBza0IwNkptaVJGSWFKQkpLSjlJd05lMGxuZzg1CmFuWE9MQTFIL2FDL1VUekpYeG9DNnA2VVJPajZQdC8xSjdNc0RCbEJHOXRhTnlKcHRkWEhrODIvUHFXbnBFSUgKMzRXSGMwdWNqMmtsV2NZLzF6a0NnZ0VCQVBnREgzTjdHR1RESitDdWhTRXdlN3JnUmYvNzlaZUtXeFdIWlM2ZApHRS9xQjZYUGNHOFJZdE5abnpobGxNQVZVdWlnM2VOR0lYdE5oUE0wcVlVeExDb0tRek1JWC9oblcxQTdoOUg5Cml5dVRoSFB5elNIK2dwNnYzaDZEYkdWUSs0Mys2d3JlZ3JhNlhjaGFNVmhBY285d2t6cXpaVWRHZk5VRVZSKzcKUDBBUkpPWDhrSlpHc0U2aHpiL3c1aXAwUFRwTlZMQzRSNEtmODgzeU5CeGVLY0MvNGYxSVlaTXBUR3ZreCthaQo5RGlJOTlqTTgxNW9vR3lNOXF4REZqaDVEdStHMzlwOVRYT2NuZDVGYjdLM2lrWDk3YjhwMXk4d0RoWEhOV3V6CkpMazRwdGtHRnJTLzJlaXdvWVo3aHhZTmdZMythODhzWVlUWEs4NlNzRVgvTXFrQ2dnRUJBTlBUSHlnNHlxL1AKZkU0NjM0NGJ6d244WUF3UFhNMnpLUmRqSFkvTlo0cEhicmRhT09qN0pBK0J6YS9KTjZkTzlsRDc4YWEyVURLYgpPNGJONTBzRDY1anBwK0lqUnp5OE82TEkzTnhVTlRHMkZmZkRyNkJmd3NwYUd2VTRrZjBwdVlnUTE2b0VUVk1aCnR4dmgvNmV1Mi9sbFdPTmRFUkZzaktucUhkNFhDYkJVLzNpOTBkdWxRRkZCcXh1Q1lMSzkzZkJBeXArZ3NiODMKYm84djdrMUJ5eFh6Q2l1ZlZONFlNYkc3WUQyK0RrY1lDSHJXWVgrdGkyL0VIeGErMEhmaXNiSmNhaWcyZjAycApHM0d5ejRPVTE5SnI5enJDM2xaQ0hzQ3ZuVE5YT21TOG85eEJOTTh1RDBvWXNlVWtKYUs4QUtOcldBVkNRclZjCkhNSzhSa3VNTUtrQ2dnRUJBT25md0FQZE81YWhkZlJwZm45YXdnTHF4UGZ0T0o0cnlWTFc5L0pxRCtna01Bd0wKUHVKdUNieDJVakFUa3A5RVBJZkVVeG1rSTZTcjZFaVVDNXZmVDk5aENCZVN1VFY4K2Q0Q0ZVVlBpN0tQREtOdQpma1NsUmJXdzhJdmpzUThsdStJZVZyVk1PUVZwWDFDMHhMMk5JTHJsRk9HUkZGdVBPOTZBbEdrMDRTTmdSMlJkCnRGY1IxK1orckpCbzhoTnN3K1E3MGpaSHdKK01pSk5YNkE0c09jRmE4Umd3N2xxZzRrRUlYLzI5QXdKaEh4K2gKdlluMHJmdFBQcm9aRlZZeHlvVFRzanJPV0lCQ1c1aWo3LzRmR0ZTQ2JYVU1WckJYNTZCZjE1OTFNcGM3dGhNSApxOWZNNXdlSHNQb3BlS3l5RmM2NThoNU9vck5yV1JNV3Z3Vnk3dWtDZ2dFQkFKUWFZd2grWE1qNzYwL1BQZ3RnClNpd1ROeHgzaVUyUlhNT3JXem4yUmRTYkNVQk5ac2tPL3pHUWNqM2NGSHQ0YkNSSFk3aEtkRnhOeVJzQjBCdlYKQzk4SVQ0ZC9Yd21LR3JCQWZKdllqTERMUFNUVXYzRUVRMit6L0hGRU1sNnQwN2pjL2MwejROU2ZnRFdRbUcybgpoc29qSURrb0V3ejV0b2YrMXc4M1VHRG5yUS9BdUlBNFZIWDcwaVVUellScjJFZHBKY0xpV2lUMkh1a2lmQjJzClNOQjU4N3g0VktCTWprSlVYb0FNNkhLd3pRMEY0M21mMzRRdnZnVHJPVnI1TjRFYnVHV1JaUVRwbmZTckx3Z3oKQTR0dVRaZmFOQlpmZUowRXJJYi9FQ2JxOWk3RHNLYkM3NUhCSG5DMkMxSnkzSWRtUUU2OCsyTk9taFZXQ2xnOApGckVDZ2dFQU53WlpnN2ZXaEl4emlrcnBia1Vsa0JQR283RVh4T2hpcG4vU3VzY3kzd01nR0RDRm9ISFdrajY1CndtN2tKMGVxL1hrTE5KVXZEWllNbkFJZ2tFYnFyeXREaW5TVnlYUGt1S3FnTURlV1ZMRnFTZ3ppdWUwUHdIRncKeHNFNnIrR3JzTGhhVHR5M3YxemRUclg2dTczclNuN0xZVWs5S2hURjR3ajF4UmJWSjdNd0RNb01PNnFDMTFzTAp5aXM4NmxSM3dwNnlSaTlHMUpqTmpsTTNSa1ZXV2VCM0VYUE1HVjkvMmlUbE5zc0ppTTJEdnJ2SEtYcHczWExiClFVOHVySlRrdG12U0ErcU1LbUJKRWMrZXFhL1M5OTYyT3hDLzBmYTBOQUMxWmxJcHpPNEFKSDBQZXdHZjhBVWUKMEh6anc1QUdoT2dGQUgySzhCTkk4dk05QXdLaE9nPT0KLS0tLS1FTkQgUFJJVkFURSBLRVktLS0tLQo=
---
apiVersion: v1
kind: Service
metadata:
  name: nacos-headless
  namespace: dev-nacos
  labels:
    app: nacos-headless
spec:
  publishNotReadyAddresses: true
  type: ClusterIP
  clusterIP: None
  selector:
    app: nacos
  ports:
    - port: 8848
      name: server
      targetPort: 8848
    - port: 9848
      name: client-rpc
      targetPort: 9848
    - port: 9849
      name: raft-rpc
      targetPort: 9849
    # 兼容1.4.x版本的选举端口
    - port: 7848
      name: old-raft-rpc
      targetPort: 7848
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nacos-cm
  namespace: dev-nacos
data:
  mysql.host: "mysql-x.dev-mysql"
  mysql.db.name: "dev_nacos" # 需提前手动创建好数据库，并执行初始化语句 https://github.com/alibaba/nacos/blob/2.3.0/distribution/conf/mysql-schema.sql
  mysql.port: "3306"
  mysql.user: "root"
  mysql.password: "xhBQ9QN7d+sF"
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nacos
  namespace: dev-nacos
spec:
  podManagementPolicy: Parallel
  serviceName: nacos-headless
  replicas: 3
  selector:
    matchLabels:
      app: nacos
  template:
    metadata:
      labels:
        app: nacos
      annotations:
        pod.alpha.kubernetes.io/initialized: "true"
    spec:
      affinity:
        podAntiAffinity:
          #requiredDuringSchedulingIgnoredDuringExecution:
          #  - labelSelector:
          #      matchExpressions:
          #        - key: "app"
          #          operator: In
          #          values:
          #            - nacos
          #    topologyKey: "kubernetes.io/hostname"
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: "app"
                  operator: In
                  values:
                  - nacos
              topologyKey: "kubernetes.io/hostname"
      initContainers:
        - name: peer-finder-plugin-install
          image: nacos/nacos-peer-finder-plugin:1.1  # 用于在Kubernetes环境中自动发现和注册 Nacos 节点
          imagePullPolicy: Always
          volumeMounts:
            - mountPath: /home/nacos/plugins/peer-finder
              name: data
              subPath: peer-finder
      containers:
        - name: nacos
          imagePullPolicy: Always
          image: nacos/nacos-server:v2.3.0
          resources:
            requests:
              memory: 256Mi
              cpu: 50m
            limits: 
              memory: 1Gi
              cpu: 1000m
          ports:
            - containerPort: 8848
              name: client
            - containerPort: 9848
              name: client-rpc
            - containerPort: 9849
              name: raft-rpc
            - containerPort: 7848
              name: old-raft-rpc
          env:
            - name: NACOS_AUTH_ENABLE # nacos.core.auth.enabled，NACOS 集群鉴权启用开关 ，默认 false
              value: "true"
            - name: NACOS_AUTH_TOKEN # nacos.core.auth.plugin.nacos.token.secret.key
              #节点间密钥需要保持一致
              #自定义密钥时，推荐将配置项设置为Base64编码的字符串，且原始密钥长度不得低于32字符
              value: "ZWVkNWYwZWItMTlkOC00OGNkLWE0MjItZWEyZjUyOGFjNTFlCg=="
            - name: NACOS_AUTH_IDENTITY_KEY # nacos.core.auth.server.identity.key 启用集群鉴权后需设置，服务端之间通信的身份识别的key（不可为空）
              value: "kUFdkKrs"
            - name: NACOS_AUTH_IDENTITY_VALUE # nacos.core.auth.server.identity.value 启用集群鉴权后需设置，服务端之间通信的身份识别的value（不可为空）
              value: "ZJbd#&m5j4Jhc,p%McBr-E8Qyi6UAAQF"
            - name: NACOS_REPLICAS
              value: "3"
            - name: SERVICE_NAME
              value: "nacos-headless"
            - name: DOMAIN_NAME
              value: "cluster.local"
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.namespace
            - name: MYSQL_SERVICE_HOST
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.host
            - name: MYSQL_SERVICE_DB_NAME
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.db.name
            - name: MYSQL_SERVICE_PORT
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.port
            - name: MYSQL_SERVICE_USER
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.user
            - name: MYSQL_SERVICE_PASSWORD
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.password
            - name: SPRING_DATASOURCE_PLATFORM
              value: "mysql"
            - name: NACOS_SERVER_PORT
              value: "8848"
            - name: NACOS_APPLICATION_PORT
              value: "8848"
            - name: PREFER_HOST_MODE
              value: "hostname"
            #- name: NACOS_SERVERS
            #  value: "nacos-0.nacos-headless.dev-nacos.svc.cluster.local:8848 nacos-1.nacos-headless.dev-nacos.svc.cluster.local:8848 nacos-2.nacos-headless.dev-nacos.svc.cluster.local:8848"
          volumeMounts:
            - name: data
              mountPath: /home/nacos/plugins/peer-finder
              subPath: peer-finder
            - name: data
              mountPath: /home/nacos/data
              subPath: data
            - name: data
              mountPath: /home/nacos/logs
              subPath: logs
            - name: timezone
              mountPath: /etc/localtime
              readOnly: true
      volumes:
        - name: timezone
          hostPath:
            path: /usr/share/zoneinfo/Asia/Shanghai
  volumeClaimTemplates:
    - metadata:
        name: data
        #annotations:
        #  volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
      spec:
        accessModes: [ "ReadWriteMany" ]
        resources:
          requests:
            storage: 2Gi
---
# 内部系统，不可暴露到公网
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nacos-headless
  namespace: dev-nacos
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - nacos-web.zhch.lan
    secretName: zhch.lan
  rules:
  - host: nacos-web.zhch.lan
    http:
      paths:
      - path: /nacos
        pathType: Prefix
        backend:
          service: 
            name: nacos-headless
            port:
              name: server
```



## Seata

* 在 Nacos 上为 Seata 创建一个新的 namespace ：空间名为 `Seata`
* 在 Nacos 的 Seata 命名空间 新建配置（Data ID 为 seataServer.properties 、 Group 为 SEATA_GROUP）
* 配置内容参考 <https://github.com/apache/incubator-seata/blob/v2.0.0/script/config-center/config.txt> 并按需修改保存。 截取本次案例修改部分的内容如下：

```properties
# 把 store.mode=file 改为 store.mode=db 代表采用数据库存储 Seata-Server 的全局事务数据
#Transaction storage configuration, only for the server. The file, db, and redis configuration values are optional.
store.mode=db
store.lock.mode=db
store.session.mode=db

# 修改 store.db 全局事务数据库配置项（驱动、jdbc url、用户名密码等），稍后还要手动创建相应的 database 和 table
store.db.url=jdbc:mysql://mysql-x.dev-mysql:3306/seata?characterEncoding=utf8&useUnicode=true&useSSL=false&serverTimezone=Asia/Shanghai&rewriteBatchedStatements=true
store.db.user=root
store.db.password=xhBQ9QN7d+sF

# 事务逻辑分组 与 Seata 服务端集群的映射关系
# 等号右侧是 seata 集群名（集群名称由 seata 服务端在 application.yml (seata.registry.nacos.cluster) 文件中设置）
# 'default_tx_group'是自定义的事务逻辑分组名称，每行配置一个事务逻辑分组，可配置多个
# 事务逻辑分组名在 seata 客户端中会被用到（ seata 客户端需要指明其所属的事务逻辑分组，通常在 yml 或 properties 文件中通过 seata.tx-service-group 配置项指定）
service.vgroupMapping.default_tx_group=default
# TC服务列表，仅注册中心为file时使用，nacos 作为注册中心时可忽略或删除该项
service.default.grouplist=127.0.0.1:8091
```

* 创建并初始化 Seata-Server 全局事务数据库
  * <https://github.com/apache/incubator-seata/blob/v2.0.0/script/server/db/mysql.sql>
    * global_table 保存全局事务数据；　　
    * branch_table 保存分支事务数据；
    * lock_table 保存锁定资源数据

```sql
-- -------------------------------- The script used when storeMode is 'db' --------------------------------
-- the table to store GlobalSession data
CREATE TABLE IF NOT EXISTS `global_table`
(
    `xid`                       VARCHAR(128) NOT NULL,
    `transaction_id`            BIGINT,
    `status`                    TINYINT      NOT NULL,
    `application_id`            VARCHAR(32),
    `transaction_service_group` VARCHAR(32),
    `transaction_name`          VARCHAR(128),
    `timeout`                   INT,
    `begin_time`                BIGINT,
    `application_data`          VARCHAR(2000),
    `gmt_create`                DATETIME,
    `gmt_modified`              DATETIME,
    PRIMARY KEY (`xid`),
    KEY `idx_status_gmt_modified` (`status` , `gmt_modified`),
    KEY `idx_transaction_id` (`transaction_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4;

-- the table to store BranchSession data
CREATE TABLE IF NOT EXISTS `branch_table`
(
    `branch_id`         BIGINT       NOT NULL,
    `xid`               VARCHAR(128) NOT NULL,
    `transaction_id`    BIGINT,
    `resource_group_id` VARCHAR(32),
    `resource_id`       VARCHAR(256),
    `branch_type`       VARCHAR(8),
    `status`            TINYINT,
    `client_id`         VARCHAR(64),
    `application_data`  VARCHAR(2000),
    `gmt_create`        DATETIME(6),
    `gmt_modified`      DATETIME(6),
    PRIMARY KEY (`branch_id`),
    KEY `idx_xid` (`xid`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4;

-- the table to store lock data
CREATE TABLE IF NOT EXISTS `lock_table`
(
    `row_key`        VARCHAR(128) NOT NULL,
    `xid`            VARCHAR(128),
    `transaction_id` BIGINT,
    `branch_id`      BIGINT       NOT NULL,
    `resource_id`    VARCHAR(256),
    `table_name`     VARCHAR(32),
    `pk`             VARCHAR(36),
    `status`         TINYINT      NOT NULL DEFAULT '0' COMMENT '0:locked ,1:rollbacking',
    `gmt_create`     DATETIME,
    `gmt_modified`   DATETIME,
    PRIMARY KEY (`row_key`),
    KEY `idx_status` (`status`),
    KEY `idx_branch_id` (`branch_id`),
    KEY `idx_xid` (`xid`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4;

CREATE TABLE IF NOT EXISTS `distributed_lock`
(
    `lock_key`       CHAR(20) NOT NULL,
    `lock_value`     VARCHAR(20) NOT NULL,
    `expire`         BIGINT,
    primary key (`lock_key`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4;

INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('AsyncCommitting', ' ', 0);
INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('RetryCommitting', ' ', 0);
INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('RetryRollbacking', ' ', 0);
INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('TxTimeoutCheck', ' ', 0);
```

* install seata server

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev-seata
---
apiVersion: v1
kind: Secret
type: kubernetes.io/tls
metadata:
  name: zhch.lan
  namespace: dev-seata
data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUZuakNDQTRhZ0F3SUJBZ0lVWXhDSDhnWVRrL2hHK0lnSnd5QXhIdEtYN2Jzd0RRWUpLb1pJaHZjTkFRRUwKQlFBd1pqRUxNQWtHQTFVRUJoTUNRMDR4RWpBUUJnTlZCQWdNQ1VkMVlXNW5aRzl1WnpFU01CQUdBMVVFQnd3SgpSM1ZoYm1kNmFHOTFNUTB3Q3dZRFZRUUtEQVI2YUdOb01RMHdDd1lEVlFRTERBUjZhR05vTVJFd0R3WURWUVFECkRBaDZhR05vTG14aGJqQWdGdzB5TXpFeU1URXdOakExTVRsYUdBOHlNVEl6TVRFeE56QTJNRFV4T1Zvd1pqRUwKTUFrR0ExVUVCaE1DUTA0eEVqQVFCZ05WQkFnTUNVZDFZVzVuWkc5dVp6RVNNQkFHQTFVRUJ3d0pSM1ZoYm1kNgphRzkxTVEwd0N3WURWUVFLREFSNmFHTm9NUTB3Q3dZRFZRUUxEQVI2YUdOb01SRXdEd1lEVlFRRERBaDZhR05vCkxteGhiakNDQWlJd0RRWUpLb1pJaHZjTkFRRUJCUUFEZ2dJUEFEQ0NBZ29DZ2dJQkFNMDNHNjV5OUlIR1VEdVMKWXpqNHdsMUpFNE9ZWlUyeXZZRWhTYzVaZngrYXVuM09MM1BRVG00eUY5cFIvMDlCYzRZNmcxU2NtSmhTWnI0dgpDNGFWS1FxcmtyYnhEM0VRREg0dEcxdnprUmZaTmxFNFA2bG5XUWYyS0g4RmVRdEtuWHpGNm0xK3Y3M0grQmlQCmJyMnlYUkRqUUNWWm5QZXM5NEhuUExGRFNhSXNpME1oNmFuc29nM0h4bnoybnZnRWQ4bndEekJhc1Rwd0wvR0QKOFlPOFN5ZmhEcE5tdmozZmx5bmQ5TWJFREpWcHBTN1pXbC9UTG1GNHJEZy9WRUJkSkVSR05lMk4xYkR3K3VQdQo1VXdYV2ZmNzduVnNuTS9VaFV6b3JlMU01U3Vwbll3ZkxQaWZxUmlSRTFOVEd1dHhpQm9LRlQ3blNsUVVRZWNhCkZwV0pNUHBTMU54M1l0eFM0eHFxakhJZEduR3V3eDQ3WFk0MjNOUGROcDlMUnM3THRHQ2lFYlhGeWVOM3FmNEgKTWxKQzdudCtVbmtob2dtQ2Urek5TajlmenpSOTdJcmEwTGVTNm9FaFg3ZXVWYU51VGNjZG1nRU9uc2xzZUZEWgp3bHVwREwvV2hjZmNjUkM1UGUwZFBqR0VBeW1ZcXAxelJ1OXNKSTRmUHFoTkVVRmZSUUpsSTExWVp3S1lmMzcvCnBOV0g4a25lNWlBU01Sd2tJdzN6V290REpyQVl3WVFjYzJIaytlUEJyeUFaaG5BcnVPaVZIUDdYQ3lwUFBpeTAKRXJmYWcrc1N3aUVjSzZsZUw1dTdDdWRQN3Y4b1FEcklENWRXZS8zMFI5MDN4Z3NJSUpYOG9qZlFDdzhwTXZ1dApaWnU1RGxmN0JVSHNuRmE2ZWZEVjVSdEdaQ0dSQWdNQkFBR2pRakJBTUI4R0ExVWRFUVFZTUJhQ0NIcG9ZMmd1CmJHRnVnZ29xTG5wb1kyZ3ViR0Z1TUIwR0ExVWREZ1FXQkJSSklHVjk2bURjNEY5OU9ybDRSWCs1MTMrc2ZEQU4KQmdrcWhraUc5dzBCQVFzRkFBT0NBZ0VBdTNLV0JJVVNWOExVK2VoamdyM2ZnSVdSWGZmekZRSUhXc2tGcXRLSgp3YVZISVdCaG1Eakx0cTdEczFJam01MlZ5ekVXZ0xiUllmbTVlbmFlWDlISm8rTVkyMytjTWlUTEFVZitVRG9DClN1Y0V1NnZaVFhKSVlDWUM1ZFJKWGVMZTlBMWhhZ1VHRUJqZmIwYitRUEh5WHhwdFpVeWloSlluNHJPUmkrd3cKMkZJdU5zdlRIMkcxOXdpeXgxTHQzUUd2ekVoSnRIWWNRNTNSVzQwazE3eHNYMTBvVlBSeFZEVFVjclN4cnljVAoyeTJxcndJNm9zb0JjTWtIYkRhQUtzRXZkZUZhdnZtbFF4Z3Y0QkZaMVR6dCsxWjZVcGVLQXB6YkJJenIzamw4CjVIQS9oVGd5V0hGRlMrY1NYdzN4bVhkbzZkMW9YUFhVQXhaSWpTd29USjJWalNCd2lxOFJocjh3ekl1cGszdk8KcUlhZ0JuczB1TmYrK3lhRlJFbXdsWjhTY21aTVhwRWptanFWTC9mQUdYRGsreTNZSnFIUm9KUjltSzdGdndXWAplMnN3UkVZNFN1TDdqbDhvSVhpSTNQQ2poR1B2VWxPc0lvVUNXQW9yZ24wYW5IQTEyYW1NQ3VFcEIxSWkvNllpClVoQ0ZLOTFlUno0ZzdiWVZpZ2VVd2JnUmtvOVEwVmt3SjJqMjFOdDdPRXorR0Z1YUNKeVZEaHY3WE5YOHR2cVcKRS9LS1VYbTkyeW1rWTIzNjNKUTRDbE5XNkpadHVSYjg5L1RyYjFtdis4dnVpUmh5dlcvU3ZPS3FkbTQvRlVaVwpPNlR5Q0tubVdXZWJZV1ZpU2c3NEQzYmhpdGNvV21kZDdFd0hBSERaeFNoN0U1TWEvbTlrbzc0ZWRFQ2lKd3BNCnE5ST0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
  tls.key: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JSUpRZ0lCQURBTkJna3Foa2lHOXcwQkFRRUZBQVNDQ1N3d2dna29BZ0VBQW9JQ0FRRE5OeHV1Y3ZTQnhsQTcKa21NNCtNSmRTUk9EbUdWTnNyMkJJVW5PV1g4Zm1ycDl6aTl6MEU1dU1oZmFVZjlQUVhPR09vTlVuSmlZVW1hKwpMd3VHbFNrS3E1SzI4UTl4RUF4K0xSdGI4NUVYMlRaUk9EK3BaMWtIOWloL0JYa0xTcDE4eGVwdGZyKzl4L2dZCmoyNjlzbDBRNDBBbFdaejNyUGVCNXp5eFEwbWlMSXRESWVtcDdLSU54OFo4OXA3NEJIZko4QTh3V3JFNmNDL3gKZy9HRHZFc240UTZUWnI0OTM1Y3AzZlRHeEF5VmFhVXUyVnBmMHk1aGVLdzRQMVJBWFNSRVJqWHRqZFd3OFByago3dVZNRjFuMysrNTFiSnpQMUlWTTZLM3RUT1VycVoyTUh5ejRuNmtZa1JOVFV4cnJjWWdhQ2hVKzUwcFVGRUhuCkdoYVZpVEQ2VXRUY2QyTGNVdU1hcW94eUhScHhyc01lTzEyT050elQzVGFmUzBiT3k3UmdvaEcxeGNuamQ2bisKQnpKU1F1NTdmbEo1SWFJSmdudnN6VW8vWDg4MGZleUsydEMza3VxQklWKzNybFdqYmszSEhab0JEcDdKYkhoUQoyY0picVF5LzFvWEgzSEVRdVQzdEhUNHhoQU1wbUtxZGMwYnZiQ1NPSHo2b1RSRkJYMFVDWlNOZFdHY0NtSDkrCi82VFZoL0pKM3VZZ0VqRWNKQ01OODFxTFF5YXdHTUdFSEhOaDVQbmp3YThnR1lad0s3am9sUnorMXdzcVR6NHMKdEJLMzJvUHJFc0loSEN1cFhpK2J1d3JuVCs3L0tFQTZ5QStYVm52OTlFZmROOFlMQ0NDVi9LSTMwQXNQS1RMNwpyV1didVE1WCt3VkI3SnhXdW5udzFlVWJSbVFoa1FJREFRQUJBb0lCLzN6emNRZG5OemxOWnN6ZTlVdGJLLzFnCjRXRGZDYytsWlgyYXB6WGRpR25WN0hkdGM3Y3d2cENhTDZ2ZkFYVmdoTmJXQ2VFYStFN0czWWd2WFBVMUhTaEMKRDdNVVZES2pjdmZndnlmZHhocWZSMU5zekZaNWR0eENKYVl4enVIeExMTXNUdkVjbStNU1B4MjFOOWlKSWVHRwpmU2hBeURLR1BxMzUvaHB3dmdUZzJtcWwyNEI3ZExDdlUwd0RYZ2Zsc0lwa2dOc1FYWmtYZGhtNEhQWDVVRW1YCjN5Z2hCdlRsajBVT3dGdkdRMk0yVUQyV1dsQytaUjgwT3FpRTV1Zkp6cXREbE5KdjZnMHlyWkRiaFFJdnRiZ28KemFqeDJRa3lmWGUydFRBb0FlSDBCTm1zb2RWQVlkVnpnRERjQ1NnU21LeENOMjExcHV4SzZWV3RyTktnRmhFOQpJUXM5Rit6cTVGTkFOS2VZbEUxZFhkY1ArU0l6SUsyRXl0TVhjWTBmd0hha1BpWlRLMW1Ud2JMYkQyb1Jvck80CjRtUEEwak1BWEJ4QkdRUlFxNldTc28rVjZFdHV6ZzhJNnRJKzBCOFIzVmpaZDQ2Mlh4SUNoTTBzNWZNbFpRYm0KNlBWNmdFTWlpMi84Umh2S1Q4eXlkdThDMDRnbExwbHlnMlB5UElWSC9LdHplN2hKWGVZeldjempmTWxCYm9NQgpqQlBwODVCSitZa2dqbmhJK3FCS09GaU5FaDN5OS9QclVMSHBza0IwNkptaVJGSWFKQkpLSjlJd05lMGxuZzg1CmFuWE9MQTFIL2FDL1VUekpYeG9DNnA2VVJPajZQdC8xSjdNc0RCbEJHOXRhTnlKcHRkWEhrODIvUHFXbnBFSUgKMzRXSGMwdWNqMmtsV2NZLzF6a0NnZ0VCQVBnREgzTjdHR1RESitDdWhTRXdlN3JnUmYvNzlaZUtXeFdIWlM2ZApHRS9xQjZYUGNHOFJZdE5abnpobGxNQVZVdWlnM2VOR0lYdE5oUE0wcVlVeExDb0tRek1JWC9oblcxQTdoOUg5Cml5dVRoSFB5elNIK2dwNnYzaDZEYkdWUSs0Mys2d3JlZ3JhNlhjaGFNVmhBY285d2t6cXpaVWRHZk5VRVZSKzcKUDBBUkpPWDhrSlpHc0U2aHpiL3c1aXAwUFRwTlZMQzRSNEtmODgzeU5CeGVLY0MvNGYxSVlaTXBUR3ZreCthaQo5RGlJOTlqTTgxNW9vR3lNOXF4REZqaDVEdStHMzlwOVRYT2NuZDVGYjdLM2lrWDk3YjhwMXk4d0RoWEhOV3V6CkpMazRwdGtHRnJTLzJlaXdvWVo3aHhZTmdZMythODhzWVlUWEs4NlNzRVgvTXFrQ2dnRUJBTlBUSHlnNHlxL1AKZkU0NjM0NGJ6d244WUF3UFhNMnpLUmRqSFkvTlo0cEhicmRhT09qN0pBK0J6YS9KTjZkTzlsRDc4YWEyVURLYgpPNGJONTBzRDY1anBwK0lqUnp5OE82TEkzTnhVTlRHMkZmZkRyNkJmd3NwYUd2VTRrZjBwdVlnUTE2b0VUVk1aCnR4dmgvNmV1Mi9sbFdPTmRFUkZzaktucUhkNFhDYkJVLzNpOTBkdWxRRkZCcXh1Q1lMSzkzZkJBeXArZ3NiODMKYm84djdrMUJ5eFh6Q2l1ZlZONFlNYkc3WUQyK0RrY1lDSHJXWVgrdGkyL0VIeGErMEhmaXNiSmNhaWcyZjAycApHM0d5ejRPVTE5SnI5enJDM2xaQ0hzQ3ZuVE5YT21TOG85eEJOTTh1RDBvWXNlVWtKYUs4QUtOcldBVkNRclZjCkhNSzhSa3VNTUtrQ2dnRUJBT25md0FQZE81YWhkZlJwZm45YXdnTHF4UGZ0T0o0cnlWTFc5L0pxRCtna01Bd0wKUHVKdUNieDJVakFUa3A5RVBJZkVVeG1rSTZTcjZFaVVDNXZmVDk5aENCZVN1VFY4K2Q0Q0ZVVlBpN0tQREtOdQpma1NsUmJXdzhJdmpzUThsdStJZVZyVk1PUVZwWDFDMHhMMk5JTHJsRk9HUkZGdVBPOTZBbEdrMDRTTmdSMlJkCnRGY1IxK1orckpCbzhoTnN3K1E3MGpaSHdKK01pSk5YNkE0c09jRmE4Umd3N2xxZzRrRUlYLzI5QXdKaEh4K2gKdlluMHJmdFBQcm9aRlZZeHlvVFRzanJPV0lCQ1c1aWo3LzRmR0ZTQ2JYVU1WckJYNTZCZjE1OTFNcGM3dGhNSApxOWZNNXdlSHNQb3BlS3l5RmM2NThoNU9vck5yV1JNV3Z3Vnk3dWtDZ2dFQkFKUWFZd2grWE1qNzYwL1BQZ3RnClNpd1ROeHgzaVUyUlhNT3JXem4yUmRTYkNVQk5ac2tPL3pHUWNqM2NGSHQ0YkNSSFk3aEtkRnhOeVJzQjBCdlYKQzk4SVQ0ZC9Yd21LR3JCQWZKdllqTERMUFNUVXYzRUVRMit6L0hGRU1sNnQwN2pjL2MwejROU2ZnRFdRbUcybgpoc29qSURrb0V3ejV0b2YrMXc4M1VHRG5yUS9BdUlBNFZIWDcwaVVUellScjJFZHBKY0xpV2lUMkh1a2lmQjJzClNOQjU4N3g0VktCTWprSlVYb0FNNkhLd3pRMEY0M21mMzRRdnZnVHJPVnI1TjRFYnVHV1JaUVRwbmZTckx3Z3oKQTR0dVRaZmFOQlpmZUowRXJJYi9FQ2JxOWk3RHNLYkM3NUhCSG5DMkMxSnkzSWRtUUU2OCsyTk9taFZXQ2xnOApGckVDZ2dFQU53WlpnN2ZXaEl4emlrcnBia1Vsa0JQR283RVh4T2hpcG4vU3VzY3kzd01nR0RDRm9ISFdrajY1CndtN2tKMGVxL1hrTE5KVXZEWllNbkFJZ2tFYnFyeXREaW5TVnlYUGt1S3FnTURlV1ZMRnFTZ3ppdWUwUHdIRncKeHNFNnIrR3JzTGhhVHR5M3YxemRUclg2dTczclNuN0xZVWs5S2hURjR3ajF4UmJWSjdNd0RNb01PNnFDMTFzTAp5aXM4NmxSM3dwNnlSaTlHMUpqTmpsTTNSa1ZXV2VCM0VYUE1HVjkvMmlUbE5zc0ppTTJEdnJ2SEtYcHczWExiClFVOHVySlRrdG12U0ErcU1LbUJKRWMrZXFhL1M5OTYyT3hDLzBmYTBOQUMxWmxJcHpPNEFKSDBQZXdHZjhBVWUKMEh6anc1QUdoT2dGQUgySzhCTkk4dk05QXdLaE9nPT0KLS0tLS1FTkQgUFJJVkFURSBLRVktLS0tLQo=
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: seata-server-config
  namespace: dev-seata
data:
  # https://github.com/apache/incubator-seata/blob/v2.0.0/server/src/main/resources/application.yml
  application.yml: |
    server:
      port: 7091
    
    spring:
      application:
        name: seata-server
    
    logging:
      config: classpath:logback-spring.xml
      file:
        path: ${log.home:${user.home}/logs/seata}
      extend:
        logstash-appender:
          destination: 127.0.0.1:4560
        kafka-appender:
          bootstrap-servers: 127.0.0.1:9092
          topic: logback_to_logstash
    
    console:
      user:
        username: seata
        password: seata
    seata:
      server:
        service-port: 8091 #If not configured, the default is '${server.port} + 1000'
      config:
        # support: nacos, consul, apollo, zk, etcd3
        type: nacos
        nacos:
          server-addr: nacos-headless.dev-nacos:8848
          username: "nacos" # NACOS 集群启用集群鉴权后，客户端需设置 username 和 password
          password: "nacos"
          namespace: 53b846ba-0cfe-417f-a225-4f0bac803abe # Nacos 中创建的名为 `Seata` 的 namespace 的 ID
          group: SEATA_GROUP
          data-id: seataServer.properties
      registry:
        # support: nacos, eureka, redis, zk, consul, etcd3, sofa
        # 当 registry.type=file 时用的不是真正的注册中心，不具备服务的健康检查机制，当 TC 不可用时无法自动剔除
        type: nacos
        nacos:
          application: seata-server
          server-addr: nacos-headless.dev-nacos:8848
          username: "nacos"
          password: "nacos"
          group: SEATA_GROUP
          namespace: 53b846ba-0cfe-417f-a225-4f0bac803abe
          cluster: default # 指定注册至nacos注册中心的集群名
      store:
        # support: file 、 db 、 redis
        # file 存储模式下未提供本地文件同步的功能，不能使用 file 存储模式搭建 Seata 集群
        mode: db

      security:
        secretKey: SeataSecretKey0c382ef121d778043159209298fd40bf3850a017
        tokenValidityInMilliseconds: 1800000
        ignore:
          urls: /,/**/*.css,/**/*.js,/**/*.html,/**/*.map,/**/*.svg,/**/*.png,/**/*.jpeg,/**/*.ico,/api/v1/auth/login,/metadata/v1/**
---
apiVersion: v1
kind: Service
metadata:
  name: seata-server
  namespace: dev-seata
  labels:
    app: seata-server
spec:
  selector:
    app: seata-server
  type: ClusterIP
  ports:
    - port: 8091
      protocol: TCP
      name: service
    - port: 7091
      protocol: TCP
      name: web
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: seata-server
  namespace:  dev-seata
  labels:
    app: seata-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: seata-server
  template:
    metadata:
      labels:
        app: seata-server
    spec:
      containers:
        - name: seata-server
          image: seataio/seata-server:2.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: service
              containerPort: 8091
              protocol: TCP
            - name: web
              containerPort: 7091
              protocol: TCP
          volumeMounts:
            - name: seata-config
              mountPath: /seata-server/resources/application.yml
              subPath: application.yml
            - name: timezone
              mountPath: /etc/localtime
              readOnly: true
      volumes:
        - name: seata-config
          configMap:
            name: seata-server-config
        - name: timezone
          hostPath:
            path: /usr/share/zoneinfo/Asia/Shanghai
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: seata-server
  namespace: dev-seata
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - seata-web.zhch.lan
    secretName: zhch.lan
  rules:
  - host: seata-web.zhch.lan
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: 
            name: seata-server
            port:
              name: web
```

## 项目实战

- 后端项目：
  - [https://github.com/hczhch/tensquare_parent/tree/k8s/](https://github.com/hczhch/tensquare_parent/tree/k8s/){:target="_blank"}
- 前端项目：
  - [https://github.com/hczhch/tensquare-admin/tree/k8s/](https://github.com/hczhch/tensquare-admin/tree/k8s/){:target="_blank"}
