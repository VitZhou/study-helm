# subchats和全局values

到目前为止，我们只用一个chart工作。 但是chart可以有称为subcharts的依赖关系，它们也有自己的values和模板。 在本节中，我们将创建一个subcharts并查看我们可以从模板中访问值的不同方式。

在我们深入了解代码之前，需要了解一些有关subcharts的重要细节。

1. Subcharts被认为是“独立的”，这意味着subcharts不能明确依赖于其父图。
2. 因此，subcharts无法访问其父项的values。
3. 父charts可以覆盖subcharts的values。
4. helm有全局values的概念，可以被所有charts访问。

当我们在本节中通过示例时，其中许多概念将变得更加清晰。

## 创建subcharts

对于这些练习，我们将从本指南开始时创建的mychart/chart开始，并在其中添加一个新的charts。

```shell
$ cd mychart/charts
$ helm create mysubchart
Creating mysubchart
$ rm -rf mysubchart/templates/*.*
```

注意，和以前一样，我们删除了所有的基本模板，以便我们可以从头开始。 在本指南中，我们专注于模板如何工作，而不是管理依赖关系。 但chart指南有更多关于subcharts工作的信息。

## 将value和模板添加到subcharts

接下来，我们为mysubchart图表创建一个简单的模板和values文件。 在mychart/charts/mysubchart中应该已经有了一个values.yaml。 我们将这样设置：

```yaml
dessert: cake
```

接下来，我们将在mychart/charts/mysubchart/templates/configmap.yaml目录中创建一个新的ConfigMap模板:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-cfgmap2
data:
  dessert: {{ .Values.dessert }}
```

由于每个subcharts都是独立的chart，因此我们可以自行测试mysubchart：

```yaml
$ helm install --dry-run --debug mychart/charts/mysubchart
SERVER: "localhost:44134"
CHART PATH: /Users/mattbutcher/Code/Go/src/k8s.io/helm/_scratch/mychart/charts/mysubchart
NAME:   newbie-elk
TARGET NAMESPACE:   default
CHART:  mysubchart 0.1.0
MANIFEST:
---
# Source: mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: newbie-elk-cfgmap2
data:
  dessert: cake
```

## 从父chart中覆盖value

我们原来的chart mychart现在是mysubchart的父chart[。 这种关系完全基于mysubchart在mychart/charts中的事实。

由于mychart是父级，我们可以在mychart中指定配置，并将该配置推入mysubchart。 例如，我们可以像这样修改mychart/values.yaml：

```yaml
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions

mysubchart:
  dessert: ice cream
```

请注意最后两行。 mysubchart部分内的任何指令都将被发送到mysubchart chart。 因此，如果我们运行helm install --dry-run --debug mychart，我们将看到的其中一个是mysubchart ConfigMap：

```yaml
# Source: mychart/charts/mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: unhinged-bee-cfgmap2
data:
  dessert: ice cream
```

顶层的value现在已经覆盖了subchart的价值。

这里有一个重要的细节需要注意。 我们没有将mychart/charts/mysubchart/templates/configmap.yaml的模板更改为指向.Values.mysubchart.dessert。 从该模板的角度来看，该值仍位于.Values.dessert。 随着模板引擎一起传递值，它会设置范围。 因此，对于mysubchart模板，只有专门用于mysubchart的值将在.Values中可用。

但有时候，您确实希望某些值可用于所有模板。 这是使用全局chart value完成的。

## 全局chart values

全局values是可以从任何chart或subchart以完全相同的名称访问的values。 全局需要明确声明。 您不能像现有全局一样使用非全局。

Values数据类型有一个名为Values.global的保留部分，可以设置全局值。 我们在mychart/values.yaml文件中设置一个。

```yaml
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions

mysubchart:
  dessert: ice cream

global:
  salad: caesar
```

由于全局变量的工作方式，mychart/templates/configmap.yaml和mysubchart/templates/configmap.yaml应该能够以\{\{.Values.global.salad\}\}的形式访问该值。

`mychart/templates/configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  salad: {{ .Values.global.salad }}
```

`mysubchart/templates/configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-cfgmap2
data:
  dessert: {{ .Values.dessert }}
  salad: {{ .Values.global.salad }}
```

现在，如果我们运行dry run install，我们会在两个输出中看到相同的值：

```shell
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: silly-snake-configmap
data:
  salad: caesar

---
# Source: mychart/charts/mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: silly-snake-cfgmap2
data:
  dessert: ice cream
  salad: caesar
```

全局变量对于传递这样的信息非常有用，但它确实需要一些计划来确保将正确的模板配置为使用全局变量。

## 与subcharts共享模板

父chart和subcharts可以共享模板。 任何chart中的任何定义块都可用于其他chart。

例如，我们可以像这样定义一个简单的模板：

```yaml
{{- define "labels" }}from: mychart{{ end }}
```

回想一下模板上的标签是如何全局共享的。 因此，可以从任何其他chart中包含标签chart。

虽然chart开发人员可以在包含和模板之间进行选择，但使用include的一个优点是可以动态引用模板：

```yaml
{{ include $mytemplate }}
```

以上将取消引用$mytemplate。 相比之下，模板函数只接受字符串文字。

## 避免使用block

Go模板语言提供了一个block关键字，它允许开发人员提供一个默认的实现，这个实现稍后会被覆盖。 在Helm Chart中，块不是重写的最佳工具，因为如果提供了同一个块的多个实现，那么所选的那个是不可预知的。

建议是改为使用include。