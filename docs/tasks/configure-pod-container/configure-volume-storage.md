
---
title: 配置Pod使用卷作为存储
---

{% capture overview %}

这篇教程指导如何配置Pod使用卷作为存储。

容器的文件系统随着容器的消失而消失，因此当一个容器结束后重启时，文件系统上所作的
修改会完全丢失。若需要独立于容器的永久存储，可以使用[Volume](/docs/concepts/storage/volumes/). 
这对于有状态的应用是相当重要的，比如键值对存储和数据库。Redis就是一个键值对的缓存
和存储。

{% endcapture %}

{% capture prerequisites %}

{% include task-tutorial-prereqs.md %}

{% endcapture %}

{% capture steps %}



## 给Pod配置卷

在这个实验里，我们会创建一个Pod运行一个容器，这个Pod有一个[emptyDir](/docs/concepts/storage/volumes/#emptydir)
类型的卷，这会持续到Pod的整个生命周期，即使容器终结并重启。下面是Pod的配置：

{% include code.html language="yaml" file="pod-redis.yaml" ghlink="/docs/tasks/configure-pod-container/pod-redis.yaml" %}

1. 创建Pod:

       kubectl create -f https://k8s.io/docs/tasks/configure-pod-container/pod-redis.yaml

1. 验证Pod的容器是否运行，并查看Pod的变动：

       kubectl get pod redis --watch

    输出类似下面:

       NAME      READY     STATUS    RESTARTS   AGE
       redis     1/1       Running   0          13s



1. 在另外一个终端里，连接到容器里的shell:

       kubectl exec -it redis -- /bin/bash

1. 在shell里，切换到`/data/redis`目录，并创建新文件:

       root@redis:/data/redis# echo Hello > test-file

1. 在shell里，列出运行的进程：

       root@redis:/data/redis# ps aux

    输出类似下面:

       USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
       redis        1  0.1  0.1  33308  3828 ?        Ssl  00:46   0:00 redis-server *:6379
       root        12  0.0  0.0  20228  3020 ?        Ss   00:47   0:00 /bin/bash
       root        15  0.0  0.0  17500  2072 ?        R+   00:48   0:00 ps aux



1. 在shell里，杀死redis进程：

       root@redis:/data/redis# kill <pid>

    这里的`<pid>`就是redis的进程ID(PID).

1. 在你原先的终端里，观察redis Pod的变化。最终你可能会看到的内容如下：

       NAME      READY     STATUS     RESTARTS   AGE
       redis     1/1       Running    0          13s
       redis     0/1       Completed  0         6m
       redis     1/1       Running    1         6m

容器结束并重新启动，这是因为redis Pod 的[重启策略](/docs/api-reference/{{page.version}}/#podspec-v1-core)
是`Always`。



1. 在重启后的容器里打开一个shell :
       kubectl exec -it redis -- /bin/bash

1. 在shell里，切换到`/data/redis`目录，并验证文件`test-file`是否还在。

{% endcapture %}

{% capture whatsnext %}

* 阅读[卷](/docs/api-reference/{{page.version}}/#volume-v1-core).

* 阅读[Pod](/docs/api-reference/{{page.version}}/#pod-v1-core).

* 除了由`emptyDir`类型提供的本地磁盘外, Kubernetes还支持多种不同的网络存储方案，
包括GCE的PD，EC2的EBS，这些更加适用于关键数据，而且需要处理细节问题包括在节点上
挂载和卸载设备等，详情请查看[Volumes](/docs/concepts/storage/volumes/).

{% endcapture %}

{% include templates/task.md %}
