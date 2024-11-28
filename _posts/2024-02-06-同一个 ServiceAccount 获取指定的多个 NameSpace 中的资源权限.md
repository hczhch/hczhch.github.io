---
layout:       post
title:        "Kubernetes 同一个 ServiceAccount 管理指定的多个 NameSpace 中的资源"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - Kubernetes
    - ServiceAccount

---


### 同一个 ServiceAccount 管理指定的多个 NameSpace 中的资源

* 普通情况下是：
  * ServiceAccount 通过 RoleBinding 与所在 namespace 中的 Role 进行绑定，获取所在 namespace 中的资源权限
  * ServiceAccount 通过 ClusterRoleBinding 与 ClusterRole 进行绑定，获取集群内的资源权限（不限定 namespace ）




* 而实现”同一个 ServiceAccount 获取指定的多个 NameSpace 中的资源权限“的做法是：

  * 同一个 **ServiceAccount** 通过 **RoleBinding** 在不同的 **namespace** 中绑定 **ClusterRole**，从而获取该  **namespace** 中的资源权限

  * **ClusterRole** 可以是同一个 ClusterRole，也可以是不同的 ClusterRole

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fdr-developer
  namespace: default
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: developer
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create", "delete", "get", "list", "patch", "update", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding # 注意这里不是 ClusterRoleBinding
metadata:
  name: fdr-developer-dev-fdr
  namespace: dev-fdr # ServiceAccount[fdr-developer] 在 namespace[dev-fdr] 内具有 ClusterRole[developer] 定义的权限
subjects:
  - kind: ServiceAccount
    name: fdr-developer
    namespace: default
roleRef:
  kind: ClusterRole # 这里是 ClusterRole
  name: developer
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: fdr-developer-stag-fdr
  namespace: stag-fdr
subjects:
  - kind: ServiceAccount
    name: fdr-developer
    namespace: default
roleRef:
  kind: ClusterRole
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

* 验证

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-auth-test
  namespace: default
spec:
  serviceAccountName: fdr-developer
  containers:
  - name: pod-auth-test
    image: alpine:3.16
    imagePullPolicy: IfNotPresent
    command: ["sh","-c","sleep 3600"]
    volumeMounts:
      - name: kubectl
        mountPath: /usr/bin/kubectl
        readOnly: true
  volumes:
    - name: kubectl
      hostPath:
        path: /usr/bin/kubectl
```

```shell
[root@k8s-master1 ~]# kubectl exec -it pod-auth-test -- sh
/ # kubectl get po -n dev-fdr
No resources found in dev-fdr namespace.
/ # kubectl get po -n stag-fdr
No resources found in stag-fdr namespace.
/ # kubectl get po -n devops
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:default:fdr-developer" cannot list resource "pods" in API group "" in the namespace "devops"
/ # kubectl get po
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:default:fdr-developer" cannot list resource "pods" in API group "" in the namespace "default"
/ # 
```

