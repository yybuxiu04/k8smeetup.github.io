---
approvers:
- bgrant0607
- mikedanese
title: Install and Set Up kubectl
---

{% capture overview %}

利用 Kubernetes 命令行工具 [kubectl](/docs/user-guide/kubectl)，用户可以在 Kubernetes 集群中部署和管理应用程序。通过 kubectl，您可以探查集群的各种资源；创建、删除和更新集群中的各种组件以及浏览您新创建的集群并在其中创建样例应用程序。
{% endcapture %}

{% capture prerequisites %}

请使用与您服务器端版本相同或者更高版本的 kubectl 与您的 Kubernetes 集群连接，使用低版本的 kubectl 连接较高版本的服务器可能引发验证错误。
{% endcapture %}


下文总结了几种可以在您的环境中安装 kubectl 的方法。
{% capture steps %}

## 通过 curl 命令安装 kubectl 可执行文件

{% capture macos %}

1. 通过以下命令下载 kubectl 的最新版本：

        curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/darwin/amd64/kubectl

    
    若需要下载特定版本的 kubectl，请将上述命令中的 `$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)` 部分替换成为需要下载的 kubectl 的具体版本即可。

    
    例如，如果需要下载用于 MacOS 的 {{page.fullversion}} 版本，需要使用如下命令：

        curl -LO https://storage.googleapis.com/kubernetes-release/release/{{page.fullversion}}/bin/darwin/amd64/kubectl


2. 修改所下载的 kubectl 二进制文件为可执行模式。

    ```
    chmod +x ./kubectl
    ```


3. 将 kubectl 可执行文件放置到系统 PATH 目录下。

    ```
    sudo mv ./kubectl /usr/local/bin/kubectl
    ```
{% endcapture %}

{% capture linux %}

1. 通过以下命令下载 kubectl 的最新版本：

        curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl

    
    若需要下载特定版本的 kubectl，请将上述命令中的 `$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)`部分替换成为需要下载的 kubectl 的具体版本即可。

    
    例如，如果需要下载用于 Linux 的 {{page.fullversion}} 版本，需要使用如下命令：

        curl -LO https://storage.googleapis.com/kubernetes-release/release/{{page.fullversion}}/bin/linux/amd64/kubectl


2. 修改所下载的 kubectl 二进制文件为可执行模式。

    ```
    chmod +x ./kubectl
    ```


3. 将 kubectl 可执行文件放置到系统 PATH 目录下。

    ```
    sudo mv ./kubectl /usr/local/bin/kubectl
    ```
{% endcapture %}

{% capture win %}

