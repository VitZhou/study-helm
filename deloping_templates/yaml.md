# YAML技术

本指南的大部分内容都集中在编写模板语言。 在这里，我们将看看YAML格式。 YAML具有一些有用的功能，我们作为模板作者可以使我们的模板更少出错并更易于阅读。

## 标志和集合

根据[YAML规范](http://yaml.org/spec/1.2/spec.html)，有两种类型的集合，以及许多标量类型。

这两种类型的集合是map和sequences：

```yaml
map:
  one: 1
  two: 2
  three: 3

sequence:
  - one
  - two
  - three
```

标量值是单个值（与集合相对）

## YAML中的标量类型

在Helm的YAML方言中，值的标量数据类型由一组复杂的规则确定，包括用于资源定义的Kubernetes模式。 但是，在推断类型时，以下规则往往成立。

如果一个整数或浮点数是不加引号的单词，它通常被视为一个数字类型：

```yaml
count: 1
size: 2.34
```

但如果他们被引用，他们被视为字符串：

```yaml
count: "1" # <-- string, not int
size: '2.34' # <-- string, not float
```

布尔值也是如此：

```yaml
isGood: true   # bool
answer: "true" # string
```

如果值为空value,那么它是null（不是Nil）。

请注意，端口：“80”是有效的YAML，并且将通过模板引擎和YAML分析器，但如果Kubernetes预期端口为整数，则会失败。

在某些情况下，您可以使用YAML节点标签强制进行特定的类型推断：

```yaml
coffee: "yes, please"
age: !!str 21
port: !!int "80"
```

在上面，!! str告诉解析器年龄是一个字符串，即使它看起来像一个int。 并且端口被视为int，尽管它被引用。

## YAML中的字符串

我们放在YAML文档中的大部分数据都是字符串。 YAML有多种表示字符串的方式。 本节将介绍这些方法并演示如何使用其中的一些方法。

有三种“内联”的方式来声明一个字符串：

```yaml
way1: bare words
way2: "double-quoted strings"
way3: 'single-quoted strings'
```

所有内联样式必须位于同一行上。

- bare words没有被引用，并且没有逃脱。 出于这个原因，你必须小心你使用什么字符。
- 双引号字符串可以使用\转义特定字符。 例如“\”hello\“，she said”。 您可以使用\ n来换行。
- 单引号字符串是“literal(字面)”字符串，并且不使用\来转义字符。 唯一的转义序列是''，它被解码为一个'。

除了单行字符串外，还可以声明多行字符串：

```yaml
coffee: |
  Latte
  Cappuccino
  Espresso
```

以上将把coffee的value视为一个相当于Latte\ nCappuccino\ nEspresso\ n的字符串。

请注意，|后的第一行 必须正确缩进。 所以我们可以通过这样做来打破上面的例子：

```yaml
coffee: |
         Latte
  Cappuccino
  Espresso
```

由于Latte被错误地缩进，我们会得到这样的错误：

```shell
Error parsing file: error converting YAML to JSON: yaml: line 7: did not find expected key
```

在模板中，为了防止出现上述错误，在多行文档中放置伪造的“第一行”内容有时更安全：

```yaml
coffee: |
  # Commented first line
         Latte
  Cappuccino
  Espresso
```

> 请注意，无论第一行是什么，它都将保留在字符串的输出中。 因此，如果您使用这种技术将文件内容注入到ConfigMap中，那么该注释应该是任何正在读取该条目的预期类型。

## 控制多行字符串中的空格

在上面的例子中，我们使用了| 表示一个多行字符串。 但请注意，我们的字符串的内容后跟着\ n。 如果我们希望YAML处理器去掉尾随的换行符，我们可以在|后面添加 -

```yaml
coffee: |-
  Latte
  Cappuccino
  Espresso
```

现在，caffee的价值将是：Latte\nCappuccino\nEspresso（没有拖尾\ n）。

其他时候，我们可能希望保留所有尾随空格。 我们可以用| +符号来做到这一点：

```yaml
coffee: |+
  Latte
  Cappuccino
  Espresso


another: value
```

现在caffee的价值将是Latte\nCappuccino\nEspresso\n\n\n。

文本块内部的缩进被保留，并导致保留换行符：

```yaml
coffee: |-
  Latte
    12 oz
    16 oz
  Cappuccino
  Espresso
```

在上述情况下，咖啡将是拿铁\n 12 oz\n 16 oz\nCappuccino\nEspresso。

## 缩进和模板

在编写模板时，您可能会发现自己希望将文件内容注入模板。 正如我们在前面的章节中看到的，有两种方法可以做到这一点：

- 使用\{\{.Files.Get“ FILENAME”\}\}获取chart中文件的内容。
- 使用\{\{include “tempalet” .  \}\}渲染模板，然后将其内容放入chart中。

将文件插入YAML时，最好理解上面的多行规则。 通常情况下，插入静态文件的最简单方法是做这样的：

```yaml
myfile: |
{{ .Files.Get "myfile.txt" | indent 2 }}
```

请注意我们如何执行上面的缩进：indent 2告诉模板引擎使用两个空格缩进“myfile.txt”中的每一行。 请注意，我们不缩进该模板行。 那是因为如果我们做了，第一行的文件内容会缩进两次

## 折叠多行字符串

有时候你想在你的YAML中用多行代表一个字符串，但是当它被解释时，要把它当作一个长行。 这被称为“折叠”。 要声明折叠块，请使用>代替|：

```yaml
coffee: >
  Latte
  Cappuccino
  Espresso
```

以上coffee的value将是Latte Cappuccino Espresso\n。 请注意，除最后一个换行符之外的所有内容都将转换为空格。 您可以将空格控件与折叠的文本标记组合起来，因此> - 将替换或修剪所有换行符。

```yaml
coffee: >-
  Latte
    12 oz
    16 oz
  Cappuccino
  Espresso
```

以上将产生Latte\n 12 oz\n 16 oz\nCappuccino Espresso。 请注意，间距和换行符都在那里。

## 在一个文件中嵌入多个文档

可以将多个YAML文档放入单个文件中。 这是通过在一个新的文档前添加开始标识符—和结束标识符...

```yaml
---
document:1
...
---
document: 2
...
```

在许多情况下，---或...可能被省略。

Helm中的某些文件不能包含多个文档。 例如，如果在values.yaml文件内部提供了多个文档，则只会使用第一个文档。

但是，模板文件可能有多个文档。 发生这种情况时，文件（及其所有文档）在模板呈现过程中被视为一个对象。 但是，最终的YAML在被送到Kubernetes之前被分成多个文件。

我们建议每个文件只在绝对必要时使用多个文档。 在一个文件中有多个文件可能很难调试。

## YAML是JSON的一个超集

因为YAML是JSON的超集，所以任何有效的JSON文档都应该是有效的YAML。

```json
{
  "coffee": "yes, please",
  "coffees": [
    "Latte", "Cappuccino", "Espresso"
  ]
}
```

另一种表达方式：

```yaml
coffee: yes, please
coffees:
- Latte
- Cappuccino
- Espresso
```

这两者可以混合使用（小心）：

```yaml
coffee: "yes, please"
coffees: [ "Latte", "Cappuccino", "Espresso"]
```

以上这三个都应该解析为相同的内部表示。

虽然这意味着像values.yaml这样的文件可能包含JSON数据，但Helm不会将文件扩展名.json视为有效的后缀。

## YAML锚点

YAML规范提供了一种方法来存储对某个值的引用，并稍后通过引用来引用该值。 YAML将此称为“锚点(anchoring)”：

```yaml
coffee: "yes, please"
favorite: &favoriteCoffee "Cappucino"
coffees:
  - Latte
  - *favoriteCoffee
  - Espresso
```

在上面，＆favoriteCoffee设置了对Cappuccino的引用。 之后，该引用被用作*favoriteCoffee。 所以coffees变成Latte，Cappucino，浓Espresso。

虽然在少数情况下锚点是有用的，但它们的一个方面可能导致细微的错误：第一次使用YAML时，引用被扩展，然后被丢弃。

所以如果我们要解码然后重新编码上面的例子，那么产生的YAML将是：

```yaml
coffee: yes, please
favorite: Cappucino
coffees:
- Latte
- Cappucino
- Espresso
```

因为Helm和Kubernetes经常读取，修改并重写YAML文件，锚将会丢失。

