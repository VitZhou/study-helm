# Charts

Helm使用名为Charts的包装格式。 chat是描述相关的一组Kubernetes资源的文件集合。 单个chat可能用于部署简单的东西，比如memcached pod，或者一些复杂的东西，比如完整的具有HTTP服务器，数据库，缓存等的Web应用程序堆栈。

图表被创建为特定目录树中的文件，然后将它们打包到版本化的归档中进行部署。

本文档解释了chat格式，并提供了使用Helm构建图表的基本指导。

## chart文件结构

chart被组织为一个目录内的文件集合。 目录名称是chart的名称（没有版本信息）。 因此，描述WordPress的chart将被存储在wordpress /目录中。

在这个目录里面，Helm会期望一个匹配的结构:

```hxml
wordpress/
  Chart.yaml          # 包含有关chart信息的YAML文件
  LICENSE             # OPTIONAL: 包含chart许可证的纯文本文件
  README.md           # OPTIONAL: 一个可读的README文件
  requirements.yaml   # OPTIONAL: 一个YAML文件，列出了chart的依赖关系
  values.yaml         # 该chart的默认配置值
  charts/             # OPTIONAL: 包含此chart所依赖的任何chart的目录。
  templates/          # OPTIONAL: 一个模板目录，当与values相结合时，
                      # 将生成有效的Kubernetes清单文件
  templates/NOTES.txt # OPTIONAL: 包含简短使用说明的纯文本文件
```

Helm保留使用chart/和templates/目录以及列出的文件名称。 其他文件将保留原样。

尽管chart和tempaltes目录是可选的，但chart必须至少有一个chart依赖项或模板文件才能生效。

### chart.yaml文件

Chart.yaml文件是chart所必需的。 它包含以下字段：

```html
name: chart的名称 (required)
version: 一个SemVer 2(语义化版本)版本(required)
description: 这个项目的单句描述 (optional)
keywords:
  - 关于此项目的关键字列表 (optional)
home: 该项目主页的URL(optional)
sources:
  - 此项目的源代码URL列表 (optional)
maintainers: # (optional)
  - name: 维护者的名称 (每个维护者都需要)
    email: 维护者的email (optional for each maintainer)
    url: 维护者的url (optional for each maintainer)
engine: gotpl＃模板引擎的名称（可选，默认为gotpl）
icon: 要用作图标的SVG或PNG图像的URL (optional).
appVersion: 包含的应用程序版本（可选）。 这个不一定是SemVer。
deprecated: 此chart是否已被弃用（可选，布尔型）
tillerVersion: 该图表要求的Tiller版本。 这应该表示为SemVer范围：“> 2.0.0”（可选）
```

如果您熟悉Helm Classic的Chart.yaml文件格式，则会注意到指定相关性的字段已被删除。 这是因为新的Chart格式使用chart/目录表示依赖关系。

其他字段将被默默忽略。

#### chart和版本控制

每张chart都必须有一个版本号。 版本必须遵循SemVer 2标准。 与helm Classic不同，Kubernetes Helm使用版本号作为release标记。 存储库中的软件包由名称加版本标识。

例如，版本字段设置为版本1.2.3的nginx chart将被命名为：

```html
nginx-1.2.3.tgz
```

还支持更复杂的SemVer 2名称，例如版本：1.2.3-alpha.1 + ef365。 但非SemVer名称被系统明确禁止。

> 虽然Helm Classic和Deployment Manager在chart方面都非常适合GitHub，但Kubernetes Helm并不依赖或需要GitHub甚至Git。 因此，它根本不使用Git SHA进行版本控制

许多Helm工具都使用Chart.yaml中的版本字段，包括CLI和Tiller服务器。 生成包时，helm package命令将使用它在Chart.yaml中找到的版本作为包名称中的一个标记。 系统假定chart包名中的版本号与Chart.yaml中的版本号相匹配。 不符合这个假设会导致错误。

#### appVersion字段

请注意，appVersion字段与version字段无关。 这是一种指定应用程序版本的方法。 例如，drupal chart可能具有appVersion：8.2.1，表示chart中包含的Drupal版本（默认情况下）为8.2.1。 该字段是信息性的，对chart版本计算没有影响。

