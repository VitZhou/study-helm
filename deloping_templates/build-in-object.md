# 内置对象

对象从模板引擎传递到模板中。 而且你的代码可以传递对象（当我们看到with和range语句时，我们会看到一些例子）。 甚至有几种方法可以在你的模板中创建新的对象，比如我们稍后会看到的tuple函数。

对象可以很简单，只有一个values。 或者他们可以包含其他对象或功能。 例如。 Release对象包含多个对象（如Release.Name），而Files对象有几个功能。

在上一节中，我们使用\{\{.Release.Name\}\}将版本的名称插入到模板中。 Release是您可以在模板中访问的顶级对象之一。

- Release: 这个对象描述了这个release本身。 它里面有几个对象：
  - Release.Name: release的名称
  - Release.Time: 发布的时间
  - Release.Namespace: release的版本(如果声明中未覆盖)
  - Release.Service: Release service的名称（通常为Tiller）。
  - Release.Revision: 此release的修订版号。 它从1开始，每调用一次helm upgrade都会增加
  - Release.IsUpgrade: 如果当前操作是upgrade或rollback，则设置为true。
  - Release.IsInstall: 如果当前操作是install，则设置为true。
- Values: values从values.yaml文件和用户提供的文件传递到模板中。 默认情况下，值为空。
- Chart: Chart.yaml文件的内容。 Chart.yaml中的任何数据都可以在这里访问。 例如\{\{.Chart.Name\}\} - \{\{.Chart.Version\}\}将打印出mychart-0.1.0
  - [chart指南](../charts/charts.md)中列出了可用字段
- Files: 这提供了对chat中所有非特殊文件的访问。 虽然您无法使用它来访问模板，但您可以使用它来访问chart中的其他文件。
  - Files.Get: 是一个按名称获取文件的函数（.Files.Get config.ini）
  - Files.GetBytes: 是将文件内容作为字节数组而不是字符串获取的函数。 这对像chart这样的东西很有用。
- Capabilities: 这提供了关于Kubernetes集群支持哪些功能的信息。
  - Capabilities.APIVersions: 是一组版本号
  - Capabilities.APIVersions.Has $version: 指示是否在群集上启用版本（batch / v1）
  - Capabilities.KubeVersion: 提供了查找Kubernetes版本的方法。 它具有以下值：Major，Minor，GitVersion，GitCommit，GitTreeState，BuildDate，GoVersion，Compiler和Platform。
  - Capabilities.TillerVersion: 提供了查找Tiller版本的方法。 它具有以下值：SemVer，GitCommit和GitTreeState。
- Template: 包含有关正在执行的当前模板的信息
  - Name: 到当前模板的namespace文件路径（例如mychart/templates/mytemplate.yaml）
  - BasePath: 当前chart模板目录的namespace路径（例如mychart/templates）。

这些值可用于任何顶级模板。 我们稍后会看到，这并不一定意味着它们将在任何地方都可用。

内置Values始终以大写字母开头。 这符合Go的命名约定。 当你创建自己的name时，你可以自由地使用适合你的团队的惯例。 一些团队，如[Kubernetes chart](https://github.com/kubernetes/charts)团队，选择仅使用首字母小写来区分本地名称与内置名称。 在本指南中，我们遵循该约定。