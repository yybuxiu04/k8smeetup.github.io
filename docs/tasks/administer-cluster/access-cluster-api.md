---
title:  通过 Kubernetes API 访问集群
---




{% capture overview %}
This page shows how to access clusters using the Kubernetes API.
这一页显示如何通过 Kubernetes API 来访问集群。
{% endcapture %}

{% capture prerequisites %}

{% include task-tutorial-prereqs.md %}
{% endcapture %}

{% capture steps %}



## 访问集群API
###  通过kubectl访问API
如果第一次访问 Kubernetes 的 API ， 可以使用 Kubernetes 的命令行工具  `kubectl`。
要访问一个集群， 您需要知道集群所在的位置，并有访问它的凭证。 典型的， 你可以按照[入门指南](/docs/getting-started-guides/)自动建立集群， 或者由其他人创建集群， 然后提供给你相应的凭证和集群的位置。

通过下面的命令获取集群的位置和凭证：

```shell
$ kubectl config view
```

关于如何使用 kubectl 的更多例子可以参考 [例子](https://github.com/kubernetes/kubernetes/tree/{{page.githubbranch}}/examples/) 。 完整的文档见 [kubectl 手册](/docs/user-guide/kubectl/index).




### 直接访问 REST API
Kubectl  会处理 apiserver 的定位和认证。 如果您要用一个 http 客户端， 比如 `curl` 或者 `wget`  或者 浏览器，  直接访问 REST API ， 有多种方法可以定位 apiserver 和 通过 apiserver 的认证：

1. 运行 kubectl 的代理模式（推荐）。 比较推荐使用这种方法， 因为这种方式使用了已存储的的 apiserver 的位置信息， 并使用自行签发的证书来验证 apiserver 的身份。 使用这种方式不存在中间人攻击。
2. 另外一种可选的方式，  您可以把 apiserver 的位置信息和凭证信息直接提供给http 客户端。 这种方式可以用于那些被代理拒绝的客户端代码。 为了确保不受到中间人攻击， 你需要往您的浏览器里导入根证书。

使用Go 或者 Python 的客户端库， 可以访问代理模式下 kubectl 。


#### 使用 kubectl 代理

通过下面的命令运行 kubectl 在某个模式下， 在该模式下， 它充当了一个反向代理， 起到了处理 apiserver 的定位和认证的作用。

运行命令如下：

```shell
$ kubectl proxy --port=8080 &
```

查看 [kubectl proxy](/docs/user-guide/kubectl/v1.6/#proxy) 获取更多细节 。

紧接着你可以通过 curl， wget， 或者浏览器访问 API， 访问方式如下：

```shell
$ curl http://localhost:8080/api/
{
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "10.0.1.149:443"
    }
  ]
}
```




#### 不使用 kubectl 代理

如果要避免使用 kubectl 代理的话，可以通过直接传递一个认证 token 给 apiserver， 比如:

``` shell
$ APISERVER=$(kubectl config view | grep server | cut -f 2- -d ":" | tr -d " ")
$ TOKEN=$(kubectl describe secret $(kubectl get secrets | grep default | cut -f1 -d ' ') | grep -E '^token' | cut -f2 -d':' | tr -d '\t')
$ curl $APISERVER/api --header "Authorization: Bearer $TOKEN" --insecure
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "10.0.1.149:443"
    }
  ]
}
```

以上的例子中使用了参数 `--insecure` 。 这样的话会使集群很容易受到中间人攻击.  当 kubectl 访问集群时， 它会使用一个已经保存的根证书和客户端证书。 ( 这些证书会被安装在 `~/.kube` 目录下).  由于集群的证书一般是自行签发的，可能需要特殊的配置使你的 http 客户端可以使用到根证书。

在一些集群，  apiserver  不需要验证； 它是可能是在本机上提供服务的， 或者被防火墙隔离起来。 没有一个针对这方面的标准。[配置 API 的访问](/docs/admin/accessing-the-api) 这篇文档介绍了集群管理员如何进行这方面的配置。
这些方法可能会和将来的高可用支持互相冲突。



### 通过编程方式访问 API
Kubernetes 官方支持的客户端库有 [Go](#go-client) 和  [Python](#python-client)  客户端库。

#### Go 客户端

* 要获得客户端库， 可以运行如下命令 : `go get k8s.io/client-go/<version number>/kubernetes` 参考 [https://github.com/kubernetes/client-go](https://github.com/kubernetes/client-go) 来了解哪些版本是被支持的。
* 写一个基于 client-go 客户端库的应用。 注意到 client-go 定义了它自己的 API 对象， 因此在需要时需要导入 client-go 的 API 定义，而不是从仓库。 比如, `import "k8s.io/client-go/1.4/pkg/api/v1"` 是正确的做法。

Go 的客户端可以使用和 kubectl 命令行工具相同的 [kubeconfig 配置文件](/docs/concepts/cluster-administration/authenticate-across-clusters-kubeconfig/) 来定位和认证 apiserver。 参考这个 [例子](https://git.k8s.io/client-go/examples/out-of-cluster-client-configuration/main.go)：

```golang
import (
   "fmt"
   "k8s.io/client-go/1.4/kubernetes"
   "k8s.io/client-go/1.4/pkg/api/v1"
   "k8s.io/client-go/1.4/tools/clientcmd"
)
...
   // uses the current context in kubeconfig
   config, _ := clientcmd.BuildConfigFromFlags("", "path to kubeconfig")
   // creates the clientset
   clientset, _:= kubernetes.NewForConfig(config)
   // access the API to list pods
   pods, _:= clientset.Core().Pods("").List(v1.ListOptions{})
   fmt.Printf("There are %d pods in the cluster\n", len(pods.Items))
...
```

如果应用以 Pod 的方式部署在集群中， 可以参考[下一小节](#accessing-the-api-from-a-pod).



#### Python 客户端

要使用  [Python 客户端](https://github.com/kubernetes-incubator/client-python), 运行如下命令: `pip install kubernetes` 参考 [Python 客户端库页面](https://github.com/kubernetes-incubator/client-python) 获取更多的安装说明。

The Python 客户端使用和 kubectl 命令行相同的 [kubeconfig 配置文件](/docs/user-guide/kubeconfig-file)
来定位和认证 apiserver。 参考这个 [例子](https://github.com/kubernetes-incubator/client-python/tree/master/examples/example1.py):

```python
from kubernetes import client, config

config.load_kube_config()

v1=client.CoreV1Api()
print("Listing pods with their IPs:")
ret = v1.list_pod_for_all_namespaces(watch=False)
for i in ret.items:
    print("%s\t%s\t%s" % (i.status.pod_ip, i.metadata.namespace, i.metadata.name))
```

#### 其它语言

 使用其它语言访问 API 可以参考这个文档[客户端库](/docs/reference/client-libraries/)  。 您可以了解到其它语言的库如何认证 API 的。




### 从 Pod 里 访问 API

当从 pod 里去访问集群的API时，对 apiserver 的定位和认证就有点不太一样了。

在 pod 里定位 apiserver 推荐的方式， 是使用 `kubernetes` 的 DNS 名称，这个名称可以解析到一个可以路由到 apiserver 的服务IP.

对 apiserver 的认证推荐的方式是使用一个[服务账号](/docs/user-guide/service-accounts) 凭证.  在 kube-system 命名空间下，  一个 pod关联的服务账号和与服务账号相关的凭证(token) 位于那个 pod 的每个容器的文件系统树之上 ， 具体路径是 `/var/run/secrets/kubernetes.io/serviceaccount/token`.

如果有可用的证书， 那么这个证书会在每个容器的文件系统树下的这个路径 `/var/run/secrets/kubernetes.io/serviceaccount/ca.crt`,  这个证书可以用来验证服务端的 apiserver 证书。

最后的， 可以用于命名空间 API 操作的默认命名空间, 位于一个文件里， 这个文件在每个容器的`/var/run/secrets/kubernetes.io/serviceaccount/namespace` 文件路径下。

在一个 pod 之内连接 API 的推荐方式有：

  - 在 pod 里的某个容器运行 kubectl 代理， 或者以后台进程运行在容器内。 这个代理提供了 Kubernetes API 到 pod 的本地主机端口的代理。 因此 pod 的任何一个容器的其它进程都可以访问这个代理。 参考这个例子 [在 pod 里使用 kubectl proxy](https://github.com/kubernetes/kubernetes/tree/{{page.githubbranch}}/examples/kubectl-container/).
  - 使用 Go 客户端库， 使用`rest.InClusterConfig()` 和  `kubernetes.NewForConfig()` 函数创建一个客户端。 它们负责处理 apiserver 的定位和认证。参考 [例子](https://git.k8s.io/client-go/examples/in-cluster-client-configuration/main.go)

在每种情形下， pod 里的凭证都被用于与 apiserver 的安全通信。

{% endcapture %}

{% include templates/task.md %}
