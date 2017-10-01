---
title: 从私有仓库拉取镜像
---

{% capture overview %}

这个教程指导如何创建一个 Pod 使用 Secret 从私有镜像或者镜像源拉取镜像。

{% endcapture %}

{% capture prerequisites %}

* {% include task-tutorial-prereqs.md %}

* 要完成这个使用，您需要有一个
[Docker ID](https://docs.docker.com/docker-id/) 和密码.

{% endcapture %}

{% capture steps %}

## 登陆到Docker

    docker login

出现提示符时，输入你的 Docker 用户名和密码。

登陆过程会创建或者更新 `config.json` 文件，这个文件包含了验证口令。

查看 `config.json` 文件内容:

    cat ~/.docker/config.json

输出会有类似这样的一段内容：

    {
        "auths": {
            "https://index.docker.io/v1/": {
                "auth": "c3R...zE2"
            }
        }
    }

**注意:** 如果你使用了 Docker 的证书存储功能，您可能不会看到 'auth' 的字眼，而是 `credsStore` ， 而它的值就是存储的名字。
{: .note}

## 创建一个 Secret 来保存您的验证口令

创建一个名为 `regsecret` 的 Secret :

    kubectl create secret docker-registry regsecret --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>

在这里:

* `<your-registry-server>` 是你的私有仓库的FQDN.
* `<your-name>` 是你的 Docker 用户名.
* `<your-pword>` 是你的 Docker  密码.
* `<your-email>` 是你的 Docker 邮箱.

## 理解你的 Secret

想要知道你刚刚创建的 Secret 是什么东西，可以看看 YAML 格式的 Secret :

    kubectl get secret regsecret --output=yaml

输出类似这样:

    apiVersion: v1
    data:
      .dockercfg: eyJodHRwczovL2luZGV4L ... J0QUl6RTIifX0=
    kind: Secret
    metadata:
      ...
      name: regsecret
      ...
    type: kubernetes.io/dockercfg

`.dockercfg` 的值是一个经过 base64 加密的数据。

把这串 base64 加密的数据复制到一个名为 `secret64` 的文件里.


**重要**: 确保你的 `secret64` 的文件内容没有任何换行。

想知道 `.dockercfg` 的内容是什么意思，只要将 secret 数据转换成可读格式即可：

    base64 -d secret64

输出类似这样:

    {"yourprivateregistry.com":{"username":"janedoe","password":"xxxxxxxxxxx","email":"jdoe@example.com","auth":"c3R...zE2"}}

注意到 secret 数据其实包含了你的 `config.json` 文件里的验证口令。

## 创建一个 Pod 来使用你的 Secret

下面是一个需要读取你的 Secret 数据的 Pod 的配置文件：

{% include code.html language="yaml" file="private-reg-pod.yaml" ghlink="/docs/tasks/configure-pod-container/private-reg-pod.yaml" %}

把 `private-reg-pod.yaml` 的内容复制到你自己名为 `my-private-reg-pod.yaml` 的文件。
在你的文件里，将 `<your-private-image>` 覆盖为私有仓库里的镜像地址。

Docker Hub 的私有镜像例子：

    janedoe/jdoe-private:v1

要从私有镜像拉取镜像， Kubernetes 需要有验证口令。这里 `imagePullSecrets` 告诉 Kubernets 应该从名为
`regsecret` 的 Secret 里获取验证口令。

创建一个 Pod 来使用你的 Secret, 并验证是否运行成功：

    kubectl create -f my-private-reg-pod.yaml
    kubectl get pod private-reg


{% endcapture %}

{% capture whatsnext %}

* 了解更多关于 [Secrets](/docs/concepts/configuration/secret/).
* 了解更多关于
[如何使用私有仓库](/docs/concepts/containers/images/#using-a-private-registry).
* 查看 [kubectl 创建 secret docker 仓库](/docs/user-guide/kubectl/v1.6/#-em-secret-docker-registry-em-).
* 查看 [Secret](/docs/api-reference/{{page.version}}/#secret-v1-core)
* 查看 [PodSpec](/docs/api-reference/{{page.version}}/#podspec-v1-core) 的 `imagePullSecrets` 段
.

{% endcapture %}

{% include templates/task.md %}