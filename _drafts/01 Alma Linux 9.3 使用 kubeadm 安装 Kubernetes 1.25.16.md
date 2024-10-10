### 官方文档
https://v1-25.docs.kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/  
https://github.com/kubernetes/kubernetes  

### 环境

| Hostname             | IP             | 程序                                           |
| -------------------- | -------------- | ---------------------------------------------- |
| k8s-master1.zhch.lan | 192.168.99.201 | Docker、cri-dockerd、kubelet、kubeadm、kubectl |
| k8s-node1.zhch.lan   | 192.168.99.204 | Docker、cri-dockerd、kubelet、kubeadm、kubectl |
| k82-node2.zhch.lan   | 192.168.99.205 | Docker、cri-dockerd、kubelet、kubeadm、kubectl |

### 安装 Docker（Master、Node）

* [01 Rocky Linux 安装 Docker.md](https://gitee.com/hczhch/docker---learning-notes/blob/master/01%20Rocky%20Linux%20%E5%AE%89%E8%A3%85%20Docker.md)

* Docker 在默认情况下使用的 cgroup driver 为 cgroupfs ，而 Kubernetes 推荐使用 systemd 来替代 cgroupfs

  ```shell
  [root@k8s-master1 ~]# vim /etc/docker/daemon.json 
  {
    "registry-mirrors": ["https://y5kpfirm.mirror.aliyuncs.com"],
    "exec-opts": ["native.cgroupdriver=systemd"]
  }
  [root@k8s-master1 ~]# systemctl daemon-reload
  [root@k8s-master1 ~]# systemctl restart docker
  ```
### 安装 cri-dockerd（Master、Node）

* https://github.com/Mirantis/cri-dockerd  

```shell
[zc@k8s-master1 ~]$ mkdir software
[zc@k8s-master1 ~]$ cd software
[zc@k8s-master1 software]$ wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.8/cri-dockerd-0.3.8.amd64.tgz
[zc@k8s-master1 software]$ tar -zxf cri-dockerd-0.3.8.amd64.tgz
[zc@k8s-master1 software]$ cd cri-dockerd
[root@k8s-master1 software]# install -o root -g root -m 0755 cri-dockerd /usr/local/bin/cri-dockerd
```
* https://github.com/Mirantis/cri-dockerd/blob/v0.3.8/packaging/systemd/cri-docker.service
* https://github.com/kubernetes/kubernetes/blob/v1.25.16/build/pause/Makefile
```shell
[root@k8s-master1 ~]# vim /etc/systemd/system/cri-docker.service
[Unit]
Description=CRI Interface for Docker Application Container Engine
Documentation=https://docs.mirantis.com
After=network-online.target firewalld.service docker.service
Wants=network-online.target
Requires=cri-docker.socket

[Service]
Type=notify
ExecStart=/usr/local/bin/cri-dockerd --network-plugin=cni --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.8 --container-runtime-endpoint fd://
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always

# Note that StartLimit* options were moved from "Service" to "Unit" in systemd 229.
# Both the old, and new location are accepted by systemd 229 and up, so using the old location
# to make them work for either version of systemd.
StartLimitBurst=3

# Note that StartLimitInterval was renamed to StartLimitIntervalSec in systemd 230.
# Both the old, and new name are accepted by systemd 230 and up, so using the old name to make
# this option work for either version of systemd.
StartLimitInterval=60s

# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity

# Comment TasksMax if your systemd version does not support it.
# Only systemd 226 and above support this option.
TasksMax=infinity
Delegate=yes
KillMode=process

[Install]
WantedBy=multi-user.target

```
* https://github.com/Mirantis/cri-dockerd/blob/v0.3.8/packaging/systemd/cri-docker.socket
```shell
[root@k8s-master1 ~]# vim /etc/systemd/system/cri-docker.socket
[Unit]
Description=CRI Docker Socket for the API
PartOf=cri-docker.service

[Socket]
ListenStream=%t/cri-dockerd.sock
SocketMode=0660
SocketUser=root
SocketGroup=docker

[Install]
WantedBy=sockets.target

```

```shell
# [root@k8s-master1 ~]# systemctl enable --now cri-docker.socket cri-docker.service
# Created symlink /etc/systemd/system/sockets.target.wants/cri-docker.socket → /etc/systemd/system/cri-docker.socket.
# Created symlink /etc/systemd/system/multi-user.target.wants/cri-docker.service → /etc/systemd/system/cri-docker.service.
[root@k8s-master1 ~]# systemctl enable --now cri-docker
Created symlink /etc/systemd/system/multi-user.target.wants/cri-docker.service → /etc/systemd/system/cri-docker.service.
```

### Hosts（Master、Node）

```shell
# 主机名解析：在内网自建DNS解析，或者编辑所有节点的 /etc/hosts文件，添加下面内容
[root@k8s-master1 ~]# vim /etc/hosts
192.168.99.201 k8s-master1.zhch.lan
192.168.99.204 k8s-node1.zhch.lan
192.168.99.205 k8s-node2.zhch.lan
```

### 时间同步（Master、Node）

```shell
# 查看时区
date -R

# 修改时区
timedatectl set-timezone Asia/Shanghai

# 安装 chrony
yum -y install chrony

# 修改配置文件
vim /etc/chrony.conf
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
# pool cn.ntp.org.cn iburst
# server ntp.aliyun.com iburst
pool ntp.ntsc.ac.cn iburst
server ntp.tencent.com iburst prefer minpoll 3 maxpoll 5
# prefer 指定优先使用的时间服务器
# maxpoll 3 ：轮询时间服务器的最小时间间隔是 2 的 3 次方

# 启动服务
systemctl start chronyd

# 设置开机启动
systemctl enable chronyd

# 查看时间同步源
chronyc sources -v

# 手动同步时间
chronyc makestep / chronyc -a makestep
```

### ~~开放端口~~

* https://kubernetes.io/zh-cn/docs/reference/networking/ports-and-protocols/

* Master

  ```shell
  [root@k8s-master1 ~]# firewall-cmd --add-port={6443,2379-2380,10250,10259,10257}/tcp --permanent
  [root@k8s-master1 ~]# firewall-cmd --reload
  ```

* Node

  ```shell
  [root@k8s-node1 ~]# firewall-cmd --add-port={10250,30000-32767}/tcp --permanent
  [root@k8s-node1 ~]# firewall-cmd --add-port=30000-32767/udp --permanent
  [root@k8s-node1 ~]# firewall-cmd --reload
  ```

### 关闭防火墙（Master、Node）

* 某些 Linux 发行版的默认防火墙规则可能会阻止 Kubernetes 集群内的通信。从 Kubernetes v1.19 开始，最好关闭 firewalld，因为它与 Kubernetes 网络插件冲突。

* 我尝试不关闭 firewalld ，而是设置放行规则，在使用 kubernetes 过程中遇到了很多麻烦


```shell
[root@k8s-master1 ~]# systemctl stop firewalld
[root@k8s-master1 ~]# systemctl disable firewalld
Removed "/etc/systemd/system/multi-user.target.wants/firewalld.service".
Removed "/etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service".
```

### 禁用交换分区（Master、Node）

```shell
# 临时禁用交换分区
[root@k8s-node1 ~]# swapoff -a
# 永久禁用交换分区，注释掉 /etc/fstab 中加载 swap 分区的行
[root@k8s-node1 ~]# vim /etc/fstab
/dev/mapper/almalinux-root /                       xfs     defaults        0 0
UUID=3068f70c-73d5-40a4-8e69-05901ec13170 /boot                   xfs     defaults        0 0
#/dev/mapper/almalinux-swap none                    swap    defaults        0 0
```

### 将 SELinux 设置为 permissive 模式（相当于将其禁用）（Master、Node）

```shell
[root@k8s-master1 ~]# setenforce 0
[root@k8s-master1 ~]# sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

### 修改linux的内核参数，添加网桥过滤和地址转发功能（Master、Node）

```shell
[root@k8s-master1 ~]# vim /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
[root@k8s-master1 ~]# sysctl -p /etc/sysctl.d/kubernetes.conf

# 加载网桥过滤模块
[root@k8s-master1 ~]# modprobe br_netfilter
# 查看网桥过滤模块是否加载成功
[root@k8s-master1 ~]# lsmod | grep br_netfilter
br_netfilter           36864  0
bridge                405504  1 br_netfilter

```

### 配置 ipvs 功能（Master、Node）
* 在 Kubernetes 中 Service 有两种模型，一种是基于 iptables，一种是基于 ipvs。两者比较，ipvs 的性能明显要高一些，但是如果要使用它，需要手动载入 ipvs 模块

```shell
# 安装 ipset 和 ipvsadm
[root@k8s-master1 ~]# yum install ipset ipvsadm -y
# 添加需要加载的模块
[root@k8s-master1 ~]# mkdir -p /etc/sysconfig/modules
[root@k8s-master1 ~]# vim /etc/sysconfig/modules/ipvs.modules
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack

# use nf_conntrack instead of nf_conntrack_ipv4 for Linux kernel 4.19 and later

[root@k8s-master1 ~]# chmod +x /etc/sysconfig/modules/ipvs.modules
[root@k8s-master1 ~]# /bin/bash /etc/sysconfig/modules/ipvs.modules
# 查看对应的模块是否加载成功
[root@k8s-master1 ~]# lsmod | grep -e ip_vs -e nf_conntrack
```
### 安装 kubelet，kubeadm，kubectl（Master、Node）
```shell
[root@k8s-node2 ~]# vim /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg 
       https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
       
[root@k8s-node2 ~]# yum clean all && yum makecache
```

```shell
# 原计划安装 1.25.16，可是安装的时候发现仓库里 1.25 的最新版是 1.25.14
# 事实上，github 上早已发布 1.25.16 版本

# 查询可安装的版本
[root@k8s-master1 ~]# yum list kubeadm --showduplicates | sort -r
# 安装
[root@k8s-master1 ~]# yum install -y kubelet-1.25.14 kubeadm-1.25.14 kubectl-1.25.14 --disableexcludes=kubernetes

[root@k8s-master1 ~]# vim /etc/sysconfig/kubelet
KUBELET_CGROUP_ARGS="--cgroup-driver=systemd"
KUBE_PROXY_MODE="ipvs"

[root@k8s-master1 ~]# systemctl enable kubelet
```

### 集群初始化（Master）

```shell
# --apiserver-advertise-address 只能使用 ipV4 或者 ipV6 地址，不能使用 hostname
[root@k8s-master1 ~]# kubeadm init \
    --apiserver-advertise-address=192.168.99.201 \
    --image-repository registry.aliyuncs.com/google_containers \
    --kubernetes-version=v1.25.14 \
    --service-cidr=10.96.0.0/12 \
    --pod-network-cidr=10.244.0.0/16 \
    --cri-socket unix:///var/run/cri-dockerd.sock
[init] Using Kubernetes version: v1.25.14
......
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.99.201:6443 --token zaq4bk.qwozx47isxrfl104 \
	--discovery-token-ca-cert-hash sha256:df9157315bf4755155dddd1932be5883331b30d458a42cf0d931c5f94113adfa
```

```shell
[root@k8s-master1 ~]# mkdir -p $HOME/.kube
[root@k8s-master1 ~]# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@k8s-master1 ~]# sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
### 网络插件
* NetworkManager（Master、Node）
* https://docs.tigera.io/calico/latest/operations/troubleshoot/troubleshooting#configure-networkmanager
```shell
# 许多 Linux 发行版都包含 NetworkManager。 默认情况下，NetworkManager 不允许 Calico 管理接口。 如果您的节点具有 NetworkManager，在安装 Calico 之前请完成以下步骤，防止 NetworkManager 控制 Calico 接口。
[root@k8s-master1 ~]# vim /etc/NetworkManager/conf.d/calico.conf
[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:tunl*;interface-name:vxlan.calico;interface-name:vxlan-v6.calico;interface-name:wireguard.cali;interface-name:wg-v6.cali
 
[root@k8s-master1 ~]# systemctl restart NetworkManager
```
* 安装 calico（Master）
```shell
[root@k8s-master1 ~]# mkdir /opt/kubernetes
[root@k8s-master1 ~]# cd /opt/kubernetes
[root@k8s-master1 kubernetes]# wget https://docs.projectcalico.org/manifests/calico.yaml

# 修改内容1：IP_AUTODETECTION_METHOD ， 目前发现不修改 IP_AUTODETECTION_METHOD 也可以，可能是跟我本地网卡的实际情况有关
# 修改内容2：CALICO_IPV4POOL_CIDR
[root@k8s-master1 kubernetes]# vim calico.yaml
            - name: CLUSTER_TYPE
              value: "k8s,bgp"
            - name: IP_AUTODETECTION_METHOD
              value: "interface=ens.*"

            # The default IPv4 pool to create on startup if none exists. Pod IPs will be
            # chosen from this range. Changing this value after installation will have
            # no effect. This should fall within `--cluster-cidr`.
            - name: CALICO_IPV4POOL_CIDR
              value: "10.244.0.0/16"

[root@k8s-master1 kubernetes]# kubectl apply -f calico.yaml
```

```shell
# 重新安装 clico
[root@k8s-master1 kubernetes]# kubectl delete -f calico.yaml
[root@k8s-master1 kubernetes]# rm -rf /var/lib/cni
[root@k8s-master1 kubernetes]# kubectl apply -f calico.yaml
```

* ~~防火墙（Master、Node）~~
  * https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements#network-requirements
```shell
# Calico networking (BGP)
# Calico networking with Typha enabled
# flannel networking (VXLAN)
[root@k8s-master1 ~]# firewall-cmd --zone=public --add-port=179/tcp --permanent
[root@k8s-master1 ~]# firewall-cmd --zone=public --add-port=5473/tcp --permanent
[root@k8s-master1 ~]# firewall-cmd --zone=public --add-port=4789/udp --permanent
[root@k8s-master1 ~]# firewall-cmd --reload
```

### Node 加入集群（Node）

```shell
[root@k8s-node1 ~]# kubeadm join 192.168.99.201:6443 --token zaq4bk.qwozx47isxrfl104 \
	--discovery-token-ca-cert-hash sha256:df9157315bf4755155dddd1932be5883331b30d458a42cf0d931c5f94113adfa \
	--cri-socket unix:///var/run/cri-dockerd.sock
```

```shell
[root@k8s-master1 kubernetes]# kubectl get pods -n kube-system
NAME                                           READY   STATUS    RESTARTS      AGE
calico-kube-controllers-74cfc9ffcc-f7w46       1/1     Running   0             2m50s
calico-node-kprx7                              1/1     Running   0             2m50s
calico-node-kptmd                              1/1     Running   0             2m50s
calico-node-r5pzf                              1/1     Running   0             2m50s
coredns-c676cc86f-2nrv2                        1/1     Running   1 (47m ago)   156m
coredns-c676cc86f-pnd8t                        1/1     Running   1 (47m ago)   156m
etcd-k8s-master1.zhch.lan                      1/1     Running   1 (47m ago)   156m
kube-apiserver-k8s-master1.zhch.lan            1/1     Running   1 (47m ago)   156m
kube-controller-manager-k8s-master1.zhch.lan   1/1     Running   1 (47m ago)   156m
kube-proxy-4skmx                               1/1     Running   1 (47m ago)   120m
kube-proxy-jszfp                               1/1     Running   2 (46m ago)   119m
kube-proxy-z4x7h                               1/1     Running   1 (47m ago)   156m
kube-scheduler-k8s-master1.zhch.lan            1/1     Running   1 (47m ago)   156m

# 查看 pod 详情
[root@k8s-master1 ~]# kubectl describe pod PodName -n kube-system

[root@k8s-master1 kubernetes]# kubectl get node
NAME                   STATUS   ROLES           AGE    VERSION
k8s-master1.zhch.lan   Ready    control-plane   155m   v1.25.14
k8s-node1.zhch.lan     Ready    <none>          119m   v1.25.14
k8s-node2.zhch.lan     Ready    <none>          118m   v1.25.14
```

* 忘记 `token` 怎么办

```shell
# 查看所有 token
[root@k8s-master1 ~]# kubeadm token list
# 如果 token 都过期了，则创建新的 token
[root@k8s-master1 ~]# kubeadm token create
```

* 忘记 `hash` 怎么办
  
```shell
[root@k8s-master1 ~]# openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
df9157315bf4755155dddd1932be5883331b30d458a42cf0d931c5f94113adfa
```


### 测试

```shell
[root@k8s-master1 test]# kubectl create deployment nginx --image=nginx:1.14-alpine
deployment.apps/nginx created
[root@k8s-master1 test]# kubectl expose deploy nginx --port=80 --target-port=80 --type=NodePort
service/nginx exposed

[root@k8s-master1 test]# kubectl get svc
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        10h
service/nginx        NodePort    10.108.160.86   <none>        80:31824/TCP   7m52s

[root@k8s-master1 test]# kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP              NODE                 NOMINATED NODE   READINESS GATES
nginx-6bcdf89b89-5hcxf   1/1     Running   0          10m   10.244.37.193   k8s-node1.zhch.lan   <none>           <none>

# 验证是否可以正常访问：
#   http://k8s-master1.zhch.lan:31824
#   http://k8s-node1.zhch.lan:31824
#   http://k8s-node2.zhch.lan:31824

# 验证后删除
[root@k8s-master1 test]# kubectl delete deployment nginx
deployment.apps "nginx" deleted
[root@k8s-master1 test]# kubectl delete service nginx
service "nginx" deleted
```

### 让 Node 节点也能使用 kubectl

```shell
[root@k8s-master1 ~]# scp /root/.kube/config root@k8s-node1.zhch.lan:/root/.kube/

[root@k8s-master1 ~]# scp /root/.kube/config root@k8s-node2.zhch.lan:/root/.kube/
```



### Kubernetes 国内镜像仓库

* https://github.com/lank8s
  * `registry.k8s.io`  ->   `lank8s.cn`

* `quay.io`  ->  `quay.mirrors.ustc.edu.cn`
