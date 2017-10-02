---
assignees:
- eparis
- pmorie
title: 使用ConfigMap配置容器
redirect_from:
- "/docs/user-guide/configmap/index/"
- "/docs/user-guide/configmap/index.html"
cn-approvers:
- rootsongjc
cn-reviewers:
- markthink
---


{% capture overview %}


本文将向您展示如何使用ConfigMap来配置应用。ConfigMap允许您将配置文件从容器镜像中解耦，从而增强容器应用的可移植性。

{% endcapture %}

{% capture prerequisites %}

* {% include task-tutorial-prereqs.md %}

{% endcapture %}


{% capture steps %}

### 使用kubectl创建ConfigMap

可以使用`kubectl create configmap`命令，可以根据[目录](#从目录中创建ConfigMap), [文件](#使用文件创建configmap)或 [字面值](#使用字面值创建configmap)创建ConfigMap。

```shell
kubectl create configmap <map-name> <data-source>
```
\<map-name>代表ConfigMap的名字，\<data-source>代表目录、文件或者字面值。

数据源对应于ConfigMap中的键值对，

- key = 文件名或者命令行中提供的key
- value = 文件内容或命令中提供的字面值

可以使用[`kubectl describe`](/docs/user-guide/kubectl/v1.6/#describe) or [`kubectl get`](/docs/user-guide/kubectl/v1.6/#get) 获取ConfigMap的信息。前者仅展示ConfigMap的概要，后者将展示ConfigMap的全部内容。

### 利用目录创建ConfigMap

使用`kubectl create configmap`命令从同一目录下的一组文件中创建ConfigMap，例如：

```shell
kubectl create configmap game-config --from-file=docs/user-guide/configmap/kubectl
```


将`docs/user-guide/configmap/kubectl`目录的内容

```shell
ls docs/user-guide/configmap/kubectl/
game.properties
ui.properties
```

组合成下面的ConfigMap：

```shell
kubectl describe configmaps game-config
Name:           game-config
Namespace:      default
Labels:         <none>
Annotations:    <none>

Data
====
game.properties:        158 bytes
ui.properties:          83 bytes
```


`docs/user-guide/configmap/kubectl/`目录下的`game.properties`和`ui.properties`文件代表ConfigMap中的`data`部分。

```shell
kubectl get configmaps game-config-2 -o yaml
```

```yaml
apiVersion: v1
data:
  game.properties: |
    enemies=aliens
    lives=3
    enemies.cheat=true
    enemies.cheat.level=noGoodRotten
    secret.code.passphrase=UUDDLRLRBABAS
    secret.code.allowed=true
    secret.code.lives=30
  ui.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
    how.nice.to.look=fairlyNice
kind: ConfigMap
metadata:
  creationTimestamp: 2016-02-18T18:52:05Z
  name: game-config-2
  namespace: default
  resourceVersion: "516"
  selfLink: /api/v1/namespaces/default/configmaps/game-config-2
  uid: b4952dc3-d670-11e5-8cd0-68f728db1985
```

### 利用文件中创建ConfigMap

使用`kubectl create configmap`命令从单个文件或一组文件中创建ConfigMap，例如：

```shell
kubectl create configmap game-config-2 --from-file=docs/user-guide/configmap/kubectl/game.properties 
```

将产生如下的ConfigMap：

```shell
kubectl describe configmaps game-config-2
Name:           game-config-2
Namespace:      default
Labels:         <none>
Annotations:    <none>

Data
====
game.properties:        158 bytes
```

您可以多次传递`--from-file`参数使用不同的数据源来创建ConfigMap。

```shell
kubectl create configmap game-config-2 --from-file=docs/user-guide/configmap/kubectl/game.properties --from-file=docs/user-guide/configmap/kubectl/ui.properties 
```

```shell
kubectl describe configmaps game-config-2
Name:           game-config-2
Namespace:      default
Labels:         <none>
Annotations:    <none>

Data
====
game.properties:        158 bytes
ui.properties:          83 bytes
```

#### 利用文件创建ConfigMap是定义key

当您使用`--from-file`参数时，可以在ConfigMap的`data`小节内定义key替代默认的文件名：

```shell
kubectl create configmap game-config-3 --from-file=<my-key-name>=<path-to-file>
```

`<my-key-name>`是ConfigMap中的key，`<path-to-file>`是key代表的数据源文件位置。

```shell
kubectl create configmap game-config-3 --from-file=game-special-key=docs/user-guide/configmap/kubectl/game.properties

kubectl get configmaps game-config-3 -o yaml
```

```yaml
apiVersion: v1
data:
  game-special-key: |
    enemies=aliens
    lives=3
    enemies.cheat=true
    enemies.cheat.level=noGoodRotten
    secret.code.passphrase=UUDDLRLRBABAS
    secret.code.allowed=true
    secret.code.lives=30
kind: ConfigMap
metadata:
  creationTimestamp: 2016-02-18T18:54:22Z
  name: game-config-3
  namespace: default
  resourceVersion: "530"
  selfLink: /api/v1/namespaces/default/configmaps/game-config-3
  uid: 05f8da22-d671-11e5-8cd0-68f728db1985
```

### 利用字面值创建ConfigMap

使用`kubectl create configmap`时使用`--from-literal`参数在命令中定义字面值：

```shell
kubectl create configmap special-config --from-literal=special.how=very --from-literal=special.type=charm
```

可以传递多个键值对。每个对都代表ConfigMap的`data`小节中独立的一项。

```shell
kubectl get configmaps special-config -o yaml
```

```yaml
apiVersion: v1
data:
  special.how: very
  special.type: charm
kind: ConfigMap
metadata:
  creationTimestamp: 2016-02-18T19:14:38Z
  name: special-config
  namespace: default
  resourceVersion: "651"
  selfLink: /api/v1/namespaces/default/configmaps/special-config
  uid: dadce046-d673-11e5-8cd0-68f728db1985
```

{% endcapture %}

{% capture discussion %}

## 理解Config Map

ConfigMap允许您将配置文件从容器镜像中解耦，从而增强容器应用的可移植性。

ConfigMap API resource将配置数据以键值对的形式存储。这些数据可以在pod中消费或者为系统组件提供配置，例如controller。ConfigMap与[Secret](/docs/concepts/configuration/secret/)类似，但是通常只保存不包含敏感信息的字符串。用户和系统组件可以以同样的方式在ConfigMap中存储配置数据。

注意：ConfigMap只引用属性文件，而不会替换它们。可以把ConfigMap联想成Linux中的`/etc`目录和它里面的内容。例如，假如您使用ConfigMap创建了[Kubernetes Volume](/docs/concepts/storage/volumes/)，ConfigMap中的每个数据项都代表该volume中的一个文件。

ConfigMap的`data`项中包含了配置数据。如下所示，可以是很简单的——如使用 `—from-literal` 参数定义的每个属性；也可以很复杂——如使用`—from-file`参数定义的配置文件或者json对象。

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  creationTimestamp: 2016-02-18T19:14:38Z
  name: example-config
  namespace: default
data:
  # example of a simple property defined using --from-literal
  example.property.1: hello
  example.property.2: world
  # example of a complex property defined using --from-file
  example.property.file: |-
    property.1=value-1
    property.2=value-2
    property.3=value-3
```

{% endcapture %}

{% capture whatsnext %}

* 参考 [在Pod中使用ConfigMap数据](/docs/tasks/configure-pod-container/configure-pod-configmap).
* 参考实际案例[使用ConfigMap配置Redis](/docs/tutorials/configuration/configure-redis-using-configmap/).
  {% endcapture %}

{% include templates/task.md %}
