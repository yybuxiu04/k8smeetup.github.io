---
approvers:
- caesarxuchao
- mikedanese
title: 获取一个运行容器的 Shell
---



{% capture overview %}



本页面展示如何通过 `kubectl exec` 获取一个运行容器的 Shell

{% endcapture %}


{% capture prerequisites %}

{% include task-tutorial-prereqs.md %}

{% endcapture %}


{% capture steps %}



## 获取容器的 Shell

在这个练习中，创建只有一个容器的 Pod，容器中运行 nginx 镜像。如下是 Pod  的配置文件：

{% include code.html language="yaml" file="shell-demo.yaml" ghlink="/docs/tasks/debug-application-cluster/shell-demo.yaml" %}



创建 Pod ：

```shell
kubectl create -f https://k8s.io/docs/tasks/debug-application-cluster/shell-demo.yaml
```



验证容器正在运行：

```shell
kubectl get pod shell-demo
```



获取运行容器的 Shell ：

```shell
kubectl exec -it shell-demo -- /bin/bash
```



在 shell 中列出正在运行的进程：

```shell
root@shell-demo:/# ps aux
```



在 shell 中，列出 nginx 进程：

```shell
root@shell-demo:/# ps aux | grep nginx
```



在 shell 中，尝试其它命令。例如：

```shell
root@shell-demo:/# ls /
root@shell-demo:/# cat /proc/mounts
root@shell-demo:/# cat /proc/1/maps
root@shell-demo:/# apt-get update
root@shell-demo:/# apt-get install tcpdump
root@shell-demo:/# tcpdump
root@shell-demo:/# apt-get install lsof
root@shell-demo:/# lsof
```



## 编辑 nginx 的主页

再次查看 Pod 的配置文件。 Pod 有一个 `emptyDir` 卷，挂载在容器的 `/usr/share/nginx/html`。



在 shell 中，在 `/usr/share/nginx/html` 目录下创建 `index.html` 文件：

```shell
root@shell-demo:/# echo Hello shell demo > /usr/share/nginx/html/index.html
```



在 shell 中，发送 GET 请求到 nginx 服务器：

```shell
root@shell-demo:/# apt-get update
root@shell-demo:/# apt-get install curl
root@shell-demo:/# curl localhost
```



输出的正是 `index.html` 文件中的内容：

```shell
Hello shell demo
```



当完成工作后，可以通过输入 `exit` 退出 shell 。



## 在容器中运行单个命令

在普通的命令窗口中(而不是容器中的 shell ) ，列出运行容器中的环境变量：

```shell
kubectl exec shell-demo env
```



尝试运行其它命令。例如：

```shell
kubectl exec shell-demo ps aux
kubectl exec shell-demo ls /
kubectl exec shell-demo cat /proc/1/mounts
```

{% endcapture %}

{% capture discussion %}



## 当 Pod 中有多个容器时打开一个 shell

如果 Pod 中有多个容器，使用 `--container` 或 `-c` 在 `kubectl exec` 命令中指定一个容器。例如，假设你有一个 Pod 叫 my-pod ，并且这个 Pod 中有两个容器，分别叫做 main-app 和 helper-app 。下面的命令将会打开 main-app 容器的 shell 。

```shell
kubectl exec -it my-pod --container main-app -- /bin/bash
```

{% endcapture %}


{% capture whatsnext %}

* [kubectl exec](/docs/user-guide/kubectl/v1.6/#exec)

{% endcapture %}


{% include templates/task.md %}
