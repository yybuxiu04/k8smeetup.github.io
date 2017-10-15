---
approvers:
- david-mcmahon
- jbeda
title: 下载或构建 Kubernetes
cn-reviewers:
- chentao1596
---


您可以选择使用源代码构建版本或者下载预先构建好的 Kubernetes 版本。如果不打算开发 Kubernetes 本身，建议您使用预先构建好的版本。


如果只是想在本地运行 Kubernetes 来做开发，我们建议使用 Minikube。您能够从 [这里](https://github.com/kubernetes/minikube/releases/latest) 下载 Minikube。Minikube 创建一个运行安全的 Kubernetes 集群的本地虚拟机，使用它处理集群非常的容易。

* TOC
{:toc}


### 预先构建的二进制版本


从 [Kubernetes GitHub 仓库的版本页](https://github.com/kubernetes/kubernetes/releases) 能够找到可下载的二进制版本列表。


下载最新的版本，在 Linux 或者 OS X 上解包 tar 文件，使用 cd 命令进入创建的 `kubernetes/` 目录，然后根据您的云环境按照相应的入门指南进行操作。


如果是在 OS X 上，您也能够使用 [homebrew](http://brew.sh/) 包管理器：`brew install kubernetes-cli`


### 从源代码构建


获取 Kubernetes 源代码。如果只是想从源代码简单地构建版本，您没有必要创建一个完整的 golang 环境，只需要在 Docker 容器中执行构建。


构建版本非常的简单。

```shell
git clone https://github.com/kubernetes/kubernetes.git
cd kubernetes
make release
```


更多构建版本的细节请查看 [`build`](http://releases.k8s.io/{{page.githubbranch}}/build/) 目录


### 下载 Kubernetes 并自动创建一个默认的集群


使用 `wget` 或者 `curl` 命令从 [`https://get.k8s.io`](https://get.k8s.io) 下载 bash 脚本并执行，就会自动下载 Kubernetes，并基于您想要的云服务提供商创建一个集群环境。

```shell
# wget version
export KUBERNETES_PROVIDER=YOUR_PROVIDER; wget -q -O - https://get.k8s.io | bash

# curl version
export KUBERNETES_PROVIDER=YOUR_PROVIDER; curl -sS https://get.k8s.io | bash
```


支持的 `YOUR_PROVIDER` 包括：


* `gce` - 谷歌计算引擎 [默认]
* `gke` - 谷歌容器引擎
* `aws` - 亚马逊 EC2
* `azure` - 微软 Azure
* `vagrant` - Vagrant（在本地虚拟机上）
* `vsphere` - VMWare VSphere
* `rackspace` - Rackspace


如果想要了解脚本支持的最新的完整的供应商名单，可以在查看 Kubernetes 主代码仓库下面的 [`/cluster`](https://github.com/kubernetes/kubernetes/tree/{{page.githubbranch}}/cluster) 目录，目录下面每个文件夹都代表了 `YOUR_PROVIDER` 支持的一个值。如果没有找到您想要的供应商，可以试着查看我们的 [入门指南](/docs/getting-started-guides)，很有可能我们的文档上对它进行了描述。
