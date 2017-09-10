---
approvers:
- bprashanth
- erictune
- foxish
- smarterclayton
title: 强制删除StatefulSet中的Pod
---

{% capture overview %}

本文档说明了如何删除StatefulSet中的Pod，并解释了在删除操作时需要牢记的注意事项。
{% endcapture %}

{% capture prerequisites %}


* 本任务相对高阶，包含一些可能违背StatefulSet固有属性或者常规规则的内容。
* 在阅读前，请您先熟悉以下列出的注意事项。

{% endcapture %}

{% capture steps %}



## 有关StatefulSet的注意事项


在正常的StatefulSet操作中，**从不**需要强制删除StatefulSet中的Pod。StatefulSet控制器（StatefulSet Controller）负责创建、伸缩和删除StatefulSet中的所有成员。控制器将确保序号从0到N-1的Pod处于活跃健康的状态，而StatefulSet保证了在任何时候，集群中至多有一个拥有给定ID的Pod存在。这一特性被称为StatefulSet的*至多一个*原则。


手动强制删除应谨慎进行，因为这种操作导致的结果可能违背了StatefulSet所固有的至多一个原则。StatefulSet可被用于运行需要稳定网络关联关系和稳定持久化存储的集群化或分布式应用程序。通常，这些应用程序的配置需要依赖一个拥有固定成员数量的集群，而这些集群成员自己也需要拥有固定ID从而能够在集群中识别它们。如果多个集群中的成员使用相同的ID可能引发灾难性的后果并可能导致数据丢失（例如在基于法定数量的系统中出现分片等情况）。


## 删除Pod


以下命令可以优雅地删除Pod（graceful delete）：

```shell
kubectl delete pods <pod>
```


为了能够顺利使用上述命令终止Pod，Pod的属性`pod.Spec.TerminationGracePeriodSeconds`**不能**被设置为0。设置`pod.Spec.TerminationGracePeriodSeconds`为0的做法对于StatefulSet的Pod来讲是不安全的，强烈不推荐使用。上述Graceful Delete的方式是安全的，并可保证在kubelet删除apiserver上的数据之前[Pod被顺利终止](/docs/user-guide/pods/#termination-of-pods)。


Kubernetes(1.5或者更高版本)不会因为集群节点不可达（unreachable node）而删除Pod。在不可达节点上运行的Pod在[超时](/docs/admin/node/#node-condition)后将进入'Terminating'或者'Unknown'状态。如果用户尝试使用Graceful Delete删除不可达节点上的Pod时，Pod可能进入这些状态。以下列举了可以从apiserver上删除处于这些状态的Pod的方法：
   * 删除节点对象（可以由您删除，也可以由[节点控制器(Node Controller)](/docs/admin/node)删除）。
   * 不可达节点上的kubelet开始响应，终止了Pod并从apiserver上删除了相关数据条目。
   * 用户强制删除Pod。


推荐的最佳做法是使用上述第一或者第二种方法。如果节点被确认为死机（例如永久断开网络连接，掉电等），则删除节点对象。如果有网络分片发生，则应先解决分片问题或者等待分片问题解决。网络分片问题解决后，kubelet将完成对Pod的删除并删除apiserver上的相关数据。


通常而言，一旦Pod不再在节点上运行或者节点被管理员删除，系统将会完成删除操作。您也可以通过强制删除来完成删除操作。


### 强制删除


强制删除**并不**等待kubelet确认待删除的Pod已经终止。无论是否成功终止了Pod，强制删除都会立即删除apiserver上的相关对象和数据。这会让StatefulSet控制器创建一个具有相同ID的Pod来替代被删除的Pod。这一过程可能导致具有重复ID的Pod同时出现在集群中，而此时被删除的Pod如果仍能与StatefulSet中的其他成员通信的话，则违背了StatefulSet所维护的至多一个原则。


在您强制删除StatefulSet中的Pod时，请务必确认待删除的Pod将永不与StatefulSet中的其他Pod通信并且其数据可以安全地从apiserver中删除从而能让StatefulSet控制器创建一个新的Pod来替换待删除的Pod。


若使用1.5或者更高版本的kubectl强制删除Pod，请执行以下命令：

```shell
kubectl delete pods <pod> --grace-period=0 --force
```


如果您使用1.4或者更低版本的kubectl，需要省略`--force`选项：

```shell
kubectl delete pods <pod> --grace-period=0
```


在强制删除StatefulSet的Pod前，请务必充分了解这一操作所带来的各种风险。

{% endcapture %}

{% capture whatsnext %}


更多信息请参阅[StatefulSet的调试](/docs/tasks/manage-stateful-set/debugging-a-statefulset/)。

{% endcapture %}

{% include templates/task.md %}
