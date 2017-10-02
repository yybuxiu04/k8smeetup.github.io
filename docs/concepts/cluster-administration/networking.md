
---
assignees:
- thockin
title: 集群网络
redirect_from:
- "/docs/admin/networking/"
- "/docs/admin/networking.html"

---



Kubernetes 对网络的处理方式与 docker 有些不同，有4个不同的网络问题需要解决：

1. 高度耦合的容器到容器之间的通信：解决 [pods](/docs/concepts/workloads/pods/pod/) 和 `localhost` 的通信。
2. 抽象的 pod 到 pod 之间的通信：本文档的重点。
3. pod 到 service 之间的通信：涵盖的[服务](/docs/concepts/services-networking/service/)。
4. 集群外部与内部组件之间的通信：涵盖的[服务](/docs/concepts/services-netwoking/service/)。




## 总结

Kubernetes 假设 pod 可以与其他 pod 进行通信，不管他们是在哪个宿主机上。每个 pod 都有一个独立的 IP 地址，所以不需要额外考虑如何建立 pod 之间的连接，也不需要考虑将容器端口映射到主机端口的问题。创建一个简单的兼容性较好的模型。从该模型的网络的端口分配、域名解析、服务发现、负载均衡、应用配置和迁移等角度来看，pod 都能够被看做是一台独立的“虚拟机”或“服务器”。



为了实现这一点，我们必须对如何设置集群网络提出一些要求。



## Docker 模型

