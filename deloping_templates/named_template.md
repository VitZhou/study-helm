# 命名模板

现在是时候超越一个模板，并开始创建其他模板。 在本节中，我们将看到如何在一个文件中定义命名模板，然后在别处使用它们。 命名模板（有时称为部分模板或子模板）只是一个在文件内定义的模板，并给定了一个名称。 我们将看到两种创建方法，以及几种不同的使用方法。

在“流程控制”部分，我们介绍了三种用于声明和管理模板的操作：define，template和block。 在本节中，我们将介绍这三个操作，并介绍一个与模板操作类似的特殊用途包含功能。

## 部分和_文件

到目前为止，我们已经使用了一个文件，而且一个文件包含一个模板。 但Helm的模板语言允许您创建指定的嵌入模板，可以通过其他名称访问。

在开始编写这些模板之前，有一些文件命名约定值得一提：

- 模板中的大多数文件被视为包含Kubernetes声明
- NOTES.txt是一个例外
- 但是，名称以下划线（_）开头的文件被假定为没有内部声明。 这些文件不会呈现给Kubernetes对象定义，而是在其他chart模板中随处可用以供使用。

这些文件用于存储部分和帮助程序。 事实上，当我们第一次创建mychart时，我们看到一个名为_helpers.tpl的文件。 该文件是模板部分的默认位置。

## 声明和使用DEFINE,TEMPALTE函数使用模板

define操作允许我们在模板文件内创建一个命名模板。 它的语法如下所示：

```yaml
{{ define "MY_NAME" }}
  # body of template here
{{ end }}
```

例如，我们可以define一个模板来封装一个Kubernetes标签块：

```yaml
{{- define "my_labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
```

现在我们可以将此模板嵌入到现有的ConfigMap中，然后将其包含在模板操作中：

```yaml
{{- define "my_labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "my_labels" }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

当模板引擎读取该文件时，它将保存对my_labels的引用，直到调用模板“my_labels”。 然后它将内联呈现该模板。 所以结果如下所示：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: running-panda-configmap
  labels:
    generator: helm
    date: 2016-11-02
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
```

传统上，Helm chart将这些模板放在partials文件中，通常是_helpers.tpl。 让我们在这里移动这个功能：

```yaml
{{/* Generate basic labels */}}
{{- define "my_labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
```

按照惯例，定义函数应该有一个简单的文档块（\{\{/ * ... * /\}\}）来描述它们的功能。

尽管这个定义在_helpers.tpl中，但它仍然可以在configmap.yaml中访问：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "my_labels" }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

命名模板时要记住一个非常重要的细节：模板名称是全局的。 如果您声明两个具有相同名称的模板，则无论最后加载哪一个模板都将是可使用的模板。 由于子chart中的模板与顶级模板一起编译，因此应该小心地使用特定于chart的名称命名模板。

一种流行的命名约定是为每个define的模板添加chart名称：

```yaml
{{ define "mychart.labels" }} or {{ define "mychart_labels" }}.
```

## 设置模板的范围

在我们上面定义的模板中，我们没有使用任何对象。 我们只是使用函数。 让我们修改我们定义的模板以包含chart名称和chart版本：

```Yaml
{{/* Generate basic labels */}}
{{- define "my_labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
    chart: {{ .Chart.Name }}
    version: {{ .Chart.Version }}
{{- end }}
```

如果我们这样做，结果将不会是我们所期望的：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: moldy-jaguar-configmap
  labels:
    generator: helm
    date: 2016-11-02
    chart:
    version:
```

名称和版本发生了什么变化？ 他们不在我们定义的模板的范围内。 当一个已命名的模板（由define创建）被呈现时，它将接收由模板调用传入的作用域。 在我们的例子中，我们包含了这样的模板：

```yaml
{{- template "my_labels" }}
```

没有范围被传入，所以在模板中，我们无法访问任何内容.(句号).但这很容易解决。 我们只需将范围传递给模板：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "my_labels" . }}
```

请注意，在模板调用结束位置我们使用了.(句号)。 我们可以轻松地通过.Values或.Values.favorite或任何我们想要的范围。 但是我们想要的是顶级范围。

现在当我们用helm install --dry-run --debug ./mychart执行这个模板时，我们得到了这个：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: plinking-anaco-configmap
  labels:
    generator: helm
    date: 2016-11-02
    chart: mychart
    version: 0.1.0
```

现在\{\{.Chart.Name\}\}解析为mychart，\{\{.Chart.Version\}\}解析为0.1.0。

## include函数

假设我们已经定义了一个如下所示的简单模板：

```yaml
{{- define "mychart_app" -}}
app_name: {{ .Chart.Name }}
app_version: "{{ .Chart.Version }}+{{ .Release.Time.Seconds }}"
{{- end -}}
```

现在说我想将它插入到我的模板的lables：部分以及data：部分：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
    {{ template "mychart_app" .}}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
  {{ template "mychart_app" . }}
```

结果将不会是我们所期望的：

```Shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: measly-whippet-configmap
  labels:
    app_name: mychart
app_version: "0.1.0+1478129847"
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
  app_name: mychart
app_version: "0.1.0+1478129847"
```

请注意，app_version上的缩进在两个地方都是错误的。 为什么？ 因为被替换的模板将文本对齐到右侧。 由于模板是一个动作，而不是一个函数，因此无法将模板调用的输出传递给其他函数; 数据只是内嵌插入。

为了解决这个问题，Helm提供了一个模板的替代方法，它可以将模板的内容导入到当前的管道中，并将它传递给管道中的其他功能。

以上是上面的示例，更正为使用缩进来正确缩进mychart_app模板：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
{{ include "mychart_app" . | indent 4 }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
{{ include "mychart_app" . | indent 2 }}
```

现在生成的YAML对每个部分都正确缩进：

```Shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: edgy-mole-configmap
  labels:
    app_name: mychart
    app_version: "0.1.0+1478129987"
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
  app_name: mychart
  app_version: "0.1.0+1478129987"
```

> 在Helm模板中使用include 导入外部模板被认为是最好的方式，这样可以更好地为YAML文档处理输出格式。

有时我们想要导入内容，但不是作为模板。 也就是说，我们要逐字输入文件。 我们可以通过下一节中介绍的.Files对象访问文件来实现这一点。