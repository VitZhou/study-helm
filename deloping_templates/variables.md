# 变量

通过我们的function，pipeline，object和控制结构，我们可以在许多编程语言中找到更基本的想法之一：变量。 在模板中，它们使用的频率较低。 但我们将看到如何使用它们来简化代码，并更好地使用with和range。

在前面的例子中，我们看到这段代码会失败：

```yaml
 {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  release: {{ .Release.Name }}
  {{- end }}
```

Release.Name不在with块中限制的范围内。 解决范围问题的一种方法是将对象分配给可以在不考虑当前范围的情况下访问的变量。

在Helm模板中，变量是对另一个对象的命名引用。 它遵循形式$name。 变量分配一个特殊的赋值运算符：:=。 我们可以为Release.Name使用一个变量来重写上面的内容。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- $relname := .Release.Name -}}
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  release: {{ $relname }}
  {{- end }}
```

注意，我们在with块之前，分配\$ relname：= .Release.Name。 现在在with块中，$ relname变量仍指向发布名称。

运行会产生这样的结果：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: viable-badger-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  release: viable-badger
```

变量在rang循环中特别有用。 它们可以用于类似列表的对象以同时捕获索引和值：

```yaml
toppings: |-
    {{- range $index, $topping := .Values.pizzaToppings }}
      {{ $index }}: {{ $topping }}
    {{- end }}
```

请注意，rang首先是变量，然后是赋值运算符，然后是list。 这会将整数索引（从零开始）分配给$ index，并将值分配给$ topping。 运行它将产生：

```yaml
 toppings: |-
      0: mushrooms
      1: cheese
      2: peppers
      3: onions
```

对于同时具有键和值的数据结构，我们可以使用rang来获取两者。 例如，我们可以像这样循环访问.Values.favorite：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end}}
```

现在在第一次迭代中，\$ key将是drink，\$ val将是coffee，第二次，\$key将是food，$ val将是比pizza。 运行上面的代码会生成这个：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: eager-rabbit-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
```

变量通常不是“全局”的。 它们的范围是它们所在的块。 之前，我们在模板的顶层分配了\$ relname。 该变量将在整个模板的范围内。 但在我们的最后一个例子中，\$ key和\$ val只会在\{\{range …\}\} \{\{end\}\}块的范围内。

然而，总有一个变量是全局的 - $ - 这个变量总是指向root上下文。 当你在需要知道chart release名称的范围内循环时，这可能非常有用。

例如:

```yaml
{{- range .Values.tlsSecrets }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ .name }}
  labels:
    # Many helm templates would use `.` below, but that will not work, 
    # however `$` will work here 
    app: {{ template "fullname" $ }}
    # I cannot reference .Chart.Name, but I can do $.Chart.Name
    chart: "{{ $.Chart.Name }}-{{ $.Chart.Version }}"
    release: "{{ $.Release.Name }}"
    heritage: "{{ $.Release.Service }}"
type: kubernetes.io/tls
data:
  tls.crt: {{ .certificate }}
  tls.key: {{ .key }}
---
{{- end }}
```

到目前为止，我们只查看了一个文件中声明的一个模板。 但是Helm模板语言的强大功能之一是它能够声明多个模板并将它们一起使用。 我们将在下一节谈谈。