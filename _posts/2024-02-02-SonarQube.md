---
layout:       post
title:        "Kubernetes SonarQube"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - Kubernetes
    - SonarQube

---

### Install

```yaml
apiVersion: v1
kind: Service 
metadata:
  name: sonar-postgres
  namespace: devops
spec:
  selector: 
    app: sonar-postgres
  type: ClusterIP
  ports:
  - port: 5432 
    targetPort: 5432
    protocol: TCP
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sonar-postgres
  namespace: devops
  labels:
    app: sonar-postgres
spec:
  serviceName: "sonar-postgres"
  replicas: 1
  selector:
    matchLabels:
      app: sonar-postgres
  template:
    metadata:
      labels:
        app: sonar-postgres
    spec:
      containers:
      - name: sonar-postgres
        image: postgres:16.1-alpine3.19
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: "sonar"
        - name: POSTGRES_USER
          value: "sonar"
        - name: POSTGRES_PASSWORD 
          value: "59m5p>@wYe"
        - name: ALLOW_IP_RANGE
          value: "0.0.0.0/0"
        volumeMounts:
          - name: sonar-postgres
            mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: sonar-postgres
      labels:
        app: sonar-postgres
    spec:
      accessModes: [ "ReadWriteOnce" ]
      #storageClassName: "managed-nfs-storage"
      resources:
        requests:
          storage: 2Gi
---
apiVersion: v1
kind: Service 
metadata:
  name: sonarqube
  namespace: devops
spec:
  selector: 
    app: sonarqube
  type: ClusterIP
  ports:
  - port: 9000 
    targetPort: 9000
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sonarqube
  namespace: devops
  labels:
    app: sonarqube
spec:
  serviceName: "sonarqube"
  replicas: 1
  selector:
    matchLabels:
      app: sonarqube
  template:
    metadata:
      labels:
        app: sonarqube
    spec:
      initContainers:
      - name: init-sysctl
        image: busybox
        imagePullPolicy: IfNotPresent
        #command: ["sysctl", "-w", "vm.max_map_count=524288"]
        #command: [ "/bin/sh", "-c", "--" ]
        #args: [ "sysctl -w vm.max_map_count=524288; sysctl -w fs.file-max=131072; ulimit -n 131072; ulimit -u 8192;" ]
        # ulimit -n 131072; ulimit -u 8192; 只在当前会话有效，因此从命令中去除了
        command: [ "/bin/sh", "-c", "--" ]
        args: [ "sysctl -w vm.max_map_count=524288; sysctl -w fs.file-max=131072;" ]
        securityContext:
          privileged: true
      containers:
      - name: sonarqube
        image: sonarqube:9.9.3-community
        ports:
        - containerPort: 9000
        env:
        - name: SONAR_JDBC_USERNAME
          value: "sonar"
        - name: SONAR_JDBC_PASSWORD
          value: "59m5p>@wYe"
        - name: SONAR_JDBC_URL
          value: "jdbc:postgresql://sonar-postgres:5432/sonar"
        livenessProbe:
          httpGet:
            path: /sessions/new
            port: 9000
          initialDelaySeconds: 60
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /sessions/new
            port: 9000
          initialDelaySeconds: 60
          periodSeconds: 30
          failureThreshold: 6
        resources: 
          requests: 
            memory: 512Mi
            cpu: 10m
          limits: 
            memory: 1.5Gi
            cpu: 1000m
        volumeMounts:
        - name: sonarqube
          mountPath: /opt/sonarqube/data          
          subPath: data
        - name: sonarqube
          mountPath: /opt/sonarqube/extensions
          subPath: extensions
  volumeClaimTemplates:
  - metadata:
      name: sonarqube
      labels:
        app: sonarqube
    spec:
      accessModes: [ "ReadWriteOnce" ]
      #storageClassName: "managed-nfs-storage"
      resources:
        requests:
          storage: 2Gi
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sonarqube
  namespace: devops
spec: 
  ingressClassName: nginx
  tls:
  - hosts:
    - sonar.zhch.lan
    secretName: zhch.lan
  rules:
  - host: sonar.zhch.lan
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: sonarqube
            port:
              number: 9000
```

### 使用

* https://sonar.zhch.lan/    admin / admin

* 安装中文插件：Administration -> Marketplace  ,  点击 'I understand the risk' ， 搜索 Chinese ，安装插件
* 创建 token：我的账号 -> 安全 -> 创建令牌
  * 创建好后立即复制，该令牌不会显示第二次，本例创建的令牌是：`sqa_3cf5259d5c4db59ba29de9f15f42e6a18f5545c8`