#### 弃用chart

在管理chart存储库中的chart时，有时需要弃用chart。 Chart.yaml中可选的deprecated字段可用于将chart标记为不赞成使用。 如果存储库中最新版本的chart标记为已弃用，则整个chart被视为已弃用。 chart名称稍后可以通过发布未标记为已弃用的较新版本来重用。 弃用chart的工作流程以及kubernetes/charts项目的后续工作流程如下： - 更新chart的Chart.yaml，将chart标记为不推荐使用，颠覆版本 - 在chart存储库中发布新chart版本 - 将chart从源存储库中移除 （例如git）

### chart的LICENSE,README,NOTE文件

Chart还可以包含描述chart的安装，配置，使用和许可证的文件。图表的自述文件应在Markdown（README.md）中格式化，并且通常应包含：

- Chart提供的应用程序或服务的描述
- 运行chart的任何先决条件或要求
- values.yaml中的选项说明和默认值
- 任何其他可能与安装或配置chart相关的信息

该chart还可以包含一个简短的纯文本templates/ NOTES.txt文件，该文件将在安装后以及查看版本状态时打印出来。此文件将作为模板进行评估，并可用于显示使用说明，下一步或任何其他与发布chart相关的信息。例如，可以提供用于连接到数据库或访问Web UI的指令。由于此文件在运行helm install或helm status时会打印到STDOUT(标准输出)，因此建议保持内容简洁并指向自述文件以获取更多详细信息。

### chart依赖

在Helm中，一个chart可能取决于任何数量的其他chart。 这些依赖关系可以通过requirements.yaml文件动态链接，或者引入charts/目录并手动管理。

尽管手动管理依赖关系有一些团队需要的优势，但声明依赖关系的首选方法是使用chart内的requirements.yaml文件。

> Helm Classic的Chart.yaml的dependencies：部分已被完全删除。

#### 使用requirements.yaml管理依赖关系

requirements.yaml文件是列出您的依赖关系的简单文件。

```yaml
dependencies:
  - name: apache
    version: 1.2.3
    repository: http://example.com/charts
  - name: mysql
    version: 3.2.1
    repository: http://another.example.com/charts
```

- Name: 你想要的chart的名称
- Version: 你想要的chart的版本
- repository字段是图表存储库的完整URL。 请注意，您还必须使用helm repo add在本地添加该repository。

一旦你有一个依赖关系文件，你可以运行helm dependency update，它会使用你的依赖关系文件为你下载所有指定的chart到你的charts/目录中。

```shell
$ helm dep up foochart
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "local" chart repository
...Successfully got an update from the "stable" chart repository
...Successfully got an update from the "example" chart repository
...Successfully got an update from the "another" chart repository
Update Complete. Happy Helming!
Saving 2 charts
Downloading apache from repo http://example.com/charts
Downloading mysql from repo http://another.example.com/charts
```

当实用helm dependency update更新检索chart时，它会将它们作为chart归档存储在charts/目录中。 因此，对于上面的示例，人们会期望在charts目录中看到以下文件：

```html
charts/
  apache-1.2.3.tgz
  mysql-3.2.1.tgz
```

使用requirements.yaml管理chart是轻松保持chart更新的好方法，并且还可以在整个团队中共享需求信息。

#### requirements.yaml中的别名字段

除上述其他字段外，每个需求条目可能包含可选的字段别名。

为依赖关系图添加别名会将chart放入依赖关系中，并使用别名作为新依赖关系的名称。

在需要使用其他名称访问图表的情况下，可以使用别名。

```yaml
# parentchart/requirements.yaml
dependencies:
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
    alias: new-subchart-1
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
    alias: new-subchart-2
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
```

在上面的例子中，我们会在parentchart中得到3个依赖关系

```shell
subchart
new-subchart-1
new-subchart-2
```

实现这一目的的手动方法是将多个不同名称的charts/目录中的相同chart复制/粘贴。

#### requirements.yaml中的tag和condition字段

