
---
approvers:
- jpeeler
- pmorie
title: 配置Pod使用配套的存储卷作为存储
---

{% capture overview %}
这篇教程指导如何使用[`配套`](/docs/concepts/storage/volumes/#projected)卷来挂载几个存在的卷到相同的目录。目前，`secret`, `configMap`,和`downwardAPI` 卷都可以作为配套卷使用。
{% endcapture %}

{% capture prerequisites %}
{% include task-tutorial-prereqs.md %}
{% endcapture %}

{% capture steps %}



## 给Pod配置配套卷

在这个例子里，我们会从本地文件里创建用户名和密码Secretes。然后新建一个Pod来运行这个容器，使用[`配套`](/docs/concepts/storage/volumes/#projected) 卷来挂载Secretes到同一个共享目录。

下面是Pod的配置文件:

{% include code.html language="yaml" file="projected-volume.yaml" ghlink="/docs/tasks/configure-pod-container/projected-volume.yaml" %}

1. 创建Secrets:

       # 创建文件包含用户名和密码：
       echo -n "admin" > ./username.txt
       echo -n "1f2d1e2e67df" > ./password.txt

       # 将这些文件打包到secretes:
       kubectl create secret generic user --from-file=./username.txt
       kubectl create secret generic pass --from-file=./password.txt

1. 创建Pod:

       kubectl create -f projected-volume.yaml



1. 验证Pod的容器是否运行，然后查看容器变化：

       kubectl get --watch pod test-projected-volume

    输出类似这样:

       NAME                    READY     STATUS    RESTARTS   AGE
       test-projected-volume   1/1       Running   0          14s

1. 在另外一个终端里，连接到运行的容器的shell：

       kubectl exec -it test-projected-volume -- /bin/sh



1. 在shell里，验证`projected-volume`目录是否包含了你的内容：
       / # ls /projected-volume/
{% endcapture %}

{% capture whatsnext %}
* 更多关于 [`配套`](/docs/concepts/storage/volumes/#projected) 卷.
* 阅读 [卷大全](https://github.com/kubernetes/community/blob/{{page.githubbranch}}/contributors/design-proposals/all-in-one-volume.md) 设计文档.
{% endcapture %}

{% include templates/task.md %}

