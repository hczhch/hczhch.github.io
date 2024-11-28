---
layout:       post
title:        "kube-prometheus"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - Kubernetes
    - kube-prometheus

---

### download

* 版本匹配：<https://github.com/prometheus-operator/kube-prometheus?tab=readme-ov-file#compatibility>

```shell
[root@k8s-master1 ~]# cd software
[root@k8s-master1 software]# mkdir kube-prometheus
[root@k8s-master1 software]# cd kube-prometheus
[root@k8s-master1 kube-prometheus]# wget https://github.com/prometheus-operator/kube-prometheus/archive/refs/tags/v0.12.0.tar.gz
[root@k8s-master1 kube-prometheus]# tar -zxf v0.12.0.tar.gz
[root@k8s-master1 kube-prometheus]# ls
kube-prometheus-0.12.0  v0.12.0.tar.gz
```

### replace image

```shell
[root@k8s-master1 kube-prometheus]# cd kube-prometheus-0.12.0/manifests

[root@k8s-master1 manifests]# grep "image: " * -r
alertmanager-alertmanager.yaml:  image: quay.io/prometheus/alertmanager:v0.25.0
blackboxExporter-deployment.yaml:        image: quay.io/prometheus/blackbox-exporter:v0.23.0
blackboxExporter-deployment.yaml:        image: jimmidyson/configmap-reload:v0.5.0
blackboxExporter-deployment.yaml:        image: quay.io/brancz/kube-rbac-proxy:v0.14.0
grafana-deployment.yaml:        image: grafana/grafana:9.3.2
kubeStateMetrics-deployment.yaml:        image: registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.7.0
kubeStateMetrics-deployment.yaml:        image: quay.io/brancz/kube-rbac-proxy:v0.14.0
kubeStateMetrics-deployment.yaml:        image: quay.io/brancz/kube-rbac-proxy:v0.14.0
nodeExporter-daemonset.yaml:        image: quay.io/prometheus/node-exporter:v1.5.0
nodeExporter-daemonset.yaml:        image: quay.io/brancz/kube-rbac-proxy:v0.14.0
prometheusAdapter-deployment.yaml:        image: registry.k8s.io/prometheus-adapter/prometheus-adapter:v0.10.0
prometheusOperator-deployment.yaml:        image: quay.io/prometheus-operator/prometheus-operator:v0.62.0
prometheusOperator-deployment.yaml:        image: quay.io/brancz/kube-rbac-proxy:v0.14.0
prometheus-prometheus.yaml:  image: quay.io/prometheus/prometheus:v2.41.0
```

