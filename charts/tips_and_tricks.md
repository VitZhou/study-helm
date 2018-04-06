# chart开发提示和技巧

本指南涵盖了Helm chart开发人员在建立生产质量chart时学到的一些提示和技巧。

## 了解你的模板功能

Helm使用[Go模板](https://godoc.org/text/template)模板化资源文件。 虽然Go提供了几个内置函数，但我们还添加了许多其他功能。

首先，我们在[Sprig库](https://godoc.org/github.com/Masterminds/sprig)中添加了几乎所有的功能。 出于安全原因，我们删除了两个：env和expandenv(这会让chart作者访问Tiller的环境变量).

我们还添加了两个特殊的模板函数：include和required。 include函数允许您引入另一个模板，然后将结果传递给其他模板函数。

例如，这个模板片段包含一个名为mytpl的模板，然后将结果小写，然后用双引号将其换行。

```yaml
value: {{include "mytpl" . | lower | quote}}
```

required的功能允许您根据模板渲染的要求声明特定的values条目。 如果该值为空，则模板渲染将失败并显示用户提交的错误消息。

以下示例函数声明了required的.Values.who条目，并且在缺少该条目时将输出错误消息：

```yaml
value: {{required "A valid .Values.who entry required!" .Values.who }}
```

## 引用字符串，请勿引用整数

当你使用字符串数据时，你总是引用字符串比把它们留为空白字符更安全：

```yaml
name: {{.Values.MyName | quote }}
```

但是，使用整数时不要引用值。 在很多情况下，这可能会导致Kubernetes内部的解析错误。

```yam
port: {{ .Values.Port }}
```

这个注解不适用于为字符串的预先设定env变量值，即使它们表示整数：

```yaml
env:
  -name: HOST
    value: "http://host"
  -name: PORT
    value: "1234"
```

## 使用`include`函数

Go提供了一种使用内置模板指令将一个模板包含在另一个模板中的方法。 但是，Go模板pipeline中不能使用内置函数。

为了能够包含模板，然后对该模板的输出执行操作，Helm有一个特殊的包含函数:

```yaml
{{ include "toYaml" $value | indent 2 }}
```

上面包含一个名为toYaml的模板，将它传递给$value，然后将该模板的输出传递给indent函数。

因为YAML赋予缩进级别和空白的重要性，所以这是包含代码片段的好方法，但是需要再相关的上下文中处理缩进。

## 使用`REQUIRED`函数

Go提供了一种设置模板选项以控制map使用map中不存在的key编制索引时的行为的方法。 这通常使用template.Options（“missingkey = option”）设置，其中选项可以是默认值，零或错误。 将此选项设置为错误将停止执行并出现错误，这将应用于map中每个缺失的key。 可能会出现chart开发人员想要为values.yml文件中的选择values强制实施此行为的情况。

required的函数使开发人员能够根据模板渲染的要求声明value条目。 如果values.yml中的条目为空，模板将不会呈现，并会返回开发人员提供的错误消息。

例如:

```yam
{{ required "A valid foo is required!" .Values.foo }}
```

上面将在定义了.Values.foo时渲染模板，但在未定义.Values.foo时将无法渲染并退出。

## 创建镜像拉取secret

镜像拉的秘钥实质上是注册表，用户名和密码的组合。 您可能需要它们在您正在部署的应用程序中，但要创建它们需要多次运行base64。 我们可以编写一个帮助程序模板来组合Docker配置文件，以用作Secret的有效负载。 这里是一个例子：

首先，假设凭证是在values.yaml文件中定义的，如下所示：

```yaml
imageCredentials:
  registry: quay.io
  username: someone
  password: sillyness
```

然后我们定义我们的帮助模板如下:

```yaml
{{- define "imagePullSecret" }}
{{- printf "{\"auths\": {\"%s\": {\"auth\": \"%s\"}}}" .Values.imageCredentials.registry (printf "%s:%s" .Values.imageCredentials.username .Values.imageCredentials.password | b64enc) | b64enc }}
{{- end }}
```

最后，我们在更大的模板中使用助手模板来创建Secret声明:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myregistrykey
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: {{ template "imagePullSecret" . }}
```

## 在配置或secret变更时自动进行卷展开

通常情况下，configmaps或secrets被作为配置文件注入容器中。 根据应用程序的不同，可能需要重新启动才能使用后续的helm upgrade进行更新，但如果deployment spec本身没有更改，则应用程序会继续与旧配置一起运行，导致部署不一致。

sha256sum函数可用于确保在另一个文件发生更改时更新部署的注解部分:

```yaml
kind: Deployment
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
[...]
```

另请参阅helm upgrade --recreate-pods标志，以解决此问题。

## 告诉TILLER不要删除资源

有时在Helm运行helm删除时不应删除资源。 chart开发人员可以将注解添加到资源以防止被删除。

```yaml
kind: Secret
metadata:
  annotations:
    "helm.sh/resource-policy": keep
[...]
```

(引用是必须的)

注解“helm.sh/resource-policy”：保持在helm delete操作期间指示Tiller跳过该资源。 但是，此资源变成孤儿。 helm将不再以任何方式管理它。 这可能会导致问题，如果使用helm install - 替换已被删除的版本，但保留了资源。

## 使用“PARTIALS”和模板包含

有时候你想在chart中创建一些可重用的部分，无论它们是blockss还是模板部分。 通常，将这些文件保存在自己的文件中会更干净。

在templates /目录中，任何以下划线（\_）开头的文件都不会输出Kubernetes声明文件。 因此，按照惯例，辅助模板和PARTIALS被放置在_helpers.tpl文件中。

## 复杂的chart与许多相关内容

官方chart存储库中的许多chart是用于创建更高级应用程序的“构建块(buiding block)”。 但chart可能用于创建大规模应用程序的实例。 在这种情况下，一张伞形chart可能有多个子chart，每个子chart都起着整体的作用。

从离散部分组成复杂应用程序的当前最佳做法是创建一个公开全局配置的顶级伞形chart，然后使用charts /子目录来嵌入每个组件。

这些项目说明了两种强大的设计模式：

- **SAP**的[**OpenStack chart**](https://github.com/sapcc/openstack-helm) 此chart在Kubernetes上安装完整的OpenStack IaaS。 所有chart都收集在一个GitHub存储库中。

- **Deis**的[**Workflow**](https://github.com/deis/workflow/tree/master/charts/workflow): 这chart表显示了整个Deis PaaS系统的一张chart。 但与SAP chart不同的是，该workflow chart是从每个组件构建而来的，每个组件都在不同的Git存储库中进行跟踪。 查看requirements.yaml文件以查看此chart是如何由其CI / CD管道组成的。

  这两个chart都说明了使用Helm复杂环境的成熟技术。

## YAML是JSON的一个超级

根据YAML规范，YAML是JSON的超集。 这意味着任何有效的JSON结构都应该在YAML中有效。

这有一个优点：有时模板开发人员可能会发现使用类似JSON的语法来表达数据结构更容易，而不是处理YAML的空白敏感度。

作为最佳实践，模板应遵循类似YAML的语法，除非JSON语法大幅降低了格式问题的风险。

## 请小心生成随机值

Helm中有一些函数允许您生成随机数据，加密密钥等。 这些都很好用。 但请注意，在升级过程中，模板会被重新执行。 当模板运行产生与上次运行不同的数据时，将触发该资源的更新。

## 立即升级版本

为了在安装和升级发行版时使用相同的命令，请使用以下命令：

```shell
helm upgrade --install <release name> --values <values file> <chart directory>
```

