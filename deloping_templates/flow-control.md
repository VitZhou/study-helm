# 流控制

控制结构（模板说法中称为“操作”）为模板作者提供了控制模板生成流程的能力。 Helm的模板语言提供了以下控制结构：

- `if`/`else: 用于创建条件块
- with: 指定范围
- range: 它提供了“for-each”风格的循环

除此之外，它还提供了一些声明和使用命名模板段的操作：

- define: 在模板中声明一个新的命名模板
- template: 导入一个Named模板
- block: 声明了一种特殊的可填写模板区域

在本节中，我们将讨论if，with和range。 其他内容在本指南后面的“Named模板”一节中介绍。

## IF/ELSE

我们要看的第一个控制结构是用于在模板中有条件地包含文本块。 这是if/else块。

条件的基本结构如下所示：

```yaml
{{ if PIPELINE }}
  # Do something
{{ else if OTHER PIPELINE }}
  # Do something else
{{ else }}
  # Default case
{{ end }}
```

请注意，我们现在讨论的是管道而不是values。 其原因是要明确控制结构可以执行整个管道，而不仅仅是评估一个values。

如果values是以下内容，则管道评估为false:

- 一个为false的布尔值
- 一个为0的数字
- 一个空字符串
- nil(empty或者null)
- 一个空集合(`map`, `slice`, `tuple`, `dict`, `array`)

在所有其他条件下，条件都是true。

让我们为ConfigMap添加一个简单的条件。 如果drink被设置为coffee，我们将添加另一个设置：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{ if eq .Values.favorite.drink "coffee" }}mug: true{{ end }}
```

由于我们在最后一个例子中注入了drink：coffee，所以输出不应该包括mug：true; 但是，如果我们将该行添加到我们的values.yaml文件中，则输出应如下所示：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: eyewitness-elk-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  mug: true
```

> 这个例子必须在Values:yaml文件中设置drink的值,否则会抛出error calling eq: invalid type for comparison异常.因为如果不设置的话,.Values.favorite.drink会返回nil.

## 控制空白区域

在查看条件时，我们应该快速查看模板中的空白区域的控制方式。 让我们看一下前面的例子，为更容易阅读将其格式化：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{if eq .Values.favorite.drink "coffee"}}
    mug: true
  {{end}}
```

起初，这看起来不错。 但是如果我们通过模板引擎运行它，我们会得到一个不幸的结果：

```shell
$ helm install --dry-run --debug ./mychart
SERVER: "localhost:44134"
CHART PATH: /Users/mattbutcher/Code/Go/src/k8s.io/helm/_scratch/mychart
Error: YAML parse error on mychart/templates/configmap.yaml: error converting YAML to JSON: yaml: line 9: did not find expected key
```

发生了什么？ 由于上面的空白区域，我们生成了不正确的YAML。

```Shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: eyewitness-elk-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
    mug: true
```

Mug被错误地缩进。 让我们简单地调整一下空白，然后重新运行：

```shell
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{if eq .Values.favorite.drink "coffee"}}
  mug: true
  {{end}}
```

当我们发送该信息时，我们会得到有效的YAML，但仍然看起来有点滑稽：

```shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: telling-chimp-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"

  mug: true
  
```

请注意，我们在YAML中收到了一些空行。 为什么？ 当模板引擎运行时，它将删除\{\{和\}\}中的内容，但它完全按原样保留剩余的空白。

YAML将保留空白，因此管理空白变得非常重要。 幸运的是，Helm模板有一些工具可以帮助。

首先，可以使用特殊字符修改模板声明的大括号语法，以告诉模板引擎填充空白。\{\{ - （添加了短划线和空格）表示应该将空白空格左移，而 - \}\}表示空格右侧的空格应该被使用。 小心！ 换行符也是空格！

> 确保 - 符号和您的指令的其余部分之间有空格。\{\{ - 3\}\}表示“修剪左空白并打印3”，\{\{-3\}\}表示“打印-3”。

使用这个语法，我们可以修改我们的模板来摆脱这些行:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{- if eq .Values.favorite.drink "coffee"}}
  mug: true
  {{- end}}
```

为了清楚说明这一点，让我们调整上述内容，然后将\*替换为将按照此规则删除的所有空白。 该行末尾的\*表示将被删除的换行符.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}*
**{{- if eq .Values.favorite.drink "coffee"}}
  mug: true*
**{{- end}}
```

我们可以通过Helm运行我们的模板并查看结果：

```shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: clunky-cat-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  mug: true
```