```shell
[root@k8s-master1 manifests]# sed -i 's/quay.io/quay.mirrors.ustc.edu.cn/g' alertmanager-alertmanager.yaml
[root@k8s-master1 manifests]# sed -i 's/quay.io/quay.mirrors.ustc.edu.cn/g' blackboxExporter-deployment.yaml
[root@k8s-master1 manifests]# sed -i 's/registry.k8s.io/lank8s.cn/g' kubeStateMetrics-deployment.yaml
[root@k8s-master1 manifests]# sed -i 's/quay.io/quay.mirrors.ustc.edu.cn/g' kubeStateMetrics-deployment.yaml
[root@k8s-master1 manifests]# sed -i 's/quay.io/quay.mirrors.ustc.edu.cn/g' nodeExporter-daemonset.yaml
[root@k8s-master1 manifests]# sed -i 's/registry.k8s.io/lank8s.cn/g' prometheusAdapter-deployment.yaml
[root@k8s-master1 manifests]# sed -i 's/quay.io/quay.mirrors.ustc.edu.cn/g' prometheusOperator-deployment.yaml
[root@k8s-master1 manifests]# sed -i 's/quay.io/quay.mirrors.ustc.edu.cn/g' prometheus-prometheus.yaml

[root@k8s-master1 manifests]# grep "image: " * -r
alertmanager-alertmanager.yaml:  image: quay.mirrors.ustc.edu.cn/prometheus/alertmanager:v0.25.0
blackboxExporter-deployment.yaml:        image: quay.mirrors.ustc.edu.cn/prometheus/blackbox-exporter:v0.23.0
blackboxExporter-deployment.yaml:        image: jimmidyson/configmap-reload:v0.5.0
blackboxExporter-deployment.yaml:        image: quay.mirrors.ustc.edu.cn/brancz/kube-rbac-proxy:v0.14.0
grafana-deployment.yaml:        image: grafana/grafana:9.3.2
kubeStateMetrics-deployment.yaml:        image: lank8s.cn/kube-state-metrics/kube-state-metrics:v2.7.0
kubeStateMetrics-deployment.yaml:        image: quay.mirrors.ustc.edu.cn/brancz/kube-rbac-proxy:v0.14.0
kubeStateMetrics-deployment.yaml:        image: quay.mirrors.ustc.edu.cn/brancz/kube-rbac-proxy:v0.14.0
nodeExporter-daemonset.yaml:        image: quay.mirrors.ustc.edu.cn/prometheus/node-exporter:v1.5.0
nodeExporter-daemonset.yaml:        image: quay.mirrors.ustc.edu.cn/brancz/kube-rbac-proxy:v0.14.0
prometheusAdapter-deployment.yaml:        image: lank8s.cn/prometheus-adapter/prometheus-adapter:v0.10.0
prometheusOperator-deployment.yaml:        image: quay.mirrors.ustc.edu.cn/prometheus-operator/prometheus-operator:v0.62.0
prometheusOperator-deployment.yaml:        image: quay.mirrors.ustc.edu.cn/brancz/kube-rbac-proxy:v0.14.0
prometheus-prometheus.yaml:  image: quay.mirrors.ustc.edu.cn/prometheus/prometheus:v2.41.0

```

### install

```shell
[root@k8s-master1 manifests]# cd ..
[root@k8s-master1 kube-prometheus-0.12.0]# kubectl apply --server-side -f manifests/setup
customresourcedefinition.apiextensions.k8s.io/alertmanagerconfigs.monitoring.coreos.com serverside-applied
customresourcedefinition.apiextensions.k8s.io/alertmanagers.monitoring.coreos.com serverside-applied
customresourcedefinition.apiextensions.k8s.io/podmonitors.monitoring.coreos.com serverside-applied
customresourcedefinition.apiextensions.k8s.io/probes.monitoring.coreos.com serverside-applied
customresourcedefinition.apiextensions.k8s.io/prometheuses.monitoring.coreos.com serverside-applied
customresourcedefinition.apiextensions.k8s.io/prometheusrules.monitoring.coreos.com serverside-applied
customresourcedefinition.apiextensions.k8s.io/servicemonitors.monitoring.coreos.com serverside-applied
customresourcedefinition.apiextensions.k8s.io/thanosrulers.monitoring.coreos.com serverside-applied
namespace/monitoring serverside-applied
```
```shell
[root@k8s-master1 kube-prometheus-0.12.0]# kubectl wait \
                                           --for condition=Established \
                                           --all CustomResourceDefinition \
                                           --namespace=monitoring
customresourcedefinition.apiextensions.k8s.io/addresspools.metallb.io condition met
customresourcedefinition.apiextensions.k8s.io/alertmanagerconfigs.monitoring.coreos.com condition met
customresourcedefinition.apiextensions.k8s.io/alertmanagers.monitoring.coreos.com condition met
customresourcedefinition.apiextensions.k8s.io/bfdprofiles.metallb.io condition met
customresourcedefinition.apiextensions.k8s.io/bgpadvertisements.metallb.io condition met
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org condition met
customresourcedefinition.apiextensions.k8s.io/bgpfilters.crd.projectcalico.org condition met
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org condition met
customresourcedefinition.apiextensions.k8s.io/bgppeers.metallb.io condition met
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org condition met
customresourcedefinition.apiextensions.k8s.io/caliconodestatuses.crd.projectcalico.org condition met
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org condition met
customresourcedefinition.apiextensions.k8s.io/communities.metallb.io condition met
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org condition met
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org condition met
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org condition met
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org condition met
customresourcedefinition.apiextensions.k8s.io/ipaddresspools.metallb.io condition met
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org condition met
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org condition met
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org condition met
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org condition met
customresourcedefinition.apiextensions.k8s.io/ipreservations.crd.projectcalico.org condition met
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org condition met
customresourcedefinition.apiextensions.k8s.io/l2advertisements.metallb.io condition met
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org condition met
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org condition met
customresourcedefinition.apiextensions.k8s.io/podmonitors.monitoring.coreos.com condition met
customresourcedefinition.apiextensions.k8s.io/probes.monitoring.coreos.com condition met
customresourcedefinition.apiextensions.k8s.io/prometheuses.monitoring.coreos.com condition met
customresourcedefinition.apiextensions.k8s.io/prometheusrules.monitoring.coreos.com condition met
customresourcedefinition.apiextensions.k8s.io/servicemonitors.monitoring.coreos.com condition met
customresourcedefinition.apiextensions.k8s.io/thanosrulers.monitoring.coreos.com condition met
```