除上述其他字段外，每个需求条目可能包含可选字段tag和condition。

所有chart默认加载。 如果存在tag或condition字段，则会对它们进行评估并用于控制它们所应用的chart的加载。

- Condition: condition字段包含一个或多个YAML路径（用逗号分隔）。 如果此路径存在于顶级父级的值中并且解析为布尔值，则将根据该布尔值启用或禁用图表。 只有在列表中找到的第一个有效路径才被评估，如果没有路径存在，那么该条件不起作用。
- Tags-tags字段是与此chart关联的YAML标签列表。 在顶级父级的值中，可以通过指定tag和布尔值来启用或禁用所有带有标签的chart。

```yaml
# parentchart/requirements.yaml
dependencies:
      - name: subchart1
        repository: http://localhost:10191
        version: 0.1.0
        condition: subchart1.enabled, global.subchart1.enabled
        tags:
          - front-end
          - subchart1

      - name: subchart2
        repository: http://localhost:10191
        version: 0.1.0
        condition: subchart2.enabled,global.subchart2.enabled
        tags:
          - back-end
          - subchart2
```

```yaml
# parentchart/values.yaml

subchart1:
  enabled: true
tags:
  front-end: false
  back-end: true
```

在上面的例子中，所有具有front-end标签的chart都将被禁用，但由于subchart1.enabled路径在父项值中评估为'true'，因此该条件将覆盖front-end标签，并且将启用subchart1。

由于subchart2被标记为back-end，并且该标记的计算结果为true，所以将启用subchart2。 还要注意的是，尽管subchart2在requirements.yaml中指定了一个条件，但parentchart的值中没有对应的路径和值，因此条件无效。

##### 使用带有tags和condition的CLI

--set参数可以照常使用，以更改tags和condtion值。

```shell
helm install --set tags.front-end=true --set subchart2.enabled=false
```

##### Tags和Condition解析

- Condition（当在values设置时）总是覆盖tags。 存在胜利的第一个条件路径以及该chart的后续条件将被忽略。
- tags表示:如果任何chart的标签为真，则启用chart
- tags和condition值必须在顶层父级的values中进行设置。
- Tags：键值必须是顶级键。 全局和嵌套标记：表格当前不支持。

#### 通过requirements.yaml导入子values

在某些情况下，希望允许子chart的values传播到父chart并作为通用默认值共享。 使用导出格式的另一个好处是它可以使将来的工具反省用户可设置的值。

包含要导入的值的键可以使用YAML列表在父chart的requirements.yaml文件中指定。 列表中的每个项目都是从子chart的导出字段导入的键。

要导入未包含在exports键中的值，请使用子父级格式。 下面描述了两种格式的例子。

##### 使用exports格式

如果子chart的values.yaml文件在根目录中包含exports字段，则可以通过指定要导入的key直接将它的内容导入到父项的values中，如下例所示：

```yaml
# parent's requirements.yaml file
    ...
    import-values:
      - data
```

```yaml
# child's values.yaml file
...
exports:
  data:
    myint: 99
```

由于我们在export-values列表中指定了data key，因此Helm会在子chart的exports字段中查找data key并导入其内容。

最终的父values将包含我们的exports字段：

```yaml
# parent's values file
...
myint: 99
```

请注意，父chart的data key不包含在父chart的最终values中。 如果您需要指定父key，请使用'child-parent'格式。

##### 使用child-parent格式

要访问未包含在子chart values的exports key中的值，您需要指定要export的key的源key（子级）和父chart values（父级）中的目标路径。

下例中的export-values指示Helm在child:path处找到任何值，便将它们复制到parent中指定路径的父values：

```yaml
# parent's requirements.yaml file
dependencies:
  - name: subchart1
    repository: http://localhost:10191
    version: 0.1.0
    ...
    import-values:
      - child: default.data
        parent: myimports
```

在上面的例子中，在subchart1的values中的default.data中找到的值将被导入到父chart values中的myimports key中，详情如下：

```yaml
# parent's values.yaml file

myimports:
  myint: 0
  mybool: false
  mystring: "helm rocks!"
```

