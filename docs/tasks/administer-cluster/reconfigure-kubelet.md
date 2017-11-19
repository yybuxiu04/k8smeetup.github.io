---
approvers:
- mtaufen
- dawnchen
cn-approvers:
- xiaosuiba
title: 重新配置活动集群中节点的 Kubelet
---



{% capture overview %}
{% include feature-state-alpha.md %}

1.8 版本的 Kubernetes 中包含了一个新的 [动态 Kubelet 配置（Dynamic Kubelet Configuration）](https://github.com/kubernetes/features/issues/281) alpha 功能。该功能可以让您通过顶级 Kubernetes 概念（first-class Kubernetes concepts）改变活动集群中的 Kubelet 的配置。特别的，该功能还允许通过 ConfigMaps 配置单个节点的 Kubelet。


**警告：** 所有的 Kubelet 配置参数都可以动态修改，但并不是所有的修改都安全。此功能适用于对配置更改会如何影响行为有深刻理解的系统专家。当前没有一个文件列出“对修改安全”的配置域，但我们计划在此功能从 alpha 阶段毕业时添加这个文件。
{% endcapture %}

{% capture prerequisites %}

- 有一个活动的 Kubernetes 集群，Master 和 Node 均运行 1.8 或更高版本。Kubernetes 打开了 `DynamicKubeletConfig` 特性开关（feature gate）并设置 Kubelet 的 `--dynamic-config-dir` 参数为节点上的一个可写目录。必须设置此参数以启用动态 Kubelet 配置。
- kubectl 命令行工具也必须为 1.8 或更高版本，并能和集群正常通信。
  {% endcapture %}

{% capture steps %}


## 重新配置集群活动节点上的 Kubelet


### 基本工作流程概览


  配置活动集群中的 kubelet 的基本工作流程如下：

1. 编写一个包含 Kubelet 配置的 YAML 或 JSON 格式的配置文件。
2. 用 ConfigMap 包装该文件并保存至 Kubernetes 控制平面。
3. 更新 Kubelet 对应的节点对象以使用此 ConfigMap。


每个 Kubelet 都会监控各自节点对象上的配置引用。当这个引用发生改变时，Kubelet 将下载新的配置并退出。为了使这个功能正常工作，您必须运行一个进程管理器（例如 systemd），以便在 Kubelet 退出时自动对其进行重启。当 Kubelet 重新启动后，它将开始使用新的配置。


新的配置将完全覆盖旧配置；新配置中没有指定的字段将使用典型的默认值。某些 CLI 配置项没有与之关联的配置字段，也不会被新的配置所影响。这些字段在 KubeletFlags 结构体中进行了定义，请参考 [这个文件](https://github.com/kubernetes/kubernetes/blob/master/cmd/kubelet/app/options/options.go)。


节点 Kubelet 配置的状态通过节点状态中的 `ConfigOK` 状态进行报告。一旦使用新的 ConfigMap 对节点进行了升级，您就可以通过观察这个状态确认节点是否正在使用预期的配置。您可以在文章末尾找到描述所有可能状态的表格。


本文描述了通过 `kubectl edit` 编辑节点的方法。除此之外还有其它修改节点 spec 的方式，例如 `kubectl patch` 可用于通过脚本配置的工作流程。


本文仅仅介绍了单个节点使用各种 ConfigMap 的场景。这也适用于多个节点使用相同 ConfigMap 的场景。


### Node Authorizer 变通方案


Node Authorizer 不会在意哪个 ConfigMap 分配给了哪个节点。如果您目前正在使用 Node authorizer，节点的 Kubelet 将不能自动获得下载各自 ConfigMap 的权限。


本文使用的临时变通方案是为每个 ConfigMap 手动创建 RBAC Role 和 RoleBinding。在动态 Kubelet 配置功能从 alpha 阶段毕业之前，我们将对 Node Authorizer 进行扩展，所以您永远不必在生产环境中这样做。 


### 生成包含当前配置的文件


动态 Kubelet 配置功能允许您对整个配置对象进行覆盖，而不只是单个字段的替换。这种更为简单的模式使得追踪配置值的来源及调试问题变得更加容易。但是，随之而来的问题是您必须知道现存的配置，以确保只对需要的配置字段进行了修改。


将来，Kubelet 将使用磁盘上的文件进行引导（请参考 [通过配置文件设置 Kubelet 参数](/docs/tasks/administer-cluster/kubelet-config-file)），您可以简单的通过编辑这个文件的副本（作为最佳实践，应该对其进行版本控制）来创建第一个 Kubelet ConfigMap。然而当前 Kubelet 仍然使用命令行参数启动。幸运的是，您可以通过一个脏技巧（dirty trick）生成包含节点当前配置的文件。这个技巧需要通过 kubectl proxy 访问 Kubelet 服务的 `configz` endpoint。在目前的实现中，这个 endpoint 仅用于辅助调试，这就是为何这是一个脏技巧。改进这个 endpoint 的工作正在进行当中，未来这个操作将变得不是那么 “脏”。这个技巧还要求您的机器安装了 `jq` 命令行工具，用于对 endpoint 回复的 JSON 响应进行拆包和编辑。


  执行下列步骤以生成配置文件：

1. 选择一个需要重新配置的节点。我们将使用 NODE_NAME 指代这个节点的名字。
2. 通过 `kubectl proxy --port=8001 &` 命令在后台启动 kubectl proxy。
3. 运行下列命令以从 configz endpoint 下载并解包配置。

```
$ export NODE_NAME=the-name-of-the-node-you-are-reconfiguring
$ curl -sSL http://localhost:8001/api/v1/proxy/nodes/${NODE_NAME}/configz | jq '.kubeletconfig|.kind="KubeletConfiguration"|.apiVersion="kubeletconfig/v1alpha1"' > kubelet_configz_${NODE_NAME}
```


请注意，我们需要手动为下载的对象添加 `kind` 和 `apiVersion`，因为  configz endpoint 没有报告这些值。这是计划在将来进行修复的限制之一。


### 编辑配置文件


使用文本编辑器改变前面步骤生成的 `kubelet_configz_${NODE_NAME}` 文件中的某个参数。例如，QPS 参数 `eventRecordQPS` 是一个不错的选择。


### 推送配置文件至控制平面


使用下列命令推送编辑后的配置文件至控制平面：

```
$ kubectl -n kube-system create configmap my-node-config --from-file=kubelet=kubelet_configz_${NODE_NAME} --append-hash -o yaml
```


您应该看到类似的响应：

```
apiVersion: v1
data:
  kubelet: |
    {...}
kind: ConfigMap
metadata:
  creationTimestamp: 2017-09-14T20:23:33Z
  name: my-node-config-gkt4c2m4b2
  namespace: kube-system
  resourceVersion: "119980"
  selfLink: /api/v1/namespaces/kube-system/configmaps/my-node-config-gkt4c2m4b2
  uid: 946d785e-998a-11e7-a8dd-42010a800006
```


请注意，ConfigMap 的 `kubelet` 键下必须存在配置数据。


我们在 `kube-system` namespace 中创建这个 ConfigMap，这种选择是恰当的，因为这个 ConfigMap 用于配置 Kubernetes 的系统组件之一 —— Kubelet。


`--append-hash` 选项将 ConfigMap 内容的短校验和添加到其名称后面。这将为   edit->push 工作流程带来便利，因为它将自动的、确切的为新的 ConfigMap 生成新的名称。


我们使用 -o yaml 输出格式，以便创建的时候输出 name、namespace 和 uid。在后续步骤中我们将需要这些值。我们将使用 CONFIG_MAP_NAME 指代 name，使用 CONFIG_MAP_UID 指代 uid。


### 授权节点读取新的 ConfigMap


现在您已经创建了一个新的 ConfigMap，您需要授权节点对其进行读取。首先使用下列命令为新的 ConfigMap 创建一个 Role：

```
$ export CONFIG_MAP_NAME=name-from-previous-output
$ kubectl -n kube-system create role ${CONFIG_MAP_NAME}-reader --verb=get --resource=configmap --resource-name=${CONFIG_MAP_NAME}
```


接下来，创建一个 RoleBinding 以将节点和新的 Role 进行关联：

```
$ kubectl -n kube-system create rolebinding ${CONFIG_MAP_NAME}-reader --role=${CONFIG_MAP_NAME}-reader --user=system:node:${NODE_NAME}
```


一旦将来 Node Authorizer 可以自动进行该工作时，您就可以跳过这个步骤。


### 设置节点使用新的配置


使用下列命令编辑节点的引用，使其指向新的 ConfigMap。

```
kubectl edit node ${NODE_NAME}
```


使用编辑器，在 `spec` 下添加如下 YAML：

```
configSource:
    configMapRef:
        name: CONFIG_MAP_NAME
        namespace: kube-system
        uid: CONFIG_MAP_UID
```


请确保指定了 `name`、`namespace` 和 `uid` 三个配置。


### 观察节点是否开始使用新的配置


通过 `kubectl get node ${NODE_NAME} -o yaml` 命令获取节点信息，并在 `status.conditions` 下查找 `ConfigOK` 状态。如果 Kubelet 已经开始使用新的配置，您应该能看到类似 `Using current (UID: CONFIG_MAP_UID)` 的信息。


方便起见，您可以使用下面的命令（使用 `jq`）筛选 `ConfigOK` 状态。

```
$ kubectl get no ${NODE_NAME} -o json | jq '.status.conditions|map(select(.type=="ConfigOK"))'
[
  {
    "lastHeartbeatTime": "2017-09-20T18:08:29Z",
    "lastTransitionTime": "2017-09-20T18:08:17Z",
    "message": "using current (UID: \"2ebc8d1a-9e2a-11e7-a8dd-42010a800006\")",
    "reason": "passing all checks",
    "status": "True",
    "type": "ConfigOK"
  }
]
```


如果出现问题，您可能会看到几种不同错误状态之一，详情请参阅下面的 ConfigOK Conditions 表格。当这个情况发生时，请检查 Kubelet 日志以获取详细信息。


### 再次编辑配置文件


如果想要再次改变配置，我们可以简单的重复上述步骤。请尝试编辑 `kubelet` 文件，为前面修改过的参数设置一个新的值。


### 推送重新编辑过的配置到控制平面


使用下列命令生成新的 ConfigMap 来推送新配置到控制平面：

```
$ kubectl create configmap my-node-config --namespace=kube-system --from-file=kubelet=kubelet_configz_${NODE_NAME} --append-hash -o yaml
```


由于我们改变了内容，这个新的 ConfigMap 会获得一个新的名字。我们将使用 NEW_CONFIG_MAP_NAME 指代新的名字，使用 NEW_CONFIG_MAP_UID 指代新的 uid。


### 授权节点读取新的 ConfigMap


现在您已经创建了一个新的 ConfigMap，您需要授权节点来对其进行读取。首先，使用下列命令为新的 ConfigMap 创建一个 Role：

```
$ export NEW_CONFIG_MAP_NAME=name-from-previous-output
$ kubectl -n kube-system create role ${NEW_CONFIG_MAP_NAME}-reader --verb=get --resource=configmap --resource-name=${NEW_CONFIG_MAP_NAME}
```


接下来，创建一个 RoleBinding 以对节点和新的 Role 进行关联：

```
$ kubectl -n kube-system create rolebinding ${NEW_CONFIG_MAP_NAME}-reader --role=${NEW_CONFIG_MAP_NAME}-reader --user=system:node:${NODE_NAME}
```


一旦将来 Node Authorizer 可以自动进行该工作时，您就可以跳过这个步骤。


### 配置节点使用新的配置


再次使用 `kubectl edit node ${NODE_NAME}` 命令编辑节点的 `spec.configSource`。新的 `spec.configSource` 应该和下面类似，请替换 `name` 和 `uid` 为实际值：

```
configSource:
    configMapRef:
        name: NEW_CONFIG_MAP_NAME
        namespace: kube-system
        uid: NEW_CONFIG_MAP_UID
```


### 观察 Kubelet 是否正在使用新的配置


再一次使用 `kubectl get node ${NODE_NAME} -o yaml` 获取节点信息并在 `status.conditions` 字段下查看 `ConfigOK` 状态。当 Kubelet 开始使用新的配置时，您应该可以看见类似 `Using current (UID: NEW_CONFIG_MAP_UID)` 的信息。


### 取消对节点读取旧 ConfigMap 的授权


一旦知道了节点已经在使用新的配置并且确信没有引起任何问题时，最好取消对节点读取旧 ConfigMap 的授权。执行下列命令以删除 RoleBinding 和 Role：

```
$ kubectl -n kube-system delete rolebinding ${CONFIG_MAP_NAME}-reader
$ kubectl -n kube-system delete role ${CONFIG_MAP_NAME}-reader
```


请注意，这样做并不一定能防止节点读取旧配置，因为它可能会在一段不定的时间内在本地缓存旧 ConfigMap。


你还可以选择删除旧 ConfigMap：

```
$ kubectl -n kube-system delete configmap ${CONFIG_MAP_NAME}
```


一旦将来 Node Authorizer 可以自动进行该工作时，您就可以跳过这个步骤。


### 重置节点以使用本地默认配置


最后，如果希望重置节点并使用启动时的配置，只需要简单的使用 `kubectl edit node ${NODE_NAME}` 命令编辑节点并去除 `spec.configSource` 子字段。


### 观察节点是否正在使用本地默认配置


在去除了这个子字段之后，您应该最终能够观察到 ConfigOK 状态的信息变为 `using current (default)` 或者 `using current (init)`（这取决于节点是如何配置的）。


### 取消对节点读取旧 ConfigMap 的授权


一旦知道了节点已经在使用新的配置并且确信没有引起任何问题时，最好取消对节点读取旧 ConfigMap 的授权。执行下列命令以删除 RoleBinding 和 Role：

```
$ kubectl -n kube-system delete rolebinding ${NEW_CONFIG_MAP_NAME}-reader
$ kubectl -n kube-system delete role ${NEW_CONFIG_MAP_NAME}-reader
```


请注意，这样做并不一定能防止节点读取旧配置，因为它可能会在一段不定的时间内在本地缓存旧 ConfigMap。


你还可以选择删除旧 ConfigMap：

```
$ kubectl -n kube-system delete configmap ${NEW_CONFIG_MAP_NAME}
```


一旦将来 Node Authorizer 可以自动进行该工作时，您就可以跳过这个步骤。

{% endcapture %}

{% capture discussion %}

## Kubectl Patch 示例
如上所述，存在多种改变节点配置源的方式。这是一个使用 `kubectl patch` 命令的示例：

```
kubectl patch node ${NODE_NAME} -p "{\"spec\":{\"configSource\":{\"configMapRef\":{\"name\":\"${CONFIG_MAP_NAME}\",\"namespace\":\"kube-system\",\"uid\":\"${CONFIG_MAP_UID}\"}}}}"
```


## 理解 ConfigOK 状态


下面的表格列举了您在启用了动态 Kubelet 配置的集群中可能遇到的几种 `ConfigOK` 节点状态。如果您观察到状态为 `status=False`，请检查 Kubelet 日志并搜索消息或原因的文本，以获取更多关于错误的详细信息。

<table>


<table align="left">
<tr>
    <th>Possible Messages</th>
    <th>Possible Reasons</th>
    <th>Status</th>
</tr>
<tr>
    <td><p>using current (default)</p></td>
    <td><p>current is set to the local default, and no init config was provided</p></td>
    <td><p>True</p></td>
</tr>
<tr>
    <td><p>using current (init)</p></td>
    <td><p>current is set to the local default, and an init config was provided</p></td>
    <td><p>True</p></td>
</tr>
<tr>
    <td><p>using current (UID: CURRENT_CONFIG_MAP_UID)</p></td>
    <td><p>passing all checks</p></td>
    <td><p>True</p></td>
</tr>
<tr>
    <td><p>using last-known-good (default)</p></td>
    <td>
        <ul>
            <li>failed to load current (UID: CURRENT_CONFIG_MAP_UID)</li>
            <li>failed to parse current (UID: CURRENT_CONFIG_MAP_UID)</li>
            <li>failed to validate current (UID: CURRENT_CONFIG_MAP_UID)</li>
        </ul>
    </td>
    <td><p>False</p></td>
</tr>
<tr>
    <td><p>using last-known-good (init)</p></td>
    <td>
        <ul>
            <li>failed to load current (UID: CURRENT_CONFIG_MAP_UID)</li>
            <li>failed to parse current (UID: CURRENT_CONFIG_MAP_UID)</li>
            <li>failed to validate current (UID: CURRENT_CONFIG_MAP_UID)</li>
        </ul>
    </td>
    <td><p>False</p></td>
</tr>
<tr>
    <td><p>using last-known-good (UID: LAST_KNOWN_GOOD_CONFIG_MAP_UID)</p></td>
    <td>
        <ul>
            <li>failed to load current (UID: CURRENT_CONFIG_MAP_UID)</li>
            <li>failed to parse current (UID: CURRENT_CONFIG_MAP_UID)</li>
            <li>failed to validate current (UID: CURRENT_CONFIG_MAP_UID)</li>
        </ul>
    </td>
    <td><p>False</p></td>
</tr>
<tr>
    <td>
        <p>
            The reasons in the next column could potentially appear for any of
            the above messages.
        </p>
        <p>
            This condition indicates that the Kubelet is having trouble
            reconciling `spec.configSource`, and thus no change to the in-use
            configuration has occurred.
        </p>
        <p>
            The "failed to sync" reasons are specific to the failure that
            occurred, and the next column does not necessarily contain all
            possible failure reasons.
        </p>
    </td>
    <td>
    <p>failed to sync, reason:</p>
    <ul>
        <li>failed to read Node from informer object cache</li>
        <li>failed to reset to local (default or init) config</li>
        <li>invalid NodeConfigSource, exactly one subfield must be non-nil, but all were nil</li>
        <li>invalid ObjectReference, all of UID, Name, and Namespace must be specified</li>
        <li>invalid ObjectReference, UID SOME_UID does not match UID of downloaded ConfigMap SOME_OTHER_UID</li>
        <li>failed to determine whether object with UID SOME_UID was already checkpointed</li>
        <li>failed to download ConfigMap with name SOME_NAME from namespace SOME_NAMESPACE</li>
        <li>failed to save config checkpoint for object with UID SOME_UID</li>
        <li>failed to set current config checkpoint to default</li>
        <li>failed to set current config checkpoint to object with UID SOME_UID</li>
    </ul>
    </td>
    <td><p>False</p></td>
</tr>
</table>
{% endcapture %}


{% include templates/task.md %}
