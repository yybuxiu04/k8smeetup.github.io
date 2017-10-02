
---
title: 配置Pod初始化
---

{% capture overview %}
这篇教程指导如何在应用运行之前使用Init容器来初始化一个Pod。

{% endcapture %}

{% capture prerequisites %}

{% include task-tutorial-prereqs.md %}

{% endcapture %}

{% capture steps %}



## 创建一个含有Init容器的Pod

在这个例子里，我们会创建一个包含一个应用容器和一个Init容器的Pod，这个Init
容器会在应用容器开始前完成它的工作。

这是Pod的配置文件:

{% include code.html language="yaml" file="init-containers.yaml" ghlink="/docs/tasks/configure-pod-container/init-containers.yaml" %}

在这个配置文件里，您可以看到Pod有一个Init容器和应用容器共享的存储卷。



Init容器将这个卷挂载在`/work-dir`,而应用容器则挂载在`/usr/share/nginx/html`。
Init容器运行下面的命令并结束：

     wget -O /work-dir/index.html http://kubernetes.io

注意到Init容器往nginx的根目录写入了`index.html`这个文件。

创建Pod:

    kubectl create -f https://k8s.io/docs/tasks/configure-pod-container/init-containers.yaml

验证nginx容器是否运行:

    kubectl get pod init-demo

输出显示了nginx容器正在运行：

    NAME      READY     STATUS    RESTARTS   AGE
    nginx     1/1       Running   0          43m



在init-demo容器里的nginx容器打开一个shell：

    kubectl exec -it init-demo -- /bin/bash

在Shell里，发送一个GET请求给nginx服务器：

    root@nginx:~# apt-get update
    root@nginx:~# apt-get install curl
    root@nginx:~# curl localhost

下面输出显示了nginx正在提供网页服务并使用了Init容器写入的内容：

    <!Doctype html>
    <html id="home">

    <head>
    ...
    "url": "http://kubernetes.io/"}</script>
    </head>
    <body>
      ...
      <p>Kubernetes is open source giving you the freedom to take advantage ...</p>
      ...



{% endcapture %}

{% capture whatsnext %}

* 了解更多
[Pod里的容器间通讯](/docs/tasks/configure-pod-container/communicate-containers-same-pod/).
* 了解更多[Init容器](/docs/concepts/workloads/pods/init-containers/).
* 了解更多[存储卷](/docs/concepts/storage/volumes/).
* 了解更多[Init容器纠错](/docs/tasks/debug-application-cluster/debug-init-containers/)

{% endcapture %}

{% include templates/task.md %}