```shell
[root@k8s-master1 kube-prometheus-0.12.0]# kubectl apply -f manifests/
alertmanager.monitoring.coreos.com/main created
networkpolicy.networking.k8s.io/alertmanager-main created
poddisruptionbudget.policy/alertmanager-main created
prometheusrule.monitoring.coreos.com/alertmanager-main-rules created
secret/alertmanager-main created
service/alertmanager-main created
serviceaccount/alertmanager-main created
servicemonitor.monitoring.coreos.com/alertmanager-main created
clusterrole.rbac.authorization.k8s.io/blackbox-exporter created
clusterrolebinding.rbac.authorization.k8s.io/blackbox-exporter created
configmap/blackbox-exporter-configuration created
deployment.apps/blackbox-exporter created
networkpolicy.networking.k8s.io/blackbox-exporter created
service/blackbox-exporter created
serviceaccount/blackbox-exporter created
servicemonitor.monitoring.coreos.com/blackbox-exporter created
secret/grafana-config created
secret/grafana-datasources created
configmap/grafana-dashboard-alertmanager-overview created
configmap/grafana-dashboard-apiserver created
configmap/grafana-dashboard-cluster-total created
configmap/grafana-dashboard-controller-manager created
configmap/grafana-dashboard-grafana-overview created
configmap/grafana-dashboard-k8s-resources-cluster created
configmap/grafana-dashboard-k8s-resources-namespace created
configmap/grafana-dashboard-k8s-resources-node created
configmap/grafana-dashboard-k8s-resources-pod created
configmap/grafana-dashboard-k8s-resources-workload created
configmap/grafana-dashboard-k8s-resources-workloads-namespace created
configmap/grafana-dashboard-kubelet created
configmap/grafana-dashboard-namespace-by-pod created
configmap/grafana-dashboard-namespace-by-workload created
configmap/grafana-dashboard-node-cluster-rsrc-use created
configmap/grafana-dashboard-node-rsrc-use created
configmap/grafana-dashboard-nodes-darwin created
configmap/grafana-dashboard-nodes created
configmap/grafana-dashboard-persistentvolumesusage created
configmap/grafana-dashboard-pod-total created
configmap/grafana-dashboard-prometheus-remote-write created
configmap/grafana-dashboard-prometheus created
configmap/grafana-dashboard-proxy created
configmap/grafana-dashboard-scheduler created
configmap/grafana-dashboard-workload-total created
configmap/grafana-dashboards created
deployment.apps/grafana created
networkpolicy.networking.k8s.io/grafana created
prometheusrule.monitoring.coreos.com/grafana-rules created
service/grafana created
serviceaccount/grafana created
servicemonitor.monitoring.coreos.com/grafana created
prometheusrule.monitoring.coreos.com/kube-prometheus-rules created
clusterrole.rbac.authorization.k8s.io/kube-state-metrics created
clusterrolebinding.rbac.authorization.k8s.io/kube-state-metrics created
deployment.apps/kube-state-metrics created
networkpolicy.networking.k8s.io/kube-state-metrics created
prometheusrule.monitoring.coreos.com/kube-state-metrics-rules created
service/kube-state-metrics created
serviceaccount/kube-state-metrics created
servicemonitor.monitoring.coreos.com/kube-state-metrics created
prometheusrule.monitoring.coreos.com/kubernetes-monitoring-rules created
servicemonitor.monitoring.coreos.com/kube-apiserver created
servicemonitor.monitoring.coreos.com/coredns created
servicemonitor.monitoring.coreos.com/kube-controller-manager created
servicemonitor.monitoring.coreos.com/kube-scheduler created
servicemonitor.monitoring.coreos.com/kubelet created
clusterrole.rbac.authorization.k8s.io/node-exporter created
clusterrolebinding.rbac.authorization.k8s.io/node-exporter created
daemonset.apps/node-exporter created
networkpolicy.networking.k8s.io/node-exporter created
prometheusrule.monitoring.coreos.com/node-exporter-rules created
service/node-exporter created
serviceaccount/node-exporter created
servicemonitor.monitoring.coreos.com/node-exporter created
clusterrole.rbac.authorization.k8s.io/prometheus-k8s created
clusterrolebinding.rbac.authorization.k8s.io/prometheus-k8s created
networkpolicy.networking.k8s.io/prometheus-k8s created
poddisruptionbudget.policy/prometheus-k8s created
prometheus.monitoring.coreos.com/k8s created
prometheusrule.monitoring.coreos.com/prometheus-k8s-prometheus-rules created
rolebinding.rbac.authorization.k8s.io/prometheus-k8s-config created
rolebinding.rbac.authorization.k8s.io/prometheus-k8s created
rolebinding.rbac.authorization.k8s.io/prometheus-k8s created
rolebinding.rbac.authorization.k8s.io/prometheus-k8s created
role.rbac.authorization.k8s.io/prometheus-k8s-config created
role.rbac.authorization.k8s.io/prometheus-k8s created
role.rbac.authorization.k8s.io/prometheus-k8s created
role.rbac.authorization.k8s.io/prometheus-k8s created
service/prometheus-k8s created
serviceaccount/prometheus-k8s created
servicemonitor.monitoring.coreos.com/prometheus-k8s created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io configured
clusterrole.rbac.authorization.k8s.io/prometheus-adapter created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader configured
clusterrolebinding.rbac.authorization.k8s.io/prometheus-adapter created
clusterrolebinding.rbac.authorization.k8s.io/resource-metrics:system:auth-delegator created
clusterrole.rbac.authorization.k8s.io/resource-metrics-server-resources created
configmap/adapter-config created
deployment.apps/prometheus-adapter created
networkpolicy.networking.k8s.io/prometheus-adapter created
poddisruptionbudget.policy/prometheus-adapter created
rolebinding.rbac.authorization.k8s.io/resource-metrics-auth-reader created
service/prometheus-adapter created
serviceaccount/prometheus-adapter created
servicemonitor.monitoring.coreos.com/prometheus-adapter created
clusterrole.rbac.authorization.k8s.io/prometheus-operator created
clusterrolebinding.rbac.authorization.k8s.io/prometheus-operator created
deployment.apps/prometheus-operator created
networkpolicy.networking.k8s.io/prometheus-operator created
prometheusrule.monitoring.coreos.com/prometheus-operator-rules created
service/prometheus-operator created
serviceaccount/prometheus-operator created
servicemonitor.monitoring.coreos.com/prometheus-operator created
```

