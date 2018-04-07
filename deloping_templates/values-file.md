# Values文件

在上一节中，我们看了Helm模板提供的内置对象。Values是四个内置对象之一。 该对象提供对传入chart的values的访问。 其内容来自四个来源：

- chart中的values.yaml文件
- 如果这是一个子chart，来自父chart的values.yaml文件
- Values文件如果通过-f标志传递给helm install或helm update（helm install -f myvals.yaml ./mychart）
- 个别参数通过--set设置（例如helm install --set foo = bar ./mychart）

上面的列表按照特定的顺序排列：values.yaml是默认值，可以由父chart的values.yaml覆盖，而这些values可以被用户提供的values文件覆盖，而这些值又可以被—set参数覆盖。

values文件是纯YAML文件。 让我们编辑mychart/values.yaml，然后编辑我们的ConfigMap模板。

删除values.yaml中的默认值，我们将只设置一个参数：

```yaml
favoriteDrink: coffee
```

现在我们可以在模板中使用这个：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favoriteDrink }}
```

注意最后一行，我们将favoriteDrink作为Values的一个属性进行访问：\{\{.Values.favoriteDrink\}\}。

让我们看看这是如何渲染:

```shell
$ helm install --dry-run --debug ./mychart
SERVER: "localhost:44134"
CHART PATH: /Users/mattbutcher/Code/Go/src/k8s.io/helm/_scratch/mychart
NAME:   geared-marsupi
TARGET NAMESPACE:   default
CHART:  mychart 0.1.0
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: geared-marsupi-configmap
data:
  myvalue: "Hello World"
  drink: coffee
```

由于favoriteDrink在默认的values.yaml文件中设置为coffe，这就是模板中显示的值。 我们可以通过在我们的helm install调用中添加一个--set标志来轻松覆盖它：

```shell
helm install --dry-run --debug --set favoriteDrink=slurm ./mychart
SERVER: "localhost:44134"
CHART PATH: /Users/mattbutcher/Code/Go/src/k8s.io/helm/_scratch/mychart
NAME:   solid-vulture
TARGET NAMESPACE:   default
CHART:  mychart 0.1.0
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: solid-vulture-configmap
data:
  myvalue: "Hello World"
  drink: slurm
```

由于--set比默认的values.yaml文件具有更高的优先级，因此我们的模板会生成drink:slurm。

值文件也可以包含更多结构化内容。 例如，我们可以在我们的values.yaml文件中创建一个favorite，然后在其中添加几个键：

```yaml
favorite:
  drink: coffee
  food: pizza
```

现在我们必须稍微修改模板：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink }}
  food: {{ .Values.favorite.food }}
```

虽然以这种方式构建数据是可能的，但建议您不要将values树设置得太深。 当我们看看为子chart分配values时，我们将看到如何使用树结构来命名values。

## 删除默认key

如果您需要从默认值中删除一个key，则可以将该key的值覆盖为空，在这种情况下，Helm将从覆盖的values merge中删除该key。

例如，稳定的Drupal chart允许配置liveness probe，以防配置自定义chart。 以下是默认值：

```yaml
livenessProbe:
  httpGet:
    path: /user/login
    port: http
  initialDelaySeconds: 120
```

如果您尝试使用--set livenessProbe.exec.command = [cat，docroot/CHANGELOG.txt]覆盖livenessProbe处理程序来执行而不是httpGet，Helm会将默认和覆盖的key合并在一起，从而产生以下YAML：

```yaml
livenessProbe:
  httpGet:
    path: /user/login
    port: http
  exec:
    command:
    - cat
    - docroot/CHANGELOG.txt
  initialDelaySeconds: 120
```

但是，Kubernetes会失败，因为您无法声明多个livenessProbe处理程序。 为了克服这个问题，你可以指示Helm通过设置它为null来删除livenessProbe.httpGet：

```shell
helm install stable/drupal --set image=my-registry/drupal:0.1.0 --set livenessProbe.exec.command=[cat,docroot/CHANGELOG.txt] --set livenessProbe.httpGet=null
```

到此，我们已经看到了几个内置对象，并用它们将信息注入到模板中。 现在我们来看看模板引擎的另一个方面：函数和管道。