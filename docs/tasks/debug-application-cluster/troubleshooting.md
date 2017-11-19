---
approvers:
- brendandburns
- davidopp
title: 排除故障
---





有时候会出现问题。本指南旨在使其正确。它有两个部分：

   * [排除应用程序的故障](/docs/tasks/debug-application-cluster/debug-application/) —— 适用于向 Kubernetes 部署代码，并想知道为什么它不工作的用户。
   * [排除集群故障](/docs/tasks/debug-application-cluster/debug-cluster/) —— 适用于集群管理员和定位 Kubernetes 集群问题的用户。

你还应该检查你正在使用的[发行版](https://github.com/kubernetes/kubernetes/releases)的已知问题。



## 获取帮助

如果你的问题没有得到上述指南的解答，你可以通过多种方式从 Kubernetes 团队获得帮助。



### 问题

本站已经按照问题的范围进行结构化。[Concepts](/docs/concepts/) 解释 Kubernetes 架构以及每个组件的工作原理；[Setup](/docs/setup/) 提供了开始使用的实用指导。[Tasks](/docs/tasks/) 展示了如何完成常用任务；[Tutorials](/docs/tutorials/) 是对现实世界、特定行业或端到端开发场景的更全面的演练。[Reference](/docs/reference/) 提供了有关[Kubernetes API](/docs/api-reference/{{page.version}}/) 和命令行界面（CLIs）的详细文档，例如 [`kubectl`](/docs/user-guide/kubectl-overview/)。



我们也有很多 FAQ 页面：

   * [User FAQ](https://github.com/kubernetes/kubernetes/wiki/User-FAQ)
   * [Debugging FAQ](https://github.com/kubernetes/kubernetes/wiki/Debugging-FAQ)
   * [Services FAQ](https://github.com/kubernetes/kubernetes/wiki/Services-FAQ)



你也能在 Stack Overflow 找到相关主题：

   * [Kubernetes](http://stackoverflow.com/questions/tagged/kubernetes)
   * [Google Container Engine - GKE](http://stackoverflow.com/questions/tagged/google-container-engine)



## 帮帮我！我的问题没有涵盖！我现在需要帮忙！

### Stack Overflow

社区其他人可能已经问过类似的问题，或者可能会帮助你解决问题。Kubernetes 团队也会监控 [Kubernetes 标签的帖子](http://stackoverflow.com/questions/tagged/kubernetes)。如果不存在任何有用的问题，请[提一个新的问题](http://stackoverflow.com/questions/ask?tags=kubernetes)！



### Slack

Kubernetes 团队在 Slack 中有 `#kubernetes-users` 频道。你可以在这里参加与Kubernetes团队的讨论。Slack 需要注册，但是 Kubernetes 是公开的，任何人都可以在这里[注册](http://slack.kubernetes.io)。随时来问问题。



注册后，可以浏览各种感兴趣的频道。例如，Kubernetes 新人可以加入 `#kubernetes-novice` 频道。开发者可以加入 `#kubernetes-dev` 频道。



也有很多国家或地区的频道，随时加入这些频道获取本地化支持和信息：

- France: `#fr-users`, `#fr-events`
- Germany: `#de-users`, `#de-events`
- Japan: `#jp-users`, `#jp-events`



### 邮件列表

Kubernetes / Google Container Engine 邮件列表是 [kubernetes-users@googlegroups.com](https://groups.google.com/forum/#!forum/kubernetes-users)。



### Bug 和功能请求

如果你有什么看起来像一个 bug ，或者你想要一个功能。请使用[Github 问题跟踪系统](https://github.com/kubernetes/kubernetes/issues)。



在你提出问题之前，请搜索现有问题，看看你的问题是否已经存在。

如果提交一个 bug, 请提供有关如何重现问题的详细信息，例如：

* Kubernetes 版本：`kubectl version`
* 云提供商，OS 发行版本，网络配置和 Docker 版本
* 重现问题的步骤
