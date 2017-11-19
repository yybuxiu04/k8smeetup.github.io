---
approvers:
- davidopp
- kevin-wangzefeng
- bsalamat
cn-approvers:
- linyouchong
title: Taint 和 Toleration
---



节点亲和性（详见[这里](/docs/concepts/configuration/assign-pod-node/#node-affinity-beta-feature)），是 *pod* 的一种属性（偏好或硬性要求），它使 *pod* 被吸引到一类特定的节点。Taint 则相反，它使一个 *节点* 能够 *排斥* 一类特定的 pod。


taint 和 toleration 共同工作以确保 pod 不被分配到不合适的节点。一个或多个 taint 应用于一个节点，表示该节点不应接受任何不能忍受这些 taint 的 pod 。toleration 应用于 pod ，表示这些 pod 可以被分配到具有相匹配的 taint 的节点。

* TOC
{:toc}


## 概念


您可以使用 [kubectl taint](/docs/user-guide/kubectl/{{page.version}}/#taint) 命令给一个节点增加一个 taint。比如，

```shell
kubectl taint nodes node1 key=value:NoSchedule
```


给节点 `node1` 增加一个 taint，它的 key 是 `key`，value 是 `value`，effect 是 `NoSchedule`。这表示只有拥有和这个 taint 相匹配的 toleration 的 pod 才能够被分配到 `node1` 这个节点。您可以在 PodSpec 中定义一个 pod 的 toleration。下面两个 toleration 均与上面例子中使用 `kubectl taint` 命令创建的 taint 相匹配，因此如果一个 pod 拥有其中的任何一个 toleration 都能够被分配到 `node1` ：

```yaml
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
```

```yaml
tolerations:
- key: "key"
  operator: "Exists"
  effect: "NoSchedule"
```


一个 toleration 和一个 taint 相“匹配”是指它们有一样的 key 和 effect ，并且：
* 如果 `operator` 是 `Exists` （此时 toleration 不能指定 `value`），或者
* 如果 `operator` 是 `Equal` ，则它们的 `value` 应该相等


**注意：** 存在两种特殊情况：

* 如果一个 toleration 的 `key` 为空且 operator 为 `Exists` ，表示这个 toleration 与任意的 key 、 value 和 effect 都匹配，即这个 toleration 能忍受任意 taint。

```yaml
tolerations:
- operator: "Exists"
```


* 如果一个 toleration 的 `effect` 为空，则 `key` 值与之相同的相匹配 taint 的 `effect` 可以是任意值。

```yaml
tolerations:
- key: "key"
  operator: "Exists"
```


上述例子使用到的 `effect` 的一个值 `NoSchedule`，您也可以使用另外一个值 `PreferNoSchedule`。这是“优化”或“软”版本的 `NoSchedule`  -- 系统会*尽量*避免将 pod 到分配到存在其不能忍受的 taint 的节点上，但这不是强制的。`effect` 的值还可以设置为 `NoExecute` ，下文会详细描述这个值。


您可以给一个节点添加多个 taint ，也可以给一个 pod 添加多个 toleration。Kubernetes 处理多个 taint 和 toleration 的过程就像一个过滤器：从一个节点的所有 taint 开始遍历，过滤掉那些 pod 中存在与之相匹配的 toleration 的 taint。余下未被过滤的 taint 的 effect 值决定了 pod 是否会被分配到该节点，特别是以下情况：


* 如果未被过滤的 taint 中存在一个以上 effect 值为 `NoSchedule` 的 taint，则 Kubernetes 不会将 pod 分配到该节点。
* 如果未被过滤的 taint 中不存在 effect 值为 `NoSchedule` 的 taint，但是存在 effect 值为 `PreferNoSchedule` 的 taint，则 Kubernetes 会*尝试*将 pod 分配到该节点。
* 如果未被过滤的 taint 中存在一个以上 effect 值为 `NoExecute` 的 taint，则 Kubernetes 不会将 pod 分配到该节点（如果 pod 还未在节点上运行），或者将 pod 从该节点驱逐（如果 pod 已经在节点上运行）。


例如，假设您给一个节点添加了如下的 taint

```shell
kubectl taint nodes node1 key1=value1:NoSchedule
kubectl taint nodes node1 key1=value1:NoExecute
kubectl taint nodes node1 key2=value2:NoSchedule
```


然后存在一个 pod，它有两个 toleration

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
```


在这个例子中，上述 pod 不会被分配到上述节点，因为其没有 toleration 和第三个 taint 相匹配。但是如果在给节点添加 上述 taint 之前，该 pod 已经在上述节点运行，那么它还可以继续运行在该节点上，因为第三个 taint 是三个 taint 中唯一不能被这个 pod 忍受的。


通常情况下，如果给一个节点添加了一个 effect 值为 `NoExecute` 的taint，则任何不能忍受这个 taint 的 pod 都会马上被驱逐，任何可以忍受这个 taint 的 pod 都不会被驱逐。但是，如果 pod 存在一个 effect 值为 `NoExecute` 的 toleration 指定了可选属性 `tolerationSeconds` 的值，则表示在给节点添加了上述 taint 之后，pod 还能继续在节点上运行的时间。例如，

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
  tolerationSeconds: 3600
```


这表示如果这个 pod 正在运行，然后一个匹配的 taint 被添加到其所在的节点，那么 pod 还将继续在节点上运行 3600 秒，然后被驱逐。如果在此之前上述 taint 被删除了，则 pod 不会被驱逐。


## 使用例子


通过 taint 和 toleration ，可以灵活地让 pod *避开*某些节点或者将 pod 从某些节点驱逐。下面是几个使用例子：


* **专用节点**: 如果您想将某些节点专门分配给特定的一组用户使用，您可以给这些节点添加一个 taint（即，
`kubectl taint nodes nodename dedicated=groupName:NoSchedule`），然后给这组用户的 pod 添加一个相对应的 toleration（通过编写一个自定义的[admission controller](/docs/admin/admission-controllers/)，很容易就能做到）。拥有上述 toleration 的 pod 就能够被分配到上述专用节点，同时也能够被分配到集群中的其它节点。如果您希望这些 pod 只能被分配到上述专用节点，那么您还需要给这些专用节点另外添加一个和上述 taint 类似的 label （例如：`dedicated=groupName`），同时 还要在上述 admission controller 中给 pod 增加节点亲和性要求上述 pod 只能被分配到添加了 `dedicated=groupName` 标签的节点上。


* **配备了特殊硬件的节点**: 在部分节点配备了特殊硬件（比如 GPU）的集群中，我们希望不使用这类硬件的 pod 不要被分配到这些特殊节点，以便为后继需要这类硬件的 pod 保留资源。要达到这个目的，可以先给配备了特殊硬件的节点添加 taint（例如 `kubectl taint nodes nodename special=true:NoSchedule` or `kubectl taint nodes nodename special=true:PreferNoSchedule`)，然后给使用了这类特殊硬件的 pod 添加一个相匹配的 toleration。和专用节点的例子类似，添加这个 toleration 的最简单的方法是使用自定义 [admission controller](/docs/admin/admission-controllers/)。比如，admission controller 可以从 pod 的一些特点判断出该 pod 应该被允许分配到一些特殊节点，然后 admission controller 就给 pod 添加对应的 toleration。为了保证使用了特殊硬件的 pod *只*被分配到配备了特殊硬件的节点，您还需要做一些额外的工作，比如，您可以使用[opaque integer resources](/docs/concepts/configuration/manage-compute-resources-container/#opaque-integer-resources-alpha-feature)表示这类特殊资源，然后在 PodSpec 中申请使用它；或者您也可以给配备了特殊硬件的节点添加 label，然后给需要特殊硬件的 pod 配置节点亲和性。


* **基于 taint 的驱逐 （alpha 特性）**: 这是在每个 pod 中配置的在节点出现问题时的驱逐行为，接下来的章节会描述这个特性


## 基于 taint 的驱逐


前文我们提到过 taint 的 effect 值 `NoExecute`  , 它会影响已经在节点上运行的 pod

 * 如果 pod 不能忍受effect 值为 `NoExecute` 的 taint，那么 pod 将马上被驱逐
 * 如果 pod 能够忍受effect 值为 `NoExecute` 的 taint，但是在 toleration 定义中没有指定 `tolerationSeconds`，则 pod 还会一直在这个节点上运行。
 * 如果 pod 能够忍受effect 值为 `NoExecute` 的 taint，而且指定了 `tolerationSeconds`，则 pod 还能在这个节点上继续运行这个指定的时间长度。


上述特性行为目前处于 beta 阶段。此外，Kubernetes 1.6 已经支持（alpha阶段）节点问题的表示。换句话说，当某种条件为真时，node controller会自动给节点添加一个 taint。当前内置的 taint 包括：

 * `node.kubernetes.io/not-ready`：节点未准备好。这相当于节点状态 `Ready` 的值为 "`False`"。
 * `node.alpha.kubernetes.io/unreachable`：node controller 访问不到节点. 这相当于节点状态 `Ready` 的值为 "`Unknown`"。
 * `node.kubernetes.io/out-of-disk`：节点磁盘耗尽。
 * `node.kubernetes.io/memory-pressure`：节点存在内存压力。
 * `node.kubernetes.io/disk-pressure`：节点存在磁盘压力。
 * `node.kubernetes.io/network-unavailable`：节点网络不可用。
 * `node.cloudprovider.kubernetes.io/uninitialized`：如果 kubelet 启动时指定了一个 "外部" cloud provider，它将给当前节点添加一个 taint 将其标志为不可用。在 cloud-controller-manager 的一个 controller 初始化这个节点后，kubelet 将删除这个 taint。


在启用了 `TaintBasedEvictions` 这个 alpha 功能特性后（在 Kubernetes controller manager 的 `--feature-gates` 参数中包含`TaintBasedEvictions=true` 开启这个功能特性，例如：`--feature-gates=FooBar=true,TaintBasedEvictions=true`），NodeController (或 kubelet)会自动给节点添加这类 taint，上述基于节点状态 Ready 对 pod 进行驱逐的逻辑会被禁用。
（注意：为了保证由于节点问题引起的 pod 驱逐[rate limiting](/docs/concepts/architecture/nodes/)行为正常，系统实际上会以 rate-limited 的方式添加 taint。在像 master 和 node 通讯中断等场景下，这避免了 pod 被大量驱逐。使用这个 alpha 功能特性，结合 `tolerationSeconds` ，pod 就可以指定当节点出现一个或全部上述问题时还将在这个节点上运行多长的时间。


比如，一个使用了很多本地状态的应用程序在网络断开时，仍然希望停留在当前节点上运行一段较长的时间，愿意等待网络恢复以避免被驱逐。在这种情况下，pod 的 toleration 可能是下面这样的：

```yaml
tolerations:
- key: "node.alpha.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 6000
```


注意，Kubernetes 会自动给 pod 添加一个 key 为 `node.kubernetes.io/not-ready` 的 toleration 并配置 `tolerationSeconds=300`，除非用户提供的 pod 配置中已经已存在了 key 为 `node.kubernetes.io/not-ready` 的 toleration。同样，Kubernetes 会给 pod 添加一个 key 为 `node.kubernetes.io/unreachable` 的 toleration 并配置 `tolerationSeconds=300`，除非用户提供的 pod 配置中已经已存在了 key 为 `node.kubernetes.io/unreachable` 的 toleration。


这种自动添加 toleration 机制保证了在其中一种问题被检测到时 pod 默认能够继续停留在当前节点运行 5 分钟。这两个默认 toleration 是由 [DefaultTolerationSeconds
admission controller](https://git.k8s.io/kubernetes/plugin/pkg/admission/defaulttolerationseconds)添加的。


[DaemonSet](/docs/concepts/workloads/controllers/daemonset/) 中的 pod 被创建时，针对以下 taint 自动添加的 `NoExecute` 的 toleration 将不会指定 `tolerationSeconds`：

  * `node.alpha.kubernetes.io/unreachable`
  * `node.kubernetes.io/not-ready`

这保证了出现上述问题时 DaemonSet 中的 pod 永远不会被驱逐，这和 `TaintBasedEvictions` 这个特性被禁用后的行为是一样的。


## 基于节点状态添加 taint


1.8 版本引入了一个 alpha 功能特性，该特性使 node controller 根据节点状态创建相应的 taint。当启用了该功能特性（您可以通过在 scheduler 的命令行参数 `--feature-gates` 中包含 `TaintNodesByCondition=true` 来开启这个功能，例如 `--feature-gates=FooBar=true,TaintNodesByCondition=true`），scheduler 不会检查节点状态；scheduler 检查的是 taint。这保证了节点状态不会影响到哪些 pod 会被分配到节点。用户可以通过给 pod 添加适当的 toleration 来忽略节点的一些故障（表示为节点状态）。


为了保证开启这个功能特性不会对 DaemonSet 造成破坏，从 1.8 版本开始，DaemonSet controller 会自动地给所有的 daemon 添加如下 effect 为 `NoSchedule` 的 toleration：

  * `node.kubernetes.io/memory-pressure`
  * `node.kubernetes.io/disk-pressure`
  * `node.kubernetes.io/out-of-disk` (*只适合 critical pod*)


上述设置确保了向后兼容，但我们需要明白它们可能不符合所有用户的需求，这就是为什么集群管理员还可以选择自由的向 DaemonSet 添加 toleration。
