---
assignees:
- derekwaynecarr
- janetkuo
title: 使用命名空间共享一个集群  
redirect_from:
- "/docs/admin/namespaces/"
- "/docs/admin/namespaces/index.html"
---



命名空间是用户资源管理的逻辑分区


## 动机

通常集群应该能够满足多个用户或用户组（以下称为“用户社区”）的需求。

每个用户社区希望能够与其他社区隔离工作。

每个用户社区都有自己的：


1. 资源（pods, 服务, rc等）
2. 策略（能否在他们的社区执行）
3. 限制约束（这个社区允许这么多配额等）

集群操作员可以为每个唯一用户社区创建一个命名空间。


命名空间提供了一个唯一的范围：

1. 命名资源（以避免基本的命名冲突）
2. 委托管理权限给受信任的用户
3. 限制社区资源消耗的能力


## 用例

1. 作为集群操作员，我想在一个集群上支持多个用户社区。  
2. 作为集群操作员，我想将集群分区的权限委托给在这些社区的受信任的用户。  
3. 作为集群操作员，我想限制每个社区可以消耗的资源数量以便限制对该集群中其它社区的影响。  
4. 作为集群用户，我想与与用户社区相关的资源进行交互，而与群集中的其他用户社区无关。  


## 查看命名空间

您可以使用以下命令列出群集中的当前命名空间：

```shell
$ kubectl get namespaces
NAME          STATUS    AGE
default       Active    11d
kube-system   Active    11d
```


Kubernetes开始时有两个初始化的命名空间：

   * `default` 没有其他命名空间的对象的默认命名空间
   * `kube-system` 由Kubernetes系统创建的对象的命名空间
   
您还可以使用以下命令获取特定命名空间的摘要：

```shell
$ kubectl get namespaces <name>
```


或者你可以获得详细的信息通过：

```shell
$ kubectl describe namespaces <name>
Name:       default
Labels:       <none>
Status:       Active

No resource quota.

Resource Limits
 Type        Resource    Min    Max    Default
 ----                --------    ---    ---    ---
 Container            cpu            -    -    100m
```


注意这些详细信息显示了资源配额（如果存在）以及资源限制范围。

资源配额记录*命名空间*中资源的总体使用情况并且允许集群操作员定义*硬性*资源使用率限制一个命名空间可能消耗的资源。


限制范围定义了在*命名空间*中单个实体可以使用的资源量的最小/最大约束。

参考[准入控制：限制范围](https://git.k8s.io/community/contributors/design-proposals/admission_control_limit_range.md)。

命名空间可以分为两个阶段：

   * `Active` 命名空间正在使用中
   * `Terminating` 命名空间正在被删除，不能被新的对象使用

更多细节请参考[设计文档](https://git.k8s.io/community/contributors/design-proposals/namespaces.md#phases)。


## 创建一个新的命名空间

要想创建一个新的命名空间，首先创建一个名为`my-namespace.yaml`的新的YAML文件，其内容如下：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <insert-namespace-name-here>
```


然后运行：

```shell
$ kubectl create -f ./my-namespace.yaml
```


注意你的命名空间的名称必须是一个DNS兼容的标签。

这里有一个可选的`终止器`域，每当这个命名空间被删除时允许观察清除资源。
记住如果你指定了一个不存在的终止器，这个命名空间将会被创建但是如果用户试图删除它时，它将卡在`Terminating`态。

有关终止器的更多信息可以在[命名空间设计文档](https://git.k8s.io/community/contributors/design-proposals/namespaces.md#finalizers)中找到。


### 在命名空间中工作

参阅[设置命名空间请求](/docs/user-guide/namespaces/#setting-the-namespace-for-a-request)和[设置命名空间优先级](/docs/user-guide/namespaces/#setting-the-namespace-preference)。

## 删除一个命名空间

你可以通过以下命令删除一个命名空间：

```shell
$ kubectl delete namespaces <insert-some-namespace-name>
```


**警告，这将删除在命名空间下的一切！**

这个删除是异步的，因此一段时间你会看到命名空间停留在`Terminating`状态。


## 命名空间与DNS

当你创建了一个[服务](/docs/user-guide/services)，它会创建一个相应的[DNS入口](/docs/admin/dns)。
这个入口是`<service-name>.<namespace-name>.svc.cluster.local`的形式，这意味着如果一个容器仅使用`<service-name>`，它将使用本地的命名空间解析这个服务。
这对于使用相同的配置跨越多个命名空间是很有用的，比如开发，迭代和生产命名空间。如果你想要达到跨越命名空间的目的，你需要使用完整的域名（FQDN）。


## 设计

在Kubernetes中命名空间的详细设计，包括[详细示例](https://git.k8s.io/community/contributors/design-proposals/namespaces.md#example-openshift-origin-managing-a-kubernetes-namespace)可以在[命名空间设计文档](https://git.k8s.io/community/contributors/design-proposals/namespaces.md)中找到。
