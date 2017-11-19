---
approvers:
- bprashanth
- enisoc
- erictune
- foxish
- janetkuo
- kow3ns
- smarterclayton
title: 调试 Init 容器
---



{% capture overview %}



本页将展示如何确定执行 Init 容器时相关的问题。示例命令行中使用 `<pod-name>` 表示 Pod ，使用 `<init-container-1>` 和 `<init-container-2>` 表示 Init 容器。

{% endcapture %}

{% capture prerequisites %}

{% include task-tutorial-prereqs.md %}



* 您应该已经熟悉 [Init 容器](/docs/concepts/abstractions/init-containers/) 的基础知识。
* 您应该已经 [配置一个 Init 容器](/docs/tasks/configure-pod-container/configure-pod-initialization/#creating-a-pod-that-has-an-init-container/)。


{% endcapture %}

{% capture steps %}



## 查看 Init 容器的状态

显示 Pod 的状态：

```shell
kubectl get pod <pod-name>
```



例如： `Init:1/2` 状态表示2个初始化容器中，有一个已经完成。

```
NAME         READY     STATUS     RESTARTS   AGE
<pod-name>   0/1       Init:1/2   0          7s
```

有关状态值及其含义的更多示例，请参阅[理解 Pod 状态](#understanding-pod-status)。



## 获取 Init 容器详细信息

查看关于 Init 容器更详细信息可以执行：

```shell
kubectl describe pod <pod-name>
```



例如，具有两个 Init 容器的 Pod 可能显示以下内容：

```
Init Containers:
  <init-container-1>:
    Container ID:    ...
    ...
    State:           Terminated
      Reason:        Completed
      Exit Code:     0
      Started:       ...
      Finished:      ...
    Ready:           True
    Restart Count:   0
    ...
  <init-container-2>:
    Container ID:    ...
    ...
    State:           Waiting
      Reason:        CrashLoopBackOff
    Last State:      Terminated
      Reason:        Error
      Exit Code:     1
      Started:       ...
      Finished:      ...
    Ready:           False
    Restart Count:   3
    ...
```

<--
You can also access the Init Container statuses programmatically by reading the
`status.initContainerStatuses` field on the Pod Spec:
-->

您还也可以通过编程的方式读取 Pod Spec 中的 `status.initContainerStatuses` 字段获取 Init 容器的状态：

{% raw %}
```shell
kubectl get pod nginx --template '{{.status.initContainerStatuses}}'
```
{% endraw %}



这个命令将返回与原始 JSON 相同的信息。

## 访问 Init 容器的日志

在 Pod 名称后传递 Init 容器名称访问它的日志。

```shell
kubectl logs <pod-name> -c <init-container-2>
```



运行 shell 脚本的 Init 容器将会打印其执行命令。例如，您可以在执行脚本之前在 Bash 中运行 `set -x` 。

{% endcapture %}

{% capture discussion %}



## 理解 Pod 状态

以 `Init:` 开始的 Pod 状态概括表示 Init 容器的执行状态。下表展示了在调试 Init 容器时可能见到的状态值。



状态 | 含义
------ | -------
`Init:N/M` | Pod 中有 `M` 个 Init  容器，其中 `M` 已经完成
`Init:Error` | Init 容器执行错误
`Init:CrashLoopBackOff` | Init 容器已经失败多次
`Pending` | Pod 还没有开始执行 Init 容器
`PodInitializing` or `Running` | Pod 已经完成执行 Init 容器

{% endcapture %}

{% include templates/task.md %}