从[本链接](https://storage.googleapis.com/kubernetes-release/release/{{page.fullversion}}/bin/windows/amd64/kubectl.exe)下载 kubectl 的最新版本 {{page.fullversion}}。

    
    或者，如果您已经在系统中安装了 `curl` 工具，也可以通过以下命令下载：

        curl -LO https://storage.googleapis.com/kubernetes-release/release/{{page.fullversion}}/bin/windows/amd64/kubectl.exe

    
    若需要了解最新的稳定版本（例如脚本等），请查看 https://storage.googleapis.com/kubernetes-release/release/stable.txt


2. 将 kubectl 可执行文件添加到系统 PATH 目录。

{% endcapture %}

{% assign tab_names = "macOS,Linux,Windows" | split: ',' | compact %}
{% assign tab_contents = site.emptyArray | push: macos | push: linux | push: win %}

{% include tabs.md %}


## 将 kubectl 作为 Google Cloud SDK 的一部分下载


kubectl 可以作为 Google Cloud SDK 的一部分进行安装。


1. 安装 [Google Cloud SDK](https://cloud.google.com/sdk/)。
2. 运行以下命令安装 `kubectl`：

        gcloud components install kubectl


3. 运行 `kubectl version` 命令来验证所安装的 kubectl 版本是最新的。


## 在 Ubuntu 上通过 snap 安装 kubectl


kubectl 可作为 [snap](https://snapcraft.io/) 应用程序安装使用。


如果您使用的是 Ubuntu 系统或者其他支持 [snap](https://snapcraft.io/docs/core/install) 包管理器的 Linux 发行版，可以通过以下命令安装 kubectl：

        sudo snap install kubectl --classic


2. 运行 `kubectl version` 命令来验证所安装的 kubectl 版本是最新的。


## 在 macOS 上通过 Homebrew 安装 kubectl


1. 如果您使用的是 macOS 系统并使用 [Homebrew](https://brew.sh/) 包管理器，可以通过以下命令安装 kubectl：

        brew install kubectl


2. 运行 `kubectl version` 命令来验证所安装的 kubectl 版本是最新的。


## 在 Windows 系统上通过 Chocolatey 安装 kubectl


1. 如果您使用的是 Windows 系统并使用 [Chocolatey](https://chocolatey.org) 包管理器，可以通过以下命令安装 kubectl：

        choco install kubernetes-cli


2. 运行 `kubectl version` 命令来验证所安装的 kubectl 版本是最新的。
3. 使用以下命令配置 kubectl 连接到远程 Kubernetes 集群：

        
        cd C:\users\yourusername (请进入您环境中的 %HOME% 目录)
        mkdir .kube
        cd .kube
        touch config


使用您环境中的文本编辑器编辑配置文件，例如 Notepad 等。


## 配置 kubectl


kubectl 需要一个 [kubectlconfig配置文件](/docs/concepts/cluster-administration/authenticate-across-clusters-kubeconfig/) 配置其找到并访问 Kubernetes 集群。当您使用 kube-up.sh 脚本创建 Kubernetes 集群或者成功部署 Minikube 集群后，kubectlconfig 配置文件将自动产生。请参阅[入门指南](/docs/getting-started-guides/)以了解更多创建集群相关的信息。如果您需要访问一个并非由您创建的集群，请参阅[如何共享集群的访问](/docs/tasks/administer-cluster/share-configuration/)。
默认情况下，kubectl 配置文件位于 `~/.kube/config`


## 检查 kubectl 的配置

通过获取集群状态检查 kubectl 是否被正确配置：

```shell
$ kubectl cluster-info
```

如果您看到一个 URL 被返回，那么 kubectl 已经被正确配置，能够正常访问您的 Kubernetes 集群。


如果您看到类似以下的错误信息被返回，那么 kubectl 的配置存在问题：

```shell
The connection to the server <server-name:port> was refused - did you specify the right host or port?
```


## 启用 shell 自动补全功能


kubectl 支持自动补全功能，可以节省大量输入！


自动补全脚本由 kubectl 产生，您仅需要在您的 shell 配置文件中调用即可。


以下仅提供了使用命令补全的常用示例，更多详细信息，请查阅 `kubectl completion -h` 帮助命令的输出。


### Linux 系统，使用 bash


执行 `source <(kubectl completion bash)` 命令在您目前正在运行的 shell 中开启 kubectl 自动补全功能。


可以将上述命令添加到 shell 配置文件中，这样在今后运行的 shell 中将自动开启 kubectl 自动补全：

```shell
echo "source <(kubectl completion bash)" >> ~/.bashrc
```


### macOS 系统，使用 bash

macOS 系统需要先通过 [Homebrew](https://brew.sh/) 安装 bash-completion：


```shell
## 如果您运行的是 macOS 自带的 Bash 3.2，请运行：
brew install bash-completion
## 如果您使用的是 Bash 4.1+，请运行：
brew install bash-completion@2
```


请根据 Homebrew 输出的"注意事项（caveats）"部分的内容将 bash-completion 的路径添加到本地 .bashrc 文件中。


如果您是按照[利用 Homebrew 安装 kubectl](#install-with-homebrew-on-macos) 中的步骤安装的 kubectl，那么无需其他配置，kubectl 的自动补全功能已经被启用。


如果您是手工下载并安装的 kubectl，那么您需要将 kubectl 自动补全添加到 bash-completion：

```shell
kubectl completion bash > $(brew --prefix)/etc/bash_completion.d/kubectl
```


由于 Homebrew 项目与 Kubernetes 无关，所以并不能保证 bash-completion 总能够支持 kubectl 的自动补全功能。


### 使用Oh-My-Zsh

如果使用的是 [Oh-My-Zsh](http://ohmyz.sh/)，请编辑 ~/.zshrc 文件并配置 `plugins=` 属性包含 kubectl 插件。

```shell
plugins=(git zsh-completions kubectl)
```

{% endcapture %}
{% capture whatsnext %}

[了解如何启动并对外暴露您的应用程序。](/docs/user-guide/quick-start)
{% endcapture %}
{% include templates/task.md %}
