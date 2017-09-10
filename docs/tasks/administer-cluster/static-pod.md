---
approvers:
- jsafrane
title: 静态 Pod
---


**如果需要在集群环境中使用静态 pod 的方式在每个节点上都运行一个 pod，您应该考虑使用 [DaemonSet](/docs/concepts/workloads/controllers/daemonset/)！**


*静态 pod* 在特定的节点上直接通过 kubelet 守护进程进行管理，API 服务无法管理。它没有跟任何的副本控制器进行关联，kubelet 守护进程对它进行监控，如果崩溃了，kubelet 守护进程会重启它。


Kubelet 通过 Kubernetes API 服务为每个静态 pod 创建 *镜像 pod*，这些镜像 pod 对于 API 服务是可见的，但是不受它控制。


## 创建静态 pod


静态 pod 能够通过两种方式创建：配置文件或者 HTTP。


### 配置文件


配置文件要求放在指定目录，是 json 或者 yaml 格式描述的标准的 pod 定义文件。使用 `kubelet --pod-manifest-path=<the directory>` 启动 kubelet 守护进程，它就会定期扫描目录下面 yaml/json 文件的出现/消失，从而执行 pod 的创建/删除。


这是如何启动一个简单的 Web 服务器作为静态 POD 的示例：


1. 选择一个想要运行静态 pod 的节点。在本例中，选择节点 `my-node1`。

    ```
    [joe@host ~] $ ssh my-node1
    ```


2. 选择一个命名为 `/etc/kubelet.d` 的目录，将 Web 服务的 pod 定义文件放在该目录下，例如：`/etc/kubelet.d/static-web.yaml`

    ```
    [root@my-node1 ~] $ mkdir /etc/kubernetes.d/
    [root@my-node1 ~] $ cat <<EOF >/etc/kubernetes.d/static-web.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: static-web
      labels:
        role: myrole
    spec:
      containers:
        - name: web
          image: nginx
          ports:
            - name: web
              containerPort: 80
              protocol: TCP
    EOF
    ```


3. 配置该节点上 kubelet 守护进程，将 `--pod-manifest-path=/etc/kubelet.d/` 添加到启动参数中。
    
    在 Fedora 系统上，编辑 `/etc/kubernetes/kubelet` 文件，使其包括下面行：

    ```
    KUBELET_ARGS="--cluster-dns=10.254.0.10 --cluster-domain=kube.local --pod-manifest-path=/etc/kubelet.d/"
    ```

	
	其它 Kubernetes 环境可能有不同的指令。


4. 重启 kubelet，在 Fedora 上，按照如下方式：

    ```
    [root@my-node1 ~] $ systemctl restart kubelet
    ```


## 通过 HTTP 创建静态 pod


Kubelet 定期的从参数 `--manifest-url=<URL>` 配置的地址下载文件，并将其解析为 json/yaml 格式的 pod 描述。它的工作原理与从 `--pod-manifest-path=<directory>` 中发现文件执行创建/更新静态 pod 是一样的，即，文件的每次更新都将应用到运行中的静态 pod 中(参见下面内容)。


## 静态 pod 的行为


当 kubelet 启动时，它将自动启动通过参数 `--pod-manifest-path=` 指定的目录以及 `--manifest-url=` 指定的位置下定义的 pod，例如，我们的 static-web (由于它需要一些时间拉取 nginx 镜像，请您耐心等待)

```shell
[joe@my-node1 ~] $ docker ps
CONTAINER ID IMAGE         COMMAND  CREATED        STATUS         PORTS     NAMES
f6d05272b57e nginx:latest  "nginx"  8 minutes ago  Up 8 minutes             k8s_web.6f802af4_static-web-fk-node1_default_67e24ed9466ba55986d120c867395f3c_378e5f3c
```


查看 Kubernetes API 服务(运行在主机 `my-master` 上)，我们也将看到一个新的 `mirror-pod` 被创建：

```shell
[joe@host ~] $ ssh my-master
[joe@my-master ~] $ kubectl get pods
NAME                       READY     STATUS    RESTARTS   AGE
static-web-my-node1        1/1       Running   0          2m

```


表明 pod 是静态 pod 的标签被设置到 `mirror-pod` 中，该标签可以像其它标签一样用于过滤。


请注意，我们不能通过 API 服务去删除这些 pod (例如，通过 [`kubectl`](/docs/user-guide/kubectl/) 命令)，kubelet 根本不会删除它。

```shell
[joe@my-master ~] $ kubectl delete pod static-web-my-node1
pods/static-web-my-node1
[joe@my-master ~] $ kubectl get pods
NAME                       READY     STATUS    RESTARTS   AGE
static-web-my-node1        1/1       Running   0          12s

```


回到 `my-node1` 主机上，尝试手动停止容器，我们将看到 kubelet 在一段时间之后会自动重启它：

```shell
[joe@host ~] $ ssh my-node1
[joe@my-node1 ~] $ docker stop f6d05272b57e
[joe@my-node1 ~] $ sleep 20
[joe@my-node1 ~] $ docker ps
CONTAINER ID        IMAGE         COMMAND                CREATED       ...
5b920cbaf8b1        nginx:latest  "nginx -g 'daemon of   2 seconds ago ...
```


## 动态增加和删除静态 pod


运行中的 kubelet 会定期扫描配置的目录(在我们的例子中是 `/etc/kubelet.d`)，根据目录中文件的出现/消失来确定 pod 新增/删除之类的变化。

```shell
[joe@my-node1 ~] $ mv /etc/kubelet.d/static-web.yaml /tmp
[joe@my-node1 ~] $ sleep 20
[joe@my-node1 ~] $ docker ps
// no nginx container is running
[joe@my-node1 ~] $ mv /tmp/static-web.yaml  /etc/kubelet.d/
[joe@my-node1 ~] $ sleep 20
[joe@my-node1 ~] $ docker ps
CONTAINER ID        IMAGE         COMMAND                CREATED           ...
e7a62e3427f1        nginx:latest  "nginx -g 'daemon of   27 seconds ago
```
