---
title: 创建一个文档 PR(Pull Request)
cn-approvers:
- chentao1596
---


{% capture overview %}


想要对 Kubernetes 文档提交贡献，需要在 [kubernetes/website](https://github.com/kubernetes/website){: target="_blank"} 仓库中创建 PR。本页展示怎么样创建一个 PR。

{% endcapture %}

{% capture prerequisites %}


1. 创建一个 [GitHub 账户](https://github.com){: target="_blank"}。


1. 签署 [Linux 基金会贡献许可协议](https://identity.linuxfoundation.org/projects/cncf){: target="_blank"}。


文档将通过 [CC BY SA 4.0](https://git.k8s.io/kubernetes.github.io/LICENSE) 协议发布。

{% endcapture %}

{% capture steps %}


## 创建一个 Kubernetes 文档仓库的 fork


1. 进入仓库 [kubernetes/website](https://github.com/kubernetes/website){: target="_blank"}。


1. 在右上角，点击 **Fork**。这将在您的 GitHub 账户下创建一个 Kubernetes 文档仓库的副本，这个副本被称为一个 *fork*。


## 进行您的修改


1. 在 GitHub 账户的 Kubernetes 文档 fork 下，创建一个用于进行贡献的新分支。


1. 在新分支中，做出更改并提交它们。如果您想 [写一个新的主题](/docs/home/contribute/write-new-topic/)，请选择最适合您的内容的 [页面类型](/docs/home/contribute/page-templates/)。


## 提交 PR 到主分支（当前版本）


如果希望变更在 Kubernetes 文档发布的版本中进行发布，您需要在 Kubernetes 文档仓库的主分支下创建一个 PR。


1. 在 GitHub 账户的新分支中，创建一个 kubernetes/website 仓库的主分支下的 PR。之后会打开一个页面，显示您 PR 的状态。


1. 点击 **Show all checks**。等待 **deploy/netlify** 检查完成。在 **deploy/netlify** 的右边点击 **Details** 将打开一个预览站点，您可以在此验证更改是否正确。


1. 接下来的时间里，检查 PR 的评审意见。如果有必要的话，通过在 fork 中对新分支进行变更，修改您的 PR。


## 提交 PR 到&lt;vnext&gt; 分支（即将发布）


如果对文档的变更应该在下个 Kubernetes 产品版本中发布，那么您需要在 Kubernetes 文档仓库的 &lt;vnext&gt; 分支中创建 PR。&lt;vnext&gt; 分支的格式为 `release-<version-number>`，例如 release-1.5。


1. 在 GitHub 账户的新分支中，创建一个 kubernetes/website 仓库的 &lt;vnext&gt; 分支下的 PR。之后会打开一个页面，显示您 PR 的状态。


1. 点击 **Show all checks**。等待 **deploy/netlify** 检查完成。在 **deploy/netlify** 的右边，点击 **Details**，将打开一个验证您的更改是否正确的预览站点。


1. 接下来的时间里，检查 PR 的评审意见。如果有必要的话，通过在 fork 中对新分支进行变更，修改您的 PR。


Kubernetes 即将发布的版本的预览站点为：[http://kubernetes-io-vnext-staging.netlify.com/](http://kubernetes-io-vnext-staging.netlify.com/)。预览站点反映了将要合入到发布分支的内容的当前状态，换句话说，也就是接下来的版本的内容。当新的PR合入的时候，它会自动更新。

{% endcapture %}

{% capture whatsnext %}

* 学习 [写一个主题](/docs/home/contribute/write-new-topic/)。
* 学习 [使用网页模板](/docs/home/contribute/page-templates/)。
* 学习 [预览您的变更](/docs/home/contribute/stage-documentation-changes/)。
{% endcapture %}

{% include templates/task.md %}
