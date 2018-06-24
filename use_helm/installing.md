# 安装Helm

Helm有两个部分：Helm客户端（helm）和Helm服务器（Tiller）。 本指南介绍如何安装客户端，然后继续介绍安装服务器的两种方式。

重要提示：如果您有责任确保您的群集是受控制的环境，尤其是在共享资源时，强烈建议使用安全配置安装Tiller。 有关指导，请参阅[安装Helm安装](https://docs.helm.sh/using_helm/#securing-your-helm-installation)。

## 安装helm客户端

Helm客户端可以从源代码安装，也可以从预先构建的二进制版本安装。

#### 用二进制发现版安装

Helm的每个[版本](https://github.com/kubernetes/helm/releases)都为各种操作系统提供二进制版本。 这些二进制版本可以手动下载和安装。

1. 下载你[想要的版本](https://github.com/kubernetes/helm/releases)
2. 解压它 (`tar -zxvf helm-v2.0.0-linux-amd64.tgz`)
3. 在解压后的目录中找到helm二进制文件，并将其移到它想要的目的地（mv linux-amd64/helm/usr/local/bin/helm）

然后，你就可以运行客户端了：helm help。

#### 用 Homebrew (macOS)

Kubernetes社区的成员为Homebrew贡献了Helm方案。 这个方案通常是最新的。

```shell
brew install kubernetes-helm
```

（注意：emacs-helm也有一个方案，这是一个不同的项目。）

//TBD

## 安装Tiller

Helm的服务器部分Tiller通常运行在您的Kubernetes集群内部。 但是对于开发，它也可以在本地运行，并配置为与远程Kubernetes群集通信。

### 在集群内轻松安装

将tiller安装到集群中最简单的方法就是运行helm init。 这将验证helm的本地环境设置是否正确（并在必要时进行设置）。 然后它会连接到kubectl默认连接的任何集群（kubectl config view）。 一旦连接，它将把tiller安装到kube-system namespace中。

在helm init之后，你应该可以运行kubectl get pods --namespace kube-system并且看到Tiller正在运行。

你可以更精确的安装helm init ...

- 使用--canary-image标志安装canary版本
- 使用--tiller-image安装特定镜像（版本）
- 使用--kube-context安装到特定群集
- 使用--tiller-namespace安装到特定的namespace中

一旦安装了Tiller，运行helm version应该会显示客户端和服务器版本。 （如果只显示客户端版本，helm还不能连接到服务器，请使用kubectl来查看是否有任何tiller pod正在运行。）

除非设置了--tiller-namespace或TILLER_NAMESPACE，否则Helm将在kube-system namespace中查找Tiller。

### 安装tiller canary版本 

金丝雀图像是从主分支建立的。 他们可能不稳定，但他们为您提供测试最新功能的机会。

安装Canary镜像最简单的方法是使用helm init和--canary-image标志：

```shell
$ helm init --canary-image
```

这将使用最近建立的容器镜像。 您可以使用kubectl从kube-system namespace中删除Tiller部署，随时卸载Tiller。

## 本地运行Tiller

对于开发而言，在本地处理Tiller有时更容易，并将其配置为连接到远程Kubernetes群集。

上面介绍了建立Tiller的过程。

一旦Tiller已经建成，只需启动它：

```shell
$ bin/tiller
Tiller running on :44134
```

当Tiller在本地运行时，它将尝试连接到由kubectl配置的Kubernetes群集。 （运行kubectl config view查看是哪个群集。）

您必须告诉helm连接到这个新的本地Tiller主机，而不是连接到一个群集中。 有两种方法可以做到这一点。 首先是在命令行中指定--host选项。 第二个是设置$HELM_HOST环境变量。

```shell
$ export HELM_HOST=localhost:44134
$ helm version # Should connect to localhost.
Client: &version.Version{SemVer:"v2.0.0-alpha.4", GitCommit:"db...", GitTreeState:"dirty"}
Server: &version.Version{SemVer:"v2.0.0-alpha.4", GitCommit:"a5...", GitTreeState:"dirty"}
```

重要的是，即使在本地运行，Tiller也会将发布配置存储在Kubernetes内的ConfigMaps中。

## 升级TILLER

从Helm 2.2.0开始，可以使用helm init --upgrade升级Tiller。

对于旧版本的Helm可以手动升级，或使用kubectl来修改Tiller镜像：

```shell
$ export TILLER_TAG=v2.0.0-beta.1        # Or whatever version you want
$ kubectl --namespace=kube-system set image deployments/tiller-deploy tiller=gcr.io/kubernetes-helm/tiller:$TILLER_TAG
deployment "tiller-deploy" image updated
```

设置TILLER_TAG = canary将获得master分支的最新快照。

## 删除或重新安装TILLER

由于Tiller将其数据存储在Kubernetes ConfigMaps中，因此您可以安全地删除并重新安装Tiller，而无需担心丢失任何数据。 推荐删除Tiller的方式是使用kubectl delete deploy tiller-deploy --namespace kube-system，或者更简洁的helm reset。

然后可以从客户端重新安装Tiller：

```shell
$ helm init
```

### 高级用法

helm init在安装之前为修改Tiller的deployment声明提供了额外的标志。

#### 使用`--node-selectors`

--node-selectors标志允许我们指定调度Tiller pod所需的节点标签。

下面的例子将在nodeSelector属性下创建指定的标签。

```shell
helm init --node-selectors "beta.kubernetes.io/os"="linux"
```

已安装的deloyment声明将包含我们的节点选择器标签。

```yaml
...
spec:
  template:
    spec:
      nodeSelector:
        beta.kubernetes.io/os: linux
...
```

#### 使用`--override`

--override允许您指定Tiller的deployment声明的属性。 与helm中其他地方使用的--set命令不同，helm init --override操作最终声明的指定属性（没有“values”文件）。 因此，您可以为deployment声明中的任何有效属性指定任何有效值。

#####  Override注解

在下面的示例中，我们使用--override来添加修订版本属性并将其值设置为1:

```shell
helm init --override metadata.annotations."deployment\.kubernetes\.io/revision"="1"
```

输出:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
...
```

#### override affinity

在下面的例子中，我们为affinity节点设置了属性。 可以组合多个--override命令来修改同一列表项的不同属性。

```shell
helm init --override "spec.template.spec.affinity.nodeAffinity.preferredDuringSchedulingIgnoredDuringExecution[0].weight"="1" --override "spec.template.spec.affinity.nodeAffinity.preferredDuringSchedulingIgnoredDuringExecution[0].preference.matchExpressions[0].key"="e2e-az-name"
```

指定的属性组合到“preferredDuringSchedulingIgnoredDuringExecution”属性的第一个列表项中。

```yaml
...
spec:
  strategy: {}
  template:
    ...
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - preference:
              matchExpressions:
              - key: e2e-az-name
                operator: ""
            weight: 1
...
```

#### 使用`--output`

--output标志允许我们跳过安装Tiller的deployment声明，只需将deployment声明以JSON或YAML格式输出到stdout。 然后可以使用jq等工具修改输出，并使用kubectl手动安装。

在下面的例子中，我们用--output json标志执行helm init。

```shell
helm init --output json
```

Tiller安装被跳过，声明以JSON格式输出到stdout。

```shell
"apiVersion": "extensions/v1beta1",
"kind": "Deployment",
"metadata": {
    "creationTimestamp": null,
    "labels": {
        "app": "helm",
        "name": "tiller"
    },
    "name": "tiller-deploy",
    "namespace": "kube-system"
},
...
```

#### 后端存储

默认情况下，Tiller会将release信息存储在ConfigMaps的运行的namespace中。 从Helm 2.7.0开始，现在有一个测试版存储后端使用Secrets来存储发布信息。 添加了此功能，以便在Kubernetes中发布Secret加密时保护chart的安全。

要启用后端加密，您需要使用以下选项启动Tiller：

```shell
helm init --override 'spec.template.spec.containers[0].command'='{/tiller,--storage=secret}'
```

目前，如果您想从默认后端切换到加密后端，您必须自行为此进行迁移。 当这个后端从测试版本毕业时，将会有更正式的移徙路径

## 结论

在大多数情况下，安装和获得预先构建的helm二进制文件和运行helm init一样简单。 本文档涵盖了那些想要使用Helm做更复杂事情的人的其他案例。

一旦成功安装了Helm Client和Tiller，您可以继续使用Helm来管理chart。

