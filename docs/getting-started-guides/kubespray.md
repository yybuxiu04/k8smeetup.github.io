---
title: 使用 Kubespray 在基础设施或云平台上安装 Kubernetes
---



## 概述

本文介绍了如何使用 [`Kubespray`](https://github.com/kubernetes-incubator/kubespray)
工具在 GCE, Azure, OpenStack, AWS 托管的主机或者裸机上安装 Kubernetes 集群。

Kubespray 由一系列的 [Ansible](http://docs.ansible.com/) playbook、生成 [inventory](https://github.com/kubernetes-incubator/kubespray/blob/master/docs/ansible.md) 的命令行工具以及生成 OS/Kubernetes 集群配置管理任务的专业知识构成。它有以下特性：
 
* [高可用集群](https://github.com/kubernetes-incubator/kubespray/blob/master/docs/ha-mode.md)
* [Composable](https://github.com/kubernetes-incubator/kubespray/blob/master/docs/vars.md)
  (例如：使用网络插件)
* 支持[主流 Linux](https://github.com/kubernetes-incubator/kubespray#supported-linux-distributions) 发行版
* 持续集成测试

为了选择更适合项目的工具，您可以参考它跟 [kubeadm](../kubeadm) 以及 [kops](../kops) 的[对比说明](https://github.com/kubernetes-incubator/kubespray/blob/master/docs/comparisons.md)。



## 创建集群

### (1/4) 确保满足承载[要求](https://github.com/kubernetes-incubator/kubespray#requirements)

#### 清单

* 方式一：使用 [kubespray-cli 工具](https://github.com/kubernetes-incubator/kubespray/blob/master/docs/getting-started.md) 配置裸机或者云实例节点，以支持部署 Kubernetes 集群。
  
* 方式二：使用自己选择的工具提供裸机或者启动云实例，然后使用[工具](https://github.com/kubernetes-incubator/kubespray/blob/master/contrib/inventory_builder/inventory.py)为 Ansible 创建 inventory 文件。



### (2/4) 编写配置文件

#### 清单

* 可以使用[配置默认值](https://github.com/kubernetes-incubator/kubespray/blob/master/docs/vars.md)直接进行部署( Kubespray 将默认使用 calico 作为网络插件)，也可以通过生成 [inventory](https://github.com/kubernetes-incubator/kubespray/blob/master/docs/getting-started.md#building-your-own-inventory) 文件以及覆盖默认变量的方式自定义您的部署。可以配置的内容包括：
  * DNS [配置选项](https://github.com/kubernetes-incubator/kubespray/blob/master/docs/dns-stack.md)
  * 使用的[网络插件](https://github.com/kubernetes-incubator/kubespray#network-plugins) 
  * 组件[版本](https://github.com/kubernetes-incubator/kubespray#versions-of-supported-components)
  * 附加节点组，例如 [bastion hosts](https://github.com/kubernetes-incubator/kubespray/blob/master/docs/ansible.md#bastion-host) 或者
    [Calico BGP 路由反射器](https://github.com/kubernetes-incubator/kubespray/blob/master/docs/calico.md#optional--bgp-peering-with-border-routers)
* 如果选用自定义部署或是使用[集群配置文件](https://github.com/kubernetes-incubator/kubespray/blob/master/cluster.yml)中的默认组合，Kubespray 允许用户使用 ``ansible-playbook`` 指定 inventory 、playbooks 、data 重写和标记、limits 、batches 节点等任意参数。
  
* 对于大型部署（100多个节点），可以根据[此](https://github.com/kubernetes-incubator/kubespray/blob/master/docs/large-deployments.md)调整优化配置。



### (3/4) 运行部署

#### 清单

* 使用 [kubespray-cli](https://github.com/kubernetes-incubator/kubespray/blob/master/docs/getting-started.md) 工具或者 ``ansible-playbook`` [命令](https://github.com/kubernetes-incubator/kubespray/blob/master/docs/getting-started.md#starting-custom-deployment)部署应用。



### (4/4) （可选）使用 [Netchecker](https://github.com/kubernetes-incubator/kubespray/blob/master/docs/netcheck.md) 验证 pod 连通性和 DNS 解析

#### 清单

* 确保 netchecker-agent 的 pod 可以解析 DNS 请求，并在默认命名空间内 ping 通。这些 pod 模仿其他工作负载的类似行为，并充当群集健康指标。



## 查找可附加的组件

查看[可附加组件列表文件](https://github.com/kubernetes-incubator/kubespray/tree/master/contrib)找其他可部署选项。



## 下一步

Kubespray 有不少的[标记在雷达](https://github.com/kubernetes-incubator/kubespray/blob/master/docs/roadmap.md)上。



## 清理集群

要删除临时集群，可以使用 ``ansible-playbook`` 命令 [reset role](https://github.com/kubernetes-incubator/kubespray/blob/master/roles/reset/tasks/main.yml)。

注意，不建议使用 reset 命令删除生产环境！ 



## 反馈

* Slack 通道: [kubespray](https://kubernetes.slack.com/messages/kubespray/)
* [GitHub 问题](https://github.com/kubernetes-incubator/kubespray/issues)
