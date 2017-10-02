
---
assignees:
- crassirostris
- piosz
title: 使用 Stackdriver 查看日志
redirect_from:
- "/docs/user-guide/logging/stackdriver/"
- "/docs/user-guide/logging/stackdriver.html"
---



在阅读本页之前，强烈建议您熟悉[Kubernetes 日志概述](/docs/concepts/cluster-administration/logging)。

**注意：** 默认情况下，Stackdriver 仅收集容器的标准输出和标准错误流日志。如果要收集应用程序写入文件的日志，请参阅 Kubernetes 日志概述中的 [sidecar 方法](/docs/concepts/cluster-administration/logging#using-a-sidecar-container-with-the-logging-agent)。



## 部署

为了收集日志，需要在集群的每个节点上都部署 Stackdriver 代理(agent)。使用 `fluentd` 实例做为日志代理，并使用 `ConfigMap` 存储配置，使用 Kubernetes 的 `DaemonSet` 管理实例。集群中 `ConfigMap` 和 `DaemonSet` 实际部署情况取决于你的集群的设置。



### 部署一个新的集群

#### Google Container Engine

在 Google Container Engine 中部署的集群，Stackdriver 是默认的日志解决方案。除非明确的选择，Stackdriver 默认部署到新集群。



#### 其它平台

在使用 `kube-up.sh` 创建的 *新* 集群中，可以使用如下的方法部署 Stackdriver Logging：

1. 设置 `KUBE_LOGGING_DESTINATION` 环境变量为 `gcp`。
2. **如果没有运行在 GCE **，需要在 `KUBE_NODE_LABELS` 变量中包含 `beta.kubernetes.io/fluentd-ds-ready=true`。

一旦集群已经启动，每个节点都已经运行 Stackdriver 日志代理。`DaemonSet` 和 `ConfigMap` 配置为插件。如果没有使用 `kube-up.sh`，需要先启动一个没有配置日志解决方案的集群，然后部署 Stackdriver 日志代理到运行的集群。



### 部署到现有集群

1. 设置每个节点的标签（如果尚未设置）



    使用节点标签决定哪个节点部署 Stackdriver 日志代理。标签是 Kubernetes 在 1.6 版本或更高版本中被引入的，用来区分不同的节点。在 1.5.X 或更低版本中，如果创建的集群中使用 Stackdriver 日志配置，fluentd 将作为静态 Pod 。节点中不能有多个 fluentd 实例，因此需要只在没有部署 fluentd 的节点设置标签。通过运行 `kubectl describe` 命令确保节点标签的正确性： 

    ```shell
    kubectl describe node $NODE_NAME
    ```

    会有类似的输出：

    ```
    Name:           NODE_NAME
    Role:
    Labels:         beta.kubernetes.io/fluentd-ds-ready=true
    ...
    ```



    确保容器的标签中包含 `beta.kubernetes.io/fluentd-ds-ready=true`。如果不存在，可以使用 `kubectl label` 命令添加：

    ```shell
    kubectl label node $NODE_NAME beta.kubernetes.io/fluentd-ds-ready=true
    ```

    **注意：**如果节点挂掉后重新上线，需要重新设置节点标签。为了更简单，可以在节点启动脚本中增加 Kubelet 命令行参数设置节点标签。



1. 通过运行以下命令，部署日志代理的配置的 `ConfigMap`：

    ```shell
    kubectl create -f https://k8s.io/docs/tasks/debug-application-cluster/fluentd-gcp-configmap.yaml
    ```

    这个命令在 `default` 命名空间中创建 `ConfigMap`。可以先下载这个文件，修改后再创建 `ConfigMap` 对象。



1. 通过运行以下命令，部署 `DaemonSet` 日志代理：

    ```shell
    kubectl create -f https://k8s.io/docs/tasks/debug-application-cluster/fluentd-gcp-ds.yaml
    ```

    你也可以下载编辑后再应用。



## 验证日志代理的部署

在 Stackdriver `DaemonSet` 已经部署后，可以通过以下命令获取日志代理的部署状态：

```shell
kubectl get ds --all-namespaces
```



如果集群中有3个节点，它的输出应该类似于这样：

```
NAMESPACE     NAME               DESIRED   CURRENT   READY     NODE-SELECTOR                              AGE
...
kube-system   fluentd-gcp-v2.0   3         3         3         beta.kubernetes.io/fluentd-ds-ready=true   6d
...
```



如果想要了解 Stackdriver 如何收集日志的，可以查看伪造的日志生成器 [counter-pod.yaml](/docs/tasks/debug-application-cluster/counter-pod.yaml)：

{% include code.html language="yaml" file="counter-pod.yaml" ghlink="/docs/tasks/debug-application-cluster/counter-pod.yaml" %}



在这个 pod 中，运行一个脚本，不断每秒输出一个计数和时间。让我们在默认命名空间中创建这个 pod。

```shell
kubectl create -f https://k8s.io/docs/tasks/debug-application-cluster/counter-pod.yaml
```

你可以查看运行的 pod ：

```shell
$ kubectl get pods
NAME                                           READY     STATUS    RESTARTS   AGE
counter                                        1/1       Running   0          5m
```



因为 kubelet 刚开始需要下载容器镜像，所以 pod 会短暂处在 `Pending` 状态。当 pod 处于 `Running` 状态，可以使用 `kubectl logs` 命令查看 counter pod 的输出。

```shell
$ kubectl logs counter
0: Mon Jan  1 00:00:00 UTC 2001
1: Mon Jan  1 00:00:01 UTC 2001
2: Mon Jan  1 00:00:02 UTC 2001
...
```



就像日志概述中描述的一样，这个命名从容器的日志文件中获取日志。如果容器被 Kubernetes 杀掉或重启，仍然可以访问前一个容器的日志。但是如果 pod 被从节点中驱逐，日志文件将丢失。我们可以通过删除当前正在运行的 counter 容器验证这个：

```shell
$ kubectl delete pod counter
pod "counter" deleted
```



然后重新创建它：

```shell
$ kubectl create -f https://k8s.io/docs/tasks/debug-application-cluster/counter-pod.yaml
pod "counter" created
```

过一段时间，可以再次访问 counter 的日志：

```shell
$ kubectl logs counter
0: Mon Jan  1 00:01:00 UTC 2001
1: Mon Jan  1 00:01:01 UTC 2001
2: Mon Jan  1 00:01:02 UTC 2001
...
```



正如所料，只有最近的日志。实际上，你想要获取所有容器中的日志，尤其是在调试的时候。Stackdriver 日志正是解决这个问题。



## 查看日志

Stackdriver 日志代理会在每个日志项中附加 metadata，这样就可以在后面的查询中选择特定消息：例如，特定 pod 的消息。



最重要的 metadata 是资源类型和日志名称。容器日志的资源类型是 `container`，它在 UI 中被称作 `GKE Containers` （即使 Kubernetes 集群不在 GKE 上）。日志名称是容器的名称，因此如果一个 pod 中有两个容器，叫做 `container_1` 和  `container_2`，他们的日志名称分别是 `container_1` 和  `container_2`



系统组件的资源类型是 `compute`， 在接口中被叫做 `GCE VM Instance`。系统组件的日志名称是固定的。对于 GKE 节点，系统组件中的日志条目会有下面某一个的日志名称：

* docker
* kubelet
* kube-proxy



你可以在 [Stackdriver 页面](https://cloud.google.com/logging/docs/view/logs_viewer) 了解更多查看日志的信息。

一种可能的查看日志的方式是使用 [Google Cloud SDK](https://cloud.google.com/sdk/) 中的 [`gcloud logging`](https://cloud.google.com/logging/docs/api/gcloud-logging) 命令行接口。它使用 Stackdriver Logging [过滤语法](https://cloud.google.com/logging/docs/view/advanced_filters) 查询特定日志。例如可以运行如下命令：



```shell
$ gcloud beta logging read 'logName="projects/$YOUR_PROJECT_ID/logs/count"' --format json | jq '.[].textPayload'
...
"2: Mon Jan  1 00:01:02 UTC 2001\n"
"1: Mon Jan  1 00:01:01 UTC 2001\n"
"0: Mon Jan  1 00:01:00 UTC 2001\n"
...
"2: Mon Jan  1 00:00:02 UTC 2001\n"
"1: Mon Jan  1 00:00:01 UTC 2001\n"
"0: Mon Jan  1 00:00:00 UTC 2001\n"
```

正像你所看到的，他会输出第一个和第二个 counter 的日志，即使第一个容器已经被删除。



## 导出日志

可以导出日志到 [Google Cloud Storage](https://cloud.google.com/storage/) 或 [BigQuery](https://cloud.google.com/bigquery/) 以便进一步的分析。Stackdriver Logging 提供接收器的概念，可以指定日志的目标。更多信息参见 Stackdriver 的 [导出日志页面](https://cloud.google.com/logging/docs/export/configure_export_v2)。



## 配置 Stackdriver Logging 代理

有时候，Stackdriver Logging 默认安装可能并不能满足你的需求，例如：

* 你可能需要更多资源，因为默认性能并不能满足需求
* 你可能引入额外的解析来从日志中消息中抽取更多的 metadata，例如日志级别或源码位置。
* 你可能不只发送到 Stackdriver，或只部分发送到 Stackdriver。

在这种情况下，你就需要更改`DaemonSet` 和 `ConfigMap` 中的参数



### 前提

如果你的集群在 GKE 并已经启动 Stackdriver Logging，你并不能更改它的参数。同理，如果你不在 GKE 中，但是 Stackdriver Logging 已经作为插件启动，也不能通过 Kubernetes API 更改部署参数。如果想要更改 Stackdriver Logging 代理的参数，当 Stackdriver Logging 已经安装，还没有安装集群日志解决方案，你应该切换到 API 对象部署。

你可以在[Deploying section](#deploying) 找到如何在运行的集群中安装 Stackdriver Logging 代理。



### 更改 `DaemonSet` 参数

如果集群中已经有Stackdriver Logging `DaemonSet`，你只需要更改 spec 中 `template` 域，daemonset controller 将会更新 pods。例如，假设你已经按照以上的步骤安装了 Stackdriver Logging，你想要更改内存限制，使 fluentd 更多的内存处理更多的日志。



获取当前集群中 `DaemonSet` 的配置文件：

```shell
kubectl get ds fluentd-gcp-v2.0 --namespace kube-system -o yaml > fluentd-gcp-ds.yaml
```



然后编辑配置文件中资源需求，然后使用如下的命令更新 apiserver 中 `DaemonSet` 对象：

```shell
kubectl replace -f fluentd-gcp-ds.yaml
```

一段时间后，Stackdriver Logging 代理 pod 将使用新的配置文件重启。



### 更改 fluentd 参数

Fluentd 配置存储在 `ConfigMap` 对象。它实际上包含一组配置文件。你可以在[官网](http://docs.fluentd.org)找到更多 fluentd 配置。



假设你想要增加一个新的解析逻辑到配置文件中，以便 fluentd 可以解析 Python 默认的日志格式。一个恰当的 fluentd filter 可能是这样的：

```
<filter reform.**>
  type parser
  format /^(?<severity>\w):(?<logger_name>\w):(?<log>.*)/
  reserve_data true
  suppress_parse_error_log true
  key_name log
</filter>
```



现在你不得不将它放入配置文件，以便 Stackdriver Logging 代理可以接收它。使用如下命令获取集群中 Stackdriver Logging `ConfigMap` 的当前版本：

```shell
kubectl get ds fluentd-gcp-v2.0 --namespace kube-system -o yaml > fluentd-gcp-ds.yaml
```



然后，在 `containers.input.conf` 的值中，将新的 filter 正好插入到 `source` 后面。**注意：**顺序很重要。

在 apiserver 中，更新 `ConfigMap` 要比更新 `DaemonSet` 更加复杂。最好认为 `ConfigMap` 是不可变的。然后，为了更新配置，您应该使用新名称创建 `ConfigMap`，然后使用上面的[指南](#changing-daemonset-parameters)更改 `DaemonSet` 以指向它。



### 增加 fluentd 插件

Fluentd 是使用 Ruby 编写，并且允许使用 [插件](http://www.fluentd.org/plugins) 扩展它的能力。如果你想用一个插件，但是并不在默认的 Stackdriver Logging 容器镜像中，因此需要编译一个自定义镜像。想象一下，你想为特定容器的消息添加 Kafka sink 进行其它处理。你可以使用默认 [容器镜像源](https://git.k8s.io/contrib/fluentd/fluentd-gcp-image) 进行微小更改：



* 更改 Makefile 指向你的镜像仓库，例如：`PREFIX=gcr.io/<your-project-id>`
* 增加依赖到 Gemfile，例如： `gem 'fluent-plugin-kafka'`

然后在这个目录下运行 `make build push`。在使用新的镜像更新 `DaemonSet` 后，你可以在 fluentd 配置中使用新的插件。
