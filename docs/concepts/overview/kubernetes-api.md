---
approvers:
- chenopis
cn-approvers:
- brucehex
cn-reviewers:
- zjj2wry
- xiaosuiba
title: Kubernetes API
---



总体 API 规范在 [API 规范文档](https://git.k8s.io/community/contributors/devel/api-conventions.md)中描述。



API 端点，资源类型和样例在[API 参考](/docs/reference)中描述。



对 API 的远程调用的讨论可参考[访问文档](/docs/admin/accessing-the-api)。


Kubernetes API 也是系统的声明式配置模式的基础。可以使用[Kubectl](/docs/user-guide/kubectl)命令行工具来创建、更新、删除和获取 API 对象。



Kubernetes 还存储了和 API 资源相关的序列化状态(目前存储在 [etcd](https://coreos.com/docs/distributed-configuration/getting-started-with-etcd/))。


Kubernetes 本身被分解成多个组件，通过其 API 进行交互。

## API 变更


根据我们的经验，任何成功的系统需要随着新用例的出现或现有的变化而发展和变化。因此，我们预计 Kubernetes API 将不断变化和发展。但是，我们打算在很长一段时间内不会破坏与现有客户的兼容性。一般来说，新的 API 资源和新的资源领域通常可以被频繁添加。消除资源或领域将需要遵循弃用过程，消除功能的精确弃用政策是 TBD, 但一旦达到我们的 1.0 里程碑，就会有一个具体的政策。

根据什么构成兼容的更改和如何更改 API 由[API 更改文档](https://git.k8s.io/community/contributors/devel/api_changes.md)进行详细说明。

## OpenAPI 和 Swagger 定义


使用 [Swagger v1.2](http://swagger.io/) 和 [OpenAPI](https://www.openapis.org/) 记录完整的 API 详细信息。Kubernetes apiserver(又名 "Master")公开了一个 API，可用于检索位于 `/swaggerapi` 的 Swagger v1.2 Kubernetes API 规范。您还可以通过传递 `--enable-swagger-ui=true` 标志给 apiserver 以启用 UI 浏览 `/swagger-ui` 上的 API 文档。



从 Kubernetes 1.4 开始， OpenAPI 规范也可以在[`/swagger.json`](https://git.k8s.io/kubernetes/api/openapi-spec/swagger.json)中找到。当我们从 Swagger v1.2 转换到 OpenAPI（又名 Swagger v2.0）时，一些诸如 kubectl 和 swagger-ui 的工具仍在使用 v1.2 规范。OpenAPI 规范从 Kubernetes 1.5 的 Beta 版开始启用。


Kubernetes 为主要用于集群内通信的 API 实现了另一种基于 Protobuf 的序列化格式，在[设计方案(design proposal)](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/protobuf.md)中记录，每个模式的 IDL 文件位于定义 API 对象的 Go 包中。

## API 版本


为了更容易地消除字段或重组资源表示，Kubernetes 支持多个 API 版本，每个都有不同的 API 路径，例如 `/api/v1` 或 `/apis/extensions/v1beta1`。



我们在选择在 API 级别而不是在资源或字段级别进行版本定位，以确保 API 提供清晰、一致的系统资源和行为视图，并且能够控制对即将停用的和实验性 API 的访问。JSON 和 Protobuf 序列化模式遵循相同的模式更改准则 - 以下所有描述都涵盖了这两种格式。


请注意，API 版本控制和软件版本控制仅间接相关。[API 和 发布版本提案(release versioning proposal)](https://git.k8s.io/community/contributors/design-proposals/versioning.md)
描述了 API 版本控制和软件版本控制。

不同的 API 版本意味着不同程度的稳定性和支持。描述每个级别的标准在[API 变更文档](https://git.k8s.io/community/contributors/devel/api_changes.md#alpha-beta-and-stable-versions)中有更详细的说明。总结如下：


- Alpha 级别:
  - 版本名称包含 `alpha` (例如 `v1alpha1`).
  - 实验性支持(May be buggy)  启用该功能可能会出现错误，默认禁用。
  - 功能的支持可能随时丢弃，恕不另行通知。
  - API 可能在以后的软件版本中以不兼容的方式更改，恕不另行通知。
  - 建议仅在短期测试集群中使用，因为漏洞的风险增加和缺乏长期的支持。
- 测试等级:
  - 版本名称包含 `beta` (例如 `v2beta3`).
  - 代码测试良好，启用该功能被认为是安全的，默认启用。
  - 整体功能的支持不会被删除，尽管细节可能会改变。
  - 对象的模式或语义可能会在后续 beta 版本或稳定版本中以不兼容的方式发生变化。发生这种情况时，我们将提供迁移到下一个版本的说明。这可能需要删除、编辑或重新创建 API 对象。编辑过程可能需要一些思考，这可能需要依赖该功能的应用程序一些停机时间。
  - 建议仅用于非关键业务用途，因为后续版本中可能会出现不兼容的更改。如果您有可以独立升级的多个集群，您可以放宽此限制。
  - **请尝试我们的测试版功能并给予反馈！一旦退出 beta 版本，我们可能不会做更多的更改。**
- 稳定等级:
  - 版本名称为 `vX` 其中 `X` 为整数。
  - 功能的稳定版本将出在许多后续版本的发行软件中。

## API 组

为了更容易地扩展 Kubernetes API，我们实现了 [*API 组*](https://git.k8s.io/community/contributors/design-proposals/api-group.md)。API 组在 REST 路径和序列化对象的 `apiVersion` 字段中指定。


目前有几个 API 组正在使用中：

1. "核心" (通常称为"遗产", 由于没有明确的组名称) 组， 位于 REST 的路径 `/api/v1` 并没有被指定为 `apiVersion` 字段中的一部分。例如 `apiVersion: v1`。
1. 命名组在 REST 路径 `/apis/$GROUP_NAME/$VERSION`,并使用`apiVersion:$GROUP_NAME/$VERSION`(例如 `apiVersion: batch/v1`)。支持的 API 组的完整列表可以在 [Kubernetes API 参考](/docs/reference/)中看到。


使用[自定义资源](/docs/concepts/api-extension/custom-resources/)扩展 API 有两个支持的路径：

1. [CustomResourceDefinition](/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/) 适用于具有非常基本 CRUD 需求的用户。
1. 即将推出: 需要完整 Kubernetes API 语义的用户可以实现自己的 apiserver 并使用 [aggregator](https://git.k8s.io/community/contributors/design-proposals/aggregated-api-servers.md)以使客户端进行无缝连接。



## 启用 API 组

某些资源或 API 组已默认启用。可以在 apiserver 上通过设置  `--runtime-config` 来启用或禁用它们。 `--runtime-config` 接受逗号分隔的值。例如: 要禁用 batch/v1, 请设置 `--runtime-config=batch/v1=false`, 要启用 `batch/v2alpha1`, 请设置 `--runtime-config=batch/v2alpha1`。
该标志接受一组描述 `apiserver` 运行时配置的键值对（`key=value`），以逗号分隔。


重要信息: 启用或禁用组或资源需要重新启动 apiserver 和 controller-manager 以获取 `--runtime-config` 的变更。

## 启用组中的资源


默认情况下，DaemonSets、Deployments、Horizo​​ntalPodAutoscalers、Ingress、Job 和 ReplicaSets 都被启用。
可以通过在 API 服务器上设置 `--runtime-config` 来启用其他扩展资源。 `--runtime-config`接受逗号分隔的值。 例如: 禁用部署和入口，设置`--runtime-config=extensions/v1beta1/deployments=false,extensions/v1beta1/ingress=false`。
