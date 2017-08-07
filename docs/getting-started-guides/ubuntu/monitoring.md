---
title: 监控
---

{% capture overview %}
本文介绍了怎样将各种日志分析解决方案对接集成到已经用 Juju 部署好的 Kubernetes 集群.
{% endcapture %}
{% capture prerequisites %}
本文前提假设你已经通过 Juju 部署好了一个 Kubernetes 集群.


{% endcapture %}
{% capture steps %}
## 对接 Datadog

Datadog是一个 SaaS 平台，集成了对多种应用的支持，包括 Kubernetes 和 ETCD 。在提供商业版本的同时也支持通过如下方式免费使用：使用 Datadog 开箱即用功能部署一个完整的 Kubernetes 栈：`juju deploy canonical-kubernetes-datadog`


### 安装 Datadog

首先, 从 Juju 的 Charm Store 下载部署最新版本的 Datadog :

```
juju deploy datadog
```


使用在 Datadog 控制台找到的 api-key 配置您的 Datadog。

```
juju configure datadog api-key=XXXX
```


最后, 将 `datadog` 绑定到需要监控的所有应用上。例如：kubernetes-master, kubernetes-worker, and etcd:

```
juju add-relation datadog kubernetes-worker
juju add-relation datadog kubernetes-master
juju add-relation datadog etcd
```


## 对接 Elastic stack

Elastic 栈,正规的说是 "ELK" 栈, 是指 Elastic Search 和日志收集，监控，界面展示的套件. 使用 elastic 开箱即用功能部署一个完整的 Kubernetes 栈： `juju deploy canonical-kubernetes-elastic`

### 初始化安装 ElasticSearch

首先, 从 Juju 的 Charm store 下载部署最新版本的 ElasticSearch, Kibana, Filebeat, and Topbeat:

如下一行命令即可完成:

```
juju deploy beats-core
```


此外，如果你要定制化部署或手工安装，可使用以下命令：

```
juju deploy elasticsearch
juju deploy kibana
juju deploy filebeat
juju deploy topbeat

juju add-relation elasticsearch kibana
juju add-relation elasticsearch topbeat
juju add-relation elasticsearch filebeat
```


最后, 将 `datadog` 绑定到需要监控的所有应用上. 例如：kubernetes-master and kubernetes-worker:

```
juju add-relation kubernetes-master topbeat
juju add-relation kubernetes-master filebeat
juju add-relation kubernetes-worker topbeat
juju add-relation kubernetes-worker filebeat
```


### ElasticSearch集群存在

在 ElasticSearch 集群已经存在的情况下，你可以使用下面的方式来连接和使用它而不是重新创建一个单独的新集群。首先部署 filebeat 和 topbeat 两个组件

```
juju deploy filebeat
juju deploy topbeat
```


按照如下方式可配置 filebeat 和 topbeat 对接 ElasticSearch 集群，将 `255.255.255.255` 替换成自己配置的IP。
```
juju configure filebeat elasticsearch=255.255.255.255
juju configure topbeat elasticsearch=255.255.255.255
```


按照上面的教程将 topbeat 和 filebeat 对接到需要监控的应用上。


## 对接 Nagios

Nagios 将 Nagions 远程执行协议 (NRPE) 优化成在各个节点的 agent 来采集节点的健康检查和应用检查的详细信息。


### 初始化安装Nagios

首先, 从Juju的 Charm store 部署最新版本的Nagois和NRPE:

```
juju deploy nagios
juju deploy nrpe
```


将 Nagois 和 NRPE 对接

```
juju add-relation nagios nrpe
```


最后，将 NRPE 添加到所有需要部署的应用，例如：`kubernetes-master`, `kubernetes-worker`, `etcd`, `easyrsa`, and `kubeapi-load-balancer`。

```
juju add-relation nrpe kubernetes-master
juju add-relation nrpe kubernetes-worker
juju add-relation nrpe etcd
juju add-relation nrpe easyrsa
juju add-relation nrpe kubeapi-load-balancer
```


### Nagios存在

如果存在已经安装好的 Nagios ,可以使用 `nrpe-external-master` charm 。这样支持配置将外部安装好的相应 Nagios 对接到 NRPE 。将 `255.255.255.255` 替换成nagois实例的IP。

```
juju deploy nrpe-external-master
juju configure nrpe-external-master nagios_master=255.255.255.255
```

配置完后，如上述一样添加 nrpe-external-master 到所有需要部署的应用。
{% endcapture %}

{% include templates/task.md %}