---
approvers:
- erictune
- lavalamp
- thockin
title: 从零开始搭建自定义集群
---





本指南适用于想要搭建一个定制化 Kubernetes 集群的人员。如果您在 [列表](/docs/getting-started-guides/) 中找到现有的入门指南可以满足您的需求，那么建议使用它们，因为可从他人的经验中获益。但是，如果您使用特定的 IaaS，网络，配置管理或操作系统，同时又不符合这些指南的要求，那么本指南会为您提供所需的步骤大纲。请注意，比起其他预定义的指南，研习本指南需做出相当多的努力。



本指南对那些想要从更高层次了解现有集群安装脚本执行步骤的人员也很有用。

* TOC
{:toc}



## 设计和准备

### 学习

  1. 您应该已经熟悉使用 Kubernetes 集群。建议按照如下入门指南启动一个临时的集群。首先帮您熟悉 CLI（[kubectl](/docs/user-guide/kubectl/)）和概念（[pods](/docs/user-guide/pods)，[services](/docs/user-guide/services)等）。
  1. 您的工作站应该已经存在 'kubectl'。这是完成其他入门指南后的一个附加安装。如果没有，请遵循 [说明](/docs/tasks/kubectl/install/)。



### Cloud Provider

Kubernetes 的 Cloud Provider 是一个模块，它提供一个管理 TCP 负载均衡，节点（实例）和网络路由的接口。此接口定义在 `pkg/cloudprovider/cloud.go`。未实现 Cloud Provider 也可以建立自定义集群（例如使用裸机），并不是所有的接口功能都必须实现，这取决于如何在各组件上设置标识。



### 节点

- 您可以使用虚拟机或物理机。
- 虽然可以使用一台机器构建集群，但为了运行所有的例子和测试，至少需要4个节点。
- 许多入门指南对主节点和常规节点进行区分。  这不是绝对必要的。
- 节点需要使用 x86_64 架构运行某些版本的 Linux。在其他操作系统和架构上运行是可行的，但本指南不会协助指导。
- Apiserver 和 etcd 可以运行在1个核心和 1GB RAM 的机器上，这适用于拥有数十个节点的集群。
  更大或更活跃的集群可能受益于更多的核心。
- 其他节点可以配备任何合理的内存和任意数量的内核。它们不需要相同的配置。



### 网络

#### 网络连接
Kubernetes 有一个独特的 [网络模型](/docs/admin/networking)。

Kubernetes 为每个 pod 分配一个 IP 地址。创建集群时，需要为 Kubernetes 分配一段 IP 以用作 pod 的 IP。最简单的方法是为集群中的每个节点分配不同的 IP 段。 pod 中的进程可以访问其他 pod 的 IP 并与之通信。这种连接可以通过两种方式实现：



- **使用 overlay 网络**
  - overlay 网络通过流量封装（例如 vxlan）来屏蔽 pod 网络的底层网络架构。
  - 封装会降低性能，但具体多少取决于您的解决方案。
- **不使用 overlay 网络**  
  - 配置底层网络结构（交换机，路由器等）以熟知 Pod IP 地址。
  - 不需要 overlay 的封装，因此可以实现更好的性能。



选择哪种方式取决于您的环境和需求。有多种方法来实现上述的某种选项：



