
---
title: 日志
---

{% capture overview %}
这个页面演示在 Juju 部署的集群中日志如何工作
{% endcapture %}

{% capture prerequisites %}
这个页面假设你已经有一个可用的 Juju 部署的集群
{% endcapture %}




## 日志代理

通过 `juju debug-log` 可以查看集群中每一个部署 Juju 代理节点所有的日志。它可以帮助我们确定为什么一个节点没有部署或处于错误状态。这些代理节点的日志存放在每个节点的 `/var/lib/juju/agents` 。

更多信息查看[Juju 文档](https://jujucharms.com/docs/stable/troubleshooting-logs)



## 管理日志级别

在 Juju 中设置的日志级别是 model 级别。你可以随时调整它：

```
juju add-model k8s-development --config logging-config='<root>=DEBUG;unit=DEBUG'
```

然后

```
juju config-model k8s-production --config logging-config='<root>=ERROR;unit=ERROR'
```

另外，所有控制器上的 jujud 服务默认使用 debug 级别启动。如果想要移除这种行为可以编辑控制节点 ```/var/lib/juju/init/jujud-machine-0/exec-start.sh``` 并注释 ```--debug``` 段。 



修改后：

```
#!/usr/bin/env bash

# Set up logging.
touch '/var/log/juju/machine-0.log'
chown syslog:syslog '/var/log/juju/machine-0.log'
chmod 0600 '/var/log/juju/machine-0.log'
exec >> '/var/log/juju/machine-0.log'
exec 2>&1

# Run the script.
'/var/lib/juju/tools/machine-0/jujud' machine --data-dir '/var/lib/juju' --machine-id 0 # --debug
```



日用如下的方式重启服务：

```
sudo systemctl restart jujud-machine-0.service
```

更多日志的信息和 Juju 其它模式的设置参考[官方文档](https://jujucharms.com/docs/stable/models-config)。
