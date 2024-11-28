

### Namespace

```shell
[root@k8s-master1 ~]# kubectl create namespace devops
namespace/devops created
```

### Secret

```shell
[root@k8s-master1 ~]# kubectl create secret tls zhch.lan -n devops \
                      --cert=/home/zc/cert/zhch.lan.crt \
                      --key=/home/zc/cert/zhch.lan.key
secret/zhch.lan created
```
### ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gitlab-config
  namespace: devops
data:
  gitlab.rb: |
    external_url 'https://git.zhch.lan'
    nginx['listen_port'] = 8180
    nginx['listen_https'] = false
    gitlab_rails['time_zone'] = 'Asia/Shanghai'
    gitlab_rails['smtp_enable'] = true
    gitlab_rails['smtp_address'] = "smtp.163.com"
    gitlab_rails['smtp_port'] = 465
    gitlab_rails['smtp_user_name'] = "nest0321@163.com"
    gitlab_rails['smtp_password'] = "********"
    gitlab_rails['smtp_domain'] = "163.com"
    gitlab_rails['smtp_authentication'] = "login"
    gitlab_rails['smtp_enable_starttls_auto'] = false
    gitlab_rails['smtp_tls'] = true
    gitlab_rails['smtp_pool'] = false
    gitlab_rails['gitlab_email_from'] = "nest0321@163.com"
    user["git_user_email"] = "nest0321@163.com"
    postgresql['shared_buffers'] = "128MB"
    prometheus_monitoring['enable'] = false
```

### Service

```yaml
apiVersion: v1
kind: Service 
metadata:
  name: gitlab
  namespace: devops
spec:
  selector: 
    app: gitlab-ce
  type: ClusterIP
  ports:
  - port: 80 
    targetPort: web
    protocol: TCP
```

### StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: gitlab-ce
  namespace: devops
spec:
  serviceName: "gitlab"
  replicas: 1
  selector:
    matchLabels: 
      app: gitlab-ce
  template:
    metadata:
      labels:
        app: gitlab-ce
    spec:
      containers:
      - image: gitlab/gitlab-ce:16.7.0-ce.0
        imagePullPolicy: IfNotPresent
        name: gitlab-ce
        ports:
        - containerPort: 8180
          name: web
          protocol: TCP
        env:
        - name: TZ
          value: 'Asia/Shanghai'
        volumeMounts:
        - name: data
          mountPath: /var/opt/gitlab
        - name: config
          mountPath: /etc/gitlab
        - name: gitlab-rb
          mountPath: /etc/gitlab/gitlab.rb
          subPath: gitlab.rb        
      restartPolicy: Always
      volumes:
        - name: gitlab-rb
          configMap: 
            name: gitlab-config
            items: 
            - key: gitlab.rb
              path: gitlab.rb
  volumeClaimTemplates:
  - metadata:
      name: config
      labels:
        app: gitlab-ce
    spec:
      accessModes: [ "ReadWriteOnce" ]
      #storageClassName: "managed-nfs-storage" # 使用集群默认的 StorageClass
      resources:
        requests:
          storage: 100Mi
  - metadata:
      name: data
      labels:
        app: gitlab-ce
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 2Gi
```

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gitlab
  namespace: devops
spec: 
  ingressClassName: nginx
  tls:
  - hosts:
    - git.zhch.lan
    secretName: zhch.lan
  rules:
  - host: git.zhch.lan
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gitlab
            port:
              number: 80
```

### 初始密码

* 用户名 `root`
* 初始密码：进入容器内查看文件 `/etc/gitlab/initial_root_password` ，或者进入挂载的文件服务器上去查看

```shell
[root@k8s-master1 devops-config-gitlab-ce-0-pvc-f309c435-700b-47b2-8bb4-bce0d4690904]# pwd
/root/nfs_data/rw/default/devops-config-gitlab-ce-0-pvc-f309c435-700b-47b2-8bb4-bce0d4690904
[root@k8s-master1 devops-config-gitlab-ce-0-pvc-f309c435-700b-47b2-8bb4-bce0d4690904]# cat initial_root_password 
# WARNING: This value is valid only in the following conditions
#          1. If provided manually (either via `GITLAB_ROOT_PASSWORD` environment variable or via `gitlab_rails['initial_root_password']` setting in `gitlab.rb`, it was provided before database was seeded for the first time (usually, the first reconfigure run).
#          2. Password hasn't been changed manually, either via UI or via command line.
#
#          If the password shown here doesn't work, you must reset the admin password following https://docs.gitlab.com/ee/security/reset_user_password.html#reset-your-root-password.

