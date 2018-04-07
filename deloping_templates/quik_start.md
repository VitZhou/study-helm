#  Chart Template快速开始

在本指南的这一部分，我们将创建一个chart，然后添加第一个模板。 我们在这里创建的chart将在指南的其余部分使用。

开始前,我们先来看一下Helm chart。

## CHARTS

如[Chart指南](../charts/charts.md)中所述，helm charts的结构如下所示：

```html
mychart/
  Chart.yaml
  values.yaml
  charts/
  templates/
  ...
```

templates/目录用于模板文件。 当Tiller评估chart时，它将通过模板渲染引擎发送templates/目录中的所有文件。 然后，Tiller收集这些模板的结果并将它们发送给Kubernetes。

values.yaml文件对模板也很重要。 该文件包含chart的默认值。 用户在helm安装或helm升级期间可能会覆盖这些值。

Chart.yaml文件包含chart的说明。 您可以从模板中访问它。 charts/目录可能包含其他chart（我们称之为子chart）。 在本指南的后面，我们将看到如何在模板渲染方面发挥作用。

## Chart Started

对于本指南，我们将创建一个名为mychart的简单chart，然后我们将在chart内创建一些模板。

接下来的内容我们都会在mychart目录完成.

## 快速浏览 `mychart/templates/`

如果你看看mychart/templates/目录，你会发现有几个文件已经存在。

- NOTES.txt: chart的“帮助文本”。 这将在您的用户运行helm安装时显示。
- deployment.yaml: 创建Kubernetes [deployment](http://kubernetes.io/docs/user-guide/deployments/)的基本声明
- service.yaml: 用于为您的deployment创建[[service endpoint](http://kubernetes.io/docs/user-guide/services/)]的基本声明
- _helpers.tpl: 放置模板助手的地方，您可以在整个chart中重新使用

而我们要做的就是......全部删除它们！ 这样我们就可以从头开始学习我们的教程。 我们将在实际中创建自己的NOTES.txt和_helpers.tpl。

```shell
$ rm -rf mychart/templates/*.*
```

在编写生产级chart时，使用这些chart的基本版本可能非常有用。 所以在你的日常chart制作中，你可能不想删除它们。

## 第一个模板

我们要创建的第一个模板将是一个ConfigMaps。 在Kubernetes中，ConfigMap只是存储配置数据的容器。 其他的东西，比如pod，可以访问ConfigMap中的数据。

由于ConfigMaps是基础资源，它们为我们提供了一个很好的起点。

我们首先创建一个名为mychart / templates / configmap.yaml的文件：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
```

> 提示：模板名称不遵循严格的命名模式。 但是，我们建议使用YAML文件的后缀.yaml和helper的.tpl。

上面的YAML文件是一个简单的ConfigMap，具有最少的必要字段。 由于该文件位于templates/目录中，因此将通过模板引擎发送给kubernetes。

把这样一个普通的YAML文件放在templates/目录下就好了。 当Tiller读取这个模板时，它会直接发送给Kubernetes。

有了这个简单的模板，我们现在有一个可安装的chart。 我们可以像这样安装它：

```shell
$ helm install ./mychart
NAME: full-coral
LAST DEPLOYED: Tue Nov  1 17:36:01 2016
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME                DATA      AGE
mychart-configmap   1         1m
```

在上面的输出中，我们可以看到我们的ConfigMap已经创建。 使用Helm，我们可以检索版本并查看加载的实际模板。

```Shell
$ helm get manifest full-coral

---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
```

helm get manifest命令获取release名称（full-coral）并打印出上传到服务器的所有Kubernetes资源。 每个文件以---开头，表示YAML文档的开始，然后是一个自动生成的注释行，告诉我们该模板文件生成的这个YAML文档。

从那里开始，我们可以看到，YAML数据正是我们在configmap.yaml文件中放置的数据。

现在我们可以删除我们的版本：helm delete full-coral。

## 添加一个简单的模板调用

将资源的name硬编码通常被认为是不好的做法。 名称应该是唯一的一个版本。 所以我们可能希望通过插入发行版名称来生成一个名称字段。

提示：由于DNS系统的限制，名称：字段限制为63个字符。 因此，发布名称限制为53个字符。 Kubernetes 1.3及更早版本仅限于24个字符（即14个字符名称）。

让我们相应地修改configmap.yaml。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
```

name字段的值现在是\{\{.Release.Name\}\} - configmap。

> 模板指令包含在\{\{和\}\}块中。

模板指令\{\{.Release.Name\}\}将release name注入模板。 传递给模板的值可以认为是namespace对象，其中点号（.）分隔每个namespace元素。

Release之前的最前面的点表示我们从这个范围的最顶层的namespace开始（我们将在一段时间内讨论范围）。 因此，我们可以将.Release.Name读作“从顶部命名空间开始，找到Release对象，然后在其内部查找名为Name的对象”。

Release对象是Helm的内置对象之一，稍后我们将更深入地介绍它。 但就目前而言，这足以说明这将显示Tiller分配给我们Release的版本名称。

现在，当我们安装资源时，我们会立即看到使用此模板指令的结果：

```shell
$ helm install ./mychart
NAME: clunky-serval
LAST DEPLOYED: Tue Nov  1 17:45:37 2016
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME                      DATA      AGE
clunky-serval-configmap   1         1m
```

请注意，在“RESOURCES”部分中，我们看到的名称是clunky-serval-configmap而不是mychart-configmap。

你可以运行helm get manifest clunky-serval来查看整个生成的YAML。

此时，我们已经看到了最基本的模板：在\{\{和\}\}中嵌入了模板指令的YAML文件。 在下一部分中，我们将深入研究模板。 但在继续之前，有一个快速技巧可以使构建模板更快：当您要测试模板渲染但不实际安装任何东西时，可以使用helm install --debug --dry-run ./mychart。 这会将图表发送到Tiller服务器，它将渲染模板。 但不是安装chart，它会将渲染模板返回给您，以便您可以看到输出：

```shell
$ helm install --debug --dry-run ./mychart
SERVER: "localhost:44134"
CHART PATH: /Users/mattbutcher/Code/Go/src/k8s.io/helm/_scratch/mychart
NAME:   goodly-guppy
TARGET NAMESPACE:   default
CHART:  mychart 0.1.0
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: goodly-guppy-configmap
data:
  myvalue: "Hello World"
```

使用--dry-run可以更容易地测试代码，但不能确保Kubernetes本身会接受你生成的模板。 最好不要假定你的chart只会因为--dry-run正常工作而被安装。

在接下来的几节中，我们将采用我们在这里定义的基本图表，并详细探索Helm模板语言。 我们将开始使用内置对象。