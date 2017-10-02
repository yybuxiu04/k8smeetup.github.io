---
redirect_from:
- "/docs/user-guide/liveness/"
- "/docs/user-guide.liveness.html"
title: 配置Liveness和Readiness探针
cn-approvers:
- rootsongjc
cn-reviewers:
- rootsongjc
---

{% capture overview %}


本文将向您展示如何配置容器的存活和可读性探针。

[kubelet](/docs/admin/kubelet/) 使用 liveness probe（存活探针）来确定何时重启容器。例如，当应用程序处于运行状态但无法做进一步操作，liveness 探针将捕获到 deadlock，重启处于该状态下的容器，使应用程序在存在 bug 的情况下依然能够继续运行下去。

Kubelet 使用 readiness probe（就绪探针）来确定容器是否已经就绪可以接受流量。只有当 Pod 中的容器都处于就绪状态时 kubelet 才会认定该 Pod处于就绪状态。该信号的作用是控制哪些 Pod应该作为service的后端。如果 Pod 处于非就绪状态，那么它们将会被从 service 的 load balancer中移除。

{% endcapture %}

{% capture prerequisites %}

{% include task-tutorial-prereqs.md %}

{% endcapture %}

{% capture steps %}

## 定义 liveness 命令

许多长时间运行的应用程序最终会转换到 broken 状态，除非重新启动，否则无法恢复。Kubernetes 提供了 liveness probe 来检测和补救这种情况。

在本次练习将基于 `gcr.io/google_containers/busybox`镜像创建运行一个容器的 Pod。以下是 Pod 的配置文件`exec-liveness.yaml`：

{% include code.html language="yaml" file="exec-liveness.yaml" ghlink="/docs/tasks/configure-pod-container/exec-liveness.yaml" %}

该配置文件给 Pod 配置了一个容器。`periodSeconds` 规定 kubelet 要每隔5秒执行一次 liveness probe。  `initialDelaySeconds` 告诉 kubelet 在第一次执行 probe 之前要的等待5秒钟。探针检测命令是在容器中执行 `cat /tmp/healthy` 命令。如果命令执行成功，将返回0，kubelet 就会认为该容器是活着的并且很健康。如果返回非0值，kubelet 就会杀掉这个容器并重启它。

容器启动时，执行该命令：

```shell
/bin/sh -c "touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600"
```

在容器生命的最初30秒内有一个 `/tmp/healthy` 文件，在这30秒内 `cat /tmp/healthy`命令会返回一个成功的返回码。30秒后， `cat /tmp/healthy` 将返回失败的返回码。

创建Pod：

```shell
kubectl create -f https://k8s.io/docs/tasks/configure-pod-container/exec-liveness.yaml
```

在30秒内，查看 Pod 的 event：

```
kubectl describe pod liveness-exec
```

结果显示没有失败的 liveness probe：

```shell
FirstSeen    LastSeen    Count   From            SubobjectPath           Type        Reason      Message
--------- --------    -----   ----            -------------           --------    ------      -------
24s       24s     1   {default-scheduler }                    Normal      Scheduled   Successfully assigned liveness-exec to worker0
23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulling     pulling image "gcr.io/google_containers/busybox"
23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulled      Successfully pulled image "gcr.io/google_containers/busybox"
23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Created     Created container with docker id 86849c15382e; Security:[seccomp=unconfined]
23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Started     Started container with docker id 86849c15382e
```

35秒后，再次查看 Pod 的 event：

```shell
kubectl describe pod liveness-exec
```

在最下面有一条信息显示 liveness probe 失败，容器被删掉并重新创建。