在讨论 kubernetes 实现的网络之前，值得回顾一下 Docker “标准”的网络模式。默认情况下，Docker 使用私有网络。创建名称为 `docker0` 的虚拟网桥，并从[RFC1918](https://tools.ietf.org/html/rfc1918)定义的私有网络块中分配一个子网。Docker 创建出来的每一个容器，都会创建一个虚拟的以太设备(称作`Veth`），其中一端关联到网桥上，另一端使网络用 Linux 的网路命名空间技术，映射到容器内的 `eth0` 设备，然后从网桥的地址段内给容器内 `eth0` 接口分配一个IP地址。



这样做的结果就是，同一台机器(或相同的虚拟网桥）内的容器之间可以相互通信。不同机器上的容器不能够互相通信。实际上他们甚至有可能会在相同的网络地址范围内。



为了让 Docker 容器跨节点通信，他们必须在机器的  IP地址上分配端口，然后通过这个端口转发或代理到容器。这种做法显然意味着一定要在容器之间小心谨慎地协调好端口的分配，或者使用动态端口分配的技术。



## Kubernetes 模型

大多数开发者通过暴露端口动态分配端口给系统带来了很多复杂性 - 每个应用程序都将端口作为标志，API 服务必须知道如何将动态端口插入到配置块中，以及服务之间是如何查找对方等等。而 kubernetes 处理这些问题时，采用了不同的方法。



Kubernetes 对任何网络实施了以下基本要求



 * 所有容器都可以在不用 NAT 的方式下同别的容器通信；
 * 所有节点都可以在不用 NAT 的方式下同所有容器通信，反之亦然；
 * 容器的地址和别人看到的地址是同一个地址。

这些基本的要求意味着并不是只要两台计算机运行 Docker,Kubernetes 就可以工作了。具体的集群网络实现必须保障上述基本要求。



实际上，这种网络模型的整体上并没有那么复杂，主要是为了与 Kubernetes 兼容，从而实现将已有的应用程序很容易地从 VM 迁移到容器上。如果你的应用程序原先在 VM 上运行，而那些 VM 拥有独立的 IP，并且它们之间可以直接透明地相互通信。那么 Kubernetes 的网络模型与 VM 使用的网络模型是相似的。



到目前为止，文章已经讨论到了容器。实际上，kubernets 在 `Pod` 范围内使用IP地址 - `Pod` 中的容器共享它们网络命名空间 - 包括其 IP 地址。这意味着 `Pod` 中的容器都可以在 `localhost` 上达到对方的端口。`Pod` 中的容器必须协调端口的使用，但这与 VM 中的进程没什么别的区别。我们称作为 "IP-per-pod" 模型。在Docker 中实现一个 “pod容器”，需在该“应用容器”（用户指定）加入该命名空间的同时打开 Docker 的 `--net=container:<id>` 功能



与 Docker 一样，可以请求主机端口，使其减少一些操作。在本例中，将在主机的 `Node` 上分配端口并使其通信转发到 `Pod`。`Pod`  本身是看不到主机的端口存在或不存在的。



## 如何实现这些

有多种方式可以实现这些网络模型。这篇文档不对各种方法进行详细的研究，但希望以此可以作为对各种技术介绍的开始。

以下的网络模型选项按字母的顺序排序 - 此排名不分先后。



### Cilium

[Cilium](https://github.com/cilium/cilium)是一个提供在应用容器间透明连接的安全网络开源软件。其能对 L7/HTTP 进行感知，在 L3-L7 上实施基于身份的安全模型的网络策略并与网络地址相互解耦。



### Contiv

[Contiv](https://github.com/contiv/netplugin)提供了许多配置网络的案例(native l3 using BGP, overlay using vxlan,  classic l2, or Cisco-SDN/ACI)。[Contiv](http://contiv.io)的代码全部开源。



### Contrail

[Contrail](http://www.juniper.net/us/en/products-services/sdn/contrail/contrail-networking/),基于[OpenContrail](http://www.opencontrail.org)，一个开放的，集合多种云网络虚拟化的策略管理平台。Contrail/OpenContrail 结合了多种编排系统：例如 Kubernetes, OpenShift, OpenStack和Mesos，并为虚拟机、容器/pods 和裸机工作负荷提供不同的隔离模式。


### Flannel

[Flannel](https://github.com/coreos/flannel#flannel)是一个满足Kubernetes需求的简单overlay网络。 许多人反应Flannel和Kubernetes是成功的。



### 谷歌计算引擎（GCE）

对于谷歌计算引擎集群配置脚本，我们使用[高级路由](https://cloud.google.com/compute/docs/networking#routing)为每一个 VM 分配一个子网(默认为`/24` - 254 IPs)。 绑定的子网将被 GCE 网络直接路由到 VM 中。除了分配给 VM 的主 IP 地址以外，这些地址也被用于 NAT 以便用于出站互联网的访问。Linux 网桥(被称为`cbr0`)的配置存在在该子网上，并可被传递到 docker 的 `--bridge` 标记。



启动 Docker:

```shell
DOCKER_OPTS="--bridge=cbr0 --iptables=false --ip-masq=false"
```

这个网桥是由 Kubelet（由`--network-plugin=kubenet`标记控制）根据 `Node` 的 `spec.podCIDR` 创建。

Docker 从 `cbr-cidr` 块中分配 IP。在 `cbr0` 网桥上的 `Nodes`，容器可以相互到达。在 GCE 项目的网络中，这些 IPs 都是可以路由的。



GCE 本身并不知道这些 IPs,所以它不会 NAT 出站网络流量。为了实现这一点，我们使用 iptables 规则（也称为 SNAT - 以使它看起来像是来自于 `Node` 本身的数据包）在 GCE 网络（10.0.0.0/8）之外的 IPs 绑定伪装流量。

```shell
iptables -t nat -A POSTROUTING ! -d 10.0.0.0/8 -o eth0 -j MASQUERADE
```



最后，我们在内核中开启 IP 转发（内核将为网桥容器处理数据包）：

```shell
sysctl net.ipv4.ip_forward=1
```

结果是，所有的 `Pods` 都可以相互通讯，并向互联网发送流量。



### L2 网络和 Linux 网桥

如果你有一个 “dumb” 二层网络，例如一个简单的“裸机”环境，你应该能够做类似上述的 GCE 设置。请注意，这些说明只是非常随意的尝试，尚未经过彻底的测试。如果您使用这种技术并完成此过程，请通知我们。

遵循 Lars Kellogg-Stedman 这个[非常好的教程](http://blog.oddbit.com/2014/08/11/four-ways-to-connect-a-docker/)中 "With Linux Bridge devices" 的部分。




### Nuage Networks VCS(虚拟化云服务)

[Nuage](http://www.nuagenetworks.net)提供一个高度可伸缩的基于策略的软件定义网络(SDN)平台。Nuage 使用开源的 vSwitch 为数据平面的同时提供一个具有开放标准且功能丰富的 SDN 控制器。

Nuage 平台使用 overlays 为 Kubernetes Pods 和 Kubernets 环境（VMs和裸机）之间提供了无缝的基于策略的网络。Nuage 的策略抽象模型是考虑到应用程序的设计，可以容易地为应用程序声明细颗粒度的策略。该平台的实时分析引擎可以实现 Kubernetes 应用程序的可视化和安全监控。



### OpenVSwitch

[OpenVSwitch](/docs/admin/ovs-networking)是一种相对比较成熟和复杂的构建 overlay 网络方法。并且得到了几家大厂商的支持。



### OVN(开放的虚拟网络)

OVN 是 Open vSwitch 社区开发的开源网络虚拟化解决方案。它可以创建逻辑交换机、路由器，有状态 ACLs，负载均衡等来构建不同的虚拟网络拓扑。该项目具有特定的  Kubernetes 插件，文档在[ovn-kubernetes](https://github.com/openvswitch/ovn-kubernetes)。



### Calico 项目

[Calico项目](http://docs.projectcalico.org/)是一个开源的容器网络提供者和网络策略引擎。

Calico 提供了一个高度可扩展的网络和基于相同 IP 网络的 kubernetes pods 策略解决方案。Calico 可以在不封装或 overlays 的情况下部署，以提供高性能、大规模的数据中心网络。通过其分布式防火墙为 kubernetes pods 提供细颗粒度的网络安全策略。



Calico 还可以在策略实施模式下，与其他网络解决方案（如Flannel,又名[canal](https://github.com/tigera/canal) 或本地 GCE 网络相结合运行。



### Romana

[Romana](http://romana.io)是一个开源的网络和安全自动化解决方案，可以无需 Overlay 网络而部署 Kubernetes。Romana 支持 Kubernets[网络策略](/docs/concepts/services-networking/network-policies/)，通过网络命名空间提供网络间的隔离。



### Weaveworks的Weave Net

[Weave Net](https://www.weave.works/products/weave-net/)是一个具有弹性且易于使用的网络，适用于 Kubernetes 及其托管的应用程序，可作为 CNI 插件或者独立运行。在任一版本中，它不需要任何配置或额外的代码运行。在这两种情况下，kubernetes 的标准是为每个 pod 提供一个 IP 地址。



### 华为的GNI-Genie

[CNI-Genie](https://github.com/Huawei-PaaS/CNI-Genie)是一个 CNI 插件，可以使 kubernetes [在运行的同时实现不同](https://github.com/Huawei-PaaS/CNI-Genie/blob/master/docs/multiple-cni-plugins/README.md#what-cni-genie-feature-1-multiple-cni-plugins-enables)的[kubernetes网络模型](https://git.k8s.io/kubernetes.github.io/docs/concepts/cluster-administration/networking.md#kubernetes-model)。这里包括了许多可以实施的[CNI插件](https://github.com/containernetworking/cni#3rd-party-plugins)，例如[Flannel](https://github.com/coreos/flannel#flannel), [Calico](http://docs.projectcalico.org/), [Romana](http://romana.io), [Weave-net](https://www.weave.works/products/weave-net/)。

CNI-Genie 同时支持[分配多个IP地址给pod](https://github.com/Huawei-PaaS/CNI-Genie/blob/master/docs/multiple-ips/README.md#feature-2-extension-cni-genie-multiple-ip-addresses-per-pod)，每个都来自于不同的 CNI 插件。



## 额外阅读

网络模型的早期设计及其基本原理，以及一些未来的计划在[网络设计文档](https://git.k8s.io/community/contributors/design-proposals/networking.md)中有更详细的描述。


