---
title: 联邦命名空间  
redirect_from:
- "/docs/user-guide/federation/namespaces/"
- "/docs/user-guide/federation/namespaces.html"
---


{% capture overview %}

本指南介绍了如何在联邦的控制面板上使用命名空间。

联邦控制面板中的命名空间（在本指南中简称为“联邦命名空间”）与传统的Kubernetes命名空间十分相似，提供了相同的功能。  
在联邦控制面板中创建他们可确保他们在联邦的所有集群中是同步的。

{% endcapture %}

{% capture prerequisites %}

* {% include federated-task-tutorial-prereqs.md %}


* 你应该有一些基本的[Kubernetes的大致工作知识](/docs/setup/pick-right-solution/)尤其是[命名空间](/docs/concepts/overview/working-with-objects/namespaces/)。

{% endcapture %}

{% capture steps %}



## 创建一个联邦命名空间

联邦命名空间的API与传统的Kubernetes命名空间百分百兼容。你可以通过发送一个请求到联邦apiserver来创建一个命名空间。

你可以使用kubectl运行如下的命令来实现它：
``` shell
kubectl --context=federation-cluster create -f myns.yaml
```


这个'--context=federation-cluster'标志告诉kubectl提交这个请求到联邦apiserver而不是发送到一个Kubernetes集群。

一旦一个联邦命名空间被创建了，联邦控制面板将在它下面的所有Kubernetes集群中创建一个匹配的命名空间。  
你可以通过检查它下面的每一个集群来验证它，例如：

``` shell
kubectl --context=gce-asia-east1a get namespaces myns
```


上面假设你已经在那个区域中为你的集群在你的客户端配置了一个名称为'gce-asia-east1a'的上下文。  
这个底层命名空间的名称和规范将和你上面创建的联邦命名空间的这些匹配。


## 更新一个联邦命名空间

你可以更新一个联邦命名空间就像你更新一个Kubernetes命名空间一样，仅仅发送这个请求到联邦apiserver而不是发送它到一个特定的Kubernetes集群。  
联邦控制面板将会确保每当联邦命名空间更新后，更新所有底层集群中相应的命名空间来匹配它。


## 删除一个联邦命名空间

你可以删除一个联邦命名空间就像你删除一个Kubernetes命名空间那样，仅仅发送这个请求到联邦apiserver而不是发送它到一个特定的Kubernetes集群。

例如，你可以使用kubectl运行下面的命令来实现：

```shell
kubectl --context=federation-cluster delete ns myns
```


与Kubernetes一样，删除一个联邦命名空间将会删除来自联邦控制面板的命名空间的所有资源。

请注意，此时删除一个联邦命名空间将不会删除那些命名空间中来自底层集群的相应的命名空间和资源。
用户需要手动地删除它们。我们打算在将来解决这个问题。

{% endcapture %}

{% include templates/task.md %}
