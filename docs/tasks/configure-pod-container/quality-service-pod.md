---
title: 给 Pod 配置服务质量等级
---


{% capture overview %}

这篇教程指导如何给 Pod 配置特定的服务质量（QoS）等级。Kubernetes 使用 QoS 等级来确定何时调度和终结 Pod 。

{% endcapture %}


{% capture prerequisites %}

{% include task-tutorial-prereqs.md %}

{% endcapture %}


{% capture steps %}

## QoS 等级

当 Kubernetes 创建一个 Pod 时，它就会给这个 Pod 分配一个 QoS 等级：

* Guaranteed
* Burstable
* BestEffort


## 创建一个命名空间

创建一个命名空间，以便将我们实验需求的资源与集群其他资源隔离开。

```shell
kubectl create namespace qos-example
```


## 创建一个 Pod 并分配 QoS 等级为 Guaranteed

想要给 Pod 分配 QoS 等级为 Guaranteed:

* Pod 里的每个容器都必须有内存限制和请求，而且必须是一样的。
* Pod 里的每个容器都必须有 CPU 限制和请求，而且必须是一样的。

这是一个含有一个容器的 Pod 的配置文件。这个容器配置了内存限制和请求，都是200MB。它还有
CPU 限制和请求，都是700 millicpu:

{% include code.html language="yaml" file="qos-pod.yaml" ghlink="/docs/tasks/configure-pod-container/qos-pod.yaml" %}


创建 Pod:

```shell
kubectl create -f https://k8s.io/docs/tasks/configure-pod-container/qos-pod.yaml --namespace=qos-example
```

查看 Pod 的详细信息:

```shell
kubectl get pod qos-demo --namespace=qos-example --output=yaml
```

输出显示了 Kubernetes 给 Pod 配置的 QoS 等级为 Guaranteed 。也验证了容器的内存和 CPU 的限制都满足了它的请求。

```yaml
spec:
  containers:
    ...
    resources:
      limits:
        cpu: 700m
        memory: 200Mi
      requests:
        cpu: 700m
        memory: 200Mi
...
  qosClass: Guaranteed
```


**注意:** 如果一个容器配置了内存限制，但是没有配置内存申请，那 Kubernetes 会自动给容器分配一个符合内存限制的请求。
类似的，如果容器有 CPU 限制，但是没有 CPU 申请，Kubernetes 也会自动分配一个符合限制的请求。
{: .note}

删除你的 Pod:

```shell
kubectl delete pod qos-demo --namespace=qos-example
```


## 创建一个 Pod 并分配 QoS 等级为 Burstable

当出现下面的情况时，则是一个 Pod 被分配了 QoS 等级为 Burstable :

* 该 Pod 不满足 QoS 等级 Guaranteed 的要求。
* Pod 里至少有一个容器有内存或者 CPU 请求。

这是 Pod 的配置文件，里面有一个容器。这个容器配置了200MB的内存限制和100MB的内存申请。

{% include code.html language="yaml" file="qos-pod-2.yaml" ghlink="/docs/tasks/configure-pod-container/qos-pod-2.yaml" %}

创建 Pod:

```shell
kubectl create -f https://k8s.io/docs/tasks/configure-pod-container/qos-pod-2.yaml --namespace=qos-example
```

查看 Pod 的详细信息:

```shell
kubectl get pod qos-demo-2 --namespace=qos-example --output=yaml
```

输出显示了 Kubernetes 给这个 Pod 配置了 QoS 等级为 Burstable.

```yaml
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: qos-demo-2-ctr
    resources:
      limits:
        memory: 200Mi
      requests:
        memory: 100Mi
...
  qosClass: Burstable
```

删除你的 Pod:

```shell
kubectl delete pod qos-demo-2 --namespace=qos-example
```

## 创建一个 Pod 并分配 QoS 等级为 BestEffort

要给一个 Pod 配置 BestEffort 的 QoS 等级, Pod 里的容器必须没有任何内存或者 CPU　的限制或请求。

下面是一个　Pod　的配置文件，包含一个容器。这个容器没有内存或者 CPU 的限制或者请求：

{% include code.html language="yaml" file="qos-pod-3.yaml" ghlink="/docs/tasks/configure-pod-container/qos-pod-3.yaml" %}

创建 Pod:

```shell
kubectl create -f https://k8s.io/docs/tasks/configure-pod-container/qos-pod-3.yaml --namespace=qos-example
```

查看 Pod 的详细信息:

```shell
kubectl get pod qos-demo-3 --namespace=qos-example --output=yaml
```

输出显示了 Kubernetes 给 Pod 配置的 QoS 等级是 BestEffort.

```yaml
spec:
  containers:
    ...
    resources: {}
  ...
  qosClass: BestEffort
```

删除你的 Pod:

```shell
kubectl delete pod qos-demo-3 --namespace=qos-example
```

## 创建一个拥有两个容器的 Pod 

这是一个含有两个容器的 Pod 的配置文件，其中一个容器指定了内存申请为 200MB ，另外一个没有任何申请或限制。

{% include code.html language="yaml" file="qos-pod-4.yaml" ghlink="/docs/tasks/configure-pod-container/qos-pod-4.yaml" %}

注意到这个 Pod 满足了 QoS 等级 Burstable 的要求. 就是说，它不满足 Guaranteed 的要求，而且其中一个容器有内存请求。

创建 Pod:

```shell
kubectl create -f https://k8s.io/docs/tasks/configure-pod-container/qos-pod-4.yaml --namespace=qos-example
```

查看 Pod 的详细信息:

```shell
kubectl get pod qos-demo-4 --namespace=qos-example --output=yaml
```

输出显示了 Kubernetes 给 Pod 配置的 QoS 等级是 Burstable:

```yaml
spec:
  containers:
    ...
    name: qos-demo-4-ctr-1
    resources:
      requests:
        memory: 200Mi
    ...
    name: qos-demo-4-ctr-2
    resources: {}
    ...
  qosClass: Burstable
```

删除你的 Pod:

```shell
kubectl delete pod qos-demo-4 --namespace=qos-example
```

## 清理

删除你的 namespace:

```shell
kubectl delete namespace qos-example
```

{% endcapture %}

{% capture whatsnext %}


### 对于应用开发者

* [给容器或者 Pod  分配内存资源](/docs/tasks/configure-pod-container/assign-memory-resource/)

* [给容器或者 Pod  分配 CPU 资源](/docs/tasks/configure-pod-container/assign-cpu-resource/)

### 对于集群管理者

* [给命名空间配置默认的内存请求和限制](/docs/tasks/administer-cluster/memory-default-namespace/)

* [给命名空间配置默认的 CPU 请求和限制](/docs/tasks/administer-cluster/cpu-default-namespace/)

* [给命名空间配置最大和最小的内存限制](/docs/tasks/administer-cluster/memory-constraint-namespace/)

* [给命名空间配置最大和最小的 CPU 限制](/docs/tasks/administer-cluster/cpu-constraint-namespace/)

* [给命名空间配置内存和 CPU 限额](/docs/tasks/administer-cluster/quota-memory-cpu-namespace/)

* [给命名空间配置 Pod 限额](/docs/tasks/administer-cluster/quota-pod-namespace/)

* [配置 API 对象限额](/docs/tasks/administer-cluster/quota-api-object/)

{% endcapture %}


{% include templates/task.md %}
