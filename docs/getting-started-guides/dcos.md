---
approvers:
- karlkfi
title: DCOS
---

{% assign for_k8s_version="1.6" %}{% include feature-state-deprecated.md %}


这个指南将引导你在 [数据中心操作系统(DCOS)](https://mesosphere.com/product/) 上用 [DCOS CLI](https://github.com/mesosphere/dcos-cli) 安装 [Kubernetes-Mesos](https://github.com/mesosphere/kubernetes-mesos) 和用 [DCOS Kubectl plugin](https://github.com/mesosphere/dcos-kubectl) 操作 Kubernetes。



## 关于 在 DCOS 上的 Kubernetes

DCOS 是管理计算机集群硬件和软件资源并为分布式应用提供通用服务的系统软件。在其他服务中，它提供 [Apache Mesos](http://mesos.apache.org/) 作为他的集群内核，[Marathon](https://mesosphere.github.io/marathon/) 作为他的初始系统。与 DCOS CLI，Mesos 框架类似，可以用单独的命令安装 [Kubernetes-Mesos](https://github.com/mesosphere/kubernetes-mesos)。

DCOS CLI 的另一个特性是它允许插件，像 [DCOS Kubectl plugin](https://github.com/mesosphere/dcos-kubectl)。这允许通过简单的版本编译 Kubectl ，不用手工下载或安装。

更多关于在 DCOS 上安装 Kubernetes 好处的信息可以在 [Kubernetes-Mesos 文档](https://releases.k8s.io/{{page.githubbranch}}/contrib/mesos/README.md) 找到。

更多关于 Kubernetes DCOS 包的详情，看 [Kubernetes-Mesos 工程](https://github.com/mesosphere/kubernetes-mesos)。

由于 Kubernetes-Mesos 还是 alpha 版，所以最好先熟悉限制或修改 DCOS 上 Kubernetes 的行为的[当前已知问题](https://releases.k8s.io/{{page.githubbranch}}/contrib/mesos/docs/issues.md)。

如果你对下列完整的步骤有问题，请[对 kubernetes-mesos 工程提出问题](https://github.com/mesosphere/kubernetes-mesos/issues)



## 资源

探索后面的资源获取更多关于 Kubernetes，在 Mesos/DCOS 上的 Kubernetes，和 DCOS 自身。

- [DCOS 文档](https://docs.mesosphere.com/)
- [管理 DCOS 服务](https://docs.mesosphere.com/services/kubernetes/)
- [Kubernetes 例子](https://github.com/kubernetes/kubernetes/tree/{{page.githubbranch}}/examples/)
- [在 Mesos 上的 Kubernetes 文档](https://github.com/kubernetes-incubator/kube-mesos-framework/blob/master/README.md)
- [在 Mesos 上的 Kubernetes 发行纪录](https://github.com/mesosphere/kubernetes-mesos/releases)
- [在 DCOS 上的 Kubernetes 源码包](https://github.com/mesosphere/kubernetes-mesos)



## 先决条件

- 一个运行的 [DCOS 集群](https://mesosphere.com/product/)
  - [DCOS 社区版](https://docs.mesosphere.com/1.7/archived-dcos-enterprise-edition/installing-enterprise-edition-1-6/cloud/) 在 [AWS](https://mesosphere.com/amazon/) 上当前是可用的。
  - [DCOS 企业版](https://mesosphere.com/product/) 可以部署在虚拟或实体机上。联系 sales@mesosphere.com 获取更多信息和建立协议。
- [DCOS CLI](https://docs.mesosphere.com/install/cli/) 已在本地安装。



## 安装

1. 配置并验证 [Mesosphere Multiverse](https://github.com/mesosphere/multiverse) 作为软件包源码库

    ```shell
$ dcos config prepend package.sources https://github.com/mesosphere/multiverse/archive/version-1.x.zip
    $ dcos package update --validate
    ```    
2. 安装 etcd

    默认的，Kubernetes DCOS 包启动一个单节点的 etcd。为了避免在 Kubernetes 组建容器损坏情况下状态丢失，在 DCOS 上安装一个 HA [etcd-mesos](https://github.com/mesosphere/etcd-mesos) 集群。

    ```shell
$ dcos package install etcd
    ```    
3. 校验 etcd 已经安装并且健康

    etcd 集群安装很快。在继续下一步之前，校验 `/etcd` 健康。

    ```shell
$ dcos marathon app list
    ID           MEM  CPUS  TASKS  HEALTH  DEPLOYMENT  CONTAINER  CMD
    /etcd        128  0.2    1/1    1/1       ---        DOCKER   None
    ```    
4. 创建 Kubernetes 安装配置

    配置 Kubernetes 使用 DCOS 上已经安装的 HA etcd。

    ```shell
$ cat >/tmp/options.json <<EOF
    {
      "kubernetes": {
        "etcd-mesos-framework-name": "etcd"
      }
    }
    EOF
    ```
5. 安装 Kubernetes

    ```shell
$ dcos package install --options=/tmp/options.json kubernetes
    ```    
6. 校验 Kubernetes 已经安装并且健康

    Kubernetes 集群安装很快。继续下一步之前，校验 `/kubernetes` 是健康的。

    ```shell
$ dcos marathon app list
    ID           MEM  CPUS  TASKS  HEALTH  DEPLOYMENT  CONTAINER  CMD
    /etcd        128  0.2    1/1    1/1       ---        DOCKER   None
    /kubernetes  768   1     1/1    1/1       ---        DOCKER   None
    ```    
7. 校验 Kube-DNS & Kube-UI 已经部署，运行，并且就位

    ```shell
$ dcos kubectl get pods --namespace=kube-system
    NAME                READY     STATUS    RESTARTS   AGE
    kube-dns-v8-tjxk9   4/4       Running   0          1m
    kube-ui-v2-tjq7b    1/1       Running   0          1m
    ```    
Name 和 AGE 也许不一样。


现在 Kubernetes 已经安装到 DCOS 上，你也许像查看 [Kubernetes 例子](https://github.com/kubernetes/kubernetes/tree/{{page.githubbranch}}/examples/README.md) 或 [Kubetnetes 用户指南](/docs/user-guide/)。



## 卸载

1. 停止并删除每个 namespace 所有 replication controller 和 pod

    卸载 Kubernetes 前，销毁所有 pod 和 replication controller。卸载进程将尝试自己做这些，但是默认它很快超时并且也许会使你的集群处于污染状态。

    ```shell
$ dcos kubectl delete rc,pods --all --namespace=default
    $ dcos kubectl delete rc,pods --all --namespace=kube-system
    ``` 
2. 验证所有 pod 已经被删除

    ```shell
$ dcos kubectl get pods --all-namespaces
    ``` 
3. 卸载 Kubernetes

    ```shell
$ dcos package uninstall kubernetes
    ``` 



## 支持等级

IaaS Provider        | Config. Mgmt | OS     | Networking  | Docs                                              | Conforms | Support Level
-------------------- | ------------ | ------ | ----------  | ---------------------------------------------     | ---------| ----------------------------
DCOS                 | Marathon   | CoreOS/Alpine | custom | [docs](/docs/getting-started-guides/dcos)                                   |          | Community ([Kubernetes-Mesos Authors](https://github.com/mesosphere/kubernetes-mesos/blob/master/AUTHORS.md))

所有解决方案支持等级信息，查看[解决方案](/docs/getting-started-guides/#table-of-solutions)表