> 我没测试成功,目前不知道哪里出了问题,建议使用 - 修饰符

小心使用chomping(-)修饰符。 这样做很容易产生意外：

```yaml
food: {{ .Values.favorite.food | upper | quote }}
  {{- if eq .Values.favorite.drink "coffee" -}}
  mug: true
  {{- end -}}
```

这将产生food:“PIZZA” mug:true，因为它消除了双方的\{\{和\}\}符号。

> 有关模板中空白控件的详细信息，请参阅[官方Go模板文档](https://godoc.org/text/template)

最后，有时候为了更容易告诉模板系统如何缩进，而不是试图去测试清楚模板指令的间距。 因此，有时会发现使用indent函数（\{\{indent 2“mug：true”\}\}）是非常有用的。

## 使用with修改作用域

下一个要看的控制函数是with action。 这控制着变量的作用域。  .(句号) 是对当前范围的引用。 所以.Values告诉模板在当前范围内找到Values对象。

with的语法与简单的if语句相似:

```yaml
{{ with PIPELINE }}
  # restricted scope
{{ end }}
```

范围是可以改变的,witsh可以允许你设置当前的范围(.).例如:我们一直在处理.Values.favorites。 让我们重写我们的ConfigMap来改变。 范围指向.Values.favorites：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
```

（请注意，我们删除了前面练习中的if条件）

请注意，现在我们可以参考.drink和.food，而不需要限制它们。 这是因为with语句集。 指向.Values.favorite。 这个。 在\{\{end\}\}之后重置为之前的范围。

但是请注意！ 在受限范围内，您将无法从父范围访问其他对象。 例如，这将失败：

```yaml
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  release: {{ .Release.Name }}			
  {{- end }}
```

它会产生一个错误，因为Release.Name不在限制范围内。但是，如果我们交换最后两行，所有将按预期工作，因为范围在\{\{end\}\}之后重置。

```yaml
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
  release: {{ .Release.Name }}
```

查看范围之后，我们将看看模板变量，它提供了上述范围问题的一个解决方案。

## 使用rang循环

许多编程语言都支持使用for循环，foreach循环或类似的功能机制进行循环。 在Helm的模板语言中，遍历集合的方式是使用rang运算符。

首先，让我们在我们的values.yaml文件中添加一份披萨配料列表：

```yaml
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions
```

现在我们有一个pizzaToppings列表（称为模板中的一个切片(slice)）。 我们可以修改我们的模板，将这个列表打印到我们的ConfigMap中：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
  toppings: |-
    {{- range .Values.pizzaToppings }}
    - {{ . | title | quote }}
    {{- end }}
```

让我们仔细看看toppings：列表。 rang函数将“覆盖范围”(遍历)pizzaToppings列表。 但现在发生了一些有趣的事就像with范围一样，rang操作符也是如此。 每次通过循环，.被设置为当前的比萨饼toppings。 第一次，.被设置为mushrooms。 第二次迭代它被设置为cheese，等等。

我们可以在pipeline下发送value,所以当我们使用\{\{ . | title | quote \}\}时,它将.发送到title并且引用.我们运行这个模板,得到的结果:

```shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: edgy-dragonfly-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  toppings: |-
    - "Mushrooms"
    - "Cheese"
    - "Peppers"
    - "Onions"
```

现在，在这个例子中，我们做了一些棘手的事情。 topping：| - line声明一个多行字符串。 所以我们的topping声明实际上不是YAML清单。 这是一个很大的字符串。 我们为什么要这样做？ 因为ConfigMaps数据中的数据由键/值对组成，其中键和值都是简单字符串。 要理解这种情况，请查看[Kubernetes ConfigMap文档](http://kubernetes.io/docs/user-guide/configmap/)。 但对我们来说，这个细节并不重要。

> YAML中的| - 标记需要多行字符串。 这可以是一种有用的技术，用于在您的声明中嵌入大块数据，如此处所示。

有时能够快速在模板中创建一个列表，然后遍历该列表很有用。 helm模板有一个功能可以使这个变得简单：tuple。 在计算机科学中，tuple是类固定大小的列表类集合，但是具有任意数据类型。 以下粗略地表达了tuple被使用的方式:

```yaml
sizes: |-
    {{- range tuple "small" "medium" "large" }}
    - {{ . }}
    {{- end }}
```

以上将产生这样的结果：

```yaml
  sizes: |-
    - small
    - medium
    - large
```

除了list和tuple之外，范围还可以用于迭代具有键和值的集合（如map或dict）。 当我们介绍模板变量时，我们将在下一节看到如何做到这一点。