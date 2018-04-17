# chart 存储库指南

本节介绍如何创建和使用Helm chart存储库。 在高层次上，chart存储库是可以存储和共享打包chart的位置。

官方chart库由[Kubernetes chart](https://github.com/kubernetes/charts)维护，我们欢迎参与。 但是，Helm还可以轻松创建和运行自己的chart存储库。 本指南解释了如何做到这一点。

## 必备条件

- 阅读[快速入门](https://docs.helm.sh/developing_charts/#quickstart)指南
- 通读[chart](https://docs.helm.sh/developing_charts/#charts)文档

## 创建chart存储库

Chart存储库是一个HTTP服务器，它包含一个index.yaml文件和可选的一些打包chart。 当您准备好分享您的chart时，首选方法是将其上传到chart存储库。

> 对于Helm 2.0.0，chart存储库没有任何内部认证。 在GitHub中有一个跟[踪进度的issure](https://github.com/kubernetes/helm/issues/1038)。

由于chart存储库可以是任何可以提供YAML和tar文件并可以响应GET请求的HTTP服务器，因此当您托管自己的chart存储库时，您有很多选择。 例如，您可以使用Google云端存储（GCS）存储，Amazon S3存储，Github Pages，甚至可以创建自己的Web服务器。

## chart库结构

Chart存储库由打包chart和名为index.yaml的特殊文件组成，其中包含存储库中所有chart的索引。 通常情况下，index.yaml描述的chart也被托管在同一台服务器上，起源文件也是如此。

例如，存储库https://example.com/charts的布局可能如下所示：

```xml
charts/
  |
  |- index.yaml
  |
  |- alpine-0.1.2.tgz
  |
  |- alpine-0.1.2.tgz.prov
```

在这种情况下，index文件将包含有关一个chart（Alpine图表）的信息，并为该图表提供下载URL https://example.com/charts/alpine-0.1.2.tgz。

不要求chart包与index.yaml文件位于同一台服务器上。 但是，这样做通常是最简单的。

## index文件

index文件是一个叫做index.yaml的yaml文件。 它包含一些关于包的元数据，包括chart的Chart.yaml文件的内容。 一个有效的chart存储库必须有一个index文件。 index文件包含有关chart存储库中每个chart的信息。 helm repo index命令将根据包含打包chart的给定本地目录生成索引文件。

这是一个Index文件的例子:

```yaml
apiVersion: v1
entries:
  alpine:
    - created: 2016-10-06T16:23:20.499814565-06:00
      description: Deploy a basic Alpine Linux pod
      digest: 99c76e403d752c84ead610644d4b1c2f2b453a74b921f422b9dcb8a7c8b559cd
      home: https://k8s.io/helm
      name: alpine
      sources:
      - https://github.com/kubernetes/helm
      urls:
      - https://technosophos.github.io/tscharts/alpine-0.2.0.tgz
      version: 0.2.0
    - created: 2016-10-06T16:23:20.499543808-06:00
      description: Deploy a basic Alpine Linux pod
      digest: 515c58e5f79d8b2913a10cb400ebb6fa9c77fe813287afbacf1a0b897cd78727
      home: https://k8s.io/helm
      name: alpine
      sources:
      - https://github.com/kubernetes/helm
      urls:
      - https://technosophos.github.io/tscharts/alpine-0.1.0.tgz
      version: 0.1.0
  nginx:
    - created: 2016-10-06T16:23:20.499543808-06:00
      description: Create a basic nginx HTTP server
      digest: aaff4545f79d8b2913a10cb400ebb6fa9c77fe813287afbacf1a0b897cdffffff
      home: https://k8s.io/helm
      name: nginx
      sources:
      - https://github.com/kubernetes/charts
      urls:
      - https://technosophos.github.io/tscharts/nginx-1.1.0.tgz
      version: 1.1.0
generated: 2016-10-06T16:23:20.499029981-06:00
```

生成的索引和包可以从基本的网络服务器提供。 您可以使用helm serve命令在本地进行测试，该命令启动本地服务器。

```shell
$ helm serve --repo-path ./charts
Regenerating index. This may take a moment.
Now serving you on 127.0.0.1:8879
```

上面的内容启动一个本地网络服务器，服务于它在./charts中找到的图表。 serve命令将在启动过程中为您自动生成一个index.yaml文件。

## HOSTING CHART REPOSITORIES

本部分介绍了提供chart存储库的几种方法。

### Google Cloud Storage

第一步是创建您的GCS存储。 我们会调用给我们的fantastic-charts。

![](../img/create-a-bucket.png)

接下来，通过编辑存储权限使您的存储公开。

![](../img/make-bucket-public.png)

插入此订单项以公开您的存储分区：

![](../img/edit-permissions.png)

恭喜，现在你有一个空的GCS准备好服务chart！

您可以使用Google Cloud Storage命令行工具或使用GCS Web UI上传chart存储库。 这是官方Kubernetes Charts存储库托管其chart的技术，因此如果遇到困难，您可能需要[查看该项目](https://github.com/kubernetes/charts)。

> 公共GCS存储可以通过简单的HTTPS访问，地址为https://bucket-name.storage.googleapis.com/。

### Github Pages例子

以类似的方式，您可以使用GitHub Pages创建chart存储库。

GitHub允许您以两种不同的方式提供静态网页：

- 通过配置一个项目来提供其docs /目录的内容
- 通过配置一个项目来为特定分支提供服务

我们会采取第二种方法，尽管第一种方法很简单。

第一步是创建你的gh-pages分支。 你可以在本地做到这一点。

```Shell
$ git checkout -b gh-pages
```

或者通过网络浏览器使用Github存储库上的分支按钮：

![](../img/create-a-gh-page-button.png)

接下来，您需要确保您的gh-pages分支设置为Github Pages，点击您的repo Settings并向下滚动到Github页面部分并按照以下设置：

![](../img/set-a-gh-page.png)

默认情况下，Source通常被设置为gh-pages分支。 如果这不是默认设置，那么选择它。

如果您愿意，您可以在那里使用自定义域名。

并检查是否勾选了强制HTTPS，以便在提供图表时使用HTTPS。

在这种设置中，您可以使用主分支来存储图表代码，并将gh-page分支作为chart存储库，例如：https://USERNAME.github.io/REPONAME。 演示TS chart存储库可通过https://technosophos.github.io/tscharts/访问

### 普通的网络服务器

要配置普通Web服务器来服务Helm图表，您只需执行以下操作：

- 将您的index文件和chart放入服务器可以访问的目录中
- 确保index.yaml文件可以在没有认证要求的情况下被访问
- 确保yaml文件的内容类型正确（text/yaml或text/x-yaml）

例如，如果您想要将chart从$WEBROOT/charts中提取出来，请确保您的web根目录中有一个charts/目录，并将该index文件和chart放入该文件夹中。

## 管理chart库

现在您已拥有chart存储库，本指南的最后一部分介绍了如何维护该存储库中的chart。

### 将chart存储在chart存储库中

现在您已有一个chart存储库，让我们上传一个chart和一个index文件到存储库。 chart存储库中的chart必须打包（helm package chart-name/）并版本正确（遵循[SemVer 2](https://semver.org/)准则）。

接下来的这些步骤组成了一个示例工作流程，但欢迎您使用您喜欢的任何工作流程来存储和更新chart存储库中的chart。

准备好打包chart后，创建一个新目录，并将打包chart移动到该目录。

```shell
$ helm package docs/examples/alpine/
$ mkdir fantastic-charts
$ mv alpine-0.1.0.tgz fantastic-charts/
$ helm repo index fantastic-charts --url https://fantastic-charts.storage.googleapis.com
```

最后一条命令采用刚刚创建的本地目录的路径和远程chart存储库的URL，并在给定的目录路径中组成一个index.yaml文件。

现在，您可以使用同步工具或手动将chart和index文件上传到chart存储库。 如果您使用Google云端存储，请使用gsutil客户端查看此[示例工作流程](https://docs.helm.sh/developing_charts/#developing_charts_sync_example)。 对于GitHub，您可以简单地将chart放入适当的目标分支中。

### 将新chart添加到现有存储库

每次要将新chart添加到存储库时，都必须重新生成索引。 helm repo index命令将从头开始完全重建index.yaml文件，仅包括它在本地找到的chart。

但是，您可以使用--merge标志将新图表增量添加到现有的index.yaml文件（在使用像GCS这样的远程存储库时是一个很好的选择）。 运行helm repo index --help 了解更多信息，

确保您上传修改过的index.yaml文件和chart。 如果您生成了provenance文件，也要上传。

### 与他人分享您的chart

当您准备好分享您的chart时，只需让别人知道您的存储库的URL是什么。

从那里，他们将通过helm repo add [NAME][URL]命令将他们的仓库添加到他们的helm客户端，并使用他们想用来引用仓库的任何名称。

```shell
$ helm repo add fantastic-charts https://fantastic-charts.storage.googleapis.com
$ helm repo list
fantastic-charts    https://fantastic-charts.storage.googleapis.com
```

如果chart由HTTP基本认证支持，您也可以在此处提供用户名和密码：

```shell
$ helm repo add fantastic-charts https://username:password@fantastic-charts.storage.googleapis.com
$ helm repo list
fantastic-charts    https://username:password@fantastic-charts.storage.googleapis.com
```

> 如果存储库不包含有效的index.yaml，则不会添加存储库。

之后，您的用户将能够搜索您的chart。 更新存储库后，他们可以使用helm repo update命令获取最新的chart信息。

在引擎下，helm repo add和helm repo update命令获取index.yaml文件并将它们存储在$HELM_HOME/repository/cache /目录中。 这是helm search功能找到有关chart的信息的地方。