```shell
[root@k8s-master1 kube-prometheus-0.12.0]# kubectl get all -n monitoring
NAME                                       READY   STATUS    RESTARTS        AGE
pod/alertmanager-main-0                    2/2     Running   1 (2m16s ago)   6m49s
pod/alertmanager-main-1                    2/2     Running   1 (4m38s ago)   6m49s
pod/alertmanager-main-2                    2/2     Running   1 (2m17s ago)   6m49s
pod/blackbox-exporter-8655b5cfdf-rntmp     3/3     Running   0               7m37s
pod/grafana-9f58f8675-wh44c                1/1     Running   0               7m36s
pod/kube-state-metrics-689b7c7d-6rt5v      3/3     Running   0               7m36s
pod/node-exporter-b8m74                    2/2     Running   0               7m36s
pod/node-exporter-h4vcg                    2/2     Running   0               7m36s
pod/node-exporter-lg7m6                    2/2     Running   0               7m36s
pod/prometheus-adapter-86b4c59b44-nngmt    1/1     Running   0               7m35s
pod/prometheus-adapter-86b4c59b44-xw5n8    1/1     Running   0               7m35s
pod/prometheus-k8s-0                       2/2     Running   0               6m48s
pod/prometheus-k8s-1                       2/2     Running   0               6m48s
pod/prometheus-operator-67c99c5865-q75r7   2/2     Running   0               7m35s

NAME                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/alertmanager-main       ClusterIP   10.97.0.174     <none>        9093/TCP,8080/TCP            7m37s
service/alertmanager-operated   ClusterIP   None            <none>        9093/TCP,9094/TCP,9094/UDP   6m49s
service/blackbox-exporter       ClusterIP   10.108.167.8    <none>        9115/TCP,19115/TCP           7m37s
service/grafana                 ClusterIP   10.98.130.100   <none>        3000/TCP                     7m36s
service/kube-state-metrics      ClusterIP   None            <none>        8443/TCP,9443/TCP            7m36s
service/node-exporter           ClusterIP   None            <none>        9100/TCP                     7m36s
service/prometheus-adapter      ClusterIP   10.97.241.237   <none>        443/TCP                      7m35s
service/prometheus-k8s          ClusterIP   10.96.2.56      <none>        9090/TCP,8080/TCP            7m35s
service/prometheus-operated     ClusterIP   None            <none>        9090/TCP                     6m48s
service/prometheus-operator     ClusterIP   None            <none>        8443/TCP                     7m35s

NAME                           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/node-exporter   3         3         3       3            3           kubernetes.io/os=linux   7m36s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/blackbox-exporter     1/1     1            1           7m37s
deployment.apps/grafana               1/1     1            1           7m36s
deployment.apps/kube-state-metrics    1/1     1            1           7m36s
deployment.apps/prometheus-adapter    2/2     2            2           7m35s
deployment.apps/prometheus-operator   1/1     1            1           7m35s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/blackbox-exporter-8655b5cfdf     1         1         1       7m37s
replicaset.apps/grafana-9f58f8675                1         1         1       7m36s
replicaset.apps/kube-state-metrics-689b7c7d      1         1         1       7m36s
replicaset.apps/prometheus-adapter-86b4c59b44    2         2         2       7m35s
replicaset.apps/prometheus-operator-67c99c5865   1         1         1       7m35s

NAME                                 READY   AGE
statefulset.apps/alertmanager-main   3/3     6m49s
statefulset.apps/prometheus-k8s      2/2     6m48s
```