Password: EUqojCYiLsIaZzyZwQgWI0BvfN0nrVmxsKWhe40bg+U=

# NOTE: This file will be automatically deleted in the first reconfigure run after 24 hours.
```

### SSH 22

* https://kubernetes.github.io/ingress-nginx/user-guide/exposing-tcp-udp-services/

* StatefulSet 新增暴露 22 端口

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: gitlab-ce
  namespace: devops
spec:
  serviceName: "gitlab"
  replicas: 1
  selector:
    matchLabels: 
      app: gitlab-ce
  template:
    metadata:
      labels:
        app: gitlab-ce
    spec:
      containers:
      - image: gitlab/gitlab-ce:16.7.0-ce.0
        imagePullPolicy: IfNotPresent
        name: gitlab-ce
        ports:
        - containerPort: 8180
          name: web
          protocol: TCP
        - containerPort: 22 # 新增
          name: ssh
          protocol: TCP
        env:
        - name: TZ
          value: 'Asia/Shanghai'
        volumeMounts:
        - name: data
          mountPath: /var/opt/gitlab
        - name: config
          mountPath: /etc/gitlab
        - name: gitlab-rb
          mountPath: /etc/gitlab/gitlab.rb
          subPath: gitlab.rb        
      restartPolicy: Always
      volumes:
        - name: gitlab-rb
          configMap: 
            name: gitlab-config
            items: 
            - key: gitlab.rb
              path: gitlab.rb
  volumeClaimTemplates:
  - metadata:
      name: config
      labels:
        app: gitlab-ce
    spec:
      accessModes: [ "ReadWriteOnce" ]
      #storageClassName: "managed-nfs-storage" # 使用集群默认的 StorageClass
      resources:
        requests:
          storage: 100Mi
  - metadata:
      name: data
      labels:
        app: gitlab-ce
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 2Gi    
```

* Service 新增 22 端口

```yaml
apiVersion: v1
kind: Service 
metadata:
  name: gitlab
  namespace: devops
spec:
  selector: 
    app: gitlab-ce
  type: ClusterIP
  ports:
  - port: 80
    targetPort: web
    protocol: TCP
    name: web
  - port: 22 # 新增
    targetPort: ssh
    protocol: TCP
    name: ssh
```

* 新建 ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingress-tcp
  namespace: ingress-nginx
data:
  22: "devops/gitlab:22" # namespace/service:port
```

* `kubectl edit service/ingress-nginx-controller -n ingress-nginx`


```shell
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
  - name: gitlab-ssh # 新增
    port: 22
    protocol: TCP
    targetPort: 22
```

* `kubectl edit deploy/ingress-nginx-controller -n ingress-nginx`

```shell
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
```

* 测试 ssh

```shell
# 推荐使用 ED25519、RSA；选择 RSA 时，推荐的 key size 是 3072 或更大
[zc@Test ~]$ ssh-keygen -t ed25519 -C "hczhch@ymail.com"
# 私钥 ~/.ssh/id_ed25519  公钥：~/.ssh/id_ed25519.pub

[zc@Test ~]$ ssh-keygen -t rsa -b 4096 -C "hczhch@ymail.com"
# 私钥 ~/.ssh/id_rsa  公钥：~/.ssh/id_rsa.pub

# 把公钥添加到 Gitlab 中

# 如果 ~/.ssh/known_hosts 文件中存在旧的 git.zhch.lan host 记录，则先删除

# git clone ...
```
