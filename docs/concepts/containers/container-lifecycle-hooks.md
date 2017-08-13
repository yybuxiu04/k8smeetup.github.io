---
assignees:
- mikedanese
- thockin
title: 容器生命周期的钩子
redirect_from:
- "/docs/user-guide/container-environment/"
- "/docs/user-guide/container-environment.html"
---

本页描述了被`kubelet`管理的容器是如何使用容器生命周期的钩子框架通过其生命周期内的事件触发来运行代码的。


## 概述

类似于许多具有生命周期钩子组件的编程语言框架，比如[Angular](https://github.com/angular),
Kubernetes为容器提供了生命周期钩子。  
钩子能使容器感知其生命周期内的事件并且当相应的生命周期钩子被调用时运行在处理程序中实现的代码。


## 容器钩子

容器中有两个钩子：

`PostStart`

这个钩子在容器创建后立即执行。  
但是，并不能保证钩子将在容器ENTRYPOINT之前运行。  
没有参数传递给处理程序。


`PreStop`

这个钩子在容器终止之前立即被调用。  
它是阻塞的，意味着它是同步的，
所以它必须在删除容器的调用发出之前完成。


更多终止行为的详细描述可以在这找到：[Termination of Pods](/docs/concepts/workloads/pods/pod/#termination-of-pods).


### 钩子处理程序的实现

容器可以通过实现和注册该钩子的处理程序来访问钩子。  
可以为容器实现两种类型的钩子处理程序：


* Exec - 在容器的cgroups和命名空间内执行一个特定的命令，比如`pre-stop.sh`。  
该命令消耗的资源被计入容器。  
* HTTP -  对容器上的特定的端点执行HTTP请求。


### 钩子处理程序的执行

当容器生命周期管理钩子被调用后，Kubernetes管理系统执行该钩子在容器中注册的处理程序。


在含有容器的Pod的上下文中钩子处理程序的调用是同步的。  
这意味着对于`PostStart`钩子，
容器ENTRYPOINT和钩子执行是异步的。  
然而，如果钩子花费太长时间以至于不能运行或者挂起，
容器将不能达到`running`状态。


`PreStop`钩子的行为是类似的。  
如果钩子在执行期间挂起，
Pod阶段将停留在`running`状态并且永不会达到`failed`状态。  
如果`PostStart`或者`PreStop`钩子失败，
它会杀死容器。


用户应该使他们的钩子处理程序尽可能的轻量。  
然而，有些情况下，长时间运行命令是合理的，
比如在停止容器之前预先保存状态。


### 钩子交付保证

钩子交付打算*至少一次*,这意味着对于给定的事件，一个钩子可能被多次调用，
例如对于`PostStart`或者`PreStop`。  
它是由钩子的实现来正确的处理这个。


通常，只有单次交付。  
例如，如果一个HTTP钩子的接收者挂掉不能接收流量，
该钩子不会尝试重新发送。  
然而，在一些极不常见的情况下，可能发生两次交付。  
例如，如果在发送钩子的途中kubelet重启，
该钩子可能在kubelet启动之后重新发送。


### 调试钩子处理程序

在Pod的事件中没有钩子处理程序的日志。
如果一个处理程序因为某些原因运行失败，它广播一个事件。  
对于`PostStart`, 这是`FailedPostStartHook`事件，
对于`PreStop`, 这是`FailedPreStopHook`事件。  
你可以通过运行`kubectl describe pod <pod_name>`来查看这些事件。  
下面是运行这条命令的输出示例：
```
Events:
  FirstSeen    LastSeen    Count    From                            SubobjectPath        Type        Reason        Message
  ---------    --------    -----    ----                            -------------        --------    ------        -------
  1m        1m        1    {default-scheduler }                                Normal        Scheduled    Successfully assigned test-1730497541-cq1d2 to gke-test-cluster-default-pool-a07e5d30-siqd
  1m        1m        1    {kubelet gke-test-cluster-default-pool-a07e5d30-siqd}    spec.containers{main}    Normal        Pulling        pulling image "test:1.0"
  1m        1m        1    {kubelet gke-test-cluster-default-pool-a07e5d30-siqd}    spec.containers{main}    Normal        Created        Created container with docker id 5c6a256a2567; Security:[seccomp=unconfined]
  1m        1m        1    {kubelet gke-test-cluster-default-pool-a07e5d30-siqd}    spec.containers{main}    Normal        Pulled        Successfully pulled image "test:1.0"
  1m        1m        1    {kubelet gke-test-cluster-default-pool-a07e5d30-siqd}    spec.containers{main}    Normal        Started        Started container with docker id 5c6a256a2567
  38s        38s        1    {kubelet gke-test-cluster-default-pool-a07e5d30-siqd}    spec.containers{main}    Normal        Killing        Killing container with docker id 5c6a256a2567: PostStart handler: Error executing in Docker Container: 1
  37s        37s        1    {kubelet gke-test-cluster-default-pool-a07e5d30-siqd}    spec.containers{main}    Normal        Killing        Killing container with docker id 8df9fdfd7054: PostStart handler: Error executing in Docker Container: 1
  38s        37s        2    {kubelet gke-test-cluster-default-pool-a07e5d30-siqd}                Warning        FailedSync    Error syncing pod, skipping: failed to "StartContainer" for "main" with RunContainerError: "PostStart handler: Error executing in Docker Container: 1"
  1m         22s         2     {kubelet gke-test-cluster-default-pool-a07e5d30-siqd}    spec.containers{main}    Warning        FailedPostStartHook    
```




* 了解更多关于 [Container environment](/docs/concepts/containers/container-environment-variables/)。
* 亲身体验 [attaching handlers to Container lifecycle events](/docs/tasks/configure-pod-container/attach-handler-lifecycle-event/)。
