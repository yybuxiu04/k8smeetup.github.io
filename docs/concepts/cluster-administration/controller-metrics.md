---
title: Controller manager metrics
---

{% capture overview %}

控制管理器指标对深入了解Controller manager的运行健康状况具有重要意义。

{% endcapture %}

{% capture body %}

## 什么是控制管理器指标

控制管理器指标对深入了解控制管理器的运行健康状况具有重要意义。
监控指标包括：
1、常用的Go语言运行指标（例如go_routine的数量）
2、可以度量K8s集群运行健康状况的控制器特征指标（例如etcd请求延迟或云服务提供商AWS、GCE和OpenStack的API请求延迟）。


从Kubernetes 1.7版本开始，具体的云服务提供商指标包含了GCE、AWS、vSphere和OpenStack的存储运行指标。
这些指标可用于监控持久化存储卷的运行健康状况。


以GCE为例，包括以下指标：

```
cloudprovider_gce_api_request_duration_seconds { request = "instance_list"}
cloudprovider_gce_api_request_duration_seconds { request = "disk_insert"}
cloudprovider_gce_api_request_duration_seconds { request = "disk_delete"}
cloudprovider_gce_api_request_duration_seconds { request = "attach_disk"}
cloudprovider_gce_api_request_duration_seconds { request = "detach_disk"}
cloudprovider_gce_api_request_duration_seconds { request = "list_disk"}
```





## 配置


在一个K8s集群中，使用者可从正在运行控制管理器的主机上获取控制管理器指标库。地址如下所示。
`http://localhost:10252/metrics`



指标以[Prometheus格式](https://prometheus.io/docs/instrumenting/exposition_formats/)发布并且可读。


在生产环境中，您需要配置Prometheus或某些其他的指标获取工具去收集特定时间段的指标并可从时序数据库中提取。

{% endcapture %}

{% include templates/concept.md %}
