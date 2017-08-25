---
title: 使用聚合层扩展Kubernetes API  
assignees:
- lavalamp
- cheftako
- chenopis
---


聚合层允许Kubernetes使用额外的APIs进行扩展，超出了Kubernetes核心APIs所提供的范围。


## 概述

聚合层允许在你的集群中安装更多的Kubernetes风格的APIs。这些可以是预构建的，现有的第三方解决方案，例如[service-catalog](https://github.com/kubernetes-incubator/service-catalog/blob/master/README.md),
或者是可以让你使用的用户创建的APIs，如[apiserver-builder](https://github.com/kubernetes-incubator/apiserver-builder/blob/master/README.md)。


在1.7版本中，聚合层和kube-api-server一起运行。在扩展资源被注册前，聚合层不执行任何操作。
要注册其API,用户必选添加一个APIService对象，该对象需在Kubernetes API中声明URL路径。
在这一点上，聚合层将代理发送到该API路径(e.g. /apis/myextension.mycompany.io/v1/…)的一切到注册的APIService。

通常，通过在集群中的一个pod中运行一个*extension-apiserver*来实现APIService。如果已添加的资源需要主动管理，这个extension-apiserver通常需要和一个或多个控制器配对。
因此，apiserver构建器实际上为两者提供了一个架构。
另一个例子，当service-catalog安装后，它为它提供的服务提供了extension-apiserver和控制器。


* 使聚合层在你的环境中工作，[配置聚合层](/docs/tasks/access-kubernetes-api/configure-aggregation-layer/)。
* 然后，[设置拓展api-server](/docs/tasks/access-kubernetes-api/setup-extension-api-server/) 与聚合层一起工作.
* 另外，学习如何[使用CRD扩展Kubernetes API](/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/)。
