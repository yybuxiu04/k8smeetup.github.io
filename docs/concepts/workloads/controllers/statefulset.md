---
assignees:
- enisoc
- erictune
- foxish
- janetkuo
- kow3ns
- smarterclayton
title: StatefulSet
redirect_from:
- "/docs/concepts/abstractions/controllers/statefulsets/"
- "/docs/concepts/abstractions/controllers/statefulsets.html"
cn-approvers:
- rootsongjc
cn-reviewers:
- shidrdn
---

{% capture overview %}



**StatefulSet 是 1.7 版本中的 beta 功能，用来代替 1.4 版本中的 PetSet 的功能。使用 PetSet 用户可以参考 1.5 版本的 [升级参考文档](/docs/tasks/manage-stateful-set/upgrade-pet-set-to-stateful-set/)，获取更多关于如何将已存在的 PetSet 升级到 SetatefulSet。**

StatefulSet 作为 Controller 为 Pod 提供唯一的标识。它可以保证部署和 scale 的顺序。

{% endcapture %}

{% capture body %}



## 使用 StatefulSet

StatefulSet 适用于有以下某个或多个需求的应用：

- 稳定，唯一的网络标志。
- 稳定，持久化存储。
- 有序，优雅地部署和 scale。
- 有序，优雅地删除和终止。
- 有序，自动的滚动升级。

在上文中，稳定是 Pod （重新）调度中持久性的代名词。 如果应用程序不需要任何稳定的标识符、有序部署、删除和 scale，则应该使用提供一组无状态副本的 controller 来部署应用程序，例如 [Deployment](/docs/concepts/workloads/controllers/deployment/) 或 [ReplicaSet](/docs/concepts/workloads/controllers/replicaset/) 可能更适合您的无状态需求。



## 限制

