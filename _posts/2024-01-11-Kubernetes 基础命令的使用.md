---
layout:       post
title:        "Kubernetes 基础命令的使用"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - Kubernetes
---

### kubectl 命令官方文档

* <https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands>
* <https://v1-25.docs.kubernetes.io/zh-cn/docs/reference/kubectl/>

### kubectl 资源类型与缩写

| 资源类型                   | 缩写别名 |
| :------------------------- | :------- |
| `clusters`                 |          |
| `componentstatuses`        | `cs`     |
| `configmaps`               | `cm`     |
| `daemonsets`               | `ds`     |
| `deployments`              | `deploy` |
| `endpoints`                | `ep`     |
| `event`                    | `ev`     |
| `horizontalpodautoscalers` | `hpa`    |
| `ingresses`                | `ing`    |
| `jobs`                     |          |
| `limitranges`              | `limits` |
| `namespaces`               | `ns`     |
| `networkpolicies`          |          |
| `nodes`                    | `no`     |
| `statefulsets`             | `sts`    |
| `persistentvolumeclaims`   | `pvc`    |
| `persistentvolumes`        | `pv`     |
| `pods`                     | `po`     |
| `podsecuritypolicies`      | `psp`    |
| `podtemplates`             |          |
| `replicasets`              | `rs`     |
| `replicationcontrollers`   | `rc`     |
| `resourcequotas`           | `quota`  |
| `cronjob`                  |          |
| `secrets`                  |          |
| `serviceaccount`           | `sa`     |
| `services`                 | `svc`    |
| `storageclasses`           |          |
| `thirdpartyresources`      |          |

### 已弃用 API 的迁移指南

* <https://kubernetes.io/zh-cn/docs/reference/using-api/deprecation-guide/>

