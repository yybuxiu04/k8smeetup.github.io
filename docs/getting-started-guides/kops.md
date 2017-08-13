
---
标题: 使用 kops 在 AWS 上安装 Kubernetes
---



## 概览

这个快速开始展示你如何简单的在 AWS 上安装一个 Kubernetes 集群。
它使用一个工具叫 [`kops`](https://github.com/kubernetes/kops)。



kops 是一个自用的供应系统：

* 全自动安装
* 使用 DNS 定义集群
* 自我修复： 每个东西运行在自动扩展组
* 有限的系统支持（优先 Debian，支持 Ububtu 16.04，更早支持 CentOS 和 RHEL）
* 支持高可用
* 可直接提供，或生成 terraform 显现

如果你的想法区别于这些，你也许喜欢使用 [kubeadm](/docs/admin/kubeadm/)作为构建块建立你自己的集群



## 创建一个集群

### （1/5）安装 kops

#### 需求

为了 kops 正常工作，你必须安装了 [kubectl](/docs/tasks/tools/install-kubectl/)。



#### 安装

从[发行页面](https://github.com/kubernetes/kops/releases)下载 kops（从源编译也很简单）：



在 MacOS 上：
```
wget https://github.com/kubernetes/kops/releases/download/1.6.1/kops-darwin-amd64
chmod +x kops-darwin-amd64
mv kops-darwin-amd64 /usr/local/bin/kops
# 你也可以使用 Homebrew 安装
brew update && brew install kops
```

在 Linux 上：

```
wget https://github.com/kubernetes/kops/releases/download/1.6.1/kops-linux-amd64
chmod +x kops-linux-amd64
mv kops-linux-amd64 /usr/local/bin/kops
```



### （2/5）为你的集群创建一个 route53 域名

kops 使用 DNS 来发现，无论是在集群内还是在客户端都可以访问 kubeernetes API 服务器。

kops 在集群名上有个强烈的建议：它应该是一个合法的 DNS 名。通过这么做，你将不再混淆你的集群，你可以毫不犹豫的与同事共享集群，你不用记住 IP 地址就可以访问他们。

你可以，并且很可能应该，使用子域名来划分你的集群。作为我们的例子，我们将使用 `useast1.dev.example.com`。API 服务器后端将会是 `api.useast1.dev.example.com`。

一个 Route53 托管区能服务子域名。你的托管区可能是 `useast1.dev.example.com`，也可能是 `dev.example.com` 甚至是 `example.com`。kops 与他们任何一个一起工作，通常你选择由于组织的原因（例如，你可以在 `dev.example.com` 下创建记录，但是不能在 `example.com` 下）。



假定你在使用 `dev.example.com` 作为你的托管区。你使用[正常流程](http://docs.aws.amazon.com/Route53/latest/DeveloperGuide/CreatingNewSubdomain.html)创建托管区，或使用命令，例如 `aws route53 create-hosted-zone --name dev.example.com --caller-reference 1`。

你必须在父域名上设置你的 NS 记录，因此在域名的记录将被解析。这里，你将在 `example.com` 为 `dev` 创建 NS 记录。如果他是一个根域名，你将在你的域名注册商配置 NS 记录（例如，`example.com` 将需要在你购买 `example.com` 的地方配置）。

这一步很容易搞乱（它是问题的第一原因！）你可以再次检查你的集群配置正确，如果你有 dig 工具运行：

`dig NS dev.example.com`

你应该看到4个 Route53 分配给你的托管区的 NS 记录。



### (3/5) 创建一个 S3 bucket 来存储你的集群状态

kops 允许你管理你的集群甚至是安装后。为做到这个，它必须跟踪你创建的集群与其配置，它们使用的密钥等等。这个信息被存储在一个 S3 bucket。S3 权限被用来控制 bucket 的访问。

多集群可以使用同样的 S3 bucket，你可以共享一个 S3 bucket 给你管理相同集群的同事 - 这比传递 kubecfg 文件更容易。但是任何可以访问 S3 bucket 的人都可以管理你所有的集群，所以你不会想在运维团队之外去共享它。

所以通常每个 ops 团队都有有一个 S3 bucket（并且经常名称将对应上托管区名称）

在我们的例子中，我们选择 `dev.example.com` 作为我们的托管区，因此让我们选择 `clusters.dev.example.com` 作为 S3 bucket 名。

* 导出 `AWS_PROFILE` （如果你需要选择一个配置文件使 AWS CLI 工作）

* 使用 `aws s3 mb s3://clusters.dev.example.com` 创建 S3 bucket

* 你可以 `export KOPS_STATE_STORE=s3://clusters.dev.example.com` 然后 kops 将使用默认这个位置。我们建议将其放在你的 bash_profile 或者类似文件中。



### （4/5）建立你的集群配置

运行 “kops create cluster” 来创建你的集群配置文件：

`kops create cluster --zones=us-east-1c useast1.dev.example.com`

kops 将为你的集群创建配置。注意它 _只_ 创建配置，不真正创建云资源 - 你将在下一步用 `kops update cluster` 来创建它。这给你一个机会来审查配置或改变它。

你可以用打印命令来进一步的检查：

* 列出你的集群：`kops get cluster`
* 编辑这个集群：`kops edit cluster useast1.dev.example.com`
* 编辑你的节点实例组：`kops edit ig --name=useast1.dev.example.com nodes`
* 编辑你的主实例组：`kops edit ig --name=useast1.dev.example.com master-us-east-1c`

如果这是你第一次使用 kops，花几分钟来尝试这些！一个实例组是一系列实例，它将注册为 kubernetes 节点。在 AWS 上这是通过自动伸缩组来实现的。你可以有多种实例组，例如，如果你想要竞价和永久实例混合的节点，或 GPU 和无 GPU 实例。



### （5/5）在 AWS 创建集群

运行 “kops update cluster” 在 AWS 创建你的集群：

`kops update cluster useast1.dev.example.com --yes`

这将花几秒来运行，但是你的集群好像花了几分钟来确认准备完毕。当你更改集群配置时，`kops update cluster` 将成为你使用的工具；它将你改变的配置应用到你的集群 - 看需要重新配置 AWS 或 kubernetes。

举例，在你 `kops edit ig nodes` 后，然后 `kops update cluster --yes` 来生效你的配置，又是你害需要 `kops rolling-update cluster` 来立刻动态更新配置。



### 探索其他插件

查看[插件列表](/docs/concepts/cluster-administration/addons/)来探索其他插件，工具包括日志，监控，网络策略，虚拟化与控制你的 Kubernetes 集群。

## 下一步是什么

* 学习更多关于 Kubernetes [概念](/docs/concepts/)和 [`kubectl`](/docs/user-guide/kubectl-overview/)。
* 学习关于 `kops` [高级用法](https://github.com/kubernetes/kops)

## 清理

* 删除你的集群：`kops delete cluster useast1.dev.example.com --yes`

## 反馈

* Slack 频道：[#sig-aws](https://kubernetes.slack.com/messages/sig-aws/) 有很多 kops 用户
* [GitHub Issues](https://github.com/kubernetes/kops/issues)
