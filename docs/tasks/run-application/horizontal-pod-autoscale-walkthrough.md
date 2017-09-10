---
approvers:
- fgrzadkowski
- jszczepkowski
- justinsb
- directxman12
title: Pod水平自动伸缩（Horizontal Pod Autoscaling）演练
---


Horizontal Pod Autoscaling可以根据CPU利用率自动伸缩一个Replication Controller、Deployment
或者Replica Set中的Pod数量（或者基于一些应用程序提供的度量指标，目前这一功能处于alpha版本）。


本文将引导您了解如何为php-apache服务器配置和使用Horizontal Pod Autoscaling。 更多Horizontal Pod Autoscaling信息请参阅[Horizontal Pod Autoscaling用户指南](/docs/tasks/run-application/horizontal-pod-autoscale/)。


## 前提条件


本文示例需要一个1.2或者更高版本的可运行的Kubernetes集群以及kubectl。
由于Horizontal Pod Autoscaler需要使用[Heapster](https://github.com/kubernetes/heapster)所收集到的
度量数据，请确保Heapster被正确部署到Kubernetes集群中（如果您遵循[在GCE上搭建Kubernetes入门指南](/docs/getting-started-guides/gce)
中的步骤，默认情况下Heapster已经被部署并被启用）。


如果需要为Horizontal Pod Autoscaler指定多种资源度量指标，您的Kubernetes集群以及kubectl至少需要达到1.6版本。此外，如果需要使用自定义度量指标，您的Kubernetes
集群还必须能够与提供这些自定义指标的API服务器通信。更多详细信息，请参阅[Horizontal Pod Autoscaling用户指南](/docs/user-guide/horizontal-pod-autoscaling/#support-for-custom-metrics)。


## 第一步：部署和运行php-apache服务器并将其暴露成为Kubernetes服务


为了演示Horizontal Pod Autoscaler，我们将使用一个基于php-apache镜像的定制Docker镜像。
在[这里](/docs/user-guide/horizontal-pod-autoscaling/image/Dockerfile)您可以查看完整的Dockerfile定义。
镜像中包括一个[index.php](/docs/user-guide/horizontal-pod-autoscaling/image/index.php)页面，其中包含了一些可以运行CPU密集计算任务的代码。


首先，我们部署一个Deployment运行上述Docker镜像并将其暴露成为一个Kubernetes服务（service）：

```shell
$ kubectl run php-apache --image=gcr.io/google_containers/hpa-example --requests=cpu=200m --expose --port=80
service "php-apache" created
deployment "php-apache" created
```


## 第二步：创建Horizontal Pod Autoscaler


现在，php-apache服务器已经运行，我们将通过[kubectl autoscale](https://github.com/kubernetes/kubernetes/blob/{{page.githubbranch}}/docs/user-guide/kubectl/kubectl_autoscale.md)命令创建Horizontal Pod Autoscaler。
以下命令将创建一个Horizontal Pod Autoscaler用于控制Pod的副本数量在1到10之间。
大致来说，HPA将通过增加或者减少Pod副本的数量（通过Deployment）以保持所有Pod的平均CPU利用率在50%以内
（由于每个Pod通过[kubectl run](https://github.com/kubernetes/kubernetes/blob/{{page.githubbranch}}/docs/user-guide/kubectl/kubectl_run.md)
申请了200 milli-cores CPU，所以50%的CPU利用率意味着平均CPU利用率为100 milli-cores）。
相关算法的详情请参阅[这篇文档](https://git.k8s.io/community/contributors/design-proposals/horizontal-pod-autoscaler.md#autoscaling-algorithm)。

```shell
$ kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
deployment "php-apache" autoscaled
```


现在，我们可以通过以下命令查看Horizontal Pod Autoscaler的状态：

```shell
$ kubectl get hpa
NAME         REFERENCE                     TARGET    MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache/scale   0% / 50%  1         10        1          18s

```


请注意在上面的命令输出中，目前的CPU利用率是0%，这是由于我们尚未发送任何请求到服务器（
``CURRENT``列显示了相应Deployment所控制的所有Pod的平均CPU利用率）。


## 第三步：增加负载


现在，我们将看到Horizontal Pod Autoscaler如何对增加负载作出反应。
我们将启动一个容器，并通过一个循环向php-apache服务器发送无限的查询请求（请在另一个终端中运行以下命令）：

```shell
$ kubectl run -i --tty load-generator --image=busybox /bin/sh

Hit enter for command prompt

$ while true; do wget -q -O- http://php-apache.default.svc.cluster.local; done
```


在几分钟时间内，通过以下命令，我们可以看到CPU负载升高了：

```shell
$ kubectl get hpa
NAME         REFERENCE                     TARGET      CURRENT   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache/scale   305% / 50%  305%      1         10        1          3m

```


在我们的环境中，由于请求增多，CPU利用率已经升至305%。
因此，Deployment的副本数量已经增长到了7：

```shell
$ kubectl get deployment php-apache
NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
php-apache   7         7         7            7           19m
```


**注意** 有时最终副本的数量可能需要几分钟才能稳定下来。
由于环境的差异，不同环境中最终的副本数量可能与本示例中的数量不同。


## 第四步：停止负载


我们将通过停止负载来结束我们的示例。


在我们创建基于`busybox`镜像的容器的终端中，输入`<Ctrl> + C`来终止负载的产生。


然后我们可以再次查看负载状态（等待几分钟时间）：

```shell
$ kubectl get hpa
NAME         REFERENCE                     TARGET       MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache/scale   0% / 50%     1         10        1          11m

$ kubectl get deployment php-apache
NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
php-apache   1         1         1            1           27m
```


现在，CPU利用率已经降到0，所以HPA将自动缩减副本数量至1。

**Note** autoscaling the replicas may take a few minutes.
**注意** 自动伸缩完成副本数量的改变可能需要几分钟的时间。


## 基于多项度量指标和自定义度量指标的自动伸缩


利用`autoscaling/v2alpha1`API版本（API version），您可以在自动伸缩`php-apache`这个Deployment时引入其他度量指标。


首先，获取`autosacling/v2alpha1`形式的HorizontalPodAutoscaler的YAML文件：

```shell
$ kubectl get hpa.autoscaling.v2alpha1 -o yaml > /tmp/hpa-v2.yaml
```


在编辑器中打开`/tmp/hpa-v2.yaml`：

```yaml
apiVersion: autoscaling/v2alpha1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1beta1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 50
status:
  observedGeneration: 1
  lastScaleTime: <some-time>
  currentReplicas: 1
  desiredReplicas: 1
  currentMetrics:
  - type: Resource
    resource:
      name: cpu
      currentAverageUtilization: 0
      currentAverageValue: 0
```


注意到在YAML文件中，`targetCPUUtilizationPercentage`字段已经被名为`metrics`的数组所取代。
CPU利用率这个度量指标是一个*资源度量指标*（resource metric），因为它被表示为容器上指定资源的百分比。除CPU外，您还可以指定
其他资源度量指标。默认情况下，目前唯一支持的其他资源度量指标为内存。只要Heapster被部署，这些资源度量指标就是可用的，并且他们
不会在不同的Kubernetes集群中改变名称。


您还可以指定资源度量指标使用绝对数值，而不是百分比。在Horizontal Pod Autoscaler定义中使用`targetAverageValue`而不是
`targetAverageUtilization`即可做到这一点。


还有两种其他类型的度量指标，他们被认为是*自定义度量指标*：即Pod度量指标和对象度量指标（pod metrics and object metrics）。这些度量指标可能具有特定于集群的名称，并且需要更高级的集群监控设置。


第一种可选的度量指标类型是*Pod度量指标*。这些指标从某一方面描述了Pod，在不同Pod之间进行平均，并通过与一个目标值比对来
确定副本的数量。它们的工作方式与资源度量指标非常相像，差别是它们*仅*包含`targetAverageValue`字段。


Pod度量指标通过如下代码块定义：
```yaml
type: Pods
pods:
  metricName: packets-per-second
  targetAverageValue: 1k
```


第二种可选的度量指标类型是*对象度量指标*。相对于描述Pod，这些度量指标用于描述一个在相同名字空间(namespace)中的其他对象。
请注意这些度量指标用于描述这些对象，并非从对象中获取。对象度量指标并不涉及平均计算，以下示例展示了一个对象度量指标：

```yaml
type: Object
object:
  metricName: requests-per-second
  target:
    apiVersion: extensions/v1beta1
    kind: Ingress
    name: main-route
  targetValue: 2k
```


如果您指定了多个上述类型的度量指标，HorizontalPodAutoscaler将会依次考量各个指标。
HorizontalPodAutoscaler将会计算每一个指标所提议的副本数量，然后最终选择一个最高值。


比如，如果您的监控系统能够提供网络流量数据，您可以通过`kubectl edit`命令将上述Horizontal Pod Autoscaler的定义更改为：

```yaml
apiVersion: autoscaling/v2alpha1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1beta1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 50
  - type: Pods
    pods:
      metricName: packets-per-second
      targetAverageValue: 1k
  - type: Object
    object:
      metricName: requests-per-second
      target:
        apiVersion: extensions/v1beta1
        kind: Ingress
        name: main-route
      targetValue: 10k
status:
  observedGeneration: 1
  lastScaleTime: <some-time>
  currentReplicas: 1
  desiredReplicas: 1
  currentMetrics:
  - type: Resource
    resource:
      name: cpu
      currentAverageUtilization: 0
      currentAverageValue: 0
```


然后，您的HorizontalPodAutoscaler将会尝试确保每个Pod的CPU利用率在50%以内，每秒能够服务1000个数据包请求，并确保所有
在Ingress后的Pod每秒能够服务的请求总数达到10000个。


## 附录：Horizontal Pod Autoscaler状态条件


当使用`autoscaling/v2alpha1`形式的HorizontalPodAutoscaler时，您将可以看到由Kubernetes为HorizongtalPodAutoscaler
设置的*状态条件*（status conditions）。这些状态条件可以显示当前HorizontalPodAutoscaler是否能够执行伸缩以及是否受到各种限制。


`status.conditions`字段展示了这些状态条件。可以通过`kubectl describe hpa`命令查看当前影响一个HorizontalPodAutoscaler的各种状态条件信息：

```shell
$ kubectl describe hpa cm-test
Name:                           cm-test
Namespace:                      prom
Labels:                         <none>
Annotations:                    <none>
CreationTimestamp:              Fri, 16 Jun 2017 18:09:22 +0000
Reference:                      ReplicationController/cm-test
Metrics:                        ( current / target )
  "http_requests" on pods:      66m / 500m
Min replicas:                   1
Max replicas:                   4
ReplicationController pods:     1 current / 1 desired
Conditions:
  Type                  Status  Reason                  Message
  ----                  ------  ------                  -------
  AbleToScale           True    ReadyForNewScale        the last scale time was sufficiently old as to warrant a new scale
  ScalingActive         True    ValidMetricFound        the HPA was able to successfully calculate a replica count from pods metric http_requests
  ScalingLimited        False   DesiredWithinRange      the desired replica count is within the acceptible range
Events:
```


对于上面展示的这个HorizontalPodAutoscaler，我们可以看出有若干状态条件处于健康状态。首先，`AbleToScale`表明HPA是否
可以获取和更新伸缩信息，以及是否存在阻止伸缩的各种回退条件。其次，`ScalingActive` 表明HPA是否被启用（即目标的副本数量不为零）
以及是否能够完成伸缩计算。当这一状态为`False`时，通常表明获取度量指标存在问题。最后一个条件`ScalingLimitted`表明所需伸缩的值
被HorizontalPodAutoscaler所定义的最大或者最小值所限制（即已经达到最大或者最小伸缩值）。这通常表明您可能需要调整HorizontalPodAutoscaler
所定义的最大或者最小副本数量的限制了。


## 附录：其他可能的情况


### 使用YAML文件创建HorizontalAutoScaler


除了使用`kubectl autoscale`命令，也可以使用包含以下类似内容的[hpa-php-apache.yaml](/docs/user-guide/horizontal-pod-autoscaling/hpa-php-apache.yaml)文件创建autoscaler：

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1beta1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```


基于上述文件，可以通过以下命令创建autoscaler：

```shell
$ kubectl create -f docs/user-guide/horizontal-pod-autoscaling/hpa-php-apache.yaml
horizontalpodautoscaler "php-apache" created
```
