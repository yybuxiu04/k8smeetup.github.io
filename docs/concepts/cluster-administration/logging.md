---
approvers:
- crassirostris
- piosz
title: 日志架构
---





应用和系统日志可以让您了解集群内部的运行状况。日志对调试问题和监控集群活动非常有用。大部分现代化应用都有某种日志记录机制；同样地，大多数容器引擎也被设计成支持某种日志记录机制。针对容器化应用，最简单且受欢迎的日志记录方式就是写入标准输出和标准错误流。



但是，由容器引擎或 runtime 提供的原生功能通常不足以满足完整的日志记录方案。例如，如果发生容器崩溃，pod 被驱逐,或 node 宕机,此时您仍然想访问到应用日志。这种情形下，日志应该具有独立的存储和生命周期，不依赖于 node, pods,或容器。这个概念叫 _集群级的日志_ 。集群级日志方案需要一个独立的后台来存储，分析和查询日志。Kubernetes 没有为日志数据提供原生存储方案，但是您可以集成许多现有的日志解决方案到 Kubernetes 集群。


* TOC
{:toc}



集群级日志架构假定在集群内部或者外部有一个日志后台。如果您对集群级日志不感兴趣，您仍然会从本文学到如何在节点上存储和处理日志的说明。




## Kubernetes 中的常规日志记录

本节，您会看到一个常规日志记录的例子，即在 kubernetes 中将数据写入到标准输出。该演示通过 [定义 pod ](/docs/concepts/cluster-administration/counter-pod.yaml) 创建一个每秒向标准输出写入数据的容器。


{% include code.html language="yaml" file="counter-pod.yaml" ghlink="/docs/tasks/debug-application-cluster/counter-pod.yaml" %}



用下面的命令运行 pod：

```shell
$ kubectl create -f https://k8s.io/docs/tasks/debug-application-cluster/counter-pod.yaml
pod "counter" created
```



使用 `kubectl logs` 命令获取日志:

```shell
$ kubectl logs counter
0: Mon Jan  1 00:00:00 UTC 2001
1: Mon Jan  1 00:00:01 UTC 2001
2: Mon Jan  1 00:00:02 UTC 2001
...
```



