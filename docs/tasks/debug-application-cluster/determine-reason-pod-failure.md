---
title: 确定 Pod 失败的原因
cn-approvers:
- chentao1596
cn-reviewers:
- zjj2wry
---

{% capture overview %}


本页展示了怎么样读写容器的终止信息。


终止信息为容器提供了一种把致命事件的信息写入指定位置的方法，该位置的内容可以很方便的通过 dashboards、监控软件等工具检索和展示。大部分情况下，终止信息的内容应该同时写入 [Kubernetes 日志](/docs/concepts/cluster-administration/logging/)。

{% endcapture %}


{% capture prerequisites %}

{% include task-tutorial-prereqs.md %}

{% endcapture %}


{% capture steps %}


## 写入和读取终止消息


本次练习中，您将创建一个运行单个容器的 Pod。Pod 配置文件指定了容器启动时运行的命令。

{% include code.html language="yaml" file="termination.yaml" ghlink="/docs/tasks/debug-application-cluster/termination.yaml" %}


1. 基于 YAML 配置文件创建一个 Pod

       kubectl create -f https://k8s.io/docs/tasks/debug-application-cluster/termination.yaml

	
	查看 YAML 文件的 `cmd` 和 `args` 字段，您可以看到，容器将运行 10 秒钟，然后把 "Sleep expired" 写入文件 `/dev/termination-log`。容器在写完 "Sleep expired" 后终止。


1. 显示 Pod 的信息：

       kubectl get pod termination-demo

	
	重复上面的命令，直到 Pod 不再处于运行状态为止。


1. 显示 Pod 的详细信息：

       kubectl get pod --output=yaml

	
	输出的内容包含了 "Sleep expired" 信息：

        apiVersion: v1
        kind: Pod
        ...
            lastState:
              terminated:
                containerID: ...
                exitCode: 0
                finishedAt: ...
                message: |
                  Sleep expired
                ...


1. 使用 Go 模板过滤输出的内容，以便只包含终止信息：

```
{% raw %}  kubectl get pod termination-demo -o go-template="{{range .status.containerStatuses}}{{.lastState.terminated.message}}{{end}}"{% endraw %}
```


## 设置终止日志文件


默认情况下，Kubernetes 从 `/dev/termination-log` 文件检索终止信息。如果希望从其它文件检索，您需要在容器中指定 `terminationMessagePath` 字段。


例如，假如您容器的终止信息写到了 `/tmp/my-log`，而您希望 Kubernetes 能够检索到这些信息。可以按照如下方式设置 `terminationMessagePath`：

    apiVersion: v1
    kind: Pod
    metadata:
      name: msg-path-demo
    spec:
      containers:
      - name: msg-path-demo-container
        image: debian
        terminationMessagePath: "/tmp/my-log"

{% endcapture %}

{% capture whatsnext %}


* 查看 [容器](/docs/api-reference/{{page.version}}/#container-v1-core) 中的 `terminationMessagePath` 字段。
* 了解 [检索日志](/docs/concepts/cluster-administration/logging/)。
* 了解 [Go 模板](https://golang.org/pkg/text/template/)。

{% endcapture %}


{% include templates/task.md %}
