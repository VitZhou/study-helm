# 访问模板内部文件

在上一节中，我们介绍了几种创建和访问命名模板的方法。 这可以很容易地从另一个模板中导入一个模板。 但有时需要导入不是模板的文件，并注入其内容而不通过模板渲染器发送内容。

Helm通过.Files对象提供对文件的访问。 然而，在我们开始使用模板示例之前，需要注意以下几点：

- 向您的helm chart添加额外的文件是可以的。 这些文件将被捆绑并发送给Tiller。 不过要小心。 由于Kubernetes对象的存储限制，chart必须小于1M。
- 通常出于安全原因，某些文件无法通过.Files对象访问。
  - 模板中的文件/无法访问。
  - 使用.helmignore排除的文件无法访问。
- chart不保留UNIX模式信息，因此文件级权限对文件的可用性没有影响。

## 基本实例

我们编写一个模板，将三个文件读入我们的ConfigMap。 首先，我们将三个文件添加到图表中，将所有三个文件直接放在mychart/目录中。

`config1.toml`:

```yaml
message = Hello from config 1
```

`config2.toml`:

```yaml
message = This is config 2
```

config3.toml：

```yaml
message = Goodbye from config 3
```

这些都是一个简单的TOML文件（想想老派的Windows INI文件）。 我们知道这些文件的名称，因此我们可以使用范围函数来遍历它们并将其内容注入到ConfigMap中。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  {{- $files := .Files }}
  {{- range tuple "config1.toml" "config2.toml" "config3.toml" }}
  {{ . }}: |-
    {{ $files.Get . }}
  {{- end }}
```

这个配置映射使用了前几节讨论的几种技术。 例如，我们创建一个$ files变量来保存对.Files对象的引用。 我们还使用tuple函数来创建我们循环访问的文件列表。 然后我们打印每个文件名 \{\{ \. \}\}：| - ，然后打印文件\{\{\$files.Get.\}\}。

运行这个模板将产生一个包含所有三个文件内容的ConfigMap：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: quieting-giraf-configmap
data:
  config1.toml: |-
    message = Hello from config 1

  config2.toml: |-
    message = This is config 2

  config3.toml: |-
    message = Goodbye from config 3
```

## 路径助手

在处理文件时，对文件路径本身执行一些标准操作会非常有用。 为了解决这个问题，Helm从Go的路径包中导入了许多功能供您使用。 它们都可以使用Go包中的相同名称访问，但使用小写的第一个字母。 例如，Base成为base等。

导入的功能是： -Base-Dir-Ext-IsAbs-Clean

## 全局路径

随着chart的增长，您可能会发现您需要更多地组织文件，因此我们提供了一个Files.Glob（模式字符串）方法来协助提取具有所有glob模式灵活性的特定文件。

.Glob返回一个文件类型，因此您可以调用返回对象上的任何文件方法。

例如，想象一下目录结构：

```yaml
foo/: 
  foo.txt foo.yaml

bar/:
  bar.go bar.conf baz.yaml
```

Gloob有多种选择：

```yaml
{{ range $path := .Files.Glob "**.yaml" }}
{{ $path }}: |
{{ .Files.Get $path }}
{{ end }}
```

或者

```yaml
{{ range $path, $bytes := .Files.Glob "foo/*" }}
{{ $path }}: '{{ b64enc $bytes }}'
{{ end }}
```

## CONFIGMAP和SECRETS实用程序功能

> 不存在于2.0.2或更早的版本中

想要将文件内容放入configmaps和secrets中非常常见，以便在运行时安装到您的pod中。 为了解决这个问题，我们在Files类型上提供了一些实用的方法。

为进一步组织，将这些方法与Glob方法结合使用尤其有用。

根据上面的Glob示例中的目录结构：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: conf
data:
{{ (.Files.Glob "foo/*").AsConfig | indent 2 }}
---
apiVersion: v1
kind: Secret
metadata:
  name: very-secret
type: Opaque
data:
{{ (.Files.Glob "bar/*").AsSecrets | indent 2 }}
```

##  编码

您可以导入一个文件，并使用base-64模板对其进行编码以确保成功传输：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Release.Name }}-secret
type: Opaque
data:
  token: |-
    {{ .Files.Get "config1.toml" | b64enc }}
```

以上将采用我们之前使用的相同config1.toml文件并对其进行编码：

```yaml
# Source: mychart/templates/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: lucky-turkey-secret
type: Opaque
data:
  token: |-
    bWVzc2FnZSA9IEhlbGxvIGZyb20gY29uZmlnIDEK
```

## LINES

有时需要访问模板中文件的每一行。 我们为此提供了一种方便的lines方法。

```yaml
data:
  some-file.txt: {{ range .Files.Lines "foo/bar.txt" }}
    {{ . }}{{ end }}
```

目前，无法在helm安装期间将文件外部传递到chart。 因此，如果您要求用户提供数据，则必须使用helm install -f或helm install -set加载它。

这个讨论将我们潜入写作Helm模板的工具和技术中。 在下一节中，我们将看到如何使用一个特殊文件templates/NOTES.txt将安装后说明发送给chart用户。

