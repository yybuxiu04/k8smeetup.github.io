---
cn-approvers:
- xiaosuiba
cn-reviewers:
- zjj2wry
title: 限制存储使用量
---



{% capture overview %}


本示例展示了一种在 namespace 中限制存储使用量总和的简便方法。

演示使用了下列资源：[ResourceQuota](/docs/concepts/policy/resource-quotas/)、[LimitRange](/docs/tasks/administer-cluster/memory-default-namespace/) 和 [PersistentVolumeClaim](/docs/concepts/storage/persistent-volumes/)。

{% endcapture %}

{% capture prerequisites %}

* {% include task-tutorial-prereqs.md %}

{% endcapture %}

{% capture steps %}

## 场景：限制存储使用量


集群管理员正代表一个用户群体操作集群。管理员希望控制单个 namespace 能够使用的存储数量，以此来控制成本。


管理员希望限制：

1. Namespace 中的 persistent volume claim 的数量
2. 每个 claim 可以请求的存储数量
3. Namespace 可以拥有的存储总量


## 使用 LimitRange 限制存储请求


添加 `LimitRange` 到 namespace 将限制存储请求大小处于最小和最大值之间。存储通过 `PersistentVolumeClaim` 进行请求。应用了 limit range 的准入控制器会拒绝任何大于或小于管理员设置值的 PVC 请求。


本例中，PVC 请求的 10Gi 存储将被拒绝，因为它超出了 2Gi 的最大值。

```
apiVersion: v1
kind: LimitRange
metadata:
  name: storagelimits
spec:
  limits:
  - type: PersistentVolumeClaim
    max:
      storage: 2Gi
    min:
      storage: 1Gi
```

存储请求的最小值用于底层存储提供商要求某个最小值的场景。例如 AWS EBS volume 需要 1Gi 的最小值。


## 使用 StorageQuota 限制 PVC 数量和存储总量


管理员可以限制 namespace 中 PVC 的数量及其容量的总大小。任何超过最大值的新 PVC 请求都将被拒绝。


本例中， namespace 中第 6 个 PVC 请求将被拒绝，因为它超过了 5 个最大数量的限制。又或者对于拥有 5Gi 配额最大值和 2Gi 最大值限制的 namespace，不能拥有 3 个 2Gi 大小的 PVC。因为那会向具有 5Gi 上限的 namespace 请求 6Gi 的存储。

```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storagequota
spec:
  hard:
    persistentvolumeclaims: "5"
    requests.storage: "5Gi"
```

{% endcapture %}

{% capture discussion %}


## 总结


Limit range 可以为存储的请求数量设置上限，而资源配额可以通过 claim 数量和存储总大小有效的限制 namespace 使用的存储。这使得集群管理员可以规划集群的存储预算而不用担心任何单个项目超出它们的配额。

{% endcapture %}

{% include templates/task.md %}
