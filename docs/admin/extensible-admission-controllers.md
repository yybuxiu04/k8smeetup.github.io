---
approvers:
- smarterclayton
- lavalamp
- whitlockjc
- caesarxuchao
title: Dynamic Admission Control
---

* TOC
{:toc}
概述





[admission controllers 文档](/docs/admin/admission-controllers/)介绍了怎么使用标准的，插件化的
admission controller。然而，由于以下原因，admission controller 对于大多数用户来说还不够灵活：



* 需要把它编译到kube-apiserver 二进制文件中
* 只能在启动时进行配置



1.7 引入了2个alpha的功能，*Initializers* 和 *外部 Admission Webhooks*
来解决这些限制。这些功能可以让我们在它之外进行开发和在运行时改变它的配置



这篇文档详细介绍了怎么使用 Initializers 和 外部 Admission Webhooks.



## Initializers



### 什么是 Initializers



*Initializers* 有两层意思



* 存储在每一个对象元数据中的一系列的预初始化的任务，
 （例如，"AddMyCorporatePolicySidecar")。



* 用户自定义的 controller，用来执行那些初始化任务。任务的名称跟执行该任务的控制器是相关联的，
  我们在这里称之为 *initializer controllers*



一旦一个controller执行了分配给它的任务，他就会把他的名字从这个列表中删除。例如，发送一个PATCH请求，在pod中添加一个container，
然后把它的名字从`metadata.initializers.pending`中删除。 Initializers 有可能造成这个对象突变。



一个对象如果有一个 initializer 的非空列表，我们就认为它没有被初始化，而且不能在API中看到，除非我们用这个查询参数发了一个特殊的请求,
`?includeUninitialized=true`



### 什么时候使用 initializers?