### ingress

```shell
[root@k8s-master1 kube-prometheus-0.12.0]# kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.108.29.171   192.168.99.240   80:31734/TCP,443:32740/TCP   8d
ingress-nginx-controller-admission   ClusterIP      10.103.114.33   <none>           443/TCP                      8d

# 配置 DNS 域名解析，将以下域名解析到 192.168.99.240
# - alertmanager.zhch.lan
# - grafana.zhch.lan
# - prometheus.zhch.lan
```

```yaml
[root@k8s-master1 kube-prometheus-0.12.0]# kubectl create secret tls zhch.lan -n monitoring \
                                           --cert=/home/zc/cert/zhch.lan.crt \
                                           --key=/home/zc/cert/zhch.lan.key
secret/zhch.lan created

[root@k8s-master1 kube-prometheus-0.12.0]# vim ingress-kube-prometheus.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-kube-prometheus
  namespace: monitoring # 名称空间
spec: 
  ingressClassName: nginx
  tls:
  - hosts:
    - alertmanager.zhch.lan
    - grafana.zhch.lan
    - prometheus.zhch.lan
    secretName: zhch.lan
  rules:
  - host: alertmanager.zhch.lan
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: alertmanager-main
            port:
              number: 9093
  - host: grafana.zhch.lan
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grafana
            port:
              number: 3000
  - host: prometheus.zhch.lan
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus-k8s
            port:
              number: 9090
```

