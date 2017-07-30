---
assignees:
- dcbw
- freehan
- thockin
title: 网络插件
redirect_from:
- "/docs/admin/network-plugins/"
- "/docs/admin/network-plugins.html"
---


* TOC
{:toc}

__免责声明__: 网络插件还处于Alpha测试版。其内容会经常发生变化。

Kubernetes中的网络插件会带有一些特点：

* CNI plugins: 遵守appc/CNI规范，旨在实现互操作性。
* Kubenet plugins: 基于`cbr0`，使用CNI的`bridge`和`host-local`插件


## 安装

kubelet拥有单一默认的网络插件，并且是整个集群通用的默认网络。当它在启用时检测插件，记忆发现的内容，并在pod生命周期中适当地执行选定的插件（这对于docker来说是正确的，因为rkt管理自己的CNI插件）。当使用插件时，请记住两个kubelet命令行参数：

*  `network-plugin-dir`: kubelet在启动时检测这个插件目录。
*  `network-plugin`: `network-plugin-dir`要使用的网络插件。它必须与插件目录中检测的插件名相匹配。对于CNI网络，就匹配"cni"。


## 网络插件要求

除了提供[`NetworkPlugin`接口](https://github.com/kubernetes/kubernetes/tree/{{page.fullversion}}/pkg/kubelet/network/plugins.go)去配置和清理pod网络，此插件还需要对kube-proxy有特定的支持。iptables代理显然依赖于iptables，以确保容器的流量可用于iptables。例如，如果插件将容器连接到Linux网桥，插件必须设置`net/bridge/bridge-nf-call-iptables` sysctl为1，以确保iptables代理功能正常。如果插件不使用Linux网桥（而是像Open vSwitch或其他机制）,就应该确保容器流量正确地路由到代理。

默认情况下，如果没有指定kubelet网络插件。则使用`noop`插件，设置`net/bridge-nf-call-iptables=1`以确保简单的配置（如类似docker的网桥）可与iptables代理正常工作。


### CNI

通过kubelet的`--network-plugin=cni`命令行选择项来选择CNI插件。kubelet从`--cni-conf-dir`（默认为`/etc/cni/net.d`）中读取文件，并使用该文件中的CNI配置去设置每个pod网络。CNI配置文件必须与[CNI](https://github.com/containernetworking/cni/blob/master/SPEC.md#network-configuration)相匹配，并且任何所需的CNI插件的配置必须引用目前的`--cni-bin-dir`(默认为`/opt/cni/bin`)。	

如果目录中有多个CNI配置文件，则使用按文件名字典序列中的第一个。


除了由配置文件指定的CNI插件以外，Kubernetes需要标准的CNI[`lo`](https://github.com/containernetworking/plugins/blob/master/plugins/main/loopback/loopback.go)插件，最低版本为0.2.0。

限制：由于[#31307](https://github.com/kubernetes/kubernetes/issues/31307),目前`HostPort`不能使用CNI网络插件。这意味着pod中所有`HostPort`属性将被简单地忽略。


### kubenet

kubenet是一个在Linux上非常基本，简单的网络插件。它本身不会实现更高级的功能，如跨node网络或网络策略。它通常与云提供商一起使用，为跨nodes或者单node环境通信设置路由规则。



kubenet创建名为`cbr0`的Linux网桥，为每一个pod创建veth设备对，并将每一对的主机端连接到`cbr0`。每一对pod端会通过配置或控制器管理而分配到指定的IP地址，`cbr0`会分配的一个匹配主机接口上开启的最小的MTU。


此插件需要做一些事：

* 标准CNI需要插件 `网桥`,`lo`和`host-local`，其最低的版本为0.2.0。kubenet首先在`/opt/cni/bin`中搜索。指定`network-plugin-dir`以支持额外的搜索路径。第一个发现匹配的将被生效。
* kubelet必须运行`--network-plugin=kubenet`参数，以开启插件。
* kubelet也可以运行`--non-masquerade-cidr=<clusterCidr>`参数，向该IP段之外的IP地址发送的流量将使用IP masquerade技术。
* node必须被分配到一个IP子网，无论`--pod-cidr`命令行选项或是`--allcate-node-cidrs=true --cluster-cidr=<cidr>`控制管理命令行选项。


### 定制MTU（使用kubenet)

正确的配置MTU才能获得最佳的网络性能。网络插件通常会尝试去推断出合理的MTU，但有时的逻辑，推断不出最优的MTU。例如，如果Docker网桥或其他具有小MTU的接口，kubenet将会选择当前的MTU。如果你使用IPSEC封装，MTU必会被减少，此计算对于大多数的网络插件来说都不合适。


在需要的地方，你可以使用kuelet的`network-plugin-mtu`选项指定MTU，例如，在AWS上，`eth0`MTU通常为9001，你也可以指定`--network-plugin-mtu=9001`。如果你使用IPSEC，可以为了封装开销而减少MTU，例如`--network-plugin-mtu=8873`。

只有网络插件提供此选项，目前**只有kubenet支持`network-plugin-mtu`**。


## 用法总结

* `--network-plugin=cni`规定了`cni`网络插件的使用，CNI插件二进制文件位于`--cni-bin-dir`（默认是`/opt/cni/bin`），其配置位于`--cni-conf-dir`（默认是/etc/cni/net.d）。
* `--network-plugin=kubenet`规定了`kubenet`网络插件的使用，CNI的`bridge`和`host-local`插件，位于`/opt/cni/bin`或`network-plugin-dir`
* `--network-plugin-mtu=9001`规定MTU的使用，目前只能在`kubenet`网络插件中使用。


