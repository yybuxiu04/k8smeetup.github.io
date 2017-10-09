---
approvers:
- erictune
- mikedanese
- thockin
title: 给 Pod 或者容器配置安全上下文
---

{% capture overview %}

Pod 或者容器的安全上下文定义了权限和访问控制的配置。它包含了：

* 自主访问控制：访问一个对象的权限，比如说一个文件，基于
[用户 ID (UID)和组 ID (GID)](https://wiki.archlinux.org/index.php/users_and_groups).

* [安全增强 Linux (SELinux)](https://en.wikipedia.org/wiki/Security-Enhanced_Linux): 对象可以被赋予安全标签。


* 以特权或者非特权模式运行。

* [Linux Capabilities](https://linux-audit.com/linux-capabilities-hardening-linux-binaries-by-removing-setuid/): 给一个进程某些权限，但是不是所有 Root 用户的权限。

* [AppArmor](/docs/tutorials/clusters/apparmor/): 使用程序配置来限制单个进程的权限。

* [Seccomp](https://en.wikipedia.org/wiki/Seccomp): 限制进程打开文件描述符。

更多关于 Linux 的安全机制，请查看
[Linux 内核安全机制概括](https://www.linux.com/learn/overview-linux-kernel-security-features)

{% endcapture %}

{% capture prerequisites %}

{% include task-tutorial-prereqs.md %}

{% endcapture %}

{% capture steps %}

## 给 Pod 配置安全上下文环境

要给一个 Pod 配置安全设置，只需要在 Pod 的配置文件里写上 `securityContext` ，
这个 `securityContext` 是来自于
[PodSecurityContext](/docs/api-reference/{{page.version}}/#podsecuritycontext-v1-core) 
的一个对象，给 Pod 定义的安全设置适用于这个 Pod 里面的所有容器。下面是一个 Pod 的配置文件，配置了
 `securityContext` 和一个 `emptyDir` 卷:

{% include code.html language="yaml" file="security-context.yaml" ghlink="/docs/tasks/configure-pod-container/security-context.yaml" %}

在这个配置文件里， `runAsUser` 域定义的内容适用于该 Pod 里的所有容器。第一个进程以用户ID 1000运行。
然后 `fsGroup` 定义了组ID 为2000，也适用于该 Pod 里的所有容器。这个组ID 2000会绑定到该 Pod 里
的所有容器，同时还有挂载在 `/data/demo` 的卷和该卷中创建的所有文件。



创建 Pod:

```shell
kubectl create -f https://k8s.io/docs/tasks/configure-pod-container/security-context.yaml
```

验证 Pod 里的容器是否运行:

```shell
kubectl get pod security-context-demo
```

登录到运行的容器的 shell 中:

```shell
kubectl exec -it security-context-demo -- sh
```

在 shell 里，列出运行的进程:

```shell
ps aux
```



输出显示了进程以用户 1000 的ID在运行，这就是 `runAsUser` 的作用:

```shell
USER   PID %CPU %MEM    VSZ   RSS TTY   STAT START   TIME COMMAND
1000     1  0.0  0.0   4336   724 ?     Ss   18:16   0:00 /bin/sh -c node server.js
1000     5  0.2  0.6 772124 22768 ?     Sl   18:16   0:00 node server.js
...
```

在 shell 里，切换到 `/data` 目录，并列出目录内容：

```shell
cd /data
ls -l
```

输出显示了 `/data/demo` 目录的组ID 是 2000 ， 这就是 `fsGroup` 的值

```shell
drwxrwsrwx 2 root 2000 4096 Jun  6 20:08 demo
```



在您的 shell 里，切换到 `/data/demo` 目录并创建一个文件：

```shell
cd demo
echo hello > testfile
```

列出 `/data/demo` 目录里的文件：

```shell
ls -l
```

输出显示了 `testfile` 的组ID是 2000 ，这就是 `fsGroup` 的值：

```shell
-rw-r--r-- 1 1000 2000 6 Jun  6 20:08 testfile
```

退出 shell:

```shell
exit
```


## 给容器配置安全环境设置

想要给容器配置安全设置，只要在容器的配置文件里添加 `securityContext` 定义就可以。
这个 `securityContext` 其实就是一个对象来自于
[SecurityContext](/docs/api-reference/{{page.version}}/#securitycontext-v1-core) .
用户给容器设置的安全配置只对对应的容器有效，而且当有配置冲突时，它们会覆盖在 Pod 层面
上的配置。容器的设置不会影响 Pod 的卷。

下面是一个 Pod 的配置文件，包含了一个容器。Pod 和容器都有 `securityContext` 的定义配置:

{% include code.html language="yaml" file="security-context-2.yaml" ghlink="/docs/tasks/configure-pod-container/security-context-2.yaml" %}

创建 Pod:


```shell
kubectl create -f https://k8s.io/docs/tasks/configure-pod-container/security-context-2.yaml
```

验证 Pod 的容器是否运行:

```shell
kubectl get pod security-context-demo-2
```

登录到容器里的 shell:

```shell
kubectl exec -it security-context-demo-2 -- sh
```

在 shell 里，列出所有进程:

```
ps aux
```

输出显示了进程以用户ID 2000运行，这就是 `runAsUser` 的值，它覆盖了 Pod 层面设置的1000.

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
2000         1  0.0  0.0   4336   764 ?        Ss   20:36   0:00 /bin/sh -c node server.js
2000         8  0.1  0.5 772124 22604 ?        Sl   20:36   0:00 node server.js
...
```

退出您的 shell:

```shell
exit
```


## 给容器设置 Linux Capabilities

通过 [Linux capabilities](http://man7.org/linux/man-pages/man7/capabilities.7.html),
用户可以赋予进程一定的权限而避免暴露所有的root特权。想要给容器添加或者删除Linux 
Capabilities , 用户只要在容器的配置文件里的 `securityContext` 段添加 `capabilities`
字段就可以。

首先，看看没有 `capabilities` 是什么样子。下面是一个配置文件，没有添加或者删除任何
容器的 capabilities.

{% include code.html language="yaml" file="security-context-3.yaml" ghlink="/docs/tasks/configure-pod-container/security-context-3.yaml" %}



创建 Pod:

```shell
kubectl create -f https://k8s.io/docs/tasks/configure-pod-container/security-context-3.yaml
```

验证 Pod 里的容器是否运行:

```shell
kubectl get pod security-context-demo-3
```

登录到运行的容器的 shell :

```shell
kubectl exec -it security-context-demo-3 -- sh
```

在您的 shell 里，列出运行的进程：

```shell
ps aux
```


输出显示了容器的进程ID也就是PID：

```shell
USER  PID %CPU %MEM    VSZ   RSS TTY   STAT START   TIME COMMAND
root    1  0.0  0.0   4336   796 ?     Ss   18:17   0:00 /bin/sh -c node server.js
root    5  0.1  0.5 772124 22700 ?     Sl   18:17   0:00 node server.js
```

在您的 shell 里，查看进程1的 status:

```shell
cd /proc/1
cat status
```

输出显示了该进程的 capabilities 位图：

```
...
CapPrm:	00000000a80425fb
CapEff:	00000000a80425fb
...
```

记录下 capabilities 的位图并退出您的 shell :

```shell
exit
```



接下来，运行一个跟之前一样的容器，并设置额外的 capabilities.

下面是这个 Pod 的配置文件，运行一个容器。这个配置文件添加了`CAP_NET_ADMIN` 
和 `CAP_SYS_TIME` capabilities:

{% include code.html language="yaml" file="security-context-4.yaml" ghlink="/docs/tasks/configure-pod-container/security-context-4.yaml" %}

创建 Pod:

```shell
kubectl create -f https://k8s.io/docs/tasks/configure-pod-container/security-context-4.yaml
```

登录到运行的容器的 shell:

```shell
kubectl exec -it security-context-demo-4 -- sh
```


在您的 shell 里，查看进程1的 capabilities:

```shell
cd /proc/1
cat status
```

输出了进程的 capabilities 位图：

```shell
...
CapPrm:	00000000aa0435fb
CapEff:	00000000aa0435fb
...
```

对比两个容器的 capabilities:

```
00000000a80425fb
00000000aa0435fb
```


在第一个容器的 capability 位图里，第12和25位是空的。然而在第二个容器里，
第12和25却不是空的。因为第12位是 `CAP_NET_ADMIN` 而第25位是 `CAP_SYS_TIME`.
这篇文章 [capability.h](https://github.com/torvalds/linux/blob/master/include/uapi/linux/capability.h)
介绍了 capability 常量的定义。

**注意:** Linux capability 常量的格式是 `CAP_XXX`. 但是当您列出来您的容器配置里的 capabilities 时，
您需要忽略前缀 `CAP_` .比如说要添加 `CAP_SYS_TIME`,只需要在您的 capabilities 列表里添加 `SYS_TIME` 即可。
{: .note}


## 给容器配置 SELinux 标签

想要给容器分配 SELinux 标签，只需要在 Pod 或者容器的配置文件的 `securityContext`
段添加上 `seLinuxOptions` 就可以。这个 `seLinuxOptions` 字段是
[SELinuxOptions](/docs/api-reference/{{page.version}}/#selinuxoptions-v1-core)
的一个对象。下面是一个应用了 SELinux 的例子:

```yaml
...
securityContext:
  seLinuxOptions:
    level: "s0:c123,c456"
```

**注意:** 要使用 SELinux 标签, host 机器上必须加载并启用 SELinux 安全模块。{: .note}

## 参考

一个 Pod 的安全上下文环境适用于 Pod 里的容器，也适用于 Pod 所挂载的卷。参考下面内容
配置 `fsGroup` 和 `seLinuxOptions` ：

* `fsGroup`: 支持所属管理的卷，会被修改成 `fsGroup` 的组所属及可写。
参考下文
[所属权限管理向导](https://git.k8s.io/community/contributors/design-proposals/volume-ownership-management.md)


* `seLinuxOptions`: 支持 SELinux 标签的卷会被配置上`seLinuxOptions`定义的
标签权限。通常您需要配置 `level` 段。
[Multi-Category Security (MCS)](https://selinuxproject.org/page/NB_MLS)
这篇文章提供了所有容器和卷可用的标签。

**警告:** 当您给 Pod 配置 MCS 标签之后，所有拥有相同标签的 Pod 都可以访问该卷。
如果您需要内部保护，请给每个 Pod 配置不同的 MCS 标签。
{: .warning}

{% endcapture %}

{% capture whatsnext %}

* [Pod 安全上下文环境](/docs/api-reference/{{page.version}}/#podsecuritycontext-v1-core)
* [安全上下文环境](/docs/api-reference/{{page.version}}/#securitycontext-v1-core)
* [使用最新的安全机制来配置 Docker ](https://opensource.com/business/15/3/docker-security-tuning)
* [安全上下文配置向导](https://git.k8s.io/community/contributors/design-proposals/security_context.md)
* [所属权限管理向导](https://git.k8s.io/community/contributors/design-proposals/volume-ownership-management.md)
* [Pod 安全策略](/docs/concepts/policy/pod-security-policy/)


{% endcapture %}

{% include templates/task.md %}