
---
approvers:
- Random-Liu
- dchen1107
cn-approvers:
- xuyang02965
cn-reviewers:
- markthink
title: 监控节点健康状况
---

* TOC
{:toc}



## 节点问题探测器



*节点问题探测器* 是一个监控节点健康状况的[DaemonSet](/docs/concepts/workloads/controllers/daemonset/)。它从不同的守护进程中收集节点问题，并上报给 apiserver 生成[节点健康状况](/docs/concepts/architecture/nodes/#condition)
and [事件](/docs/api-reference/{{page.version}}/#event-v1-core)。



现在它支持某些已知内核问题的探测，以后会逐步支持探测更多的问题。

当前 Kubernetes 对节点问题探测器上报的节点健康状况和事件不做任何处理。将来会引入一个恢复系统来处理发现的节点问题。

更多请参见[这里](https://github.com/kubernetes/node-problem-detector)。



## 限制



* 节点问题探测器的内核文件探测功能，目前仅支持基于文件的内核日志，不支持基于类似 journald 之类的日志工具。

* 节点问题探测器的内核文件探测功能，对内核日志格式有一定假设，目前仅能用于 Ubuntu 和 Debian。不过很容易对其进行扩展以[支持其他日志格式](/docs/admin/node-problem/#support-other-log-format)。



## 在 GCE 集群中开启和关闭节点问题探测器



节点问题探测器以[集群插件形式运行](/docs/admin/cluster-large/#addon-resources)，在 GCE 集群中默认开启了该功能。

您可以在运行 `kube-up.sh` 前，通过设置 `KUBE_ENABLE_NODE_PROBLEM_DETECTOR` 环境变量来开启/关闭它。



## 在其他环境中使用节点问题探测器



在 GCE 以外的环境中开启节点健康探测器时，您可以使用 `kubectl` 命令或着插件 Pod。



### Kubectl

Kubectl 是在 GCE 之外的环境中启动节点问题探测器的推荐方式。它提供了更加灵活的管理功能，例如可以改写默认配置来使探测器更适合您自己的环境，或者探测自定义的节点问题。



* **步骤1:** 创建 `node-problem-detector.yaml`

```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: node-problem-detector-v0.1
  namespace: kube-system
  labels:
    k8s-app: node-problem-detector
    version: v0.1
    kubernetes.io/cluster-service: "true"
spec:
  template:
    metadata:
      labels:
        k8s-app: node-problem-detector
        version: v0.1
        kubernetes.io/cluster-service: "true"
    spec:
      hostNetwork: true
      containers:
      - name: node-problem-detector
        image: gcr.io/google_containers/node-problem-detector:v0.1
        securityContext:
          privileged: true
        resources:
          limits:
            cpu: "200m"
            memory: "100Mi"
          requests:
            cpu: "20m"
            memory: "20Mi"
        volumeMounts:
        - name: log
          mountPath: /log
          readOnly: true
      volumes:
      - name: log
        hostPath:
          path: /var/log/
```

***注意，您必须确认上述配置中的系统日志路径（`/var/log`）必须和您操作系统发行版的系统路径相匹配。***



* **步骤2:** 使用 `kubectl` 启动节点问题探测器：

```shell
kubectl create -f node-problem-detector.yaml
```



### 插件 Pod

这种方式适用于已有自己的集群引导方案的用户，以及不需要修改默认配置的用户。他们可以利用插件 Pod 来进一步自动化他们的部署。

只要创建 `node-problem-detector.yaml`，并且将其放入 master 节点的插件 Pod 目录 `/etc/kubernetes/addons/node-problem-detector` 即可。



## 修改配置



在构建在节点问题探测器的容器镜像时，预先嵌入了[默认配置](https://github.com/kubernetes/node-problem-detector/tree/v0.1/config)。

但是，您可以使用[ConfigMap](/docs/tasks/configure-pod-container/configmap/)来改写默认配置，步骤如下：



* **步骤1:** 修改 `config/` 目录下的配置文件。
* **步骤2:** 使用命令 `kubectl  create configmap node-problem-detector-config --from-file=config/` 创建 ConfigMap `node-problem-detector-config`。
* **步骤3:** 修改 `node-problem-detector.yaml` 以应用步骤2中创建的 ConfigMap：

```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: node-problem-detector-v0.1
  namespace: kube-system
  labels:
    k8s-app: node-problem-detector
    version: v0.1
    kubernetes.io/cluster-service: "true"
spec:
  template:
    metadata:
      labels:
        k8s-app: node-problem-detector
        version: v0.1
        kubernetes.io/cluster-service: "true"
    spec:
      hostNetwork: true
      containers:
      - name: node-problem-detector
        image: gcr.io/google_containers/node-problem-detector:v0.1
        securityContext:
          privileged: true
        resources:
          limits:
            cpu: "200m"
            memory: "100Mi"
          requests:
            cpu: "20m"
            memory: "20Mi"
        volumeMounts:
        - name: log
          mountPath: /log
          readOnly: true
        - name: config # Overwrite the config/ directory with ConfigMap volume
          mountPath: /config
          readOnly: true
      volumes:
      - name: log
        hostPath:
          path: /var/log/
      - name: config # Define ConfigMap volume
        configMap:
          name: node-problem-detector-config
```



* **步骤4:** 使用修改后的 yaml 文件重新创建节点问题探测器。

```shell
kubectl delete -f node-problem-detector.yaml # If you have a node-problem-detector running
kubectl create -f node-problem-detector.yaml
```



***注意：上述方法只适用于使用 `kubectl` 命令启动节点问题探测器的方式。***

因为插件管理器不支持 ConfigMap，所以对于以集群插件 Pod 运行节点问题探测器的方式，当前无法修改配置。



## 内核监控器

*内核监控器* 是节点问题探测器中的一个守护进程。它监控内核日志并根据预定义规则探测已知内核问题。

内核监控器根据一组预定义规则列表[`config/kernel-monitor.json`](https://github.com/kubernetes/node-problem-detector/blob/v0.1/config/kernel-monitor.json)来匹配内核问题。预定义规则列表是可扩展的，您可以通过修改配置文件来扩展它。



### 添加新的节点健康状况（NodeConditions）

为了支持新的节点健康状况，您可以在 `config/kernel-monitor.json` 文件中通过扩展 `conditions` 域来定义新的节点健康状况：

```json
{
  "type": "NodeConditionType",
  "reason": "CamelCaseDefaultNodeConditionReason",
  "message": "arbitrary default node condition message"
}
```



### 探测新问题

为了探测新问题，您可以在 `config/kernel-monitor.json` 文件中通过扩展 `rules` 新增规则定义。

```json
{
  "type": "temporary/permanent",
  "condition": "NodeConditionOfPermanentIssue",
  "reason": "CamelCaseShortReason",
  "message": "regexp matching the issue in the kernel log"
}
```



### 更改日志路径

在不同操作系统发行版中，内核日志可能存放在不同的路径中。`config/kernel-monitor.json` 文件中的 `log` 域表示的是容器内日志的路径。您可以对它进行修改，使之与您的操作系统发行版一致。



### 支持其它日志格式

内核监控器使用[`Translator`](https://github.com/kubernetes/node-problem-detector/blob/v0.1/pkg/kernelmonitor/translator/translator.go)插件将内核日志转换成内部数据结构。它允许您轻松的实现一个用于新日志格式翻译器。



## 附加说明

我们建议您在您的集群中使用节点问题探测器，但是您需要知道，它会带来每个节点额外的资源开销。通常情况下不会有什么问题，因为：

* 内核日志生成的相对缓慢。
* 已经预先设置了节点问题探测器的资源限制。
* 即使在高负载的情况下，资源占用也是可接受的。
（参见[相关基准结果](https://github.com/kubernetes/node-problem-detector/issues/2#issuecomment-220255629)）
