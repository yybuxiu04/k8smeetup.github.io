---
approvers:
- bprashanth
- erictune
- foxish
- janetkuo
- smarterclayton
title: 删除StatefulSet
---

{% capture overview %}


本任务将展示如何删除一个StatefulSet。

{% endcapture %}

{% capture prerequisites %}


* 本任务假定您有一个应用程序已经以StatefulSet的方式部署在Kubernetes集群中运行。

{% endcapture %}

{% capture steps %}


## 删除一个StatefulSet


您可以像删除Kubernetes中其他资源一样通过`kubectl delete`命令删除一个StatefulSet。待删除的StatefulSet可以通过文件或者名字指定：

```shell
kubectl delete -f <file.yaml>
```

```shell
kubectl delete statefulsets <statefulset-name>
```


在StatefulSet被删除后，您可能还需要单独删除与之关联的Headless Service。

```shell
kubectl delete service <service-name>
```


通过kubectl删除一个StatefulSet将缩减Pod数量至0，从而删除所有属于这个StatefulSet的Pod。如果仅仅想删除StatefulSet对象而保留Pod，请在删除时使用`--cascade=false`。

```shell
kubectl delete -f <file.yaml> --cascade=false
```


通过在使用`kubectl delete`命令时设置`--cascade = false`，即使在StatefulSet对象本身被删除之后，StatefulSet所管理的Pod也将会被保留下来。如果这些Pod包含标签`app=myapp`，您可以通过以下命令删除它：

```shell
kubectl delete pods -l app=myapp
```


### 持久化卷（Persistent Volumes）


删除StatefulSet中的Pod将不会删除其所关联的持久化卷，这将确保您在彻底删除卷设备之前可以拷贝和备份其中的数据。
在Pod经历了[terminating状态](/docs/user-guide/pods/index#termination-of-pods)之后删除PVC可能会导致PV的删除，这一过程需要根据StorageClass和PV Reclaim Policy的配置来决定。PVC被删除后，将不能保证用户仍能访问PV。


**注意：删除PVC时要小心，否则可能导致数据丢失。**


### StatefulSet的完整删除


如果要简单地删除StatefulSet中包含的所有内容，包括所关联的Pod，您可以尝试执行以下一系列命令：

```shell{% raw %}
grace=$(kubectl get pods <stateful-set-pod> --template '{{.spec.terminationGracePeriodSeconds}}')
kubectl delete statefulset -l app=myapp
sleep $grace
kubectl delete pvc -l app=myapp
{% endraw %}
```


在上面的例子中，所有的Pod包含标签`app=myapp`，您可以将其替换成为您环境中实际使用的Pod标签。


### 强制删除StatefulSet中的Pod


如果您发现StatefulSet中的某些Pod长期停留在'Terminating'或者'Unknown'状态，则可能需要手动干预才能强制删除API server中的Pod。这一强制操作可能包含一些潜在风险，详细信息请参阅[删除StatefulSet中的Pod](/docs/tasks/manage-stateful-set/delete-pods/)。

{% endcapture %}

{% capture whatsnext %}

Learn more about [force deleting StatefulSet Pods](/docs/tasks/run-application/force-delete-stateful-set-pod/).
更多有关强制删除StatefulSet中的Pod的信息，请参阅[强制删除StatefulSet中的Pod](/docs/tasks/run-application/force-delete-stateful-set-pod/)。

{% endcapture %}

{% include templates/task.md %}