一旦发生容器崩溃，您可以使用命令 `kubectl logs` 和参数 `--previous` 恢复之前的容器日志。如果 pod 中有多个容器，您应该向该命令附加一个容器名以访问相应的容器日志。详见 [`kubectl logs`文档](/docs/user-guide/kubectl/{{page.version}}/#logs)。



## 节点级日志记录

![Node level logging](/images/docs/user-guide/logging/logging-node-level.png)



容器化应用写入 `stdout` 和 `stderr` 的任何数据，都会被容器引擎捕获并被重定向到某个位置。例如，Docker 容器引擎将这两个输出流重定向到 [日志驱动](https://docs.docker.com/engine/admin/logging/overview) ，该日志驱动在 Kubernetes 中配置为以 json 格式写入文件。



**提示:**  Docker json 日志驱动将日志的每一行当作一条独立的消息。在使用该日志驱动时，没有直接支持多行消息。您需要在日志代理级别或更高级别处理多行消息。



默认情况下，如果容器重启，kubelet 会保留被终止的容器日志。如果 pod 在工作节点被驱逐，该 pod 中所有的容器也会被驱逐，包括容器日志。



节点级日志记录中，需要重点考虑实现日志的轮转，以此来保证日志不会消耗节点上所有的可用空间。Kubernetes 当前并不负责轮转日志，而是通过部署工具建立一个解决问题的方案。例如，在 Kubernetes 集群中，用 `kube-up.sh` 部署一个每小时运行的工具 [`logrotate`](https://linux.die.net/man/8/logrotate)。您也可以设置容器 runtime 来自动地轮转应用日志，比如，使用 Docker 的 `log-opt` 选项。在 `kube-up.sh` 脚本中，后一种方式适用于 GCP 的 COS 镜像，而前一种方式适用于任何环境。这两种方式，默认日志超过 10MB 大小时触发日志轮转。



例如，您可以找到关于 `kube-up.sh` 为 GCP 环境的 COS 镜像设置日志的详细信息，相应的脚本在 [script][cosConfigureHelper]。



和常规日志记录的例子一样，运行 [`kubectl logs`](/docs/user-guide/kubectl/{{page.version}}/#logs) 时，节点上的 kubelet 处理该请求并直接读取日志文件，同时在响应中返回日志文件内容。
**提示：** 当前，如果有其他系统机制执行日志轮转，那么 `kubectl logs` 仅可查询到最新的日志内容，比如，一个 10MB 大小的文件，通过`logrotate` 执行轮转后生成两个文件，一个 10MB 大小，一个为空，所以 `kubectl logs` 将返回空。


[cosConfigureHelper]: https://github.com/kubernetes/kubernetes/blob/{{page.githubbranch}}/cluster/gce/gci/configure-helper.sh



### 系统组件日志

有两种类型的系统组件，运行在容器中的和未运行在容器中的。例如：



* 运行在容器中的 Kubernetes scheduler 和 kube-proxy。
* 未运行在容器中的 kubelet 和容器 runtime，比如 Docker。



在使用 systemd 机制的服务器上，kubelet 和容器 runtime 写入日志到 journald。如果没有 systemd，他们写入日志到 `/var/log` 目录的 `.log` 文件。容器中的系统组件通常将日志写到 `/var/log` 目录，绕过了默认的日志机制。他们使用 [glog][glog] 日志库。您可以在 [development docs on logging](https://git.k8s.io/community/contributors/devel/logging.md) 找到这些组件的日志告警级别协议。



和容器日志类似，`/var/log` 目录中的系统组件日志也应该被轮转，通过脚本
 `kube-up.sh` 启动的 Kubernetes 集群，他们的日志被工具 `logrotate` 执行每日轮转，或者日志大小超过 100MB 时触发轮转。

[glog]: https://godoc.org/github.com/golang/glog



## 集群级别日志架构



虽然 Kubernetes 并未提供原生的集群级记录日志方案，但是您可以考虑几种常见的方式。以下是一些选项：

* 使用运行在每个节点上的节点级的日志代理。
* 在应用程序的 pod 中，包含专门记录日志的伴生容器。
* 在应用程序中将日志直接推送到后台。



### 使用节点级日志代理

![Using a node level logging agent](/images/docs/user-guide/logging/logging-with-node-agent.png)



您可以在每个节点上使用 _节点级的日志代理_ 来实现集群级日志记录。日志代理是专门的工具，它会暴露出日志或将日志推送到后台。通常来说，日志代理是一个容器，这个容器可以访问这个节点上所有应用容器的日志目录。



因为日志代理必须在每个节点上运行，所以通常的实现方式为，DaemonSet 副本，manifest pod，或者专用于本地的进程。但是后两种方式已被弃用并且不被推荐。



对于 Kubernetes 集群来说，使用节点级的日志代理是最常用和被推荐的方式，因为在每个节点上仅创建一个代理，并且不需要对节点上的应用做修改。但是，节点级的日志 _仅适用于应用程序的标准输出和标准错误输出_。



Kubernetes 并不指定日志代理，但是有两个可选的日志代理与 Kubernetes 发行版一起发布。[Stackdriver Logging](/docs/user-guide/logging/stackdriver) 适用于 Google Cloud Platform，和 [Elasticsearch](/docs/user-guide/logging/elasticsearch)。您可以在专门的文档中找到更多的信息和说明。两者都使用 [fluentd](http://www.fluentd.org/) 与自定义配置作为节点上的代理。



### 使用伴生容器和日志代理



您可以通过以下方式之一使用伴生容器：



* 伴生容器将应用程序日志传送到自己的标准输出。
* 伴生容器运行一个日志代理，配置该日志代理以便从应用容器收集日志。



#### 传输数据流的伴生容器



利用伴生容器向自己的 `stdout` 和 `stderr` 传输流的方式，您就可以利用每个节点上的 kubelet 和日志代理来处理日志。伴生容器从文件，socket 或 journald 读取日志。每个伴生容器打印其自己的 `stdout` 和 `stderr` 流。



这种方式允许您分离出不同的日志流，这些日志流来自您应用的不同功能，其中一些可能缺乏对写入 `stdout` 和 `stderr` 的支持。背后重定向的逻辑很小，所以不会是很严重的开销。除此之外，因为 kubelet 处理 `stdout` 和 `stderr`，所以您照样可以使用 `kubectl logs` 工具。



考虑接下来的例子。pod 的容器向两个文件写不同格式的日志，下面是这个 pod 的配置文件:

{% include code.html language="yaml" file="two-files-counter-pod.yaml" ghlink="/docs/concepts/cluster-administration/two-files-counter-pod.yaml" %}



在同一个日志流中有两种不同格式的日志条目，这有点混乱，即使您试图重定向它们到容器的 `stdout` 流。取而代之的是，您可以引入两个伴生容器。每一个伴生容器可以从共享卷跟踪特定的日志文件，并重定向文件内容到各自的 `stdout` 流。



这是运行两个伴生容器的 pod 文件。

{% include code.html language="yaml" file="two-files-counter-pod-streaming-sidecar.yaml" ghlink="/docs/concepts/cluster-administration/two-files-counter-pod-streaming-sidecar.yaml" %}

Now when you run this pod, you can access each log stream separately by
running the following commands:

现在当您运行这个 pod 时，您可以分别地访问每一个日志流，运行如下命令：

```shell
$ kubectl logs counter count-log-1
0: Mon Jan  1 00:00:00 UTC 2001
1: Mon Jan  1 00:00:01 UTC 2001
2: Mon Jan  1 00:00:02 UTC 2001
...
```

```shell
$ kubectl logs counter count-log-2
Mon Jan  1 00:00:00 UTC 2001 INFO 0
Mon Jan  1 00:00:01 UTC 2001 INFO 1
Mon Jan  1 00:00:02 UTC 2001 INFO 2
...
```



无需深入配置，集群中的节点级代理即可自动地收集流日志。如果您愿意，您可以配置代理程序来解析源容器的日志行。



注意，尽管 CPU 和内存使用率都很低（以多个 cpu millicores 指标排序或者按 memory 的兆字节排序），向文件写日志然后输出到 `stdout` 流仍然会成倍地增加磁盘使用率。如果您的应用向单一文件写日志，通常最好设置 `/dev/stdout` 作为目标路径，而不是使用流式的伴生容器方式。



应用本身如果不具备轮转日志文件的功能，可以通过伴生容器实现。该方式的 [例子](https://github.com/samsung-cnct/logrotate) 是运行一个定期轮转日志的容器。然而，还是推荐直接使用 `stdout` 和 `stderr`，将日志的轮转和保留策略交给 kubelet。



### 具有日志代理功能的伴生容器

![Sidecar container with a logging agent](/images/docs/user-guide/logging/logging-with-sidecar-agent.png)



如果节点级的日志代理对您的环境来说不够灵活，您可以在伴生容器中创建一个独立的、专门为您的应用而配置的日志代理。



**提示**：在伴生容器中使用日志代理会导致严重的资源损耗。此外，您不能使用 `kubectl logs` 命令访问日志，因为日志并没有被 kubelet 管理。



例如，您可以使用 [Stackdriver](/docs/tasks/debug-application-cluster/logging-stackdriver/)，它用 fluentd 作为日志代理。这是实现此种方式的两个配置文件。第一个文件包含配置 fluentd 的 [ConfigMap](/docs/tasks/configure-pod-container/configmap/)。

{% include code.html language="yaml" file="fluentd-sidecar-config.yaml" ghlink="/docs/concepts/cluster-administration/fluentd-sidecar-config.yaml" %}



**提示**：fluentd 的配置文件已经超出了本文的讨论范畴。更多配置 fluentd 的信息，请见 [官方 fluentd 文档](http://docs.fluentd.org/)。




第二个文件描述了运行 fluentd 伴生容器的 pod 。flutend 通过 pod 的挂载卷获取它的配置数据。

{% include code.html language="yaml" file="two-files-counter-pod-agent-sidecar.yaml" ghlink="/docs/concepts/cluster-administration/two-files-counter-pod-agent-sidecar.yaml" %}



一段时间后，您可以在 Stackdriver 界面看到日志消息。



记住，这只是一个例子，事实上您可以用任何一个日志代理替换 fluentd ，并从应用容器中读取任何资源。



### 从应用中直接暴露日志目录

![Exposing logs directly from the application](/images/docs/user-guide/logging/logging-from-application.png)



通过暴露或推送每个应用的日志，您可以实现集群级日志记录；然而，这种日志记录机制的实现已超出 Kubernetes 的范围。
