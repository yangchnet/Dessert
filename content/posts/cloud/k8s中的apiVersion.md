---
author: "李昌"
title: "k8s中的apiVersion"
date: "2021-09-02"
tags: ["kubernetes"]
categories: ["Kubernetes"]
ShowToc: true
TocOpen: true
---

### apiVersion可能的字段值：

| Kind                      | apiVersion                   |
| ------------------------- | ---------------------------- |
| CertificateSigningRequest | certificates.k8s.io/v1beta1  |
| ClusterRoleBinding        | rbac.authorization.k8s.io/v1 |
| ClusterRole               | rbac.authorization.k8s.io/v1 |
| ComponentStatus           | v1                           |
| ConfigMap                 | v1                           |
| ControllerRevision        | apps/v1                      |
| CronJob                   | batch/v1beta1                |
| DaemonSet                 | extensions/v1beta1           |
| Deployment                | extensions/v1beta1           |
| Endpoints                 | v1                           |
| Event                     | v1                           |
| HorizontalPodAutoscaler   | autoscaling/v1               |
| Ingress                   | extensions/v1beta1           |
| Job                       | batch/v1                     |
| LimitRange                | v1                           |
| Namespace                 | v1                           |
| NetworkPolicy             | extensions/v1beta1           |
| Node                      | v1                           |
| PersistentVolumeClaim     | v1                           |
| PersistentVolume          | v1                           |
| PodDisruptionBudget       | policy/v1beta1               |
| Pod                       | v1                           |
| PodSecurityPolicy         | extensions/v1beta1           |
| PodTemplate               | v1                           |
| ReplicaSet                | extensions/v1beta1           |
| ReplicationController     | v1                           |
| ResourceQuota             | v1                           |
| RoleBinding               | rbac.authorization.k8s.io/v1 |
| Role                      | rbac.authorization.k8s.io/v1 |
| Secret                    | v1                           |
| ServiceAccount            | v1                           |
| Service                   | v1                           |
| StatefulSet               | apps/v1                      |

### 每个字段的意义

**alpha**  
这个apiVersion是早期候选版本，可能包含一些bug

**beta**  
已经过alpha版本的使用，未来将会真正包含到kubernetes中，但其工作方式可能会发生一些改变

**stable**   
可以安全使用的版本


**v1**  
kubernetes第一个稳定的发布版本，包含许多核心对象

**apps/v1**  
`apps`是kubernetes中最常见的APi组，许多核心对象都是从这里和`v1`中提取出来的。它包括一些与kubernetes运行相关的功能，比如：Deployments, RollingUpdates, ReplicaSet等

**autoscaling/v1**  
这个APi版本允许pods根据不同的资源使用指标自动缩放，

**batch/v1**   
`batch`API组包含与批处理及类似作业任务相关的对象。

**batch/v1beta1**  
batch功能对象的beta版本，包含CronJob（轮询任务）

**extensions/v1beta1**  
此版本的 API 包括 Kubernetes 的许多新的、常用的功能。 Deployments、DaemonSets、ReplicaSets 和 Ingresses 在此版本中都发生了重大变化。

请注意，在 Kubernetes 1.6 中，其中一些对象从扩展重新定位到特定 API 组（例如应用程序）。 当这些对象退出 Beta 版时，希望它们位于特定的 API 组中，例如 apps/v1。 使用 extensions/v1beta1 已被弃用——根据您的 Kubernetes 集群版本，尽可能尝试使用特定的 API 组。

**policy/v1beta1**  
此 apiVersion 增加了设置 pod 中断预算和有关 pod 安全性的新规则的功能。

**rbac.authorization.k8s.io/v1**
此 apiVersion 包括基于 Kubernetes 角色的访问控制的额外功能。 这有助于您保护集群

### 查看可用的apiVersion
```sh
kubectl api-versions
```

```
admissionregistration.k8s.io/v1
admissionregistration.k8s.io/v1beta1
apiextensions.k8s.io/v1
apiextensions.k8s.io/v1beta1
apiregistration.k8s.io/v1
apiregistration.k8s.io/v1beta1
apps/v1
authentication.k8s.io/v1
authentication.k8s.io/v1beta1
authorization.k8s.io/v1
authorization.k8s.io/v1beta1
autoscaling/v1
autoscaling/v2beta1
autoscaling/v2beta2
batch/v1
batch/v1beta1
certificates.k8s.io/v1
certificates.k8s.io/v1beta1
coordination.k8s.io/v1
coordination.k8s.io/v1beta1
discovery.k8s.io/v1
discovery.k8s.io/v1beta1
events.k8s.io/v1
events.k8s.io/v1beta1
extensions/v1beta1
flowcontrol.apiserver.k8s.io/v1beta1
networking.k8s.io/v1
networking.k8s.io/v1beta1
node.k8s.io/v1
node.k8s.io/v1beta1
policy/v1
policy/v1beta1
rbac.authorization.k8s.io/v1
rbac.authorization.k8s.io/v1beta1
scheduling.k8s.io/v1
scheduling.k8s.io/v1beta1
storage.k8s.io/v1
storage.k8s.io/v1beta1
v1
```


### Translated from:   
https://matthewpalmer.net/kubernetes-app-developer/articles/kubernetes-apiversion-definition-guide.html
