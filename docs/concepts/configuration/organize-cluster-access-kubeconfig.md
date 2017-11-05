---
title: 使用 kubeconfig 文件组织集群访问
cn-approvers:
- chentao1596
---


{% capture overview %}


kubeconfig 文件用于组织关于集群、用户、命名空间和认证机制的信息。命令行工具 `kubectl` 从 kubeconfig 文件中得到它要选择的集群以及跟集群 API server 交互的信息。


**注意：** 用于配置集群访问信息的文件叫作 *kubeconfig 文件*，这是一种引用配置文件的通用方式，并不是说它的文件名就是 `kubeconfig`。
{: .note}


默认情况下，`kubectl` 会从 `$HOME/.kube` 目录下查找文件名为 `config` 的文件。您可以通过设置环境变量 `KUBECONFIG` 或者通过设置 [`--kubeconfig`](/docs/user-guide/kubectl/{{page.version}}/) 去指定其它 kubeconfig 文件。


关于怎么样一步一步去创建和配置 kubeconfig 文件，请查看 [配置访问多个集群](/docs/tasks/access-application-cluster/configure-access-multiple-clusters)。

{% endcapture %}


{% capture body %}


## 支持多个集群、用户和身份验证机制


假设您有几个集群，并且用户和组件以多种方式进行身份验证。例如：


- 运行中的 kubelet 可能使用证书进行身份认证。
- 用户可能使用令牌进行身份验证。
- 管理员可能有一组提供给各个用户的证书。


使用 kubeconfig 文件，可以组织您的集群、用户和命名空间的信息。并且，您还可以定义 context，以便快速轻松地在集群和命名空间之间进行切换。

## Context


kubeconfig 文件可以包含 *context* 元素，每个 context 都是一个由（集群、命名空间、用户）描述的三元组。您可以使用 `kubectl config use-context` 去设置当前的 context。命令行工具 `kubectl` 与当前 context 中指定的集群和命名空间进行通信，并且使用当前 context 中包含的用户凭证。


## 环境变量 KUBECONFIG


环境变量 `KUBECONFIG` 保存一个 kubeconfig 文件列表。对于 Linux 和 Mac 系统，列表使用冒号将文件名进行分隔；对于 Windows 系统，则以分号分隔。环境变量 `KUBECONFIG` 不是必需的，如果它不存在，`kubectl` 就使用默认的 kubeconfig 文件 `$HOME/.kube/config`。


如果环境变量 `KUBECONFIG` 存在，那么 `kubectl` 使用的有效配置，是环境变量 `KUBECONFIG` 中列出的所有文件融合之后的结果。


## 融合 kubeconfig 文件


想要查看您的配置，请输入命令：

```shell
kubectl config view
```


如前所述，输出的内容可能来自单个 kubeconfig 文件，也可能是多个 kubeconfig 文件融合之后的结果。


当配置是由多个 kubeconfig 文件融合而成时，`kubectl` 使用的规则如下：


1. 如果设置了 `--kubeconfig`，那么只使用指定的文件，不需要融合。该标志只允许设置一次。
	
   如果设置了环境变量 `KUBECONFIG`，那么应该融合文件之后再来使用。
   根据如下规则融合环境变量 `KUBECONFIG` 中列出的文件：
   
   * 忽略空的文件名。
   * 文件内容存在不能反序列化的情况时，融合出错。
   * 多个文件设置了特定的值或者映射键时，以第一个查找到的文件中的内容为准。
   * 永远不要更改值或映射键。
     例如：保存第一个文件的 context，将其设置为 `current-context`。
	 例如：如果两个文件中都指定了 `red-user`，那么只使用第一个指定的 `red-user`。
     即使第二个文件的 `red-user` 下的条目跟第一个文件中指定的没有冲突，也丢弃它们。
	 
   设置环境变量 `KUBECONFIG` 的例子，请查看 [设置环境变量 KUBECONFIG](/docs/tasks/access-application-cluster/configure-access-multiple-clusters/#set-the-kubeconfig-environment-variable)。

   如果 `--kubeconfig` 和环境变量 `KUBECONFIG` 都没有设置，则使用默认的 kubeconfig 文件：`$HOME/.kube/config`，不需要融合。
   

1. 确定要使用的 context 时按照以下顺序查找，直到找到一个可用的context：
    1. 如果命令行参数 `--context` 存在的话，使用它指定的值。
	1. 使用融合 kubeconfig 文件之后的 `current-context` 。
	
   如果还未找到可用的 context，此时允许使用空的 context。	


1. 确定集群和用户。此时，可能存在 context，也可能没有。
   按照以下顺序查找，直到找到一个可用的集群或用户。该链查找过程运行两次：一次用于查找用户，另一次用于查找集群：
   
   1. 如果存在命令行参数：`--user` 或者 `--cluster`，则使用它们指定的值。
   1. 如果 context 非空，则从 context 中取用户或者集群。
   
   如果还未找到可用的用户或者集群，此时用户和集群可以为空。


1. 确定要使用的实际集群信息。此时，集群信息可能存在，也可能不存在。
   按照以下顺序查找，选择第一个查找到的内容：
   
   1. 如果存在命令行参数：`--server`、`--certificate-authority` 和 `--insecure-skip-tls-verify`，则使用它们指定的值。
   1. 融合 kubeconfig 文件后，如果有任何集群属性存在，都使用它们。
   1. 如果没有指定服务位置，则确定集群信息失败。


1. 确定要使用的实际用户信息。除了每个用户只能使用一个身份验证技术之外，使用与构建集群信息相同的规则来构建用户信息：

   1. 如果存在命令行参数：`--client-certificate`、`--client-key`、`--username`、`--password` 和 `--token`，使用它们指定的值。
   1. 融合 kubeconfig 文件后，使用 `user` 字段。
   1. 如果存在两种矛盾的身份验证技术，则确定用户信息失败。


1. 对于仍然缺失的任何信息，使用默认值，并潜在地提示身份验证信息。


## 文件引用



kubeconfig 文件中的文件和路径引用，都是相对 kubeconfig 文件存在的。命令行中的文件引用则是相对于当前工作目录。在文件 `$HOME/.kube/config` 中，相对路径按照相对关系存储，绝对路径按照绝对关系存储。

{% endcapture %}


{% capture whatsnext %}


* [配置访问多个集群](/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)
* [kubectl 配置](/docs/user-guide/kubectl/{{page.version}}/)

{% endcapture %}

{% include templates/concept.md %}

