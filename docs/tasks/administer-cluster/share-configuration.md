---
approvers:
- mikedanese
- thockin
title: 使用 kubeconfig 共享集群访问
---


通过复制 `kubectl` 客户端配置 ([kubeconfig](/docs/concepts/cluster-administration/authenticate-across-clusters-kubeconfig/))，客户端访问运行中的 Kubernetes 集群可以共享。该配置位于 `$HOME/.kube/config`，由 `cluster/kube-up.sh` 生成。下面是共享 `kubeconfig` 的步骤。


**1. 创建集群**

```shell
$ cluster/kube-up.sh
```


**2. 复制 `kubeconfig` 到新主机**

```shell
$ scp $HOME/.kube/config user@remotehost:/path/to/.kube/config
```


**3. 在新主机上，让复制的 `config` 在使用 `kubectl` 时生效**


* 选择 A：复制到默认的位置

```shell
$ mv /path/to/.kube/config $HOME/.kube/config
```


* 选择 B：复制到工作目录（运行 kubectl 的当前位置）

```shell
$ mv /path/to/.kube/config $PWD
```


* 选项 C：通过环境变量 `KUBECONFIG` 或者命令行标志 `kubeconfig` 传递给 `kubectl`

```shell
# via environment variable
$ export KUBECONFIG=/path/to/.kube/config

# via commandline flag
$ kubectl ... --kubeconfig=/path/to/.kube/config
```


## 手动创建 `kubeconfig`


`kubeconfig` 由 `kube-up` 生成，但是，您也可以使用下面命令生成自己想要的配置（可以使用任何想要的子集）

```shell
# create kubeconfig entry
$ kubectl config set-cluster $CLUSTER_NICK \
    --server=https://1.1.1.1 \
    --certificate-authority=/path/to/apiserver/ca_file \
    --embed-certs=true \
    # Or if tls not needed, replace --certificate-authority and --embed-certs with
    --insecure-skip-tls-verify=true \
    --kubeconfig=/path/to/standalone/.kube/config

# create user entry
$ kubectl config set-credentials $USER_NICK \
    # bearer token credentials, generated on kube master
    --token=$token \
    # use either username|password or token, not both
    --username=$username \
    --password=$password \
    --client-certificate=/path/to/crt_file \
    --client-key=/path/to/key_file \
    --embed-certs=true \
    --kubeconfig=/path/to/standalone/.kube/config

# create context entry
$ kubectl config set-context $CONTEXT_NAME \
    --cluster=$CLUSTER_NICK \
    --user=$USER_NICK \
    --kubeconfig=/path/to/standalone/.kube/config
```


注：


* 生成独立的 `kubeconfig` 时，标识 `--embed-certs` 是必选的，这样才能远程访问主机上的集群。

* `--kubeconfig` 既是加载配置的首选文件，也是保存配置的文件。如果您是第一次运行上面命令，那么 `--kubeconfig` 文件的内容将会被忽略。

```shell
$ export KUBECONFIG=/path/to/standalone/.kube/config
```


* 上面提到的 ca_file，key_file 和 cert_file 都是集群创建时在 master 上产生的文件，可以在文件夹 `/srv/kubernetes` 下面找到。持有的 token 或者 基本认证也在 master 上产生。


如果您想了解更多关于 `kubeconfig` 的详细信息，请查看[使用 kubeconfig 配置集群访问认证](/docs/concepts/cluster-administration/authenticate-across-clusters-kubeconfig/)，或者运行帮助命令 `kubectl config -h`。


## 合并 `kubeconfig` 的示例


`kubectl` 会按顺序加载和合并来自下面位置的配置


1. `--kubeconfig=/path/to/.kube/config` 命令行标志

2. `KUBECONFIG=/path/to/.kube/config` 环境变量

3. `$HOME/.kube/config` 配置文件


如果在 host1 上创建集群 A 和 B，在 host2 上创建集群 C 和 D，那么，您可以通过运行下面命令，在两个主机上访问所有的四个集群

```shell
# on host2, copy host1's default kubeconfig, and merge it from env
$ scp host1:/path/to/home1/.kube/config /path/to/other/.kube/config

$ export KUBECONFIG=/path/to/other/.kube/config

# on host1, copy host2's default kubeconfig and merge it from env
$ scp host2:/path/to/home2/.kube/config /path/to/other/.kube/config

$ export KUBECONFIG=/path/to/other/.kube/config
```


如果希望查看更加详细的例子以及关于 `kubeconfig` 加载/合并规则的描述，请您参考 [kubeconfig-file](/docs/user-guide/kubeconfig-file)。
