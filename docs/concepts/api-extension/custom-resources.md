---
title: 自定义资源  
assignees:
- enisoc
- deads2k
---


本页阐释了*自定义资源*的概念，它是对Kubernetes API的扩展。

## 自定义资源


一种*资源*就是[Kubernetes API](/docs/reference/api-overview/)中的一个端点，它存储着某种[API 对象](/docs/concepts/overview/working-with-objects/kubernetes-objects/)的集合。    
例如，内建的*pods*资源包含Pod对象的集合。


*自定义资源*是对Kubernetes API的一种扩展，它对于每一个Kubernetes集群不一定可用。  
换句话说，它代表一个特定Kubernetes的定制化安装。


在一个运行中的集群内，自定义资源可以通过动态注册出现和消失，集群管理员可以独立于集群本身更新自定义资源。  
一旦安装了自定义资源，用户就可以通过[kubectl](/docs/user-guide/kubectl-overview/)创建和访问他的对象，就像操作内建资源*pods*那样。


## 自定义控制器

自定义资源本身让你简单地存储和索取结构化数据。  
只有当和*控制器*结合后，他们才成为一种真正的[declarative API](/docs/concepts/overview/working-with-objects/kubernetes-objects/#understanding-kubernetes-objects)。
控制器将结构化数据解释为用户所期望状态的记录，并且不断地采取行动来实现和维持该状态。


定制化资源是用户可以在运行中的集群内部署和更新的一个控制器，它独立于集群本身的生命周期。  
定制化控制器可以和任何一种资源一起工作，当和定制化资源结合使用时尤其有效。


## 定制化资源定义

[CustomResourceDefinition](/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/)
(CRD)是一个内建的API, 它提供了一个简单的方式来创建自定义资源。  
部署一个CRD到集群中使Kubernetes API服务端开始为你指定的自定义资源服务。


这使你不必再编写自己的API服务端来处理自定义资源，但是这种实现的一般性意味着比你使用[API server aggregation](#api-server-aggregation)缺乏灵活性。


## API 服务聚合

一般地，在Kubernetes API中的每一种资源都需要代码处理REST请求并且管理对象的持久化存储。  
主Kubernetes API服务处理内建的资源，比如*pod*和*services*，也能通过[CustomResourceDefinitions](#customresourcedefinitions)采用一般方式来处理自定义资源。


[聚合层](/docs/concepts/api-extension/)允许你通过编写和部署你自己的从API服务为你自定义的资源提供指定的实现。  
主API服务把这些该由你处理的自定义资源的请求委托给你，使它们对于它的所有客户端是可用的。


* 学习如何[使用聚合层扩展Kubernetes API](/docs/concepts/api-extension/apiserver-aggregation/)。
* 学习如何[使用定制化资源定义扩展Kubernetes API](/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/)。
* 学习如何[迁移第三方资源到定制化资源定义](/docs/tasks/access-kubernetes-api/migrate-third-party-resource/)。