```shell
[root@k8s-master1 kube-prometheus-0.12.0]# kubectl create -f ingress-kube-prometheus.yaml 
ingress.networking.k8s.io/ingress-kube-prometheus created

[root@k8s-master1 kube-prometheus-0.12.0]# kubectl get ing -n monitoring
NAME                      CLASS   HOSTS                                                                        ADDRESS          PORTS     AGE
ingress-kube-prometheus   nginx   alertmanager.zhch.lan,alertmanager1.zhch.lan,blackbox.zhch.lan + 5 more...   192.168.99.240   80, 443   89s
```

```shell
# 504
[root@k8s-master1 manifests]# find . -name "*networkPolicy*"
./alertmanager-networkPolicy.yaml
./blackboxExporter-networkPolicy.yaml
./grafana-networkPolicy.yaml
./kubeStateMetrics-networkPolicy.yaml
./nodeExporter-networkPolicy.yaml
./prometheus-networkPolicy.yaml
./prometheusAdapter-networkPolicy.yaml
./prometheusOperator-networkPolicy.yaml

[root@k8s-master1 manifests]# kubectl get networkpolicies -n monitoring
NAME                  POD-SELECTOR                                                                                                                                             AGE
alertmanager-main     app.kubernetes.io/component=alert-router,app.kubernetes.io/instance=main,app.kubernetes.io/name=alertmanager,app.kubernetes.io/part-of=kube-prometheus   97m
blackbox-exporter     app.kubernetes.io/component=exporter,app.kubernetes.io/name=blackbox-exporter,app.kubernetes.io/part-of=kube-prometheus                                  97m
grafana               app.kubernetes.io/component=grafana,app.kubernetes.io/name=grafana,app.kubernetes.io/part-of=kube-prometheus                                             97m
kube-state-metrics    app.kubernetes.io/component=exporter,app.kubernetes.io/name=kube-state-metrics,app.kubernetes.io/part-of=kube-prometheus                                 97m
node-exporter         app.kubernetes.io/component=exporter,app.kubernetes.io/name=node-exporter,app.kubernetes.io/part-of=kube-prometheus                                      97m
prometheus-adapter    app.kubernetes.io/component=metrics-adapter,app.kubernetes.io/name=prometheus-adapter,app.kubernetes.io/part-of=kube-prometheus                          97m
prometheus-k8s        app.kubernetes.io/component=prometheus,app.kubernetes.io/instance=k8s,app.kubernetes.io/name=prometheus,app.kubernetes.io/part-of=kube-prometheus        97m
prometheus-operator   app.kubernetes.io/component=controller,app.kubernetes.io/name=prometheus-operator,app.kubernetes.io/part-of=kube-prometheus                              97m

[root@k8s-master1 manifests]# kubectl delete networkpolicies alertmanager-main grafana prometheus-k8s -n monitoring
networkpolicy.networking.k8s.io "alertmanager-main" deleted
networkpolicy.networking.k8s.io "grafana" deleted
networkpolicy.networking.k8s.io "prometheus-k8s" deleted
```

* https://grafana.zhch.lan/ 
  * 初始账号密码 admin/admin

