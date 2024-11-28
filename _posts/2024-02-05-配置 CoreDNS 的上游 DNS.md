---
layout:       post
title:        "Kubernetes 配置 CoreDNS 的上游 DNS"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - Kubernetes
    - CoreDNS
    - DNS

---



### 修改 CoreDNS 的上游 DNS

```yaml
[root@k8s-master1 ~]# kubectl -n kube-system edit configmap coredns
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        #forward . /etc/resolv.conf {
        #   max_concurrent 1000
        #}
        forward . 192.168.3.157 { # 自建 DNS 服务地址
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  creationTimestamp: "2023-12-12T15:50:07Z"
  name: coredns
  namespace: kube-system
  resourceVersion: "238"
  uid: 91e2a35c-42fa-4671-aacc-84699f48cbb3
```

### 重启 CoreDNS

```shell
[root@k8s-master1 ~]# kubectl rollout restart deploy coredns -n kube-system
deployment.apps/coredns restarted
```

### 验证

```shell
# 类似于docker指令: docker run --user root -it busybox:1.28.4 sh
[root@k8s-node1 ~]# kubectl run --image busybox:1.28.4 dns-test --restart=Never --rm -it -- sh
If you don't see a command prompt, try pressing enter.
/ # 
/ # cat /etc/resolv.conf 
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local zhch.lan
options ndots:5
/ # 
/ # nslookup kube-dns.kube-system
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kube-dns.kube-system
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
/ # 
/ # nslookup sonarqube.devops
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      sonarqube.devops
Address 1: 10.109.14.183 sonarqube.devops.svc.cluster.local
/ # 
/ # nslookup harbor.zhch.lan
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      harbor.zhch.lan
Address 1: 192.168.99.240 ingress-nginx-controller.ingress-nginx.svc.cluster.local
/ # 
/ # nslookup git.zhch.lan
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      git.zhch.lan
Address 1: 192.168.99.240 ingress-nginx-controller.ingress-nginx.svc.cluster.local
/ # 
/ # nslookup baidu.com
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      baidu.com
Address 1: 110.242.68.66
Address 2: 39.156.66.10
/ # 
```