```shell
FirstSeen LastSeen    Count   From            SubobjectPath           Type        Reason      Message
--------- --------    -----   ----            -------------           --------    ------      -------
37s       37s     1   {default-scheduler }                    Normal      Scheduled   Successfully assigned liveness-exec to worker0
36s       36s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulling     pulling image "gcr.io/google_containers/busybox"
36s       36s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulled      Successfully pulled image "gcr.io/google_containers/busybox"
36s       36s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Created     Created container with docker id 86849c15382e; Security:[seccomp=unconfined]
36s       36s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Started     Started container with docker id 86849c15382e
2s        2s      1   {kubelet worker0}   spec.containers{liveness}   Warning     Unhealthy   Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
```

再等30秒，确认容器已经重启：

```shell
kubectl get pod liveness-exec
```

从输出结果来`RESTARTS`值加1了。

```shell
NAME            READY     STATUS    RESTARTS   AGE
liveness-exec   1/1       Running   1          1m
```

## 定义 liveness HTTP请求

我们还可以使用 HTTP GET 请求作为 liveness probe。下面是一个基于`gcr.io/google_containers/liveness`镜像运行了一个容器的 Pod 的例子`http-liveness.yaml`：

{% include code.html language="yaml" file="http-liveness.yaml" ghlink="/docs/tasks/configure-pod-container/http-liveness.yaml" %}

该配置文件只定义了一个容器，`livenessProbe` 指定 kubelet 需要每隔3秒执行一次 liveness probe。`initialDelaySeconds` 指定 kubelet 在该执行第一次探测之前需要等待3秒钟。该探针将向容器中的 server 的8080端口发送一个HTTP GET 请求。如果server的`/healthz`路径的 handler 返回一个成功的返回码，kubelet 就会认定该容器是活着的并且很健康。如果返回失败的返回码，kubelet 将杀掉该容器并重启它。

任何大于200小于400的返回码都会认定是成功的返回码。其他返回码都会被认为是失败的返回码。

