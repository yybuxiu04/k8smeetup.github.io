---
approvers:
- davidopp
- mml
- foxish
- kow3ns
title: 遵循应用中断预算安全移除节点
---

{% capture overview %}

本页展示如何遵循使用 PodDisruptionBudget 指定的应用级中断预算安全地移除节点。
{% endcapture %}

{% capture prerequisites %}


操作前，要求下列先决条件已经满足：


* 您正在使用的 Kubernetes 必须是 1.5 或者更高版本。

* 下面都要求满足：
  1. 在节点删除期间，不能要求您的应用是高可用的
  1. 您已经了解了 [PodDisruptionBudget 的概念](/docs/concepts/workloads/pods/disruptions/)，并且为有需要的应用[配置了 PodDisruptionBudgets](/docs/tasks/run-application/configure-pdb/)。

{% endcapture %}

{% capture steps %}


## 使用 `kubectl drain` 从集群中移除节点


对节点执行维护操作之前（例如：内核升级，硬件维护等），您可以使用 `kubectl drain` 安全驱逐节点上面所有的 pod。安全驱逐的方式将会允许 pod 里面的容器遵循指定的 `PodDisruptionBudgets` 执行[优雅的中止](/docs/tasks/#lifecycle-hooks-and-termination-notice)。


**注：** 默认情况下，`kubectl drain` 会忽略那些不能杀死的系统类型的 pod，如果您想了解更多详细的内容，请参考[kubectl drain](/docs/user-guide/kubectl/{{page.version}}/#drain)


`kubectl drain` 返回成功表明所有的 pod （除了前面排除的那些）已经被安全驱逐（遵循期望优雅的中止期，并且没有违反任何应用程序级别的中断预算）。然后，通过对物理机断电或者在云平台上删除节点所在的虚拟机，都能安全的将节点移除。


首先，需要确定希望移除的节点的名称。您可以通过下面命令列出集群里面所有的节点：

```shell
kubectl get nodes
```


接下来，告知 Kubernetes 移除节点：

```shell
kubectl drain <node name>
```


执行完成后，如果没有任何错误返回，您可以关闭节点（如果是在云平台上，可以删除支持该节点的虚拟机）。如果在维护操作期间想要将节点留在集群，那么您需要运行下面命令：

```shell
kubectl uncordon <node name>
```

然后，它将告知 Kubernetes 允许调度新的 pod 到该节点。


## 并行的移除多个节点


`kubectl drain` 命令只允许一次处理单个节点。但是，您可以在不同终端或者以后台运行的方式，使用不同的节点名称运行多次 `kubectl drain` 命令。多个移除节点命令同时运行时，依然会遵循您指定的 `PodDisruptionBudget`。


例如，假如您有一个 StatefulSet，它有 3 个副本，并设置了一个指定 `minAvailable: 2` 的 `PodDisruptionBudget`。在 3 个 pod 都准备好的情况下，`kubectl drain` 只会从这个 StatefulSet 中驱逐一个 pod。如果有多个驱逐命令并行执行，Kubernetes 也会遵循该 PodDisruptionBudget 的设定，在任何时间内，都只有一个 pod 是不可用的。任何可能导致副本数低于该预算指定值的移除都会被阻塞。


## 驱逐 API


如果不喜欢使用 [kubectl drain](/docs/user-guide/kubectl/{{page.version}}/#drain)（例如，期望避免调用外部命令，或者期望对 pod 驱逐过程进行更好的控制），您也可以使用驱逐 API 通过编程的方式触发驱逐。


您首先应该熟悉使用 [Kubernetes 语言客户端](/docs/tasks/administer-cluster/access-cluster-api/#programmatic-access-to-the-api)。


一个 pod 的驱逐子资源可以被认为是一种政策-控制删除 pod 本身的操作。为了试图完成驱逐（也许更确切的说，是试图 *创建* 一个驱逐），您需要发布一个 POST 操作。下面是操作消息的示例：

```json
{
  "apiVersion": "policy/v1beta1",
  "kind": "Eviction",
  "metadata": {
    "name": "quux",
    "namespace": "default"
  }
}
```


您能够使用 `curl` 尝试驱逐：

```bash
$ curl -v -H 'Content-type: application/json' http://127.0.0.1:8080/api/v1/namespaces/default/pods/quux/eviction -d @eviction.json
```


API 将以下面三种方式之一进行响应：


- 如果驱逐是允许的，那么 pod 被删除完成，就好像您发送了一个删除 pod 的请求命令并且获得一个 `200 OK` 回应一样。

- 如果目前的状况不允许被预算中的规定所驱逐，您将获得 `429 Too Many Requests` 的回应。这通常是用于 *任何* 请求的通用速率限制，但这里我们的意思是 *现在* 不允许这种请求，以后可能允许。目前，请求者不会得到 *稍后重试* 的提示，但在以后的版本中可能会。

- 如果存在配置错误，比如多个预算指向同一个 pod，您将获得 `500 Internal Server Error` 的回应。


对于一个给定的驱逐请求，有两种情况：


- 没有可以匹配该 pod 的预算。在这种情况下，服务器总是返回 `200 OK`。

- 至少有一个预算满足要求。在这种情况下，上面三种响应都有可能返回。


在某些情况下，应用程序可能会达到一个失败的状态，在该状态下，它将永远不会返回除了 429 或者 500 以外的任何内容。例如，应用程序控制器创建的替代的 pod 没有准备好，或者最后一个被驱逐的 pod 有很长的中止宽限期，都可能发生这种情况。


在这种情况下，有两个可能的解决方案：


- 中止或者暂停自动化操作。分析卡住应用程序的原因，再重启自动化操作。
- 经过长时间等待之后，`删除` pod 而不是调用驱逐 API。


Kubernetes 没有具体说明在这种情况下应该怎么做；它取决于应用程序所有者和集群所有者建立的在这种情况下的行为协议。

{% endcapture %}

{% capture whatsnext %}


* 按照[配置 pod 中断预算](/docs/tasks/run-application/configure-pdb/)的步骤来保护您的应用程序。

{% endcapture %}

{% include templates/task.md %}