```yaml
# subchart1's values.yaml file

default:
  data:
    myint: 999
    mybool: true
```

父chart的values最终会如下:

```yaml
# parent's final values

myimports:
  myint: 999
  mybool: true
  mystring: "helm rocks!"
```

父chart的最终values现在包含从subchart1导入的myint和mybool字段。

### 通过charts/目录手动管理依赖

如果希望更多地控制依赖关系，可以通过将依赖chart复制到charts/目录中来明确表示这些依赖关系。

依赖关系可以是chart压缩包（foo-1.2.3.tgz），也可以是未打包的chart目录。 但它的名字不能以_或..开头。这些文件将被chart加载器忽略。

例如，如果WordPress chart取决于Apache chart，则在WordPress chart的charts/目录中提供（正确版本的）Apache chart：

```html
wordpress:
  Chart.yaml
  requirements.yaml
  # ...
  charts/
    apache/
      Chart.yaml
      # ...
    mysql/
      Chart.yaml
      # ...
```

上面的例子显示了WordPress chart如何通过在charts/目录中包含这些chart来表示它对Apache和MySQL的依赖。

> 要将依赖关系放入charts/目录，请使用helm fetch命令

### Operational aspects of using dependencies

上面的部分解释了如何指定chart依赖关系，但是这会如何影响使用helm install和helm upgrade的chart安装？

假设名为“A”的chart创建以下Kubernetes对象:

- Namepace: A-Namespace
- statefulset: A-statefulset
- service: A-service

此外，A依赖于创建对象的chartB:

- Namespace: B-namespace
- statefulset: B-statefulset
- Service: B-service

安装/升级chart A后，会创建/修改单个Helm版本。 该版本将按以下顺序创建/更新所有上述Kubernetes对象：

- A-Namespace
- B-Namespace
- A-StatefulSet
- B-ReplicaSet
- A-Service
- B-Service

这是因为当Helm安装/升级chart时，chart中的Kubernetes对象及其所有依赖项都是:

1. 聚合成一个集合
2. 按类型排序，然后按名称排序
3. 按该顺序创建/更新

因此，使用chart及其依赖关系的所有对象创建单个版本。

