---
title: 使用CRD(CustomResourceDefinitions)扩展Kubernetes API  
assignees:
- deads2k
- enisoc
---


本文展示了如何通过创建一个CRD来安装一个自定义资源到Kubernetes API中。  


* 阅读[自定义资源](/docs/concepts/api-extension/custom-resources/)。
* 确保你的Kubernetes集群的主版本为1.7.0或者更高版本。


## 创建一个CRD

当创建一个新的*自定义资源定义*（CRD）时，Kubernetes API Server 通过创建一个新的RESTful资源路径进行应答，无论是在命名空间还是在集群范围内，正如在CRD的`scope`域指定的那样。  
与现有的内建对象一样，删除一个命名空间将会删除该命名空间内所有的自定义对象。  
CRD本身并不区分命名空间，对所有的命名空间可用。

例如，如果将以下的CRD保存到`resourcedefinition.yaml`中:
```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: crontabs.stable.example.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: stable.example.com
  # version name to use for REST API: /apis/<group>/<version>
  version: v1
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # singular name to be used as an alias on the CLI and for display
    singular: crontab
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: CronTab
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ct
```

创建它：

```shell
kubectl create -f resourcedefinition.yaml
```


然后一个新的区分命名空间的RESTful API 端点被创建了：

```
/apis/stable.example.com/v1/namespaces/*/crontabs/...
```


然后可以使用此端点URL来创建和管理自定义对象。
这些对象的`kind`就是你在上面创建的CRD中指定的`CronTab`对象。


## 创建自定义对象

在CRD对象创建完成之后，你可以创建自定义对象了。自定义对象可以包含自定义的字段。这些字段可以包含任意的JSON。  
以下的示例中，在一个自定义对象`CronTab`种类中设置了`cronSpec`和`image`字段。这个`CronTab`种类来自于你在上面创建的CRD对象。

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * /5"
  image: my-awesome-cron-image
```


创建它：

```shell
kubectl create -f my-crontab.yaml
```


你可以使用kubectl来管理你的CronTab对象。例如：

```shell
kubectl get crontab
```


应该打印这样的一个列表：

```console
NAME                 KIND
my-new-cron-object   CronTab.v1.stable.example.com
```


注意当使用kubectl时，资源名称是大小写不敏感的，你可以使用单数或者复数形式以及任何缩写在CRD中定义资源。

你还可以查看原始的JSON数据：

```shell
kubectl get ct -o yaml
```


你应该可以看到它包含了自定义的`cronSpec`和`image`字段，来自于你用于创建它的yaml:

```console
apiVersion: v1
items:
- apiVersion: stable.example.com/v1
  kind: CronTab
  metadata:
    clusterName: ""
    creationTimestamp: 2017-05-31T12:56:35Z
    deletionGracePeriodSeconds: null
    deletionTimestamp: null
    name: my-new-cron-object
    namespace: default
    resourceVersion: "285"
    selfLink: /apis/stable.example.com/v1/namespaces/default/crontabs/my-new-cron-object
    uid: 9423255b-4600-11e7-af6a-28d2447dc82b
  spec:
    cronSpec: '* * * * /5'
    image: my-awesome-cron-image
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```


## 高级话题

### 终止器

*终止器*允许控制器实现异步的预删除钩子。
自定义对象支持终止器就像内建对象一样。

你可以给一个自定义对象添加一个终止器，如下所示：

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  finalizers:
  - finalizer.stable.example.com
```


对于具有终止器的一个对象，第一个删除请求仅仅是为`metadata.deletionTimestamp`字段设置一个值，而不是删除它。  
这将触发监控该对象的控制器执行他们所能处理的任意终止器。

然后，每一个控制器从列表中删除它的终止器并再一次发出删除请求。
如果这个终止器列表现在是空的，则此请求仅删除该对象，意味着所有的终止器都完成了。

* 学习如何[迁移第三方资源到CRD](/docs/tasks/access-kubernetes-api/migrate-third-party-resource/)。

