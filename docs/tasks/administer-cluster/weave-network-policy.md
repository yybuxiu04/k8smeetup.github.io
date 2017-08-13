---
assignees:
- bboreham
title: 为了 NetworkPolicy 使用 Weave 网络
redirect_from:
- "/docs/getting-started-guides/network-policy/weave/"
- "/docs/getting-started-guides/network-policy/weave.html"
- "/docs/tasks/configure-pod-container/weave-network-policy/"
- "/docs/tasks/configure-pod-container/weave-network-policy.html"
---


{% capture overview %}


本页展示怎么样为了 NetworkPolicy 使用 Weave 网络

{% endcapture %}

{% capture prerequisites %}


完成 [kubeadm 入门指南](/docs/getting-started-guides/kubeadm/)中的步骤1、2和3

{% endcapture %}

{% capture steps %}


## 安装 Weave 网络插件


按照[通过插件方式集成到 Kubernetes 指南](https://www.weave.works/docs/net/latest/kube-addon/)完成安装


Kubernetes 的 Weave 网络插件配有一个[网络策略控制器](https://www.weave.works/docs/net/latest/kube-addon/#npc)，它监控所有命名空间下 NetworkPolicy 相关的注解，然后配置 iptables 规则生成允许或者阻断通信的策略

{% endcapture %}

{% capture whatsnext %}


Weave 网络插件安装完成之后，您可以通过 [NetworkPolicy 入门指南](/docs/getting-started-guides/network-policy/walkthrough)去尝试使用 Kubernetes NetworkPolicy

{% endcapture %}

{% include templates/task.md %}
