---
title: 集群 DNS Service Autoscale
---

{% capture overview %}


本页展示了如何在集群中启用和配置 DNS Service 的 autoscaling 功能。

{% endcapture %}

{% capture prerequisites %}

* {% include task-tutorial-prereqs.md %}



* 确保启用 [DNS 特性](/docs/concepts/services-networking/dns-pod-service/)。

* 推荐在 Kubernetes 1.4.0 或更新版本使用。

{% endcapture %}

{% capture steps %}



## 确定是否 DNS 水平 autoscaling 特性已经启用

列出集群中 kube-system namespace 中的 Deployment：

```shell
    kubectl get deployment --namespace=kube-system
```



输出类似如下这样：

    NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    ...
    kube-dns-autoscaler   1         1         1            1           ...
    ...



如果在输出中看到 "kube-dns-autoscaler"，说明DNS 水平 autoscaling 已经启用，可以跳过 [调优 autoscaling 参数](#tuning-autoscaling-parameters)。



## 获取 DNS Deployment 或 ReplicationController 的名称

列出集群内 kube-system namespace 中的 Deployment：

```shell
    kubectl get deployment --namespace=kube-system
```



输出类似如下这样：

    NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    ...
    kube-dns     1         1         1            1           ...
    ...



Kubernetes 1.5 或之前版本，DNS 通过使用 ReplicationController 来实现，而不是 Deployment。
所以看不到 kube-dns 或者类似的名称，在之前的输出中，列出了集群内 kube-system namespace 中的 ReplicationController：

```shell
    kubectl get rc --namespace=kube-system
```



输出类似如下这样：

    NAME            DESIRED   CURRENT   READY     AGE
    ...
    kube-dns-v20    1         1         1         ...
    ...



## 确定 scale 目标

如果有一个 DNS Deployment，scale 目标是：

    Deployment/<your-deployment-name>



这里，<dns-deployment-name> 表示 DNS Deployment 的名称。
例如，如果 DNS Deployment 是 kube-dns，则 scale 目标为 Deployment/kube-dns。

如果有一个 DNS ReplicationController，那么 scale 目标为：

    ReplicationController/<your-rc-name>



这里 <your-rc-name> 是 DNS ReplicationController 的名称。
例如，DNS ReplicationController 的名称是 kube-dns-v20，则 scale 目标为 ReplicationController/kube-dns-v20。



## 启用 DNS 水平 autoscaling

在本段，我们创建一个 Deployment。
Deployment 中的 Pod 运行一个基于 `cluster-proportional-autoscaler-amd64` 镜像的容器。

创建文件 `dns-horizontal-autoscaler.yaml`，内容如下所示：

{% include code.html language="yaml" file="dns-horizontal-autoscaler.yaml" ghlink="/docs/tasks/administer-cluster/dns-horizontal-autoscaler.yaml" %}



在文件中，将 `<SCALE_TARGET>` 替换成 scale 目标。

进入到包含配置文件的目录中，输入如下命令创建 Deployment：

```shell
    kubectl create -f dns-horizontal-autoscaler.yaml
```



一个成功的命令输出是：

    deployment "kube-dns-autoscaler" created



DNS 水平 autoscaling 现在已经启用了。

## 调优 autoscaling 参数

验证 kube-dns-autoscaler ConfigMap 存在：

```shell
    kubectl get configmap --namespace=kube-system
```



输出类似如下所示：

    NAME                  DATA      AGE
    ...
    kube-dns-autoscaler   1         ...
    ...



修改该 ConfigMap 中的数据：

```shell
    kubectl edit configmap kube-dns-autoscaler --namespace=kube-system
```



找到如下这行内容：

    linear: '{"coresPerReplica":256,"min":1,"nodesPerReplica":16}'



根据需要修改对应的字段。
"min" 字段说明 DNS backend 的最小数量。
实际 backend 的数量，通过使用如下公式来计算：

    replicas = max( ceil( cores * 1/coresPerReplica ) , ceil( nodes * 1/nodesPerReplica ) )



注意 `coresPerReplica` 和 `nodesPerReplica` 的值都是整数。

想法是，当一个集群使用具有很多 core 的节点时，由 `coresPerReplica` 来控制。
当一个集群使用具有较少 core 的节点时，由 `nodesPerReplica` 来控制。



其它的 scaling 模式也是支持的，详情查看 [cluster-proportional-autoscaler](https://github.com/kubernetes-incubator/cluster-proportional-autoscaler)。

## 禁用 DNS 水平 autoscaling

有几个 DNS 水平 autoscaling 的选项。具体使用哪个选项因环境而异。



### 选项 1：调小 kube-dns-autoscaler deployment 至 0 个副本

该选项适用于所有场景。运行如下命令：

```shell
    kubectl scale deployment --replicas=0 kube-dns-autoscaler --namespace=kube-system
```



输出如下所示：

    deployment "kube-dns-autoscaler" scaled



验证当前副本数为 0：

```shell
    kubectl get deployment --namespace=kube-system
```



输出内容中，在 DESIRED 和 CURRENT 列显示为 0：

    NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    ...
    kube-dns-autoscaler   0         0         0            0           ...
    ...



### 选项 2：删除 kube-dns-autoscaler Deployment

如果 kube-dns-autoscaler 为您所控制，该选项可以正常工作，也就说没有人会去重新创建它：

```shell
    kubectl delete deployment kube-dns-autoscaler --namespace=kube-system
```



输出内容如下所示：

    deployment "kube-dns-autoscaler" deleted



### 选项 3：从 master 节点删除 kube-dns-autoscaler manifest 文件

如果 kube-dns-autoscaler 在 [插件管理器](https://git.k8s.io/kubernetes/cluster/addons/README.md) 的控制之下，该选项可以工作，并且具有操作 master 节点的写权限。

登录到 master 节点，删除对应的 manifest 文件。
kube-dns-autoscaler 的路径一般为：

    /etc/kubernetes/addons/dns-horizontal-autoscaler/dns-horizontal-autoscaler.yaml



当 manifest 文件删除后，插件管理器将删除 kube-dns-autoscaler Deployment。

{% endcapture %}

{% capture discussion %}



## 理解 DNS 水平 autoscaling 工作原理

* cluster-proportional-autoscaler 应用独立于 DNS service 部署。

* autoscaler Pod 运行一个客户端，它通过轮询 Kubernetes API server 获取集群中节点和 core 的数量。



* 一个期望的副本数会被计算，并根据当前可调度的节点、CPU core 数、给定 scale 参数，被应用到 DNS backend。

* scale 参数和数据点会基于一个 ConfigMap 来提供给 autoscaler，它会在每次轮询时刷新它的参数表，以与最近期望的 scaling 参数保持一致。

* 允许对 scaling 参数进行修改，而不需要重建或重启 autoscaler Pod。

* autoscaler 提供了一个控制器接口来支持两种控制模式：*linear* 和 *ladder*。



## 未来功能增强

控制模式，除了 linear 和 ladder，正在考虑未来将开发自定义 metric。

基于 DNS 特定 metric 的 DNS backend 的 scaling，考虑未来会开发。
当前实现是使用集群中节点和 core 的数量，是受限制的。

支持自定义 metric，类似于 [Horizontal Pod Autoscaling](/docs/tasks/run-application/horizontal-pod-autoscale/) 所提供的，考虑未来进行开发。

{% endcapture %}

{% capture whatsnext %}

了解关于 [cluster-proportional-autoscaler 实现](https://github.com/kubernetes-incubator/cluster-proportional-autoscaler)。

{% endcapture %}


{% include templates/task.md %}
