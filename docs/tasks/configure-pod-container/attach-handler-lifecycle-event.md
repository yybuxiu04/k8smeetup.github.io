---
title: 给容器生命周期设置操作事件
---
{% capture overview %}

这篇教程指导如何给容器生命周期设置操作事件。Kubernetes支持预启动和预结束事件。
Kubernetes在容器启动的时候发送预启动事件，在容器结束的时候发送预结束事件。

{% endcapture %}


{% capture prerequisites %}

{% include task-tutorial-prereqs.md %}

{% endcapture %}


{% capture steps %}



## 定义预启动和预结束事件操作

本实验将会创建含有一个容器的Pod，我们将会给这个容器设置预启动和预结束操作。

下面是Pod的配置文件：

{% include code.html language="yaml" file="lifecycle-events.yaml" ghlink="/docs/tasks/configure-pod-container/lifecycle-events.yaml" %}

在这个配置文件里，你可以看到postStart命令在容器目录`/usr/share`下写了一个`message`文件，
preStop命令停止容器。这在容器被因错误而结束时很有帮助。

创建Pod:

    kubectl create -f https://k8s.io/docs/tasks/configure-pod-container/lifecycle-events.yaml

验证Pod里的容器是否运行:

    kubectl get pod lifecycle-demo

连接到Pod里容器的shell:

    kubectl exec -it lifecycle-demo -- /bin/bash

在shell里，验证`postStart`是否创建了`message`文件：

    root@lifecycle-demo:/# cat /usr/share/message

输出显示了文件确实被创建了：

    Hello from the postStart handler

{% endcapture %}



{% capture discussion %}


## 说明

Kubernetes在容器创建之后就会马上发送postStart事件，但是并没法保证一定会
这么做，它会在容器入口被调用之前调用postStart操作，因为postStart的操作跟容器的操作是异步的，而且Kubernetes控制台会锁住容器直至postStart完成，因此容器只有在
postStart操作完成之后才会被设置成为RUNNING状态。

Kubernetes在容器结束之前发送preStop事件，并会在preStop操作完成之前一直锁住容器
状态，除非Pod的终止时间过期了。更多详细信息请查看
[Pods的终止](/docs/user-guide/pods/#termination-of-pods).

{% endcapture %}


{% capture whatsnext %}

* 更多关于[容器生命周期](/docs/concepts/containers/container-lifecycle-hooks/).
* 更多关于 [Pod生命周期](/docs/concepts/workloads/pods/pod-lifecycle/).


### 参考

* [生命周期](/docs/resources-reference/{{page.version}}/#lifecycle-v1-core)
* [容器](/docs/resources-reference/{{page.version}}/#container-v1-core)
* 在[PodSpec]里查看章节`terminationGracePeriodSeconds`(/docs/resources-reference/{{page.version}}/#podspec-v1-core)

{% endcapture %}

{% include templates/task.md %}
