---
approvers:
- jszczepkowski
title: 建立高可用 Kubernetes Master
---

* TOC
{:toc}



Kubernetes 1.5 版本增加了对复制 Kubernetes master 的支持，通过在 `kube-up` 或 `kube-down` 脚本中可以为 GCE 实现该功能。
该文档描述了如何使用 kube-up/down 脚本去管理高可用（HA）master，以及 HA master 如何实现并在 GCE 中使用。



## 启动 HA-兼容集群

创建新的 HA-兼容集群，必须在 `kube-up` 脚本中设置如下标志：



* `MULTIZONE=true` - 防止 master 副本从某些 zone 中被移除，这不同于 server 的默认 zone。
如果希望在不同的 zone 中运行 master 副本，该标志是推荐的，而且也是必需的。

* `ENABLE_ETCD_QUORUM_READS=true` - 为了确保来自所有 API server 的读请求将会返回最新的数据。
如果为 true，读将直接请求到 etcd 的 Leader 副本上。
将该值设置为 true 是可选的：读将非常可靠但会比较慢。

可选地，可以指定一个 GCE zone，在这里创建第一个 master 副本。
设置下面的标志：

* `KUBE_GCE_ZONE=zone` - 第一个 master 副本将运行所在的 zone。

下面的示例命令在 GCE zone `europe-west1-b` 中建立了一个 HA-兼容集群：

```shell
$ MULTIZONE=true KUBE_GCE_ZONE=europe-west1-b  ENABLE_ETCD_QUORUM_READS=true ./cluster/kube-up.sh
```



注意，上面的命令创建了具有 1 个 master 的集群。
然而，可以使用后续的命令向集群中添加新的 master 副本。



## 新增 master 副本

创建一个 HA-兼容的集群后，可以向集群中添加 master 副本。
新增 master 副本，可以使用带有如下标志的 `kube-up` 脚本：



* `KUBE_REPLICATE_EXISTING_MASTER=true` - 为已存在 master 创建一个副本。

* `KUBE_GCE_ZONE=zone` - 指定 zone，master 副本将运行在其中。
必须像其他副本的 zone 一样，在相同的区域中。



没有必要设置 `MULTIZONE`  或 `ENABLE_ETCD_QUORUM_READS` 这两个标志，因为那些标志会从启动的 HA-兼容集群继承而来。

下面示例命令，在一个已存在的 HA-兼容集群中复制 master：

```shell
$ KUBE_GCE_ZONE=europe-west1-c KUBE_REPLICATE_EXISTING_MASTER=true ./cluster/kube-up.sh
```



## 删除 master 副本

通过使用带有如下标志的 `kube-down` 脚本，可以从 HA 集群中删除 master 副本：

* `KUBE_DELETE_NODES=false` - 限制删除 kubelet。

* `KUBE_GCE_ZONE=zone` - 对应的 zone，master 副本将被从该 zone 中删除。

* `KUBE_REPLICA_NAME=replica_name` - （可选）要删除的 master 副本的名称。
如果为空：任何来自给定 zone 的副本都将被删除。

下面的示例命令，从一个已存在的 HA 集群中删除一个 master 副本：

```shell
$ KUBE_DELETE_NODES=false KUBE_GCE_ZONE=europe-west1-c ./cluster/kube-down.sh
```



## 处理 master 副本失败

如果 HA 集群中某个 master 副本失败了，最佳的实践是从集群中删除该副本，并在同一个 zone 中新增一个副本。
下面的示例命令演示了这个流程：



1. 删除失败的副本：

```shell
$ KUBE_DELETE_NODES=false KUBE_GCE_ZONE=replica_zone KUBE_REPLICA_NAME=replica_name ./cluster/kube-down.sh
```



<ol start="2"><li>新增一个副本来替换旧的副本：</li></ol>

```shell
$ KUBE_GCE_ZONE=replica-zone KUBE_REPLICATE_EXISTING_MASTER=true ./cluster/kube-up.sh
```



## HA 集群复制 master 最佳实践

* 尝试在不同的 zone 中放置 master 副本。如果 zone 失败，则所有放置在该 zone 中的 master 都将失败。
为了使 zone 从失败中恢复，也需要在多个 zone 中放置节点（查看 [多个 zone](/docs/admin/multiple-zones/)  获取更多详情）。

* 不要使用两个 master 副本的集群。
两副本集群的一致性要求，当改变持久状态时，这两个副本都能够运行。
结果两个副本是必需的，任何一个副本失败都会导致集群变成大多数失败的状态。
因此就 HA 而言，两副本集群不如单副本集群。

* 当添加一个 master 副本时，集群状态（etcd）被拷贝到一个新的实例上。
如果集群很大，它可能需要花费很长时间去复制自己的状态。
通过迁移 etcd 数据目录，可能会加快这个操作的速度，查看 [这里](https://coreos.com/etcd/docs/latest/admin_guide.html#member-migration) 给出的说明（将来我们可能考虑增加对 etcd 数据目录迁移的支持）。



## 实现需要注意的事项

![ha-master-gce](/images/docs/ha-master-gce.png)

### 概览

每个 master 副本将在下面模式下运行下列组件：



* etcd 实例：所有实例将基于一致性算法被聚集起来；

* API server：每个 server 将与本地 etcd 通信 - 集群中所有 API server 都可用；

* controller、scheduler 和 集群 auto-scaler：将使用租约机制 - 集群中有且仅有它们中的一个被激活；

* 插件管理器：每个管理器将独立起作用，设法保持插件同步。

另外，在这些 API server 前面将有一个负载均衡器，它将内部和外部的流量路由到这些 API server。



### 负载均衡

当启动第二个 master 副本时，将创建包含 2 个副本的负载均衡器，第一个副本的 IP 地址将被提升为负载均衡器的 IP。
类似地，当倒数第二个 master 副本删除以后，负载均衡器将被移除，它的 IP 地址将被指派给最后留下的副本。
请注意，创建和删除负载均衡器是复杂的操作，它可能会花费一些时间（大约 20 分钟）去传播它们。



### Master Service 和 kubelet

在 Kubernetes Service 中设法保持一个最新的 apiserver 列表，作为替代，系统直接将全部流量指向指定外部 IP：

* 在单 master 集群，IP 指向单个 master。

* 在多 master 集群，IP 指向这些 master 前面的负载均衡器。

类似地，外部 IP 将被 kubelet 用来与 master 通信。



### Master 证书

Kubernetes 为外部公共 IP 和本地每个副本的 IP 生成 Master TLS 证书。
不会为副本的临时公共 IP 生成证书。
基于临时公共 IP 访问一个副本，必须要跳过 TLS 验证。



### 聚集 etcd

为了允许 etcd 聚集，需要在 etcd 实例之间进行通信的端口将被打开（为了在集群内部通信）。
为了使得部署安全，在 etcd 实例之间的通信需要使用 SSL 进行授权。



# 进一步阅读

[HA master 自动化部署 - 设计文档](https://git.k8s.io/community/contributors/design-proposals/ha_master.md)