Initializers 对管理员用来强加一些策略是非常有用的
（例如，[AlwaysPullImages](/docs/admin/admission-controllers/#alwayspullimages))



**注释** 如果你的使用场景并不总会突变这个资源，可以考虑使用外部的 admission webhooks, 它的性能会更好



### initializers 是怎么被触发的



当向一个资源发送一个POST请求的时候，所有的`initializerConfiguration`
对象(下文有介绍)都会对它进行检查。所有匹配的这些对象的
`spec.initializers[].name`会被附加在这个资源的`metadata.initializers.pending` 字段.



一个 initializer controller 应该通过查询参数`?includeUninitialized=true` 来 list 和 watch 没有被初始化的对象。
如果使用 client-go, 只需要设置[listOptions.includeUninitialized](https://github.com/kubernetes/kubernetes/blob/v1.7.0-rc.1/staging/src/k8s.io/apimachinery/pkg/apis/meta/v1/types.go#L315)
为 true.



对于观察到的没有被初始化的对象， initializer controller 应该首先检查它的名称是否和
`metadata.initializers.pending[0]` 匹配。如果匹配他就会被分配的任务然后在列表中删除它的名字



### 启用 initializers alpha 功能



*Initializers* 目前是alpha的功能，默认是被关闭的。要打开它，你需要：



* 当你启动 `kube-apiserver` 的时候，在`--admission-control` 这个参数加上"Initializers"。
  如果你有多个`kube-apiserver`副本， 你需要在所有的上面加上这个参数。



* 启动`kube-apiserver`的时候，在`--runtime-config` 参数加上`admissionregistration.k8s.io/v1alpha1`
  来打开动态 admission controller 注册 API 的功能。再一次，如果你有多个`kube-apiserver`，
  你需要在所有的上面加上这个参数



### 部署 initializer controller



你应该通过[deployment
     API](/docs/api-reference/{{page.version}}/#deployment-v1beta1-apps)
来部署 initializer controller



### 运行时配置 initializer controller



你可以通过创建 `initializerConfiguration` 资源来配置开启哪个 initializer controller 和初始化那种资源



首先你需要部署一个 initializer controller 并且确保在创建 `initializerConfiguration` 之前工作正常。
否则，任何新创建的资源都会处在 uninitialized 的状态


下面是一个 `initializerConfiguration` 的例子：

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: InitializerConfiguration
metadata:
  name: example-config
initializers:
  # the name needs to be fully qualified, i.e., containing at least two "."
  - name: podimage.example.com
    rules:
      # apiGroups, apiVersion, resources all support wildcard "*".
      # "*" cannot be mixed with non-wildcard.
      - apiGroups:
          - ""
        apiVersions:
          - v1
        resources:
          - pods
```


你在创建了 `initializerConfiguration` 之后，系统会花费几妙中时间响应这个新的配置, 然后，
`"podimage.example.com"` 回被添加到新创建Pod的 `metadata.initializers.pending` 字段。
你应该已经提前有一个工作正常的 initializer controller 来处理这些
`metadata.initializers.pending[0].name="podimage.example.com"`
的Pod。 否则这些Pod会一直出在 uninitialized 状态



确保每一条 `rule` 内所有的 `<apiGroup, apiVersions, resources>` 展开的元组是有效的，如果不是的话
把他们放在不同的 `rule` 下面。


## 外部 Admission Webhooks



### 什么是外部 admission webhooks?



外部 admission webhooks 是一些 HTTP 的 callback, 用来接受 admission 的请求，
并且做一些相应的处理。 外部 admission webhook 做什么是由你来决定的, 但是有一个
[接口](https://github.com/kubernetes/kubernetes/blob/v1.7.0-rc.1/pkg/apis/admission/v1alpha1/types.go)
必须遵从, 来响应是否允许这个 admission 请求。



跟 initializers 或者插件化的 admission controllers 不一样的是, 外部的
admission webhooks 不允许以任何方式突变这个 admission request



因为 admission 是一个很高级别的安全相关的操作, 外部的 admission webhooks 必须支持TLS。



### 什么时候使用 admission webhooks？



使用外部 admission webhook 的一个简单的例子就是对 kubernetes 的资源进行语义验证。假如说你的架构需要所有的
Pod 资源有一些共同的 label, 而且如果这些 Pod 没有满足需要的话你不想对其持久化。 你可以写一个你自己的外部
的 admission webhook 来做这些验证并且对其做相应的响应。



### 外部 admission webhooks 是怎么触发的？



无论什么时候接受到请求， `GenericAdmissionWebhook` admission 插件会从 `externalAdmissionHookConfiguration`
对象 (下文有介绍)拿到相应的外部 admission webhooks 的一个列表然后并行的调用它们。如果**所有**的外部
admission webhooks 允许了这个 admission 请求，那么请求就会继续。 如果**任何**一个
外部 admission webhooks 拒绝了这个 admission 请求， 那么这个请求就会被拒绝，拒绝请求的原因就是第一个拒绝
请求的 admission webhook 的拒绝原因。这意味着如果如果多个外部 admission webhook 拒绝了请求，只有第一个会
被返回给用户。 如果当调用外部 admission webhook 发生错误, 这个 admission webhook 将被忽略掉， 不会用来
允许/拒绝这个 admission 请求。



**注释** admission 链的执行顺序仅依赖 `kube-apiserver` 的 `--admission-control` 选项配置的顺序



### 启用外部 admission webhooks



*外部 admission webhooks* 是一个 alpha 的功能，默认似乎被关闭的，要打开它你需要：



* 当启动 apiserver 的时候在 `--admission-control` 选项中包含 "GenericAdmissionWebhook"。
  如果你有多个 `kube-apiserver`， 你需要在所有的上面加上这个参数。



* 在启动`kube-apiserver`的时候，在`--runtime-config` 选项加上`admissionregistration.k8s.io/v1alpha1`
  来打开动态 admission controller 注册 API 的功能。再一次，如果你有多个`kube-apiserver`，
  你需要在所有的上面加上这个参数。



### 实现 webhook admission controller



这[caesarxuchao/example-webhook-admission-controller](https://github.com/caesarxuchao/example-webhook-admission-controller)
是一个 webhook admission controller 的例子.



webhook admission controller, 更准取得说是 GenericAdmissionWebhook admission controller
和 apiserver 之间的通信，需要TLS加密。 你需要生更一个CA证书然后用它来对你 webhook admission controller
使用的服务器证书进行签名。 这个CA证书需要在动态注册API的时候通过
 `externaladmissionhookconfigurations.clientConfig.caBundle` 提供给 apiserver



对于每一个 apiserver 接收到的请求，GenericAdmissionWebhook admission controller 会发送一个
[admissionReview](https://github.com/kubernetes/kubernetes/blob/v1.7.0-rc.1/pkg/apis/admission/v1alpha1/types.go#L27)
到对应的 webhook admission controller。 webhook admission controller 会从
`admissionReview.spec` 收集 `object`, `oldobject`, and `userInfo` 等信息
并且以 `admissionReview` 的格式发送一个响应。 在 `status` 字段填充 admission 的结果。



部署 webhook admission controller



[caesarxuchao/example-webhook-admission-controller deployment](https://github.com/caesarxuchao/example-webhook-admission-controller/tree/master/deployment)
是一个部署的例子



我们应该通过 [deployment API](/docs/api-reference/{{page.version}}/#deployment-v1beta1-apps)
来部署 webhook controller
并且需要创建一个 [service](/docs/api-reference/{{page.version}}/#service-v1-core) 作为部署的
前端



运行时配置 webhook admission controller



你可以通过创建 externaladmissionhookconfigurations 来配置启动哪些 webhook admission controllers
和作用于哪些目标对象



首先你需要部署一个 webhook admission controller 并且确保在创建 `externaladmissionhookconfigurations`
之前工作正常。否则，处理结果将会有条件的接受或者拒绝，这取决于你的 webhook 配置为 `fail open` 还是 `fail closed`



下面是一个 `externaladmissionhookconfiguration` 的例子

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ExternalAdmissionHookConfiguration
metadata:
  name: example-config
externalAdmissionHooks:
- name: pod-image.k8s.io
  rules:
  - apiGroups:
    - ""
    apiVersions:
    - v1
    operations:
    - CREATE
    resources:
    - pods
  failurePolicy: Ignore
  clientConfig:
    caBundle: <pem encoded ca cert that signs the server cert used by the webhook>
    service:
      name: <name of the front-end service>
      namespace: <namespace of the front-end service>
```



对于 apiserver 接受到的请求，如果这个请求满足任何一个 `externalAdmissionHook` `rules` 中
的 `rule`，`GenericAdmissionWebhook` admission controller 会发送一个 `admissionReview`
请求到 `externalAdmissionHook` 来获取 admission 结果。



这里的 `rule` 和 `externalAdmissionHook` 中的 `rule` 非常相似，但有两点不同：



* 新添加了 `operations` 字段，来描述这个 webhook 会做什么样的操作



* 添加 `resources` 字段支持 `resource/subresource` 方式的 subresources



确保每一条 `rule` 内所有的`<apiGroup, apiVersions, resources>` 展开的元组是有效的，如果不是的话
把他们放在不同的 `rule` 下面。



你也可以指定 `failurePolicy`. 在1.7的时候， 系统支持 `Ignore` 和 `Fail` 策略，当与webhook
admission controller 通信发生错误的时候，`GenericAdmissionWebhook` 可以根据配置的策略来确定、
是允许还是拒绝请求。



你在创建了 `externalAdmissionHookConfiguration` 之后，系统会花费几妙中时间响应这个新的配置
