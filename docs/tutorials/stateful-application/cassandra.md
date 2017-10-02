---
title: 示例：使用 Stateful Sets 部署 Cassandra
assignees:
- ahmetb
cn-approvers:
- xiaosuiba
cn-reviewers:
- markthink
---


{% capture overview %}

本教程展示了如何在 Kubernetes 上开发一个云原生的 [Cassandra](http://cassandra.apache.org/) deployment。在这个实例中，Cassandra 使用了一个自定义的  `SeedProvider` 来发现新加入集群的节点。


在集群环境中部署类似 Cassandra 的有状态（stateful）应用可能是具有挑战性的。StatefulSets 极大的简化了这个过程。请阅读 [StatefulSets](/docs/concepts/workloads/controllers/statefulset/) 获取更多关于此教程中使用的这个特性的信息。

**Cassandra Docker**


Pod 使用了来自 Google [容器注册表（container registry）](https://cloud.google.com/container-registry/docs/) 的  [`gcr.io/google-samples/cassandra:v12`](https://github.com/kubernetes/examples/blob/master/cassandra/image/Dockerfile) 镜像。这个 docker 镜像基于 `debian:jessie` 并包含 OpenJDK 8。这个镜像包含了来自 Apache Debian 源的标准 Cassandra 安装。您可以通过环境变量来改变插入到 `cassandra.yaml` 中的值。

| ENV VAR                | DEFAULT VALUE  |
| ---------------------- | :------------: |
| CASSANDRA_CLUSTER_NAME | 'Test Cluster' |
| CASSANDRA_NUM_TOKENS   |       32       |
| CASSANDRA_RPC_ADDRESS  |    0.0.0.0     |

{% endcapture %}

{% capture objectives %}

* 创建并验证 Cassandra headless [Services](/docs/concepts/services-networking/service/)。
* 使用  [StatefulSet](/docs/concepts/workloads/controllers/statefulset/) 创建 Cassandra  环（Cassandra ring）。
* 验证 [StatefulSet](/docs/concepts/workloads/controllers/statefulset/)。
* 修改 [StatefulSet](/docs/concepts/workloads/controllers/statefulset/)。
* 删除 [StatefulSet](/docs/concepts/workloads/controllers/statefulset/) 和它的 [Pod](/docs/concepts/workloads/pods/pod/)。
  {% endcapture %}

{% capture prerequisites %}

为了完成本教程，你应该对 [Pod](/docs/concepts/workloads/pods/pod/)、 [Service](/docs/concepts/services-networking/service/) 和 [StatefulSet](/docs/concepts/workloads/controllers/statefulset/) 有基本的了解。此外，你还应该：


*  [安装并配置 ](/docs/tasks/tools/install-kubectl/) `kubectl` 命令行工具

* 下载  [cassandra-service.yaml](/docs/tutorials/stateful-application/cassandra/cassandra-service.yaml) 和 [cassandra-statefulset.yaml](/docs/tutorials/stateful-application/cassandra/cassandra-statefulset.yaml)

* 有一个支持（这些功能）并正常运行的 Kubernetes 集群


**注意：** 如果你还没有集群，请查阅 [快速入门指南](/docs/setup/pick-right-solution/)。
{: .note}


### Minikube 附加安装说明


**小心：** [Minikube](/docs/getting-started-guides/minikube/) 默认配置 1024MB 内存和 1 CPU，这在本例中将导致资源不足。
{: .caution}


为了避免这些错误，请这样运行 minikube：

    minikube start --memory 5120 --cpus=4

{% endcapture %}

{% capture lessoncontent %}

## 创建  Cassandra Headless Service

Kubernetes [Service](/docs/concepts/services-networking/service/) 描述了一个执行相同任务的 [Pod](/docs/concepts/workloads/pods/pod/) 集合。


下面的 `Service` 用于在集群内部的 Cassandra Pod 和客户端之间进行 DNS 查找。


1. 在下载清单文件的文件夹下启动一个终端窗口。
2. 使用 `cassandra-service.yaml` 文件创建一个 `Service`，用于追踪所有的 Cassandra StatefulSet  节点。

       kubectl create -f cassandra-service.yaml

{% include code.html language="yaml" file="cassandra/cassandra-service.yaml" ghlink="/docs/tutorials/stateful-application/cassandra/cassandra-service.yaml" %}


### 验证（可选）


获取 Cassandra `Service`。

    kubectl get svc cassandra


响应应该像这样：

    NAME        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
    cassandra   None         <none>        9042/TCP   45s


如果返回了任何其它消息，这个 service 就没有被成功创建。请查阅 [调试 Services](/docs/tasks/debug-application-cluster/debug-service/)，了解常见问题。


## 使用 StatefulSet 创建 Cassandra 环


上文中的 StatefulSet 清单文件将创建一个由 3 个 pod 组成的 Cassandra 环。


**注意：** 本例中的 Minikube 使用默认 provisioner。请根据您使用的云服务商更新下面的 StatefulSet。
{: .note}


1. 如有必要请修改 StatefulSet。
2. 使用  `cassandra-statefulset.yaml` 文件创建 Cassandra StatefulSet。

       kubectl create -f cassandra-statefulset.yaml

{% include code.html language="yaml" file="cassandra/cassandra-statefulset.yaml" ghlink="/docs/tutorials/stateful-application/cassandra/cassandra-statefulset.yaml" %}


## 验证 Cassandra StatefulSet


1. 获取 Cassandra StatefulSet：

       kubectl get statefulset cassandra

   响应应该是

       NAME        DESIRED   CURRENT   AGE
       cassandra   3         0         13s


   StatefulSet 资源顺序的部署 pod。


2. 获取 pod， 查看顺序创建的状态：

       kubectl get pods -l="app=cassandra"


   响应应该像是
   
       NAME          READY     STATUS              RESTARTS   AGE
       cassandra-0   1/1       Running             0          1m
       cassandra-1   0/1       ContainerCreating   0          8s


   **注意：** 部署全部三个 pod 可能需要 10 分钟时间。
   {: .note}


    一旦所有 pod 都已经部署，相同的命令将返回：

       NAME          READY     STATUS    RESTARTS   AGE
       cassandra-0   1/1       Running   0          10m
       cassandra-1   1/1       Running   0          9m
       cassandra-2   1/1       Running   0          8m


3. 运行 Cassandra nodetool 工具，显示环的状态。

       kubectl exec cassandra-0 -- nodetool status


   响应为：

       Datacenter: DC1-K8Demo
       ======================
       Status=Up/Down
       |/ State=Normal/Leaving/Joining/Moving
       --  Address     Load       Tokens       Owns (effective)  Host ID                               Rack
       UN  172.17.0.5  83.57 KiB  32           74.0%             e2dd09e6-d9d3-477e-96c5-45094c08db0f  Rack1-K8Demo
       UN  172.17.0.4  101.04 KiB  32           58.8%             f89d6835-3a42-4419-92b3-0e62cae1479c  Rack1-K8Demo
       UN  172.17.0.6  84.74 KiB  32           67.1%             a6a1e8c2-3dc5-4417-b1a0-26507af2aaad  Rack1-K8Demo


## 修改 Cassandra StatefulSet

使用 `kubectl edit`修改 Cassandra StatefulSet 的大小。 

1. 运行下面的命令：

       kubectl edit statefulset cassandra

   这个命令将在终端中打开一个编辑器。您需要修改 `replicas` 字段一行。

   **注意：** 以下示例是 StatefulSet 文件的摘录。
   {: .note}

    ```yaml   
    # Please edit the object below. Lines beginning with a '#' will be ignored,
    # and an empty file will abort the edit. If an error occurs while saving this file will be
    # reopened with the relevant failures.
    #
    apiVersion: apps/v1beta1
    kind: StatefulSet
    metadata:
     creationTimestamp: 2016-08-13T18:40:58Z
     generation: 1
     labels:
       app: cassandra
     name: cassandra
     namespace: default
     resourceVersion: "323"
     selfLink: /apis/apps/v1beta1/namespaces/default/statefulsets/cassandra
     uid: 7a219483-6185-11e6-a910-42010a8a0fc0
    spec:
     replicas: 3
    ```


2. 修改副本数量为 4 并保存清单文件。

   这个 StatefulSet 现在包含 4 个 pod。

3. 获取 Cassandra StatefulSet 来进行验证：

       kubectl get statefulset cassandra

   响应应该为：

       NAME        DESIRED   CURRENT   AGE
       cassandra   4         4         36m

{% endcapture %}

{% capture cleanup %}

删除或缩容 StatefulSet 不会删除与其相关联的 volume。这优先保证了安全性：您的数据比其它所有自动清理的 StatefulSet 资源都更宝贵。


**警告：** 取决于  storage class 和回收策略（reclaim policy），删除 Persistent Volume Claims 可能导致关联的 volume 也被删除。绝对不要假设在 volume claim 被删除后还能访问数据。
{: .warning}

1. 运行下面的命令，删除 `StatefulSet` 中所有能内容：

       grace=$(kubectl get po cassandra-0 -o=jsonpath='{.spec.terminationGracePeriodSeconds}') \
         && kubectl delete statefulset -l app=cassandra \
         && echo "Sleeping $grace" \
         && sleep $grace \
         && kubectl delete pvc -l app=cassandra


2. 运行下面的命令，删除 Cassandra `Service`。

       kubectl delete service -l app=cassandra

{% endcapture %}

{% capture whatsnext %}

* 了解如何  [扩展 StatefulSet](/docs/tasks/run-application/scale-stateful-set/)。
* 详细了解  [KubernetesSeedProvider](https://github.com/kubernetes/examples/blob/master/cassandra/java/src/main/java/io/k8s/cassandra/KubernetesSeedProvider.java)。
* 查看更多自定义 [Seed Provider 配置](https://git.k8s.io/examples/cassandra/java/README.md)。

{% endcapture %}

{% include templates/tutorial.md %}