- StatefulSet 是 beta 资源，Kubernetes 1.5 以前版本不支持。
- 对于所有的 alpha/beta 的资源，您都可以通过在 apiserver 中设置  `--runtime-config` 选项来禁用。
- 给定 Pod 的存储必须由 [PersistentVolume Provisioner](http://releases.k8s.io/{{page.githubbranch}}/examples/persistent-volume-provisioning/README.md) 根据请求的 `storage class` 进行配置，或由管理员预先配置。
- 删除或 scale StatefulSet 将_不会_删除与 StatefulSet 相关联的 volume。 这样做是为了确保数据安全性，这通常比自动清除所有相关 StatefulSet 资源更有价值。
- StatefulSets 目前要求 [Headless Service](/docs/concepts/services-networking/service/#headless-services) 负责 Pod 的网络标识。 您有责任创建此服务。



## 组件

下面的示例中描述了 StatefulSet 中的组件。

- 一个名为 nginx 的 headless service，用于控制网络域。
- 一个名为 web 的 StatefulSet，它的 Spec 中指定在有 3 副本，每个 Pod 中运行一个 nginx 容器。
- volumeClaimTemplates 使用 PersistentVolume Provisioner 提供的 [PersistentVolumes](/docs/concepts/storage/volumes/) 作为稳定存储。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: gcr.io/google_containers/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
      annotations:
        volume.beta.kubernetes.io/storage-class: anything
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```



## Pod 标识

StatefulSet Pod 具有唯一的标识，由序数、稳定的网络标识和稳定的存储组成。 标识绑定到 Pod 上，不管它（重新）调度到哪个节点上。

### 序数

对于一个有 N 个副本的 StatefulSet，每个副本都会被指定一个整数序数，在 [0,N)之间，且唯一。



## 稳定的网络 ID

StatefulSet 中的每个 Pod 从 StatefulSet 的名称和 Pod 的序数派生其主机名。构造的主机名的模式是`$（statefulset名称)-$(序数)`。 上面的例子将创建三个名为`web-0，web-1，web-2`的 Pod。

StatefulSet 可以使用 [Headless Service](/docs/concepts/services-networking/service/#headless-services) 来控制其 Pod 的域。此服务管理的域的格式为：`$(服务名称).$(namespace).svc.cluster.local`，其中 “cluster.local” 是 [集群域](http://releases.k8s.io/{{page.githubbranch}}/cluster/addons/dns/README.md)。

在创建每个Pod时，它将获取一个匹配的 DNS 子域，采用以下形式：`$(pod 名称).$(管理服务域)`，其中管理服务由 StatefulSet 上的 `serviceName` 字段定义。 

对于 Cluster Domain,、Service name、StatefulSet name 的选择，以及它们如何影响 StatefulSet 的 Pod 的DNS名字，下面是一个示例：

| Cluster Domain | Service (ns/name) | StatefulSet (ns/name) | StatefulSet Domain              | Pod DNS                                  | Pod Hostname |      |
| -------------- | ----------------- | --------------------- | ------------------------------- | ---------------------------------------- | ------------ | ---- |
| cluster.local  | default/nginx     | default/web           | nginx.default.svc.cluster.local | web-{0..N-1}.nginx.default.svc.cluster.local | web-{0..N-1} |      |
| cluster.local  | foo/nginx         | foo/web               | nginx.foo.svc.cluster.local     | web-{0..N-1}.nginx.foo.svc.cluster.local | web-{0..N-1} |      |
| kube.local     | foo/nginx         | foo/web               | nginx.foo.svc.kube.local        | web-{0..N-1}.nginx.foo.svc.kube.local    | web-{0..N-1} |      |



注意 Cluster Domain 将被设置成  `cluster.local`  除非进行了 [其他配置](http://releases.k8s.io/{{page.githubbranch}}/cluster/addons/dns/README.md)。



### 稳定存储

Kubernetes 为每个 VolumeClaimTemplate 创建一个 [PersistentVolume](/docs/concepts/storage/volumes/)。上面的 nginx 的例子中，每个 Pod 将具有一个由 `anything` 存储类创建的 1 GB 存储的 PersistentVolume。当该 Pod （重新）调度到节点上，`volumeMounts` 将挂载与 PersistentVolume Claim 相关联的 PersistentVolume。请注意，与 PersistentVolume Claim 相关联的 PersistentVolume 在 Pod 或 StatefulSet 的时候不会被删除。这必须手动完成。



## 部署和 Scale 保证

- 对于有 N 个副本的 StatefulSet，Pod 将按照 {0..N-1} 的顺序被创建和部署。
- 当 删除 Pod 的时候，将按照逆序来终结，从{N-1..0}
- 对 Pod 执行 scale 操作之前，它所有的前任必须处于 Running 和 Ready 状态。
- 在终止 Pod 前，它所有的继任者必须处于完全关闭状态。

不应该将 StatefulSet 的 `pod.Spec.TerminationGracePeriodSeconds` 设置为 0。这样是不安全的且强烈不建议您这样做。进一步解释，请参阅 [强制删除 StatefulSet Pod](/docs/tasks/run-application/force-delete-stateful-set-pod/)。

上面的 nginx 示例创建后，3 个 Pod 将按照如下顺序创建 web-0，web-1，web-2。在 web-0 处于 [运行并就绪](/docs/user-guide/pod-states) 状态之前，web-1 将不会被部署，同样当 web-1 处于运行并就绪状态之前 web-2也不会被部署。如果在 web-1 运行并就绪后，web-2 启动之前， web-0 失败了，web-2 将不会启动，直到 web-0 成功重启并处于运行并就绪状态。

如果用户通过修补 StatefulSet 来 scale 部署的示例，以使 `replicas=1`，则 web-2 将首先被终止。 在 web-2 完全关闭和删除之前，web-1 不会被终止。 如果 web-0 在 web-2 终止并且完全关闭之后，但是在 web-1 终止之前失败，则 web-1 将不会终止，除非 web-0 正在运行并准备就绪。



### Pod 管理策略

在 Kubernetes 1.7 和以上版本，StatefulSet 允许您放开顺序保证，同时通过 `.spec.podManagementPolicy ` 字段保证标识的唯一性。

#### OrderedReady Pod 管理

StatefulSet 中默认使用的是 `OrderedReady`  pod 管理。它实现了 [如上](#deployment-and-scaling-guarantees) 所述的行为。

#### 并行 Pod 管理

`Parallel` pod 管理告诉 StatefulSet  controller 并行的启动和终止 Pod，在启动和终止其他 Pod 之前不会等待 Pod 变成运行并就绪或完全终止状态。



## 更新策略

在 kubernetes 1.7 和以上版本中，StatefulSet 的  `.spec.updateStrategy`  字段允许您配置和禁用 StatefulSet 中的对容器、label、resource request/limit、annotation 的自动滚动更新。



### 删除

`OnDelete` 更新策略实现了遗留（1.6和以前）的行为。 当 `spec.updateStrategy` 未指定时，这是默认策略。 当StatefulSet 的 `.spec.updateStrategy.type` 设置为 `OnDelete` 时，StatefulSet 控制器将不会自动更新 `StatefulSet` 中的 Pod。 用户必须手动删除 Pod 以使控制器创建新的 Pod，以反映对 StatefulSet 的 `.spec.template` 进行的修改。



### 滚动更新

`RollingUpdate`  更新策略在 StatefulSet 中实现 Pod 的自动滚动更新。 当 StatefulSet 的 `.spec.updateStrategy.type`  设置为  `RollingUpdate`  时，StatefulSet 控制器将在 StatefulSet 中删除并重新创建每个 Pod。 它将以与 Pod 终止相同的顺序进行（从最大的序数到最小的序数），每次更新一个 Pod。 在更新其前身之前，它将等待正在更新的 Pod 状态变成正在运行并就绪。



#### 分区

可以通过指定  `.spec.updateStrategy.rollingUpdate.partition`  来对  `RollingUpdate`  更新策略进行分区。如果指定了分区，则当 StatefulSet 的  `.spec.template`  更新时，具有大于或等于分区序数的所有 Pod 将被更新。具有小于分区的序数的所有 Pod 将不会被更新，即使删除它们也将被重新创建。如果 StatefulSet 的 `.spec.updateStrategy.rollingUpdate.partition` 大于其  `.spec.replicas`，则其  `.spec.template` 的更新将不会传播到 Pod。

在大多数情况下，您不需要使用分区，但如果您想要进行分阶段更新，使用金丝雀发布或执行分阶段发布，它们将非常有用。

{% endcapture %}
{% capture whatsnext %}



- 查看 [部署有状态应用](/docs/tutorials/stateful-application/basic-stateful-set) 示例

{% endcapture %}
{% include templates/concept.md %}