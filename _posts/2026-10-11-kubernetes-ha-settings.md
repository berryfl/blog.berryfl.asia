---
layout: post
title: "Kubernetes 应用高可用设置"
date: 2026-10-11
---

![Kubernetes 应用高可用设置](/assets/kubernetes-ha.png)

Kubernetes 是一个相对动态的环境：Cluster Autoscaler 会根据集群负载动态增删节点，云厂商也会主动升级控制面和节点池，节点或整个可用区有时还会突然不可用。这些情况都可能导致应用副本对应的 Pod 被驱逐（evict）或变得不可用。因此，在生产环境中配置适当的高可用策略是必要的。本文主要讨论无状态多副本应用（例如 Web 后端服务）的常见设置。

---

## 多副本设置

副本数是最基本的可用性设置，一般来说至少应设为 2。这样异常出现时，往往只会影响其中一个副本。多种不可用情况偶尔也会叠加发生，适当增大副本数可以进一步降低这类故障带来的影响。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: awxseek
  namespace: dba
spec:
  replicas: 2
  # skip other fields ...
```

这里有一个常见的坑：有些应用启用了 HPA（HorizontalPodAutoscaler），却使用了默认的最小副本数 1（例如 auto-deploy-app）。即便 Deployment 或 StatefulSet 本身设置了大于 1 的副本数，HPA 也可能在低负载时把副本数缩到 1。

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: awxseek-auto-deploy
  namespace: dba
spec:
  maxReplicas: 5
  minReplicas: 2
  # skip other fields ...
```

## PDB 设置

有些情况下 Kubernetes 集群会主动驱逐节点上的 Pod（例如上面提到的 Cluster Autoscaler，或者云厂商升级）。PDB（PodDisruptionBudget）用于约束这类主动驱逐的行为。最宽松的设置是 `minAvailable: 1`，极端情况下也至少保留一个副本；最保守的设置则是 `maxUnavailable: 1`，也就是最多只允许 1 个 Pod 不可用。

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: awxseek-auto-deploy
  namespace: dba
spec:
  maxUnavailable: 1
  # skip other fields ...
```

这里也有一个常见的坑：应用副本数设置为 1，同时 PDB 又设置 `minAvailable: 1`，则 Pod 永远无法被驱逐，可能导致节点升级进程卡住。

## 分散节点及可用区

Kubernetes 调度器有可能把多个副本的 Pod 调度到同一个节点或可用区。这样当节点或可用区不可用时，多个副本会同时不可用。`topologySpreadConstraints` 就是用来应对这种场景的。云厂商通常会自动给节点打上拓扑相关的 label（例如 GKE 节点会带上 hostname 和 zone 的 label），结合 `topologySpreadConstraints`，就可以把副本分散到不同节点或可用区。

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    metadata:
      labels:
        app: awxseek
    spec:
      topologySpreadConstraints:
        # 硬约束：必须打散到不同节点
        - maxSkew: 1
          topologyKey: "kubernetes.io/hostname"
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: awxseek
          matchLabelKeys:
            - pod-template-hash
          # skip other fields ...
        # 软约束：尽量打散到不同可用区
        - maxSkew: 1
          topologyKey: "topology.kubernetes.io/zone"
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: awxseek
          matchLabelKeys:
            - pod-template-hash
          # skip other fields ...
```

`whenUnsatisfiable` 有两个可选值。`DoNotSchedule` 会在不满足条件时让 Pod 处于 Pending，通常由此触发 Cluster Autoscaler 创建新节点，直到满足条件；`ScheduleAnyway` 则是尽力满足，相当于软约束。

`topologySpreadConstraints` 也有一个需要注意的地方：应用升级时会滚动更新，前后两个版本的 Pod 会匹配相同的 label，从而互相干扰，最终 Pod 可能无法完美地分散。如果应用使用 Deployment 部署，可以在 `matchLabelKeys` 中指定 `pod-template-hash` 这个 label。应用升级时 `pod-template-hash` 总会改变，因此前后版本不会互相干扰。

这里再简单提一下反亲和性（`antiAffinity`）。它与 `topologySpreadConstraints` 功能有些相似，容易混淆。`antiAffinity` 的主要效果是排它，例如实现一个节点上最多只有一个副本，特别适用于 etcd、Elasticsearch 这类通过共识协议组成集群的应用：严格排它能够确保节点不可用时，不会失去大多数成员，从而维持可用性。而 `topologySpreadConstraints` 更强调分散，当副本数多于节点数时，一个节点上可以存在 2 个或更多副本。

## 小结

Kubernetes 相对动态的环境要求应用具备一定的高可用能力。本文列举了几种生产环境中常用的设置，以及它们各自需要注意的地方。
