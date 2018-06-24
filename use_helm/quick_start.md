# 快速开始

本指南介绍如何快速开始使用Helm。

## 必备条件

需要以下先决条件才能成功且安全地使用Helm。

- 一个k8s集群
- 决定应用于安装的安全配置（如果有的话）
- 安装和配置集群端服务Helm和Tiller。

### 安装Kubernetes或有权访问群集

- 你必须安装Kubernetes。 对于Helm的最新版本，我们推荐最新的Kubernetes Release版本，在大多数情况下它是minor版本。
- 你还应该有一个本地配置的kubectl副本

注：1.6之前的Kubernetes版本对基于角色的访问控制（RBAC）提供有限支持或不支持。

Helm将通过阅读您的Kubernetes配置文件（通常为$HOME/.kube/config）来确定在哪里安装Tiller。 这是kubectl使用的文件。

要找出Tiller将要安装的集群，可以运行kubectl config current-context或kubectl cluster-info。

```shell
$ kubectl config current-context
my-cluster
```

## 了解您的seurity上下文

如果您在完全控制的群集上使用Helm，如minikube或专用网络中不需要共享的群集，则使用默认安装就可以（不适用于安全配置），而且它绝对是最简单的。 要安装Helm而无需额外的安全措施，请查看[安装Helm](https://docs.helm.sh/using_helm/#installing-helm\)，然后[初始化Helm](https://docs.helm.sh/using_helm/#initialize-helm-and-install-tiller)。

但是，如果您的集群暴露于更大的网络中，或者您的集群与他人共享 - 生产集群属于此类别 - 则必须采取额外步骤来确保安装的安全，以防止不小心或恶意行为者损坏集群或其数据。 要应用保护Helm以便在生产环境和其他多租户方案中使用的配置，请参阅安全Helm安装```

如果您的群集启用了基于角色的访问控制（RBAC），则可能需要在继续之前配置服务帐户和规则。

## 安装Helm

下载helm client二进制文件,你可以使用类似homebrew的工具,或在[release page](https://github.com/kubernetes/helm/releases)中下载.

### 安装helm和安装Tiler

一旦安装好了Helm，您就可以初始化本地CLI一样，一步就可以将Tiller安装到您的Kubernetes集群中：

```shell
$ helm init
```

这会将Tiller安装到您用kubectl config current-context看到的Kubernetes集群中。

> 想要安装到不同的群集？ 使用--kube-context标志。
>
> 当你想升级Tiller时，只需运行helm init --upgrade。

默认情况下，当安装Tiller时，它没有启用身份验证。 要了解有关为Tiller配置强大的TLS身份验证的更多信息，请参阅[Tiller TLS指南](https://docs.helm.sh/using_helm/#using-ssl-between-helm-and-tiller`)。

### 安装chart示例

要安装chart，您可以运行helm install命令。 Helm有几种查找和安装chart的方法，但最简单的方法是使用官方稳定chart之一。

```shell
$ helm repo update              # Make sure we get the latest list of charts
$ helm install stable/mysql
Released smiling-penguin
```

在上面的例子中，stable/mysql chart已经发布，我们新版本的名字是smile-penguin。 通过运行helm inspect stable / mysql，您可以简单了解此MySQL chart的功能。

无论何时安装chart，都会创建一个新版本。 所以一张chart可以多次安装到同一个群集中。 而且每个都可以独立管理和升级。

helm install命令是一个功能非常强大的命令，具有很多功能。 要了解更多信息，请查看使用[helm指南](https://docs.helm.sh/using_helm/#using_helm)

## 了解release

很容易看到使用Helm发布的内容：

```shell
$ helm ls
NAME             VERSION   UPDATED                   STATUS    CHART
smiling-penguin  1         Wed Sep 28 12:59:46 2016  DEPLOYED  mysql-0.1.0
```

helm list功能将显示所有已部署版本的列表。

### 卸载release

要卸载release请使用helm delete命令:

```shell
$ helm delete smiling-penguin
Removed smiling-penguin
```

这将从Kubernetes卸载smiling-penguin，但您仍然可以请求有关该版本的信息：

```shell
$ helm status smiling-penguin
Status: DELETED
...
```

由于Helm在你删除它们之后就会跟踪你的release，所以你可以审核一个集群的历史记录，甚至可以取消删除一个release（使用helm rollback）。