查看server的源码：[server.go](http://k8s.io/docs/user-guide/liveness/image/server.go).

最开始的10秒该容器是活着的， `/healthz` handler 返回200的状态码。这之后将返回500的返回码。

```go
http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    duration := time.Now().Sub(started)
    if duration.Seconds() > 10 {
        w.WriteHeader(500)
        w.Write([]byte(fmt.Sprintf("error: %v", duration.Seconds())))
    } else {
        w.WriteHeader(200)
        w.Write([]byte("ok"))
    }
})
```

容器启动3秒后，kubelet 开始执行健康检查。第一次健康监测会成功，但是10秒后，健康检查将失败，kubelet将杀掉和重启容器。

创建一个 Pod 来测试一下 HTTP liveness检测：

```shell
kubectl create -f https://k8s.io/docs/tasks/configure-pod-container/http-liveness.yaml
```

10秒后，查看 Pod 的 event，确认 liveness probe 失败并重启了容器。

```shell
kubectl describe pod liveness-http
```

## 定义 TCP liveness probe

第三种 liveness probe 使用 TCP Socket。 使用此配置，kubelet 将尝试在指定端口上打开容器的套接字。 如果可以建立连接，容器被认为是健康的，如果不能就认为是失败的。

{% include code.html language="yaml" file="tcp-liveness-readiness.yaml" ghlink="/docs/tasks/configure-pod-container/tcp-liveness-readiness.yaml" %}

如您所见，TCP 检查的配置与 HTTP 检查非常相似。 此示例同时使用了 readiness 和 liveness probe。 容器启动后5秒钟，kubelet将发送第一个 readiness probe。 这将尝试连接到端口8080上的 goproxy 容器。如果探测成功，则该 Pod 将被标记为就绪。Kubelet 将每隔10秒钟执行一次该检查。

如您所见，TCP 检查的配置与 HTTP 检查非常相似。 此示例同时使用了 readiness 和 liveness probe。 容器启动后5秒钟，kubelet 将发送第一个 readiness probe。 这将尝试连接到端口8080上的 goproxy 容器。如果探测成功，则该 pod 将被标记为就绪。Kubelet 将每隔10秒钟执行一次该检查。

除了 readiness probe之外，该配置还包括 liveness probe。 容器启动15秒后，kubelet 将运行第一个 liveness probe。 就像readiness probe一样，这将尝试连接到 goproxy 容器上的8080端口。如果 liveness probe 失败，容器将重新启动。


## 使用命名的端口

可以使用命名的 [ContainerPort ](/docs/api-reference/v1.6/#containerport-v1-core)作为 HTTP 或 TCP liveness检查：

```yaml
ports:
- name: liveness-port
  containerPort: 8080
  hostPort: 8080

livenessProbe:
  httpGet:
  path: /healthz
  port: liveness-port
```

## 定义readiness probe

有时，应用程序暂时无法对外部流量提供服务。 例如，应用程序可能需要在启动期间加载大量数据或配置文件。 在这种情况下，您不想杀死应用程序，也不想发送请求。 Kubernetes提供了readiness probe来检测和减轻这些情况。 Pod中的容器可以报告自己还没有准备，不能处理Kubernetes服务发送过来的流量。

Readiness probe的配置跟liveness probe很像。唯一的不同是使用 `readinessProbe `而不是`livenessProbe`。

```yaml
readinessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

Readiness probe 的 HTTP 和 TCP 的探测器配置跟 liveness probe 一样。

Readiness 和 livenss probe 可以并行用于同一容器。 使用两者可以确保流量无法到达未准备好的容器，并且容器在失败时重新启动。

## Configure Probes

## 配置 Probe

{% comment %}
Eventually, some of this section could be moved to a concept topic.
{% endcomment %}



[Probe ](/docs/api-reference/v1.6/#probe-v1-core)中有很多精确和详细的配置，通过它们您能准确的控制 liveness 和 readiness 检查：

- `initialDelaySeconds`：容器启动后第一次执行探测是需要等待多少秒。
- `periodSeconds`：执行探测的频率。默认是10秒，最小1秒。
- `timeoutSeconds`：探测超时时间。默认1秒，最小1秒。
- `successThreshold`：探测失败后，最少连续探测成功多少次才被认定为成功。默认是 1。对于 liveness 必须是 1。最小值是 1。 
- `failureThreshold`：探测成功后，最少连续探测失败多少次才被认定为失败。默认是 3。最小值是 1。



[HTTP probe ](/docs/api-reference/v1.6/#httpgetaction-v1-core)中可以给 `httpGet`设置其他配置项：

- `host`：连接的主机名，默认连接到 pod 的 IP。您可能想在 http header 中设置 "Host" 而不是使用 IP。
- `scheme`：连接使用的 schema，默认HTTP。
- `path`: 访问的HTTP server 的 path。
- `httpHeaders`：自定义请求的 header。HTTP运行重复的 header。
- `port`：访问的容器的端口名字或者端口号。端口号必须介于 1 和 65525 之间。



对于 HTTP 探测器，kubelet 向指定的路径和端口发送 HTTP 请求以执行检查。 Kubelet 将 probe 发送到容器的 IP 地址，除非地址被`httpGet`中的可选`host`字段覆盖。 在大多数情况下，您不想设置主机字段。 有一种情况下您可以设置它。 假设容器在127.0.0.1上侦听，并且 Pod 的`hostNetwork`字段为 true。 然后，在`httpGet`下的`host`应该设置为127.0.0.1。 如果您的 pod 依赖于虚拟主机，这可能是更常见的情况，您不应该是用`host`，而是应该在`httpHeaders`中设置`Host`头。

{% endcapture %}

{% capture whatsnext %}

* 关于 [Container Probes](/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) 的更多信息

### 参考

* [Pod](/docs/api-reference/v1.6/#pod-v1-core)
* [Container](/docs/api-reference/v1.6/#container-v1-core)
* [Probe](/docs/api-reference/v1.6/#probe-v1-core)

{% endcapture %}

{% include templates/task.md %}
