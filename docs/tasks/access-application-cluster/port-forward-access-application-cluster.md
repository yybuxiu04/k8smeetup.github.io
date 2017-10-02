---
title: 通过端口转发访问集群中的应用程序
redirect_from:
- "/docs/user-guide/connecting-to-applications-port-forward/"
- "/docs/user-guide/connecting-to-applications-port-forward.html"
cn-approvers:
- rootsongjc
cn-reviewers:
- rootsongjc
---

{% capture overview %}



本页向您展示如何使用 `kubectl port-forward` 命令连接到运行在 Kubernetes 集群中的 Redis 服务器。这种类型的连接对于数据库调试很有帮助。


{% endcapture %}


{% capture prerequisites %}

* {% include task-tutorial-prereqs.md %}

* 安装 [redis-cli](http://redis.io/topics/rediscli)。

{% endcapture %}

{% capture steps %}



## 创建一个 Pod 来运行 Redis 服务器

1. 创建一个 Pod：

   ```
    kubectl create -f https://k8s.io/docs/tasks/access-application-cluster/redis-master.yaml
   ```

   命令运行成功后将有以下输出验证该 Pod 是否已经创建：

   ```
   pod "redis-master" created
   ```

2. 检查 Pod 是否正在运行且处于就绪状态：

   ```
   kubectl get pods
   ```

   当 Pod 就绪，输出显示 Running 的状态：

   ```
    NAME           READY     STATUS    RESTARTS   AGE
    redis-master   2/2       Running   0          41s
   ```

3. 验证 Redis 服务器是否已在 Pod 中运行，并监听 6379 端口：

   ```
    {% raw %}
    kubectl get pods redis-master --template='{{(index (index .spec.containers 0).ports 0).containerPort}}{{"\n"}}'
    {% endraw %}
   ```

   端口输出如下：

   ```
   6379
   ```



## 将本地端口转发到 Pod 中的端口

1. 将本地工作站上的 6379 端口转发到 redis-master pod 的 6379 端口：

   ```
    kubectl port-forward redis-master 6379:6379
   ```

   输出类似于：

   ```
    I0710 14:43:38.274550    3655 portforward.go:225] Forwarding from 127.0.0.1:6379 -> 6379
    I0710 14:43:38.274797    3655 portforward.go:225] Forwarding from [::1]:6379 -> 6379
   ```

2. 启动 Redis 命令行界面

   ```
   redis-cli
   ```

3. 在 Redis 命令行提示符下，输入 `ping` 命令：

   ```
    127.0.0.1:6379>ping
   ```

   Ping 请求成功返回 PONG。

{% endcapture %}

{% capture discussion %}



## 讨论

创建连接，将本地的 6379 端口转发到运行在 Pod 中的 Redis 服务器的 6379 端口。有了这个连接您就可以在本地工作站中调试运行在 Pod 中的数据库。

{% endcapture %}

{% capture whatsnext %}
了解更多有关 [kubectl port-forward](/docs/user-guide/kubectl/v1.6/#port-forward) 的信息。
{% endcapture %}

{% include templates/task.md %}