Kubernetes类型的安装顺序由kind_sorter.go中的枚举InstallOrder给出（请参阅[Helm源文件](https://github.com/kubernetes/helm/blob/master/pkg/tiller/kind_sorter.go#L26)）。

### 模板和values

helm chart模板是用[Go模板语言](https://golang.org/pkg/text/template/)编写的，其中添加了来自[Sprig库](https://github.com/Masterminds/sprig)的50个左右的附加模板函数以及一些其他[专用函数](https://docs.helm.sh/developing_charts/#chart-development-tips-and-tricks)。

所有模板文件都存储在chart的templates/文件夹中。 当Helm渲染chart时，它将通过模板引擎传递该目录中的每个文件。

模板的值有两种提供方式:

- Chart开发人员可以在chart内提供名为values.yaml的文件。 该文件可以包含默认值。
- Chart用户可能会提供一个包含值的YAML文件。 这可以在命令行上使用helm install来提供。

当用户提供自定义值时，这些值将覆盖chart的values.yaml文件中的值。

### 模板文件

模板文件遵循用于编写Go模板的标准约定（请参阅t[ext/template Go包文档](https://golang.org/pkg/text/template/)以了解详细信息）。 示例模板文件可能如下所示：

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    heritage: deis
spec:
  replicas: 1
  selector:
    app: deis-database
  template:
    metadata:
      labels:
        app: deis-database
    spec:
      serviceAccount: deis-database
      containers:
        - name: deis-database
          image: {{.Values.imageRegistry}}/postgres:{{.Values.dockerTag}}
          imagePullPolicy: {{.Values.pullPolicy}}
          ports:
            - containerPort: 5432
          env:
            - name: DATABASE_STORAGE
              value: {{default "minio" .Values.storage}}
```

上面的例子基于https://github.com/deis/charts，是Kubernetes复制控制器的模板。 它可以使用以下四个模板值（通常在values.yaml文件中定义):

- imageRegistry: Docker镜像的源注册表。
- dockerTag: docker镜像的tag
- pullPolicy: k8s的pull policy(拉去策略)
- storage: 存储后端的默认设置为“minio”

所有这些值都由模板作者定义。 helm不需要或指定参数。

要查看许多工作chart，请查看[Kubernetes Charts](https://github.com/kubernetes/charts)项目

### 预定义值

通过values.yaml文件（或通过--set标志）提供的值可以从模板中的.Values对象访问。 但您可以在模板中访问其他预定义的数据片段。

以下值是预定义的，可用于每个模板，并且不能被覆盖。 与所有值一样，名称区分大小写。

- Release.Name: release的名称(不是chart的名称！)
- Release.Time: chart的release time的最后更新的时间。 这将与发布对象上的最后发布时间相匹配。
- Release.Namespace: chart release的名称空间。
- Release.Service: 进行release的服务。 通常这是Tiller。
- Release.IsUpgrade: 如果当前操作是升级或回滚，则设置为true。
- Release.IsInstall: 如果当前操作是安装，则设置为true。
- Release.Revision: 修订编号。 它从1开始，随着每次helm upgrade而增加
- Chart: Chart.yaml的内容。 因此，chart版本可以作为Chart.Version获得，维护者在Chart.Maintainers中。
- Files: 包含chart中所有非特殊文件的类map对象。 这不会让您访问模板，但会让您访问其他存在的文件（除非它们使用.helmignore排除）。 可以使用{{index.Files“file.name”}}或使用{{.Files.Get name}}或{{.Files.GetString name}}功能访问文件。 您还可以使用{{.Files.GetBytes}}以字节数组的形式访问文件的内容
- Capabilities: 包含有关Kubernetes（{{.Capabilities.KubeVersion}}，Tiller（{{.Capabilities.TillerVersion}}和受支持的Kubernetes API版本（{{.Capabilities.APIVersions.Has“）版本信息的map-like对象。"batch/V1"）

> 任何未知的Chart.yaml字段将被删除。 它们不会在Chart对象内部可访问。 因此，Chart.yaml不能用于将任意结构化的数据传递到模板中。 虽然values文件可以。

### values文件

考虑到上一节中的模板，提供必要值的values.yaml文件如下所示：

```yaml
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "s3"
```

Values文件在YAML中格式化。 Chart可能包含一个默认的values.yaml文件。 Helm install命令允许用户通过提供其他YAML值来覆盖values：

```shell
$ helm install --values=myvals.yaml wordpress
```

当以这种方式传递值时，它们将被合并到默认values文件中。 例如，考虑一个如下所示的myvals.yaml文件：

```yaml
storage: "gcs"
```

当它与图表中的values.yaml合并时，生成的内容将为：

```yaml
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "gcs"
```
> 请注意，只有最后一个字段被覆盖。
> 包含在charta内的默认values文件必须命名为values.yaml。 但是在命令行上指定的文件可以被命名为任何东西。
> 如果在helm install或helm upgrade时使用--set标志，则这些值仅在客户端转换为YAML。
> 如果values文件中存在必需条目，则可以使用“required”功能在chart模板中声明它们

然后使用.Values对象在模板内部访问这些值中的任何一个:

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    heritage: deis
spec:
  replicas: 1
  selector:
    app: deis-database
  template:
    metadata:
      labels:
        app: deis-database
    spec:
      serviceAccount: deis-database
      containers:
        - name: deis-database
          image: {{.Values.imageRegistry}}/postgres:{{.Values.dockerTag}}
          imagePullPolicy: {{.Values.pullPolicy}}
          ports:
            - containerPort: 5432
          env:
            - name: DATABASE_STORAGE
              value: {{default "minio" .Values.storage}}
```

### Scope, Dependencies, 和 Values

Values文件可以声明顶级chart的值，也可以为该chart的charts/目录中包含的任何chart声明值。 或者，用不同的方式来描述它，values文件可以为chart及其任何依赖项提供值。 例如，上面的演示WordPress chart具有mysql和apache作为依赖关系。 values文件可以为所有这些组件提供值：

```yaml
title: "My WordPress Site" # Sent to the WordPress template

mysql:
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  port: 8080 # Passed to Apache
```

更高级别的chart可以访问到在下面定义的所有变量。 所以WordPress chart可以以.Values.mysql.password的形式访问MySQL密码。 但较低级别的chart无法访问父chart中的内容，因此MySQL将无法访问title属性。 但是，它可以访问apache.port。

Values是命名空间，但命名空间已被修剪。 因此，对于WordPress chart，它可以以.Values.mysql.password的形式访问MySQL密码字段。 但是对于MySQL图表来说，这些值的范围已经缩小了，并且名称空间前缀被删除了，所以它会将密码字段看作.Values.password。

#### 全局values

从2.0.0-Alpha.2开始，Helm支持特殊的“global”values。 例如前面例子的这个修改版本：

```yaml
title: "My WordPress Site" # Sent to the WordPress template

global:
  app: MyWordPress

mysql:
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  port: 8080 # Passed to Apache
```

上面添加了值为app:MyWordPress的global部分。 此值可作为.Values.global.app提供给所有chart。

例如，mysql模板可以以{{.Values.global.app}}的形式访问应用程序，而apache chart也可以。上面的values文件是这样有效地重新生成的:

```yaml
title: "My WordPress Site" # Sent to the WordPress template

global:
  app: MyWordPress

mysql:
  global:
    app: MyWordPress
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  global:
    app: MyWordPress
  port: 8080 # Passed to Apache
```

这提供了一种与所有子chart共享一个顶级变量的方法，这对设置元数据属性（如标签）很有用。

如果子chart声明了一个全局变量，则该全局变量将向下传递（到子chart的子chart），但不向上传递到父chart。 子chart无法影响父chart的值。

此外，父chart的全局变量优先于子chart中的全局变量。

### 参考

当涉及到编写模板和values文件时，有几个标准参考可以帮助您。

- [go模板](https://godoc.org/text/template)
- [Extra template functions](https://godoc.org/github.com/Masterminds/sprig)
- [YAML格式](http://yaml.org/spec/)

## 使用Helm管理chart

helm工具有几个用于处理chart的命令。

它可以为你创建一个新的chart:

```shell
$ helm create mychart
Created mychart/
```

一旦你编辑了一个chart，helm可以将它打包成一个chart压缩包给你：

```shell
$ helm package mychart
Archived mychart-0.1.-.tgz
```

您还可以使用helm帮助您查找chart格式或信息的问题：

```shell
$ helm lint mychart
No issues found
```

### chart存储库

chart存储库是一个包含一个或多个chart包的HTTP服务器。 虽然helm可用于管理本地chart目录，但在共享chart时，首选机制是chart存储库。

任何可以提供YAML文件和tar文件并可以回答GET请求的HTTP服务器都可以用作存储库服务器。

Helm附带用于开发人员测试的内置套装服务器（helm serve）。 Helm团队已经测试了其他服务器，包括启用了网站模式的Google Cloud Storage以及启用了网站模式的S3。

存储库的主要特征是存在名为index.yaml的特殊文件，其中包含存储库提供的所有软件包的列表以及允许检索和验证这些软件包的元数据。

在客户端，仓库使用helm repo命令进行管理。 但是，Helm不提供将chart上传到远程存储服务器的工具。 这是因为这样做会增加实施服务器的实质性需求，从而增加设置存储库的障碍。

### chart启动包

helm create命令带有一个可选的--starter选项，可让您指定“started chart”。

Starteds只是常规chart，位于$HELM_HOME/starters。 作为chart开发人员，您可以创作专门设计用于started的chart。 记住这些chart时应考虑以下因素：

- Chart.yaml将被生成器覆盖。
- 用户将期望修改这样的chart内容，因此文档应该指出用户如何做到这一点。
- 所有发生的<CHARTNAME>将被替换为指定的chart名称，以便started chart可用作模板。

目前将chart添加到$HELM_HOME/starters的唯一方法是手动将其复制到那里。 在你的chart的文档中，您可能想要解释该过程。