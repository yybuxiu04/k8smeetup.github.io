---
assignees:
- derekwaynecarr  

title: 资源配额  
redirect_from:
- "/docs/admin/resourcequota/"
- "/docs/admin/resourcequota/index.html"
---

当一个固定节点数的集群中存在多个用户或者团队时，就会产生某个团队使用的资源超出公平份额的问题。  
资源配额就是为管理员解决此问题的一个工具。  
通过`ResourceQuota`对象来定义一个资源配额，提供了对每个命名空间下限制资源消耗总数的约束条件。它可以按照类型限制一个命名空间下对象的数量，也可以按照资源消耗来限制计算资源总量。
  
资源配额工作如下：
* 不同的团队在不同的命名空间下工作。目前这是自发的，但是通过ACLs实现强制性已在计划内。
* 管理员为每个命名空间创建一个或多个资源配额。
* 用户在命名空间下创建资源对象（pods, services 等），这个配额系统跟踪使用率来确保它不会超出在资源配额中定义的限制。
* 如果在创建或者更新一个资源时违反了配额的限制，这个请求将会失败，并返回`403 FORBIDDEN`HTTP状态码，以及超出限制条件的信息。
* 如果在命名空间下开启了计算资源的配额，比如`cpu`和内存，用户必须为这些资源指定请求和限制；否则，配额系统将拒绝创建pod。提示： 可以使用`LimitRanger`准入控制组件来创建默认的资源请求。参考  [walkthrough](https://kubernetes.io/docs/tasks/administer-cluster/apply-resource-quota-limit/) 这个示例来解决此问题。


使用命名空间和配额的一些例子：
* 在一个内存总量为32G，cpu为16核的集群中，给A团队内存20G，cpu 10核，给B团队内存10G，cpu 4核心， 保留2G内存和2核cpu。
* 限制`testing`命名空间使用1G内存和1核cpu, 让`production`命名空间随意使用。

如果集群的总容量小于命名空间的配额总额，可能会产生资源竞争。这时会按照先到先得来处理。  
资源竞争和配额的更新都不会影响已经创建好的资源。  


#### 启用资源配额
Kubernetes 的众多发行版本默认开启了资源配额的支持。当在apiserver的`--admission-control`配置中添加`ResourceQuota`参数后，便启用了。
当一个命名空间中含有`ResourceQuota`对象时，资源配额强制执行。一个命名空间最多只能有一个`ResourceQuota`对象。


#### 计算资源的配额
你可以在一个给定的命名空间中限制可以请求的计算资源（[compute resources](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/)）的总量。
支持以下的资源类型：

|资源名称  | 描述 |
| ------ | ------ |
| cpu | 非终止态的所有pod, cpu请求总量不能超出此值。
| limits.cpu | 非终止态的所有pod， cpu限制总量不能超出此值。
| limits.memory | 非终止态的所有pod, 内存限制总量不能超出此值。
| memory | 非终止态的所有pod, 内存请求总量不能超出此值。
| requests.cpu | 非终止态的所有pod, cpu请求总量不能超出此值。
| requests.memory | 非终止态的所有pod, 内存请求总量不能超出此值。


#### 存储资源的配额
你可以在一个给定的命名空间中限制可以请求的存储资源（[storage resources](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)）的总量。
总之，你可以根据关联的存储类来限制存储资源的消耗量。

| 资源名称 | 描述 |
| ------ | ------ |
| requests.storage | 所有PVC, 存储请求总量不能超出此值。
| persistentvolumeclaims | 命名空间中可以存在的PVC（[persistent volume claims](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)）总数。
| <storage-class-name>.storageclass.storage.k8s.io/requests.storage | 和该存储类关联的所有PVC, 存储请求总和不能超出此值。
| <storage-class-name>.storageclass.storage.k8s.io/persistentvolumeclaims | 和该存储类关联的所有PVC，命名空间中可以存在的PVC（[persistent volume claims](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)）总数。

例如，如果一个操作人员想要分别定额`gold`存储类和`bronze`存储类，则这个操作人员可以按照下面这样定义配额：
* `gold.storageclass.storage.k8s.io/requests.storage: 500Gi`
* `bronze.storageclass.storage.k8s.io/requests.storage: 100Gi`


#### 对象数量的配额
一个给定类型的对象的数量可以被限制。支持以下类型：

| 资源名称 | 描述 |
| ------ | ------ |
| congfigmaps | 命名空间中可以存在的配置映射的总数。
| persistentvolumeclaims | 命名空间中可以存在的PVC总数。
| pods | 命名空间中可以存在的非终止态的pods总数。如果一个pod的`status.phase` 是 `Failed, Succeeded`, 则该pod处于终止态。
| replicationcontrollers | 命名空间中可以存在的`rc`总数。
| resourcequotas | 命名空间中可以存在的资源配额（[resource quotas](https://kubernetes.io/docs/admin/admission-controllers/#resourcequota)）总数。
| services | 命名空间中可以存在的服务总数量。
| services.loadbalancers | 命名空间中可以存在的服务的负载均衡的总数量。 
| services.nodeports | 命名空间中可以存在的服务的主机接口的总数量。
| secrets | 命名空间中可以存在的`secrets`的总数量。

例如，pods 数量配额，则表示在单个命名空间中可以创建的pod的最大值。
你可能想要在一个命名空间中定义一个pod限额来避免一个用户创建了许多小的pods从而耗光这个集群Pod IPs 的情况。


#### 限额的作用域
每个配额可以有一组关联的作用域。如果一个限额匹配枚举的作用的交集，它将只衡量一个资源的利用率。
当一个作用域被添加到配额时，它将会限制它支持的涉及到该作用域的资源的数量。在不允许设置的限额上指定资源将会导致一个验证错误。

| 作用域 | 描述 |
| ------ | ------ |
| Terminating | 匹配 `spec.activeDeadlineSeconds >= 0` 的pods
| NotTerminating | 匹配 `spec.activeDeadlineSeconds is nil` 的pods
| BestEffort | 匹配具有最佳服务质量的pods
| NotBestEffort | 匹配具有非最佳服务质量的pods

`BestEffort`作用域禁止限额跟踪以下的资源：
* pods

`Terminating` 、`NotTerminating`和`NotBestEffort`作用域禁止限额跟踪以下的资源：
* cpu
* limits.cpu
* limits.memory
* memory
* pods
* requests.cpu
* requests.memory


#### 请求 vs 限度
当分配计算资源时，每个容器可以为cpu或者内存指定一个请求值和一个限度值。可以配置限额值来限制它们中的任何一个值。  
如果指定了`requests.cpu` 或者 `requests.memory`的限额值，那么就要求传入的每一个容器显式的指定这些资源的请求。如果指定了`limits.cpu`或者`limits.memory`，那么就要求传入的每一个容器显式的指定这些资源的限度。
 
 
#### 查看和设置配额
Kubectl 支持创建，更新和查看配额：
```
$ kubectl create namespace myspace

$ cat <<EOF > compute-resources.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    pods: "4"
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
EOF
$ kubectl create -f ./compute-resources.yaml --namespace=myspace

$ cat <<EOF > object-counts.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-counts
spec:
  hard:
    configmaps: "10"
    persistentvolumeclaims: "4"
    replicationcontrollers: "20"
    secrets: "10"
    services: "10"
    services.loadbalancers: "2"
EOF
$ kubectl create -f ./object-counts.yaml --namespace=myspace

$ kubectl get quota --namespace=myspace
NAME                    AGE
compute-resources       30s
object-counts           32s

$ kubectl describe quota compute-resources --namespace=myspace
Name:                  compute-resources
Namespace:             myspace
Resource               Used Hard
--------               ---- ----
limits.cpu             0    2
limits.memory          0    2Gi
pods                   0    4
requests.cpu           0    1
requests.memory        0    1Gi

$ kubectl describe quota object-counts --namespace=myspace
Name:                   object-counts
Namespace:              myspace
Resource                Used    Hard
--------                ----    ----
configmaps              0       10
persistentvolumeclaims  0       4
replicationcontrollers  0       20
secrets                 1       10
services                0       10
services.loadbalancers  0       2
```

#### 配额和集群容量
资源配额对象与集群容量无关。它们以绝对单位表示。因此，如果在集群中新增加节点，也不会给每个命名空间自动增加消耗更多资源的能力。
有时可能需要更复杂的策略，例如：
* 在几个团队之间按比例划分集群总体资源。
* 允许每个租户根据需要增加资源使用，但有一个宽松的限制来防止意外的资源枯竭。
* 从命名空间检测需求，添加节点并增加配额。
使用`ResourceQuota`作为一个构建模块，编写一个`controller`监听配额使用率和根据其它信号调整每个命名空间下的配额的硬性限制可以实现这些策略。
注意资源配额分割集群的总资源，但是它不能对节点创建限制：来自多个命名空间下的pods可以在同一个节点上运行。

#### 范例
参考如何使用资源配额的[详细案例](https://kubernetes.io/docs/tasks/administer-cluster/apply-resource-quota-limit/)。

#### 了解更多
参阅资[源配额设计文档](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/admission_control_resource_quota.md)来了解更多信息。