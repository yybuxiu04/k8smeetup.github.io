---
title: "示例：使用 Redis 的 PHP Guestbook 应用程序"
assignees:
- ahmetb
- jeffmendoza
---



此示例展示如何使用 Kubernetes 和 [Docker](https://www.docker.com/)构建一个简单的多层 Web 应用程序。

**目录**





  - [Guestbook 示例](#guestbook-示例)
    - [先决条件](#先决条件)
    - [快速开始](#快速开始)
    - [第一步：启动 redis master](#第一步-启动-redis-master)
      - [定义 Deployment](#定义-deployment)
      - [定义 Service](#定义-service)
      - [创建 Service](#创建-service)
      - [Service 发现](#service-发现)
        - [环境变量](#环境变量)
        - [DNS 服务](#dns-服务)
      - [创建 Deployment](#创建-deployment)
      - [小插曲](#小插曲)
    - [第二步：启动 redis slave](#第二步-启动-redis-slave)
    - [第三步：启动 guestbook 的前端](#第三步-启动-guestbook-的前端)
      - [对前端服务使用 'type: LoadBalancer' （特定云提供商）](#对前端服务使用-type-loadbalancer-特定云提供商)
    - [第四步：清理](#第四步-清理)
    - [故障排除](#故障排除)
    - [附录：外部访问 guestbook 站点](#附录-外部访问-guestbook-站点)
      - [Google Compute Engine 外部负载均衡器详细信息](#google-compute-engine-外部负载均衡器详细信息)



示例包括:

- 一个 Web 前端
- 一个 [redis](http://redis.io/) master（用于存储）和一个主从复制的 redis 'slaves' 。

Web 前端通过 javascript redis API 调用与 redis master 交互。

**注意**：如果您在 [Google Container Engine](https://cloud.google.com/container-engine/) 中安装运行此示例，
请参阅 [this Google Container Engine guestbook walkthrough](https://cloud.google.com/container-engine/docs/tutorials/guestbook)。 
基本概念是相同的，但是演练是针对 Container Engine 设置的。



### 先决条件

此示例需要运行的 Kubernetes 集群。 首先，通过获取集群状态来检查 kubectl 是否正确配置：

```console
$ kubectl cluster-info
```

如果你看到一个 url 的回显信息，说明你已经准备好了。 如果没有，请阅读[入门指南](http://kubernetes.io/docs/getting-started-guides/)了解如何入门，
并按照[先决条件](http://kubernetes.io/docs/user-guide/prereqs/)来安装和配置 `kubectl` 。 如上所述，如果您设置了 Google Container Engine 集群，
请改为阅读[本示例](https://cloud.google.com/container-engine/docs/tutorials/guestbook)。

本示例中引用的所有文件都可以从 [GitHub](https://git.k8s.io/examples/guestbook) 下载。



### 快速开始

本节展示了使示例工作最简单的方法。 如果你想知道这些细节，你应该跳过这个并阅读[剩下的例子](#step-one-start-up-the-redis-master)。

用一个命令启动 guestbook ：

```console
$ kubectl create -f guestbook/all-in-one/guestbook-all-in-one.yaml
service "redis-master" created
deployment "redis-master" created
service "redis-slave" created
deployment "redis-slave" created
service "frontend" created
deployment "frontend" created
```

或者，您可以通过运行以下命令启动 guestbook ：

```console
$ kubectl create -f guestbook/
```

然后列出所有 Services ：

```console
$ kubectl get services
NAME           CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
frontend       10.0.0.117   <none>        80/TCP     20s
redis-master   10.0.0.170   <none>        6379/TCP   20s
redis-slave    10.0.0.201   <none>        6379/TCP   20s
```

现在，您可以用前端服务的 `<Cluster-IP>:<PORT>` 访问每个节点上的 guestbook ，例如 本指南中的 `10.0.0.117:80` 。 `<Cluster-IP>` 是集群内部的 IP 。 
如果要从群集外部访问 guestbook ，请在前端 Service 的 `spec` 字段中添加 `type: NodePort` 。 然后，您可以从集群外部使用 `<NodeIP>:NodePort` 访问 guestbook 。 
在支持外部负载平衡器的云提供商上，将`type: LoadBalancer`添加到前端 Service 的`spec`字段中将为您的 Service 提供负载均衡器。 有几种访问 guestbook 的方法， 
请参阅[访问集群中运行的服务](https://kubernetes.io/docs/concepts/cluster-administration/access-cluster/#accessing-services-running-on-the-cluster)。

清除 guestbook :

```console
$ kubectl delete -f guestbook/all-in-one/guestbook-all-in-one.yaml
```

或者

```console
$ kubectl delete -f guestbook/
```



### 第一步 启动 redis master

在往下浏览详细细节之前，我们建议您先阅读 Kubernetes [概念和用户指南](http://kubernetes.io/docs/user-guide/)。

**注意**：示例中的 redis master *不是* 高可用的。 使其高可用是一个有趣而复杂的事 - redis 实际上并不支持多 master 的 Deployments ，因此实现高可用这事有点棘手，可能涉及磁盘的定期序列化等等。



#### 定义 Deployment

用文件 [redis-master-deployment.yaml](https://git.k8s.io/examples/guestbook/redis-master-deployment.yaml) 启动 redis master ，
该文件描述了一个 [pod](http://kubernetes.io/docs/user-guide/pods/)在容器中运行 redis key-value server 。

虽然我们有一个单独的 redis master 实例，但是我们使用 [Deployment](http://kubernetes.io/docs/user-guide/deployments/) 来强制保证有一个 pod 在运行。 
例如，如果节点要关闭，那么 Deployment 会确保 redis master 在一个健康节点上重新启动。 （我们这个简单示例，会导致数据丢失。）

文件 [redis-master-deployment.yaml](redis-master-deployment.yaml) 定义了redis master 的 Deployment ：



```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: redis-master
  # these labels can be applied automatically 
  # from the labels in the pod template if not set
  # labels:
  #   app: redis
  #   role: master
  #   tier: backend
spec:
  # this replicas value is default
  # modify it according to your case
  replicas: 1
  # selector can be applied automatically 
  # from the labels in the pod template if not set
  # selector:
  #   matchLabels:
  #     app: guestbook
  #     role: master
  #     tier: backend
  template:
    metadata:
      labels:
        app: redis
        role: master
        tier: backend
    spec:
      containers:
      - name: master
        image: gcr.io/google_containers/redis:e2e
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 6379
```



[下载示例文件](https://raw.githubusercontent.com/kubernetes/examples/master/guestbook/redis-master-deployment.yaml)




#### 定义 Service

Kubernetes [Service](http://kubernetes.io/docs/user-guide/services/) 被命名为负载均衡器，可以将流量代理到一个或多个容器。 
这是使用我们在上面`redis-master`  pod 中定义的 [labels](http://kubernetes.io/docs/user-guide/labels/) metadata 来实现的。 
如上所述，我们只有一个 redis master ，但是我们仍然想为它创建一个 Service 。 
为什么？ 因为它为我们提供了一个使用弹性 IP 路由到单个 master 的确定方法。

Services 可以通过 pod 的 labels 找到 pod 来负载均衡。
Service 描述的 selector 字段确定哪些 pod 将接收发送到 Service 的流量，`port` 和 `targetPort` 信息定义了 Service 代理将运行的端口。

文件 [redis-master-service.yaml](https://git.k8s.io/examples/guestbook/redis-master-deployment.yaml) 定义了 redis master Service:



```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-master
  labels:
    app: redis
    role: master
    tier: backend
spec:
  ports:
    # the port that this service should serve on
  - port: 6379
    targetPort: 6379
  selector:
    app: redis
    role: master
    tier: backend
```



[下载示例文件](https://raw.githubusercontent.com/kubernetes/examples/master/guestbook/redis-master-service.yaml)




#### 创建 Service

根据 [config最佳实践](http://kubernetes.io/docs/user-guide/config-best-practices/)，在创建 Deployments 之前创建相应的 Service ，以便调度程序可以传播构成 Service 的 pod 。 
所以我们先创建 Service ：

```console
$ kubectl create -f guestbook/redis-master-service.yaml
service "redis-master" created
```

然后检查 services 列表，其中应包括 redis-master ：

```console
$ kubectl get services
NAME              CLUSTER-IP       EXTERNAL-IP       PORT(S)       AGE
redis-master      10.0.76.248      <none>            6379/TCP      1s
```



可以很明显的看到 redis master 的所有 pods 都运行在 `<CLUSTER-IP>:<PORT>` 上。 一个 Service 可以将一个入站端口映射到后端端口中的任何 'targetPort' 。 
创建后，每个节点上的 Service 代理都设置为指定代理端口（这个示例设置为“6379”端口）。

如果在配置中省略 `targetPort`，`targetPort` 将默认为 `port`。 `targetPort` 是容器接受流量的端口，`port` 是抽象的 Service 端口，
可以是其他 pod 用于访问 Service 的任何端口。 为了简单起见，我们在以下配置中省略它。

从 slaves 到 masters 的流量可以分为两个步骤：

  - *redis slave* 将连接 *redis master Service* 的 `port`
  - 流量将从 Service `port`（ Service 节点）转发到 Service 侦听的 pod 上的 `targetPort` 上。

详情请参阅 [Connecting applications](http://kubernetes.io/docs/user-guide/connecting-applications/).



#### Service 发现

Kubernetes 主要支持两种模式的 Service 发现 - 环境变量和 DNS 。



##### 环境变量

Kubernetes 集群中的 services 可以通过其他容器内部的 [环境变量](https://kubernetes.io/docs/concepts/services-networking/service/#environment-variables) 来进行服务发现 。



##### DNS 服务

如果集群已启用，可以使用另一种方法：[集群 DNS 服务](https://kubernetes.io/docs/concepts/services-networking/service/#dns)。 这将使所有 pod 根据 Service 的名称自动对服务进行解析。

此示例已配置为默认使用 DNS 服务。

如果您的群集没有启用DNS服务，那么您可以先设置 [redis-slave-deployment.yaml](https://git.k8s.io/examples/guestbook/redis-slave-deployment.yaml) 和 [frontend-deployment.yaml](https://git.k8s.io/examples/guestbook/frontend-deployment.yaml)
中的环境变量 `GET_HOSTS_FROM` 的从 `dns` 到 `env` 的环境值，然后再启动应用程序。
（但是，这不是必须的。 您可以通过运行 `kubectl --namespace = kube-system get rc -l k8s-app = kube-dns` 来检查集群服务列表中的 DNS 服务。）
请注意，变换 env 会引发创建顺序依赖关系，因为 Services 需要在他们的客户端需要环境变量之前创建出来。



#### 创建 Deployment

然后，在您的 Kubernetes 群集中创建 redis master pod ，执行如下命令：

```console
$ kubectl create -f guestbook/redis-master-deployment.yaml
deployment "redis-master" created
```

您可以运行以下命令查看集群的 Deployment ：

```console
$ kubectl get deployments
NAME           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
redis-master   1         1         1            1           27s
```

然后，您可以列出群集中的pod，以验证 master 正在运行：

```console
$ kubectl get pods
```



您将看到群集中的所有pod，包括 redis master pod 和每个 pod 的状态。
redis master 的名字将与以下列表中的名称类似：

```console
NAME                            READY     STATUS    RESTARTS   AGE
redis-master-2353460263-1ecey   1/1       Running   0          1m
...
```

（请注意，根据网络环境，初始的 `docker pull` 拉取容器镜像可能需要几分钟的时间，当镜像下载完后，pod 的状态将会显示为 `Pending` ）。

`kubectl get pods` 只显示默认 [namespace](http://kubernetes.io/docs/user-guide/namespaces/)中的 pod 。 要查看所有 namespaces 中的 pod ，请运行：

```
kubectl get pods --all-namespaces
```

更多详细信息，请参阅[配置容器](http://kubernetes.io/docs/user-guide/configuring-containers/)和[部署应用程序](http://kubernetes.io/docs/user-guide/deploying-applications/)。



#### 小插曲

您可以通过 `kubectl describe pods/<POD-NAME>` 获取有关pod的信息，包括运行的机器。 例如，对于redis master ，您应该看到如下所示（您的pod名称将不同）：

```console
$ kubectl describe pods redis-master-2353460263-1ecey
Name:		redis-master-2353460263-1ecey
Node:		kubernetes-node-m0k7/10.240.0.5
...
Labels:		app=redis,pod-template-hash=2353460263,role=master,tier=backend
Status:		Running
IP:		10.244.2.3
Controllers:	ReplicaSet/redis-master-2353460263
Containers:
  master:
    Container ID:	docker://76cf8115485966131587958ea3cbe363e2e1dcce129e2e624883f393ce256f6c
    Image:		gcr.io/google_containers/redis:e2e
    Image ID:		docker://e5f6c5a2b5646828f51e8e0d30a2987df7e8183ab2c3ed0ca19eaa03cc5db08c
    Port:		6379/TCP
...
```



`Node` 是机器的名称和 IP ，例如在上面的例子中的 `kubernetes-node-m0k7` 。 您可以使用 `kubectl describe nodes kubernetes-node-m0k7` 命令查找有关此节点的更多详细信息。

如果您要查看某个 pod 的容器日志，可以运行：

```console
$ kubectl logs <POD-NAME>
```

这些日志通常会给您提供足够的信息进行故障排除。

如果您希望 SSH 连接到所列出的主机，以便您可以直接检查各种日志。 例如，如果使用 Google Compute Engine ，使用 `gcloud` ，你可以这样 SSH ：

```console
me@workstation$ gcloud compute ssh <NODE-NAME>
```



然后，您可以查看远程机器上的 Docker 容器。 您可能会看到这样的东西（指定的 ID 会有所不同）：

```console
me@kubernetes-node-krxw:~$ sudo docker ps
CONTAINER ID        IMAGE                                 COMMAND                 CREATED              STATUS              PORTS                   NAMES
...
0ffef9649265        redis:latest                          "/entrypoint.sh redi"   About a minute ago   Up About a minute                           k8s_master.869d22f3_redis-master-dz33o_default_1449a58a-5ead-11e5-a104-688f84ef8ef6_d74cb2b5
```

如果要查看某个容器的日志，可以运行：

```console
$ docker logs <container_id>
```



### 第二步 启动 redis slave

现在 redis master 是正在运行的，我们可以启动它的 'read slaves' 。

我们同样会定义为 pod 的副本，尽管这一次 - 与 redis master 不同 - 我们将把 replicas 的数量定义为2。
在 Kubernetes 中，Deployment 负责管理 pod 副本的多个实例。 如果 replicas 的数量低于指定的数量，Deployment 将自动启动新的 pod 。
（这种特殊的 pod 副本是一个非常好的测试工具，您可以直接尝试杀死 pod 的 Docker 进程，然后过一会再查看，它们会在新节点上重新上线。）

就像 master 一样，我们要一个 Service 来代理与 redis slaves 的连接。 在这种情况下，除了服务发现之外，slave Service 将会为 Web 应用程序客户端提供透明的负载平衡。

这次我们将  Service 和 Deployment 放在一个[文件](http://kubernetes.io/docs/user-guide/managing-deployments/#organizing-resource-configurations)中。 
将相关对象放在单独一个文件中通常比分开几个文件更好。
slaves 的规范文档在 [all-in-one/redis-slave.yaml](https://git.k8s.io/examples/guestbook/all-in-one/redis-slave.yaml) 中：



```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-slave
  labels:
    app: redis
    role: slave
    tier: backend
spec:
  ports:
    # the port that this service should serve on
  - port: 6379
  selector:
    app: redis
    role: slave
    tier: backend
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: redis-slave
  # these labels can be applied automatically
  # from the labels in the pod template if not set
  # labels:
  #   app: redis
  #   role: slave
  #   tier: backend
spec:
  # this replicas value is default
  # modify it according to your case
  replicas: 2
  # selector can be applied automatically
  # from the labels in the pod template if not set
  # selector:
  #   matchLabels:
  #     app: guestbook
  #     role: slave
  #     tier: backend
  template:
    metadata:
      labels:
        app: redis
        role: slave
        tier: backend
    spec:
      containers:
      - name: slave
        image: gcr.io/google_samples/gb-redisslave:v1
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        env:
        - name: GET_HOSTS_FROM
          value: dns
          # If your cluster config does not include a dns service, then to
          # instead access an environment variable to find the master
          # service's host, comment out the 'value: dns' line above, and
          # uncomment the line below.
          # value: env
        ports:
        - containerPort: 6379
```

[下载示例文件](https://raw.githubusercontent.com/kubernetes/examples/master/guestbook/all-in-one/redis-slave.yaml)




这次 Service 的 selector 是 `app=redis,role=slave,tier=backend` ，因为它是用来识别运行中的 redis slave 的 pod。 为 Service 设置 labels 通常是有用的，
因为我们可以使用 `kubectl get services -l "app=redis,role=slave,tier=backend"` 命令轻松找到它们。 
有关 labels 使用的更多信息，请参阅 [using-labels-effectively](http://kubernetes.io/docs/user-guide/managing-deployments/#using-labels-effectively)。

现在您已经创建了规范，通过运行以下命令在集群中创建 Service ：

```console
$ kubectl create -f guestbook/all-in-one/redis-slave.yaml
service "redis-slave" created
deployment "redis-slave" created

$ kubectl get services
NAME           CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
redis-master   10.0.76.248    <none>        6379/TCP   20m
redis-slave    10.0.112.188   <none>        6379/TCP   16s

$ kubectl get deployments
NAME           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
redis-master   1         1         1            1           22m
redis-slave    2         2         2            2           2m
```



当 Deployment 运行后，您可以列出群集中的 pod ，以验证 master 和 slaves 是否正在运行。 您可以看到类似于以下内容的清单：

```console
$ kubectl get pods
NAME                            READY     STATUS    RESTARTS   AGE
redis-master-2353460263-1ecey   1/1       Running   0          35m
redis-slave-1691881626-dlf5f    1/1       Running   0          15m
redis-slave-1691881626-sfn8t    1/1       Running   0          15m
```

您会看到一个redis master pod 和两个 redis slave pod。 如上所述，您可以运行 `kubectl describe pods/<POD_NAME>` 获取有关任何一个 pod 的更多信息。 
还可以查看 [kube-ui](http://kubernetes.io/docs/user-guide/ui/) 上的资源。



### 第三步 启动 guestbook 的前端

前端 pod 是一个简单的 PHP 服务器，配置为 slave 和 master 之间的 services 进行通信，具体取决于客户端的请求是读取还是写入的。 它暴露了一个简单的 AJAX 接口，并提供 Angular-based UX 服务。
然后，我们将用 Deployment 创建由前端 pod 副本组成的实例 - 这次有3个 replicas 。

与其他 pod 一样，我们现在想创建一个 Service 来分组前端 pod 。
Deployment 和 Service 的配置文档 [all-in-one/frontend.yaml](https://git.k8s.io/examples/guestbook/all-in-one/frontend.yaml):



```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  # if your cluster supports it, uncomment the following to automatically create
  # an external load-balanced IP for the frontend service.
  # type: LoadBalancer
  ports:
    # the port that this service should serve on
  - port: 80
  selector:
    app: guestbook
    tier: frontend
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: frontend
  # these labels can be applied automatically
  # from the labels in the pod template if not set
  # labels:
  #   app: guestbook
  #   tier: frontend
spec:
  # this replicas value is default
  # modify it according to your case
  replicas: 3
  # selector can be applied automatically
  # from the labels in the pod template if not set
  # selector:
  #   matchLabels:
  #     app: guestbook
  #     tier: frontend
  template:
    metadata:
      labels:
        app: guestbook
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google-samples/gb-frontend:v4
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        env:
        - name: GET_HOSTS_FROM
          value: dns
          # If your cluster config does not include a dns service, then to
          # instead access environment variables to find service host
          # info, comment out the 'value: dns' line above, and uncomment the
          # line below.
          # value: env
        ports:
        - containerPort: 80
```

[下载示例文件](https://raw.githubusercontent.com/kubernetes/examples/master/guestbook/all-in-one/frontend.yaml)




#### 对前端服务使用 'type: LoadBalancer' (特定云提供商)

对于支持此服务的云提供商（如 Google Compute Engine 或 Google Container Engine ），您可以在service 的 `spec` 中指定使用外部负载均衡器，将 service 暴露到外部负载均衡器的 IP 。
为此，在启动 service 之前，请取消 [all-in-one/frontend.yaml](https://git.k8s.io/examples/guestbook/all-in-one/frontend.yaml) 文件中的 `type: LoadBalancer` 行的注释。

有关 guestbook 站点外部访问的更多详细信息[请参阅下面的附录](#附录：外部访问 guestbook 站点) 。

这样创建 service 和 Deployment:

```console
$ kubectl create -f guestbook/all-in-one/frontend.yaml
service "frontend" created
deployment "frontend" created
```



然后，再次列出所有 services ：

```console
$ kubectl get services
NAME           CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
frontend       10.0.63.63     <none>        80/TCP     1m
redis-master   10.0.76.248    <none>        6379/TCP   39m
redis-slave    10.0.112.188   <none>        6379/TCP   19m
```

同时列出所有 Deployments :

```console
$ kubectl get deployments 
NAME           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
frontend       3         3         3            3           2m
redis-master   1         1         1            1           39m
redis-slave    2         2         2            2           20m
```



一旦启动，即当期望的 replicas 数量与当前的 replicas 不一致时（再次创建pod可能需要三十秒），您可以在群集中列出具有指定 labels 的 pod ，以验证 master 、 slaves 和前端都在运行。 
您可以看到 label 为 'tier' 的所有pod的列表，如下所示：

```console
$ kubectl get pods -L tier
NAME                            READY     STATUS    RESTARTS   AGE       TIER
frontend-1211764471-4e1j2       1/1       Running   0          4m        frontend
frontend-1211764471-gkbkv       1/1       Running   0          4m        frontend
frontend-1211764471-rk1cf       1/1       Running   0          4m        frontend
redis-master-2353460263-1ecey   1/1       Running   0          42m       backend
redis-slave-1691881626-dlf5f    1/1       Running   0          22m       backend
redis-slave-1691881626-sfn8t    1/1       Running   0          22m       backend
```

你应该看到一个 redis master pod ，两个 redis slaves pods 和三个前端 pods 。
前端正在运行的 PHP 服务器的代码位于 `examples/guestbook/php-redis/guestbook.php` 中。 类似这样：

```php
<?

set_include_path('.:/usr/local/lib/php');

error_reporting(E_ALL);
ini_set('display_errors', 1);

require 'Predis/Autoloader.php';

Predis\Autoloader::register();

if (isset($_GET['cmd']) === true) {
  $host = 'redis-master';
  if (getenv('GET_HOSTS_FROM') == 'env') {
    $host = getenv('REDIS_MASTER_SERVICE_HOST');
  }
  header('Content-Type: application/json');
  if ($_GET['cmd'] == 'set') {
    $client = new Predis\Client([
      'scheme' => 'tcp',
      'host'   => $host,
      'port'   => 6379,
    ]);

    $client->set($_GET['key'], $_GET['value']);
    print('{"message": "Updated"}');
  } else {
    $host = 'redis-slave';
    if (getenv('GET_HOSTS_FROM') == 'env') {
      $host = getenv('REDIS_SLAVE_SERVICE_HOST');
    }
    $client = new Predis\Client([
      'scheme' => 'tcp',
      'host'   => $host,
      'port'   => 6379,
    ]);

    $value = $client->get($_GET['key']);
    print('{"data": "' . $value . '"}');
  }
} else {
  phpinfo();
} ?>
```



请注意，如果使用 `redis-master` 和 `redis-slave' 的主机名 - 我们可以通过Kubernetes集群的 DNS 服务找到这些 Services 。 所有前端的 replicas 将写入负载均衡 redis-slave service，这也是高可用的。



### 第四步 清理

如果你在一个运行中的 Kubernetes 集群，你可以通过删除 Deployments 和 Services 来杀死pod。 使用 labels 来选择要删除的资源也是一种简单的方法。

```console
$ kubectl delete deployments,services -l "app in (redis, guestbook)"
```

要完全停止 Kubernetes 群集，如果您是用源代码运行的，可以使用：

```console
$ <kubernetes>/cluster/kube-down.sh
```



### 故障排除

如果您无法启动您的 guestbook 应用程序，请仔细检查您的外部IP是否正确定义了您的前端 Service ，并且群集节点的防火墙是否打开了80端口。

然后，请参阅[故障排除文档](http://kubernetes.io/docs/troubleshooting/)以进一步获取常见问题的列表以及如何诊断它们。



### 附录 外部访问 guestbook 站点

您将需要设置您的 guestbook Service ，以便可以从 Kubernetes 外部网络访问。 上面我们介绍了一种方法，通过设置 Service `spec` 中的 `type: LoadBalancer` 。
更普遍的，Kubernetes 支持将 Service 暴露在外部 IP 地址上的两种方式：`NodePort` 和 `LoadBalancer` ，详情请浏览[这里](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services---service-types)。
如果使用了 `LoadBalancer`，外部IP可能很快就会显示在 `kubectl get services` 命令的回显信息中，您之后可以看到类似于这样的列表：

```console
$ kubectl get services
NAME           CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
frontend       10.0.63.63     23.236.59.54  80/TCP     1m
redis-master   10.0.76.248    <none>        6379/TCP   39m
redis-slave    10.0.112.188   <none>        6379/TCP   19m
```



一旦您将 service 暴露给外部 IP ，请使用 IP 来访问您的 guestbook ，即`http://<EXTERNAL-IP>:<PORT>` 。

您应该看到一个看起来像这样的网页（没有消息）。 尝试添加一些内容！

<img width="50%" src="http://amy-jo.storage.googleapis.com/images/gb_k8s_ex1.png">

如果您觉得您的技术比较牛逼，您还可以查看 `kubectl get pods,services` 的输出来手动获取服务 IP ，并修改防火墙的标准工具和服务 (firewalld, iptables, selinux) 。



#### Google Compute Engine 外部负载均衡器详细信息

在 Google Compute Engine 中，Kubernetes 会自动为 `LoadBalancer` 创建 services 的转发规则。

您可以列出这样的转发规则（转发规则也表示外部IP）：

```console
$ gcloud compute forwarding-rules list
NAME                  REGION      IP_ADDRESS     IP_PROTOCOL TARGET
frontend              us-central1 130.211.188.51 TCP         us-central1/targetPools/frontend
```

在Google Compute Engine中，您还可能需要使用 [console][cloud-console] 或 `gcloud` 工具打开防火墙的80端口。 以下命令将允许流量从任何源到 tagged 为 `kubernetes-node`的实例（适当替换为您的 tags ）：

```console
$ gcloud compute firewall-rules create --allow=tcp:80 --target-tags=kubernetes-node kubernetes-node-80
```



对于GCE Kubernetes的启动细节，请参阅[开始使用 Google Compute Engine](http://kubernetes.io/docs/getting-started-guides/gce/)

有关Google Compute Engine限制特定源的流量的详细信息，请参阅[Google Compute Engine防火墙文档][gce-firewall-docs].

[cloud-console]: https://console.developer.google.com
[gce-firewall-docs]: https://cloud.google.com/compute/docs/networking#firewalls


[![Analytics](https://kubernetes-site.appspot.com/UA-36037335-10/GitHub/examples/guestbook/README.md?pixel)]()
