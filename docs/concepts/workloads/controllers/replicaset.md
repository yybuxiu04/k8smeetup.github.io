---
assignees:
- Kashomon
- bprashanth
- madhusudancs
title: Replica Sets
redirect_from:
- "/docs/user-guide/replicasets/"
- "/docs/user-guide/replicasets.html"
---

* TOC
{:toc}



## 什么是ReplicaSet
ReplicaSet 是下一代的Replication Controller.一个_ReplicaSet_ 和一个 [_Replication Controller_](/docs/concepts/workloads/controllers/replicationcontroller/)
之间唯一的不同目前是对选择器的支持. ReplicaSet 支持最新的基于集合的选择器需求,这描述在[标签用户指南](/docs/user-guide/labels/#label-selectors)然而一个Replication Controller
仅仅支持基于等号的选择器需求.

大部分的[`kubectl`](/docs/user-guide/kubectl/)命令不仅支持Replication Controllers也支持ReplicaSets.一个例外是[`rolling-update`](/docs/user-guide/kubectl/{{page.version}}/#rolling-update)命令.
如果你想功能上滚动升级,请考虑使用Deployments来替代.并且[`rolling-update`](/docs/user-guide/kubectl/{{page.version}}/#rolling-update)是命令式的而Deployments则是陈述式的,所以我们推荐
通过[`rollout`](/docs/user-guide/kubectl/{{page.version}}/#rollout)这个命令来使用Deployments.

当ReplicaSets能够被独立地使用的时候,今天它主要地被用在[Deployments](/docs/concepts/workloads/controllers/deployment/)上作为精心策划pod创建,删除和升级的一个机制.当你使用Deployments的时候,你无需去
担心Deployments建立的ReplicaSets怎么去管理.Deployments 拥有和管理他们自己的ReplicaSets.




## 什么时候去使用一个ReplicaSet
一个ReplicaSet保证pod副本为一个指定的数目在给定的任何时间内.然而,一个Deployment是一个更高级别的概念来去管理ReplicaSets
和提供描述性的pods升级以及很多其他有用的特性.因此,我们推荐使用Deployments来替代直接使用ReplicaSets,除非你需要定制的更新编排
或者一点也不需要更新.
这个事实上意味着你可能从不需要操作ReplicaSet对象：
直接使用一个Deployment然后在声明部分定义你的应用.
## 例如

{% include code.html language="yaml" file="frontend.yaml" ghlink="/docs/concepts/workloads/controllers/frontend.yaml" %}


保存这个配置在frontend.yaml文件里并且提交它到一个kubernetes集群可以创建一个定义的ReplicaSet和pods来管理.

```shell
$ kubectl create -f frontend.yaml
replicaset "frontend" created
$ kubectl describe rs/frontend
Name:          frontend
Namespace:     default
Image(s):      gcr.io/google_samples/gb-frontend:v3
Selector:      tier=frontend,tier in (frontend)
Labels:        app=guestbook,tier=frontend
Replicas:      3 current / 3 desired
Pods Status:   3 Running / 0 Waiting / 0 Succeeded / 0 Failed
No volumes.
Events:
  FirstSeen    LastSeen    Count    From                SubobjectPath    Type        Reason            Message
  ---------    --------    -----    ----                -------------    --------    ------            -------
  1m           1m          1        {replicaset-controller }             Normal      SuccessfulCreate  Created pod: frontend-qhloh
  1m           1m          1        {replicaset-controller }             Normal      SuccessfulCreate  Created pod: frontend-dnjpy
  1m           1m          1        {replicaset-controller }             Normal      SuccessfulCreate  Created pod: frontend-9si5l
$ kubectl get pods
NAME             READY     STATUS    RESTARTS   AGE
frontend-9si5l   1/1       Running   0          1m
frontend-dnjpy   1/1       Running   0          1m
frontend-qhloh   1/1       Running   0          1m
```


## ReplicaSet 作为一个pod水平自动扩展的服务对象.
一个ReplicaSet也能是一个[Horizontal Pod Autoscalers (HPA)](/docs/tasks/run-application/horizontal-pod-autoscale/)的目标.
一个ReplicaSet能够通过HPA来完成自动扩展或者收缩.这里有一个基于我们前一个创建的ReplicaSet来使用HPA的例子.

{% include code.html language="yaml" file="hpa-rs.yaml" ghlink="/docs/concepts/workloads/controllers/hpa-rs.yaml" %}


保存配置到hpa-rs.yaml然后提交它到一个kubernetes集群可以创建一个定义好的HPA基于副本pods的CPU使用率上来自动扩展这个目标ReplicaSet.

```shell
kubectl create -f hpa-rs.yaml
```

或者,你可以仅仅使用`kubectl autoscale`命令去实现相同的目的.(这个更容易些)

```shell
kubectl autoscale rs frontend
```
