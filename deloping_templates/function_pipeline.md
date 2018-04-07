# 模板的函数和pipeline

到目前为止，我们已经看到如何将信息放入模板中。 但是这些信息未经修改就被放入模板中。 有时我们想要以提供给我们更多可用的方式来转换提供的数据。

让我们从一个最佳实践开始：当从.Values对象注入字符串到模板中时，我们应该引用这些字符串。 我们可以通过在模板指令中调用quote函数来实现：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ quote .Values.favorite.drink }}
  food: {{ quote .Values.favorite.food }}
```

模板函数遵循句式functionName arg1 arg2 ....在上面的代码片段中，quote .Values.favorite.drink调用quote函数并将其传递给一个参数。

Helm拥有超过60种可用功能。 其中一些是由[Go模板语言](https://godoc.org/text/template)本身定义的。 其他大多数都是[Sprig模板库](https://godoc.org/github.com/Masterminds/sprig)的一部分。 在我们通过例子进行的过程中，我们会看到其中的很多。

> 虽然我们将Helm模板语言视为Helm特有的，但它实际上是Go模板语言，一些额外函数和各种包装器的组合，以将某些对象暴露给模板。 Go模板上的许多资源在您了解模板时可能会有所帮助。

## PIPELINES

模板语言的强大功能之一是其管道概念。 利用UNIX的一个概念，pipeline是一个链接在一起的一系列模板命令的工具，可以紧凑地表达一系列转换。 换句话说，pipeline是按顺序完成几件事情的有效方式。 我们用pipeline重写上面的例子。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}
  food: {{ .Values.favorite.food | quote }}
```

在这个例子中，我们不是调用quote ARGUMENT，而是颠倒了顺序。 我们使用管道符（|）将“参数”发送给函数：.Values.favorite.drink | quote。 使用管道符，我们可以将几个功能链接在一起：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
```

> 反转订单是模板中的常见做法。 你会看到.val | quote比queto .val更频繁出现.

评估时，该模板将产生以下结果：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: trendsetting-p-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
```

请注意，我们原来的“pizza”现在已经变成了“PIZZA”。

当像这样使用pipeline参数时，第一次评估的结果（.Values.favorite.drink）作为函数的最后一个参数发送。 我们可以修改上面的drink示例来说明一个带有两个参数的函数：repeat COUNT STRING：

```Yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | repeat 5 | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
```

repeat函数将给定的字符串回显给定的次数，所以我们将得到这个输出：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: melting-porcup-configmap
data:
  myvalue: "Hello World"
  drink: "coffeecoffeecoffeecoffeecoffee"
  food: "PIZZA"
```

## 使用DEFUALT功能

模板中经常使用的一个函数是defualt函数：默认DEFAULT_VALUE GIVEN_VALUE。 该功能允许您在模板内部指定默认值，以防该值被省略。 让我们用它来修改上面的drink示例：

```yaml
drink: {{ .Values.favorite.drink | default "tea" | quote }}
```

如果我们照常运行，我们会得到我们的coffee：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: virtuous-mink-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
```

现在，我们将从values.yaml中移除最喜欢的drink设置：

```yaml
favorite:
  #drink: coffee
  food: pizza
```

现在重新运行helm install --dry-run --debug ./mychart会产生这个YAML：

```shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fair-worm-configmap
data:
  myvalue: "Hello World"
  drink: "tea"
  food: "PIZZA"
```

在实际的chart中，所有静态默认值都应该存在于values.yaml中，并且不应该使用default命令重复（否则它们将是多余的）。 但是，default命令无法在values.yaml中声明的计算是非常有用的。 例如：

```yaml
drink: {{ .Values.favorite.drink | default (printf "%s-tea" (include "fullname" .)) }}
```

在某些地方,有if条件的使用default比在values.yaml声明更适合. 我们将在下一节中看到这些。

模板函数和管道是一种转换信息并将其插入到YAML中的强大方法。 但有时候需要添加一些比插入字符串更复杂一些的模板逻辑。 在下一节中，我们将看看模板语言提供的控制结构。

## 运算符也是函数

对于模板，运算符（eq，ne，lt，gt等等）都是作为函数实现的。 在管道中，操作可以用括号（（和））分组。

现在我们可以使用条件，循环和范围修饰符从函数和pipeline转向流控制(flow control)。

