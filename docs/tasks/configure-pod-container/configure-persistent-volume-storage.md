---
标题: 配置Pod使用持久卷存储
---



{% capture overview %}

这篇教程指导如何配置Pod使用持久卷（PV）存储（以下简称PV），下面是步骤的总结：

1. 集群管理者创建一个基于物理存储之上的PV，但是没有分配给任何Pod。

1. 用户创建一个持久卷申请（PVC）（以下简称PVC），这个会自动绑定合适的PV。

1. 用户创建一个Pod来使用这个PVC作为存储。

{% endcapture %}

{% capture prerequisites %}

* 你需要有一个单节点的Kubernetes集群，而且kubectl命令行必须配置好可以跟集群交互。
如果你还没有一个单节点集群，可以参照这个文档搭建一个。
[Minikube](/docs/getting-started-guides/minikube).

* 请先熟悉一下这个文档的内容
[持久卷（PV）](/docs/concepts/storage/persistent-volumes/).

{% endcapture %}

{% capture steps %}



## 在你的节点上创建一个index.html文件

连接节点里的shell,如何打开一个shell取决于你如何搭建你的集群。比如说，如果使用Minikube，
可以使用命令`minikube ssh`来打开一个shell。

在shell里创建一个目录`/tmp/data`:

    mkdir /tmp/data

在`/tmp/data`目录里，创建`index.html`文件:

    echo 'Hello from Kubernetes storage' > /tmp/data/index.html



## 创建一个PV

在这个实验里，我们会创建一个*hostPath*PV,Kubernetes在单节点集群上
支持hostPath用于开发和测试。hostPathPV使用节点上一个文件或者目录模拟
一个网络存储。

但是在生产环境中，我们不建议使用hostPath. 相反集群管理者可能会提供一个网络存储比如
Google Compute Engine永久磁盘，NFS共享磁盘，Amazon Elastic Block Store等等。集群管理者
也可以使用[存储类](/docs/resources-reference/{{page.version}}/#storageclass-v1-storage)
来配置[动态分配](http://blog.kubernetes.io/2016/10/dynamic-provisioning-and-storage-in-kubernetes.html).

下面是hostPath PV的配置文件:

{% include code.html language="yaml" file="task-pv-volume.yaml" ghlink="/docs/tasks/configure-pod-container/task-pv-volume.yaml" %}

配置文件写明了这个卷存在集群节点上的`/tmp/data`,同时配置了10G的存储空间和
`ReadWriteOnce`的读取权限，表示这个卷可以被一个单一节点挂载为读写模式。它还
定义了[StorageClass 名](/docs/concepts/storage/persistent-volumes/#class)
为`manual`，这会被用于绑定PV和PVC。

创建一个PV:

    kubectl create -f https://k8s.io/docs/tasks/configure-pod-container/task-pv-volume.yaml

查看PV的信息:

    kubectl get pv task-pv-volume

输出显示了这个PV的`STATUS`是`Available`.表示这个卷还没有被绑定到任何
PVC上。

    NAME             CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
    task-pv-volume   10Gi       RWO           Retain          Available             manual                   4s



## 创建一个PVC

下一步是创建一个PVC，Pod使用PVC来申请物理存储。在这个实验里，我们会
创建一个PVC，申请至少3G并且可以读写的存储空间。

下面是PVC的配置文件:

{% include code.html language="yaml" file="task-pv-claim.yaml" ghlink="/docs/tasks/configure-pod-container/task-pv-claim.yaml" %}

创建PVC:

    kubectl create -f https://k8s.io/docs/tasks/configure-pod-container/task-pv-claim.yaml

创建完PVC之后，Kubernetes控制中心会查找适合的满足PVC要求的持久卷。如果控制中心
找到了一个适合的持久卷并且有相同的存储类，它会把这个PVC绑定到找到的持久卷上。

再看看我们这个持久卷:

    kubectl get pv task-pv-volume

现在的`STATUS`是`Bound`.

    NAME             CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS    CLAIM                   STORAGECLASS   REASON    AGE
    task-pv-volume   10Gi       RWO           Retain          Bound     default/task-pv-claim   manual                   2m

再看看PVC:

    kubectl get PVC task-pv-claim

输出显示了这个PVC已经绑定到了我们的持久卷，`task-pv-volume`.

    NAME            STATUS    VOLUME           CAPACITY   ACCESSMODES   STORAGECLASS   AGE
    task-pv-claim   Bound     task-pv-volume   10Gi       RWO           manual         30s



## 创建一个Pod

下一步是创建一个Pod来使用我们的PVC。

下面是Pod的配置文件：

{% include code.html language="yaml" file="task-pv-pod.yaml" ghlink="/docs/tasks/configure-pod-container/task-pv-pod.yaml" %}

注意到Pod的配置文件里指定了一个PVC，但是没有指定PV。从Pod的角度来看，PVC就是
一个存储卷。

创建Pod:

    kubectl create -f https://k8s.io/docs/tasks/configure-pod-container/task-pv-pod.yaml

验证Pod里的容器是否运行:

    kubectl get pod task-pv-pod

登录到Pod里容器的shell里：

    kubectl exec -it task-pv-pod -- /bin/bash

在Shell里，验证nginx是否使用了hostpath卷的`index.html`文件：

    root@task-pv-pod:/# apt-get update
    root@task-pv-pod:/# apt-get install curl
    root@task-pv-pod:/# curl localhost

输出显示了你写到`index.html`的内容：

    Hello from Kubernetes storage

{% endcapture %}


{% capture discussion %}



## 访问控制

配置了组ID（GID）的存储，也可以让拥有同样的GID的Pod访问。不匹配或者丢失GID会
导致权限错误。管理者可以给PV分配一个GID，然后这个GID会自动分配给任何使用该PV
的Pod。

使用`pv.beta.kubernetes.io/gid`格式定义如下：

    kind: PersistentVolume
    apiVersion: v1
    metadata:
      name: pv1
      annotations:
        pv.beta.kubernetes.io/gid: "1234"

当一个Pod使用了一个具有GID声明的PV，这个GID就会应用到这个Pod里的所有容器。
所有的GID，不管是从PV继承而来，还是来自Pod的声明，都会被应用到每个容器的第一个
进程。

**注意**: 当一个Pod使用了一个PV， PV所带的GID并不会提醒在Pod的资源上。

{% endcapture %}


{% capture whatsnext %}

* 学习更多关于 [持久卷](/docs/concepts/storage/persistent-volumes/).
* 阅读 [持久存储设计方案](https://git.k8s.io/community/contributors/design-proposals/persistent-storage.md).



### 参考

* [持久卷](/docs/resources-reference/{{page.version}}/#persistentvolume-v1-core)
* [持久卷特点](/docs/resources-reference/{{page.version}}/#persistentvolumespec-v1-core)
* [持久卷申请](/docs/resources-reference/{{page.version}}/#persistentvolumeclaim-v1-core)
* [持久卷申请特点](/docs/resources-reference/{{page.version}}/#persistentvolumeclaimspec-v1-core)

{% endcapture %}

{% include templates/task.md %}