- **使用 Kubernetes 调用的网络插件**
  - Kubernetes 支持 [CNI](https://github.com/containernetworking/cni) 网络插件接口。
  - 有许多解决方案为 Kubernetes 提供插件（按字母顺序排列）：
    - [Calico](http://docs.projectcalico.org/)
    - [Flannel](https://github.com/coreos/flannel)
    - [Open vSwitch (OVS)](http://openvswitch.org/)
    - [Romana](http://romana.io/)
    - [Weave](http://weave.works/)
    - [更多](/docs/admin/networking#how-to-achieve-this)
  - 您也可以编写自己的插件。
- **将网络插件直接编译进 Kubernetes**
  - 可以通过 cloud provider 模块的 "Routes" 接口来实现。
  - Google Compute Engine（[GCE](/docs/get-started-guide/gce)）和 [AWS](/docs/get-started-guide/aws) 指南使用此方法。
- **为 Kubernetes 配置外部网络**
  - 这可以通过手工执行命令或通过一组外部维护的脚本来完成。
  - 您不得不自己实现，此功能可以带来额外的灵活性。



需要为 Pod IP 选择一个地址范围。请注意，Pod IP 尚不支持 IPv6。



- 多种方法:
  - GCE：每个项目都有自己的 `10.0.0.0/8`。从该空间为每个 Kubernetes 集群分配子网 `/16`，多个集群都拥有自己的空间。
  每个节点从该网段获取进一步的子网细分。
  - AWS：整个组织使用一个 VPC ，为每个集群划分一个块，或者为不同的集群使用不同的 VPC。
- 为每个节点的 PodIP 分配同一个 CIDR 子网，或者分配单个大型 CIDR，该大型 CIDR 由每个节点上较小的 CIDR 所组成的。
  - 您一共需要 max-pods-per-node * max-number-of-nodes 个 IP。每个节点配置子网 `/24`，即每台机器支持 254 个 pods，这是常见的配置。如果 IP 不充足，配置 `/26`（每个机器62个 pod）甚至是 `/27`（30个 pod）也是足够的。
  - 例如，使用 `10.10.0.0/16` 作为集群范围，支持最多256个节点各自使用 10.10.0.0/24 到 10.10.255.0/24 的 IP 范围。
  - 需要使它们路由可达或通过 overlay 连通。



Kubernetes 也为每个 [service](/docs/user-guide/services) 分配一个 IP。但是，Service IP 无须路由。在流量离开节点前，kube-proxy 负责将 Service IP 转换为 Pod IP。您需要利用 `SERVICE_CLUSTER_IP_RANGE` 为 service 分配一段 IP。例如，设置 `SERVICE_CLUSTER_IP_RANGE="10.0.0.0/16"` 以允许激活 65534 个不同的服务。请注意，您可以增大此范围，但在不中断使用它的 service 和 pod 时，您不能移动该范围（指增大下限或减小上限）。


此外，您需要为主节点选择一个静态 IP。

- 称为 `MASTER_IP`。
- 打开防火墙以允许访问 apiserver 的端口 80 和/或 443。
- 启用 ipv4 转发，`net.ipv4.ip_forward = 1`


#### 网络策略

Kubernetes 可以在 Pods 之间使用 [网络策略](/docs/concepts/services-networking/network-policies/) 定义细粒度的网络策略。

并非所有网络提供商都支持 Kubernetes NetworkPolicy API，参阅 [使用网络策略](/docs/tasks/configure-pod-container/declare-network-policy/) 获取更多内容。



### 集群命名

您应该为集群选择一个名称。为每个集群选择一个简短的名称并在以后的集群使用中将其作为唯一命名。以下几种方式中都会用到集群名称：

  - 通过 kubectl 来区分您想要访问的各种集群。有时候您可能想要第二个集群，比如测试新的 Kubernetes 版本，运行在不同地区的 Kubernetes 等。
  - Kubernetes 集群可以创建 cloud provider 资源（例如AWS ELB），并且不同的集群需要区分每个创建的资源。称之为 `CLUSTER_NAME`。



### 软件的二进制文件

您需要以下二进制文件：

- etcd
- 以下 Container 运行工具之一:
  - docker
  - rkt
- Kubernetes
  - kubelet
  - kube-proxy
  - kube-apiserver
  - kube-controller-manager
  - kube-scheduler



#### 下载并解压 Kubernetes 二进制文件

Kubernetes 发行版包括所有的 Kubernetes 二进制文件以及受支持的 etcd 发行版。
您可以使用 Kubernetes 的发行版（推荐）或按照 [开发人员文档](https://git.k8s.io/community/contributors/devel/) 中的说明构建您的 Kubernetes 二进制文件。本指南仅涉及使用 Kubernetes 发行版。



下载并解压 [最新的发行版](https://github.com/kubernetes/kubernetes/releases/latest)。服务器二进制 tar 包不再包含在 Kubernetes 的最终 tar 包中，因此您需要找到并运行 `./kubernetes/cluster/get-kube-binaries.sh` 来下载客户端和服务器的二进制文件。
然后找到 `./kubernetes/server/kubernetes-server-linux-amd64.tar.gz` 并解压缩。接着在被解压开的目录 `./kubernetes/server/bin` 中找到所有必要的二进制文件。



#### 选择镜像

您将在容器之外运行 docker，kubelet 和 kube-proxy，与运行系统守护进程的方式相同，这些程序需要单独的二进制文件。对于 etcd，kube-apiserver，kube-controller-manager 和 kube-scheduler，我们建议您将其作为容器运行，因此需要构建相应的镜像。



获取 Kubernetes 镜像的几种方式：

- 使用谷歌容器仓库（GCR）上托管的镜像：
  - 例如 `gcr.io/google_containers/hyperkube:$TAG`，其中 `TAG` 是最新的版本标签，可在 [最新版本页面](https://github.com/kubernetes/kubernetes/releases/latest) 上找到。
  - 确保 $TAG 与您使用的 kubelet 和 kube-proxy 的发行版标签相同。
  - [hyperkube]（https://releases.k8s.io/ {{page.githubbranch}}/cmd/hyperkube）是一个包含全部组件的二进制文件。
    - `hyperkube kubelet ...` 表示运行 kubelet，`hyperkube apiserver ...` 表示运行一个 apiserver，以此类推。  
- 构建私有镜像
  - 在使用私有镜像库时很有用
  - 包含诸如 `./kubernetes/server/bin/kube-apiserver.tar` 之类的文件，可以使用诸如 `docker load -i kube-apiserver.tar` 之类的命令将其转换为 docker 镜像。
  - 您可以使用命令 `docker images` 验证镜像是否加载正确的仓库和标签。



使用 etcd：

- 使用谷歌容器仓库（GCR）上托管的 `gcr.io/google_containers/etcd:2.2.1`
- 使用 [Docker Hub](https://hub.docker.com/search/?q=etcd) 或 [Quay.io](https://quay.io/repository/coreos/etcd) 上托管的镜像，比如 `quay.io/coreos/etcd:v2.2.1`。
- 使用操作系统安装源中的 etcd 发行版。
- 构建自己的镜像
  - 执行：`cd kubernetes/cluster/images/etcd; make`



我们建议您使用 Kubernetes 发行版中提供的 etcd。Kubernetes 程序已经使用此版本的 etcd 进行了广泛的测试，而不是与任何其他版本的 etcd。推荐的版本号也可以在 `kubernetes/cluster/images/etcd/Makefile` 中作为 `TAG` 的值被找到。

该文档的剩余部分假定镜像标签已被选定并存储在相应的环境变量中。例子（替换最新的标签和适当的仓库源）：

  - `HYPERKUBE_IMAGE=gcr.io/google_containers/hyperkube:$TAG`
  - `ETCD_IMAGE=gcr.io/google_containers/etcd:$ETCD_VERSION`



### 安全模型

两种主要的安全方式：

- 用 HTTP 访问 apiserver
  - 使用防火墙进行安全防护
  - 安装简单
- 用 HTTPS 访问 apiserver
  - 使用带证书的 https 和用户凭证。
  - 这是推荐的方法
  - 配置证书可能很棘手。

如果遵循 HTTPS 方法，则需要准备证书和凭证。



#### 准备证书

您需要准备几种证书：

- 作为 HTTPS 服务端的主节点需要一个证书。
- 作为主节点的客户端，kubelet 可以选择使用证书来认证自己，并通过 HTTPS 提供自己的 API 服务。

除非您打算用一个真正的 CA 生成证书，否则您需要生成一个根证书，并使用它来签署主节点，kubelet 和 kubectl 证书。在 [认证文档](/docs/admin/authentication/#creating-certificates) 中描述了如何做到这一点。



最终您会得到以下文件（稍后将用到这些变量）

- `CA_CERT`
  - 放在运行 apiserver 的节点上，例如位于 `/srv/kubernetes/ca.crt`。
- `MASTER_CERT`
  - 被 CA_CERT 签名
  - 放在运行 apiserver 的节点上，例如位于 `/srv/kubernetes/server.crt`。
- `MASTER_KEY `
  - 放在运行 apiserver 的节点上，例如位于 `/srv/kubernetes/server.key`。
- `KUBELET_CERT`
  - 可选
- `KUBELET_KEY`
  - 可选



#### 准备凭证
管理员用户（和任何用户）需要：

  - 用于识别他们的令牌或密码。
  - 令牌只是长字母数字的字符串，例如32个字符，生成方式
  - `TOKEN=$(dd if=/dev/urandom bs=128 count=1 2>/dev/null | base64 | tr -d "=+/" | dd bs=32 count=1 2>/dev/null)`

您的令牌和密码需要存储在文件中才能被 apiserver 读取。本指南使用 `/var/lib/kube-apiserver/known_tokens.csv`。此文件的格式在 [身份验证文档](/docs/admin/authentication) 中有描述。



为了向客户端分发凭证，Kubernetes 约定将凭证放入 [kubeconfig 文件](/docs/concepts/cluster-administration/authenticate-across-clusters-kubeconfig/) 中。



可以创建管理员的 kubeconfig 文件，如下所示：

- 如果您已经使用非定制集群的 Kubernetes（例如，遵循入门指南），那么您已拥有 `$HOME/.kube/config` 文件。
- 您需要添加证书，密钥和主节点 IP 到 kubeconfig 文件：
    - 如果使用仅防火墙安全选项，则以此方式设置 apiserver：
      - `kubectl config set-cluster $CLUSTER_NAME --server=http://$MASTER_IP --insecure-skip-tls-verify=true`
    - 否则，请设置 apiserver ip，客户端证书和用户凭证。
      - `kubectl config set-cluster $CLUSTER_NAME --certificate-authority=$CA_CERT --embed-certs=true --server=https://$MASTER_IP`
      - `kubectl config set-credentials $USER --client-certificate=$CLI_CERT --client-key=$CLI_KEY --embed-certs=true --token=$TOKEN`
    - 将集群设置为要使用的默认集群
      - `kubectl config set-context $CONTEXT_NAME --cluster=$CLUSTER_NAME --user=$USER`
      - `kubectl config use-context $CONTEXT_NAME`



接下来，为 kubelet 和 kube-proxy 创建一个 kubeconfig 文件。创建多少不同文件有几种选择：  

  1. 使用与管理员相同的凭证
    - 这是最简单的设置。
  1. 所有 kubelet 用一套令牌和 kubeconfig 文件，kube-proxy 用一套，管理端用一套。
    - 这是如今在 GCE 上的方式
  1. 每个 kubelet 使用不同的凭证
    - 我们正在改进，但所有的细节还没有准备好。



您可以通过拷贝 `$HOME/.kube/config`、参考 `cluster/gce/configure-vm.sh` 中的代码或者使用下面的模板来创建这些文件：

```yaml
apiVersion: v1
kind: Config
users:
- name: kubelet
  user:
    token: ${KUBELET_TOKEN}
clusters:
- name: local
  cluster:
    certificate-authority: /srv/kubernetes/ca.crt
contexts:
- context:
    cluster: local
    user: kubelet
  name: service-account-context
current-context: service-account-context
```



将 kubeconfig 文件放在每个节点上。以后本指南的例子假设在 `/var/lib/kube-proxy/kubeconfig` 和 `/var/lib/kube-proxy/kubeconfig` 中有 kubeconfig。



## 在节点上配置和安装基础软件

本节讨论如何将机器配置为 Kubernetes 节点。

您应该在每个节点上运行三个守护进程：

  - docker 或者 rkt
  - kubelet
  - kube-proxy

您还需要在安装操作系统后进行各种其他配置。

提示：比较可行的方法是先使用现有的入门指南来设置集群。在集群运行后，您可以从该集群复制 init.d 脚本或 systemd 单元文件，然后修改它们以便在您的自定义集群上使用。



### Docker

所需 Docker 的最低版本随 kubelet 版本的更改而变化。推荐使用最新的稳定版。如果版本太旧，Kubelet 将抛出警告并拒绝启动 pod，请尝试更换合适的 Docker 版本。

如果您已经安装过 Docker，但是该节点并没有配置过 Kubernetes，那么节点上可能存在 Docker 创建的网桥和 iptables 规则。您可能需要像下面这样删除这些内容，然后再为 Kubernetes 配置 Docker。



```shell
iptables -t nat -F
ip link set docker0 down
ip link delete docker0
```



配置 docker 的方式取决于您是否为网络选择了可路由的虚拟 IP 或 overlay 网络方式。Docker 的建议选项：



  - 为每个节点的 CIDR 范围创建自己的网桥，将其称为 cbr0，并在 docker 上设置 `--bridge=cbr0` 选项。
  - 设置 `--iptables=false`，docker 不会操纵有关主机端口的 iptables（Docker 的旧版本太粗糙，可能会在较新的版本中修复），以便 kube-proxy 管理 iptables 而不是通过 docker。
  - `--ip-masq=false`
    - 在您把 PodIP 设置为可路由时，需要设置此选项，否则，docker 会将 PodIP 源地址重写为 NodeIP。
    - 某些环境（例如 GCE）需要您对离开云环境的流量进行伪装。这类环境比较特殊。
    - 如果您正在使用 overlay 网络，请参阅这些说明。
  - `--mtu=`
    - 使用 Flannel 时可能需要该选项，因为 udp 封装会增大数据包
  - `--insecure-registry $CLUSTER_SUBNET`
    - 使用非 SSL 方式连接到您设立的私有仓库。



如果想增加 docker 的文件打开数量，设置：

   - `DOCKER_NOFILE=1000000`

此配置方式取决于您节点上的操作系统。例如在 GCE 上，基于 Debian 的发行版使用 `/etc/default/docker`。

通过 Docker 文档中给出的示例，确保在继续安装其余部分之前，您系统上的 docker 工作正常。



### rkt

[rkt](https://github.com/coreos/rkt) 是 Docker 外的另一种选择，您只需要安装 Docker 或 rkt 之一。其最低版本要求 [v0.5.6](https://github.com/coreos/rkt/releases/tag/v0.5.6)。

您的节点需要 [systemd](http://www.freedesktop.org/wiki/Software/systemd/) 来运行 rkt。匹配 rkt v0.5.6 的最小版本是 [systemd 215](http://lists.freedesktop.org/archives/systemd-devel/2014-July/020903.html)。



对 rkt 的网络支持还需要 [rkt 元数据服务](https://github.com/coreos/rkt/blob/master/Documentation/networking/overview.md)。您可以使用命令启动 rkt 元数据服务 `sudo systemd-run rkt metadata-service`

然后，您需要将该参数配置到 kubelet：

  - `--container-runtime=rkt`



### kubelet

所有节点都应该运行 kubelet。参阅 [软件的二进制文件](#software-binaries)。

要考虑的参数：

  - 如果遵循 HTTPS 安全方法：
    - `--api-servers=https://$MASTER_IP`
    - `--kubeconfig=/var/lib/kubelet/kubeconfig`
  - 否则，如果采取基于防火墙的安全方法
    - `--api-servers=http://$MASTER_IP`
  - `--config=/etc/kubernetes/manifests`
  - `--cluster-dns=` 指定要设置的 DNS 服务器地址，（请参阅[启动群集服务](#starting-cluster-services)。）
  - `--cluster-domain=` 指定用于集群 DNS 地址的 dns 域前缀。
  - `--docker-root=`
  - `--root-dir=`
  - `--configure-cbr0=` （如下面所描述的）
  - `--register-node` （在 [节点](/docs/admin/node) 文档中描述 ）



### kube-proxy

所有节点都应该运行 kube-proxy。（并不严格要求在“主”节点上运行 kube-proxy，但保持一致更简单。） 下载使用 kube-proxy 的方法和 kubelet 一样。

要考虑的参数：

  - 如果遵循 HTTPS 安全方法：
    - `--master=https://$MASTER_IP`
    - `--kubeconfig=/var/lib/kube-proxy/kubeconfig`
  - 否则，如果采取基于防火墙的安全方法
    - `--master=http://$MASTER_IP`



### 网络

每个节点需要分配自己的 CIDR 范围，用于 pod 网络。称为 `NODE_X_POD_CIDR`。

需要在每个节点上创建一个名为 `cbr0` 的网桥。在 [网络文档](/docs/admin/networking) 中进一步说明了该网桥。网桥本身需要从 `$NODE_X_POD_CIDR` 获取一个地址，按惯例是第一个 IP。称为 `NODE_X_BRIDGE_ADDR`。例如，如果 `NODE_X_POD_CIDR` 是 `10.0.0.0/16`，则`NODE_X_BRIDGE_ADDR` 是 `10.0.0.1/16`。 注意：由于以后使用这种方式，因此会保留后缀 `/16`。



- 推荐自动化方式：

  1. 在 kubelet init 脚本中设置 `--configure-cbr0=true` 选项并重新启动 kubelet 服务。Kubelet 将自动配置 cbr0。
     直到节点控制器设置了 Node.Spec.PodCIDR，网桥才会配置完成。由于您尚未设置 apiserver 和节点控制器，网桥不会立即被设置。
- 或者使用手动方案：



  1. 在 kubelet 上设置 `--configure-cbr0=false` 并重新启动。
  1. 创建一个网桥。

        ```
        ip link add name cbr0 type bridge
        ```

  1. 设置适当的 MTU。 注意：MTU 的实际值取决于您的网络环境。

        ```
        ip link set dev cbr0 mtu 1460
        ```

  1. 将节点的网络添加到网桥（docker 将在桥的另一侧）。

        ```
        ip addr add $NODE_X_BRIDGE_ADDR dev cbr0
        ```        

  1. 启动网桥

        ```
        ip link set dev cbr0 up
        ```



如果您已经关闭 Docker 的 IP 伪装，以允许pod相互通信，那么您可能需要为集群网络之外的目标 IP 进行伪装。例如：

```shell
iptables -t nat -A POSTROUTING ! -d ${CLUSTER_SUBNET} -m addrtype ! --dst-type LOCAL -j MASQUERADE
```



对于集群外部的流量，这将重写从 PodIP 到节点 IP 的源地址，并且内核 [连接跟踪](http://www.iptables.info/en/connection-state.html) 将确保目的地为节点地址的流量仍可抵达 pod。

注意：以上描述适用于特定的环境。其他环境根本不需要伪装。如 GCE，不允许 Pod IP 向外网发送流量，但在您的 GCE 项目内部之间没有问题。



### 其他

- 如果需要，为您的操作系统软件包管理器启用自动升级。
- 为所有节点组件配置日志轮转（例如使用 [logrotate](http://linux.die.net/man/8/logrotate)）。
- 设置活动监视（例如使用[supervisord](http://supervisord.org/)）。
- 设置卷支持插件（可选）
  - 安装可选卷类型的客户端，例如 GlusterFS 卷的 `glusterfs-client`。



### 使用配置管理

之前的步骤都涉及用于设置服务器的“常规”系统管理技术。您可能希望使用配置管理系统来自动执行节点配置过程。在各种入门指南中有 Ansible，Juju 和 CoreOS Cloud Config 的示例 [Saltstack](/docs/admin/salt)。



## 引导集群启动

虽然基础节点服务（kubelet, kube-proxy, docker）还是由传统的系统管理/自动化方法启动和管理，但是 Kubernetes 中其余的 *master* 组件都由 *Kubernetes* 配置和管理：

  - 它们的选项在 Pod 定义（yaml 或 json）中指定，而不是 /etc/init.d 文件或 systemd 单元中。
  - 它们由 Kubernetes 而不是 init 运行。



### etcd

您需要运行一个或多个 etcd 实例。
  - 高可用且易于恢复 - 运行3或5个 etcd 实例，将其日志写入由持久性存储（RAID，GCE PD）支持的目录，
  - 非高可用，但易于恢复 - 运行一个 etcd 实例，其日志写入由持久性存储（RAID，GCE PD）支持的目录，
  **注意：** 如果实例发生中断可能导致操作中断。
  - 高可用 - 运行3或5个非持久性存储 etcd 实例。
  **注意：** 由于存储被复制，日志可以写入非持久性存储。



有关影响集群可用性因素的更多讨论，请参阅 [cluster-troubleshooting](/docs/admin/cluster-troubleshooting)。

运行一个 etcd 实例：

1. 复制 `cluster/saltbase/salt/etcd/etcd.manifest`
1. 按需要进行修改
1. 通过将其放入 kubelet 清单目录来启动 pod。



### Apiserver, Controller Manager, and Scheduler

Apiserver，Controller Manager和 Scheduler 将分别以 pod 形式在主节点上运行。。



对于这些组件，启动它们的步骤类似：

1. 从所提供的 pod 模板开始。
1. 将 [选取镜像](#selecting-images) 中的值设置到 `HYPERKUBE_IMAGE`。
1. 使用每个模板下面的建议，确定集群需要哪些参数。
1. 将完成的模板放入 kubelet 清单目录中启动 pod。
1. 验证 pod 是否启动。



#### Apiserver pod 模板

```json
{
  "kind": "Pod",
  "apiVersion": "v1",
  "metadata": {
    "name": "kube-apiserver"
  },
  "spec": {
    "hostNetwork": true,
    "containers": [
      {
        "name": "kube-apiserver",
        "image": "${HYPERKUBE_IMAGE}",
        "command": [
          "/hyperkube",
          "apiserver",
          "$ARG1",
          "$ARG2",
          ...
          "$ARGN"
        ],
        "ports": [
          {
            "name": "https",
            "hostPort": 443,
            "containerPort": 443
          },
          {
            "name": "local",
            "hostPort": 8080,
            "containerPort": 8080
          }
        ],
        "volumeMounts": [
          {
            "name": "srvkube",
            "mountPath": "/srv/kubernetes",
            "readOnly": true
          },
          {
            "name": "etcssl",
            "mountPath": "/etc/ssl",
            "readOnly": true
          }
        ],
        "livenessProbe": {
          "httpGet": {
            "scheme": "HTTP",
            "host": "127.0.0.1",
            "port": 8080,
            "path": "/healthz"
          },
          "initialDelaySeconds": 15,
          "timeoutSeconds": 15
        }
      }
    ],
    "volumes": [
      {
        "name": "srvkube",
        "hostPath": {
          "path": "/srv/kubernetes"
        }
      },
      {
        "name": "etcssl",
        "hostPath": {
          "path": "/etc/ssl"
        }
      }
    ]
  }
}
```



以下是您可能需要设置的一些 apiserver 参数：

- `--cloud-provider=` 参阅 [cloud providers](#cloud-providers)
- `--cloud-config=` 参阅 [cloud providers](#cloud-providers)
- `--address=${MASTER_IP}` *或者* `--bind-address=127.0.0.1` 和 `--address=127.0.0.1` 如果要在主节点上运行代理。
- `--service-cluster-ip-range=$SERVICE_CLUSTER_IP_RANGE`
- `--etcd-servers=http://127.0.0.1:4001`
- `--tls-cert-file=/srv/kubernetes/server.cert`
- `--tls-private-key-file=/srv/kubernetes/server.key`
- `--admission-control=$RECOMMENDED_LIST`
  - 参阅 [admission controllers](/docs/admin/admission-controllers) 获取推荐设置。
- `--allow-privileged=true`，只有当您信任您的集群用户以 root 身份运行 pod 时。



如果您遵循仅防火墙的安全方法，请使用以下参数：

- `--token-auth-file=/dev/null`
- `--insecure-bind-address=$MASTER_IP`
- `--advertise-address=$MASTER_IP`

如果您使用 HTTPS 方法，请设置：

- `--client-ca-file=/srv/kubernetes/ca.crt`
- `--token-auth-file=/srv/kubernetes/known_tokens.csv`
- `--basic-auth-file=/srv/kubernetes/basic_auth.csv`



pod 使用 `hostPath` 卷挂载几个节点上的文件系统目录。这样的目的是：

- 挂载 `/etc/ssl` 以允许 apiserver 找到 SSL 根证书，以便它可以验证外部服务，例如一个 cloud provider。
  - 如果未使用 cloud provider，则不需要（例如裸机）。
- 挂载 `/srv/kubernetes` 以允许 apiserver 读取存储在节点磁盘上的证书和凭证。这些（证书和凭证）也可以存储在永久性磁盘上，例如 GCE PD，或烧录到镜像中。
- 可选，您也可以挂载 `/var/log`，并将输出重定向到那里。（未显示在模板中）。
  - 如果您希望使用 journalctl 等工具从根文件系统访问日志，请执行此操作。



*待办* 文档化 proxy-ssh 的安装



##### Cloud Provider

Apiserver 支持若干 cloud providers。

- `--cloud-provider` 标志的选项有 `aws`，`azure`，`cloudstack`，`fake`， `gce`， `mesos`， `openstack`， `ovirt`，`photon`，`rackspace`，`vsphere` 或者不设置。
- 在使用裸机安装的情况下无需设置。
- 通过在 [这里](https://releases.k8s.io/{{page.githubbranch}}/pkg/cloudprovider/providers) 贡献代码添加对新 IaaS 的支持。



一些 cloud provider 需要一个配置文件。如果是这样，您需要将配置文件放入 apiserver 镜像或通过 hostPath 挂载。

- `--cloud-config=` 在 cloud provider 需要配置文件时设置。
- 由 `aws`, `gce`, `mesos`, `openshift`, `ovirt` 和 `rackspace` 使用。
- 您必须将配置文件放入 apiserver 镜像或通过 hostPath 挂载。
- Cloud 配置文件语法为 [Gcfg](https://code.google.com/p/gcfg/)。
- AWS 格式由 [AWSCloudConfig](https://releases.k8s.io/{{page.githubbranch}}/pkg/cloudprovider/providers/aws/aws.go) 定义。
- 其他 cloud provider 的相应文件中也有类似的类型。
- GCE 例子：在 [这个文件](https://releases.k8s.io/{{page.githubbranch}}/cluster/gce/configure-vm.sh) 中搜索`gce.conf`。



#### Scheduler pod 模板

完成 Scheduler pod 模板：

```json
{
  "kind": "Pod",
  "apiVersion": "v1",
  "metadata": {
    "name": "kube-scheduler"
  },
  "spec": {
    "hostNetwork": true,
    "containers": [
      {
        "name": "kube-scheduler",
        "image": "$HYBERKUBE_IMAGE",
        "command": [
          "/hyperkube",
          "scheduler",
          "--master=127.0.0.1:8080",
          "$SCHEDULER_FLAG1",
          ...
          "$SCHEDULER_FLAGN"
        ],
        "livenessProbe": {
          "httpGet": {
            "scheme": "HTTP",
            "host": "127.0.0.1",
            "port": 10251,
            "path": "/healthz"
          },
          "initialDelaySeconds": 15,
          "timeoutSeconds": 15
        }
      }
    ]
  }
}
```



通常，调度程序不需要额外的标志。

或者，您也可能需要挂载 `/var/log`，并重定向输出到这里。



#### Controller Manager 模板

Controller Manager pod 模板

```json
{
  "kind": "Pod",
  "apiVersion": "v1",
  "metadata": {
    "name": "kube-controller-manager"
  },
  "spec": {
    "hostNetwork": true,
    "containers": [
      {
        "name": "kube-controller-manager",
        "image": "$HYPERKUBE_IMAGE",
        "command": [
          "/hyperkube",
          "controller-manager",
          "$CNTRLMNGR_FLAG1",
          ...
          "$CNTRLMNGR_FLAGN"
        ],
        "volumeMounts": [
          {
            "name": "srvkube",
            "mountPath": "/srv/kubernetes",
            "readOnly": true
          },
          {
            "name": "etcssl",
            "mountPath": "/etc/ssl",
            "readOnly": true
          }
        ],
        "livenessProbe": {
          "httpGet": {
            "scheme": "HTTP",
            "host": "127.0.0.1",
            "port": 10252,
            "path": "/healthz"
          },
          "initialDelaySeconds": 15,
          "timeoutSeconds": 15
        }
      }
    ],
    "volumes": [
      {
        "name": "srvkube",
        "hostPath": {
          "path": "/srv/kubernetes"
        }
      },
      {
        "name": "etcssl",
        "hostPath": {
          "path": "/etc/ssl"
        }
      }
    ]
  }
}
```



使用 controller manager 时需要考虑的标志：

 - `--cluster-cidr=`，集群中 pod 的 CIDR 范围。
 - `--allocate-node-cidrs=`，如果您使用 `--cloud-provider=`，请分配并设置云提供商上的 pod 的 CIDR。
 - `--cloud-provider=` 和 `--cloud-config` 如 apiserver 部分所述。
 - `--service-account-private-key-file=/srv/kubernetes/server.key`，由  [service account](/docs/user-guide/service-accounts) 功能使用。
 - `--master=127.0.0.1:8080`



#### 启动和验证 Apiserver，Scheduler 和 Controller Manager

将每个完成的 pod 模板放入 kubelet 配置目录中（kubelet 的参数 `--config=` 参数设置的值，通常是 `/etc/kubernetes/manifests`）。顺序不重要：scheduler 和 controller manager 将重试到 apiserver 的连接，直到它启动为止。



使用 `ps` 或 `docker ps` 来验证每个进程是否已经启动。例如，验证 kubelet 是否已经启动了一个 apiserver 的容器：

```shell
$ sudo docker ps | grep apiserver:
5783290746d5        gcr.io/google_containers/kube-apiserver:e36bf367342b5a80d7467fd7611ad873            "/bin/sh -c '/usr/lo'"    10 seconds ago      Up 9 seconds                              k8s_kube-apiserver.feb145e7_kube-apiserver-kubernetes-master_default_eaebc600cf80dae59902b44225f2fc0a_225a4695
```

然后尝试连接到 apiserver：

```shell
$ echo $(curl -s http://localhost:8080/healthz)
ok
$ curl -s http://localhost:8080/api
{
  "versions": [
    "v1"
  ]
}
```



如果您为 kubelet 选择了 `--register-node=true` 选项，那么它们向 apiserver 自动注册。
您应该很快就可以通过运行 `kubectl get nodes` 命令查看所有节点。
否则，您需要手动创建节点对象。



### 启动集群服务

您将希望通过添加集群范围的服务来完成您的 Kubernetes 集群。这些有时被称为 *addons*，查阅在 [管理指南](/docs/admin/cluster-components/#addons) 中的概述。



下面给出了设置每个集群服务的注意事项：

* 集群 DNS：
  * 许多 Kubernetes 的例子都需要
  * [安装说明](http://releases.k8s.io/{{page.githubbranch}}/cluster/addons/dns/)
  * [管理指南](/docs/concepts/services-networking/dns-pod-service/)
* 集群级日志
  * [集群级日志概述](/docs/user-guide/logging/overview)
  * [使用 Elasticsearch 的集群级日志](/docs/user-guide/logging/elasticsearch)
  *  [使用 Stackdriver 的集群级日志](/docs/user-guide/logging/stackdriver)
* 容器资源监控
  * [安装说明](http://releases.k8s.io/{{page.githubbranch}}/cluster/addons/cluster-monitoring/)
* GUI
  * 集群 [安装说明](https://github.com/kubernetes/kube-ui)



## 故障排除

### 运行 validate-cluster

`cluster/kube-up.sh` 调用 `cluster/validate-cluster.sh` 用于确定集群启动是否成功。



使用和输出示例：

```shell
KUBECTL_PATH=$(which kubectl) NUM_NODES=3 KUBERNETES_PROVIDER=local cluster/validate-cluster.sh
Found 3 node(s).
NAME                    STATUS    AGE     VERSION
node1.local             Ready     1h      v1.6.9+a3d1dfa6f4335
node2.local             Ready     1h      v1.6.9+a3d1dfa6f4335
node3.local             Ready     1h      v1.6.9+a3d1dfa6f4335
Validate output:
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-1               Healthy   {"health": "true"}
etcd-2               Healthy   {"health": "true"}
etcd-0               Healthy   {"health": "true"}
Cluster validation succeeded
```



### 检查 pod 和 service

尝试通过其他入门指南中的 “检查集群” 部分，例如 [GCE](/docs/getting-started-guides/gce/#inspect-your-cluster)。您应该看到一些服务。还应该看到 apiserver，scheduler 和 controller-manager 的 “镜像 pod”，以及您启动的任何加载项。



### 试一试例子

此时，您应该能够运行一个基本的例子，例如 [nginx 例子](/docs/tutorials/stateless-application/deployment.yaml)。



### 运行一致性测试

您可能想尝试运行 [一致性测试](http://releases.k8s.io/ {{page.githubbranch}} /test/e2e_node/conformance/run_test.sh)。任何失败都会给一个提示，您需要更多地关注它们。



### 网络

节点必须能够使用其私有 IP 通讯。从一个节点到另一个节点使用 ping 或 SSH 命令进行验证。



### 帮助

如果您遇到问题，请参阅 [troubleshooting](/docs/getting-started-guides/gce#troubleshooting)，联系 [kubernetes 用户组](https://groups.google.com/forum/#!forum/kubernetes-users)，或者在 [Slack](/docs/troubleshooting#slack) 提问。



## 支持级别


IaaS 提供商        | 配置管理 | 系统     | 网络  | 文档                                              | 整合 | 支持级别
-------------------- | ------------ | ------ | ----------  | ---------------------------------------------     | ---------| ----------------------------
任何                  | 任何          | 任何    | 任何         | [文档](/docs/getting-started-guides/scratch)                                |          | 社区 ([@erictune](https://github.com/erictune))


有关所有解决方案的支持级别信息，请参阅图表 [解决方案表](/docs/getting-started-guides/#table-of-solutions)。
