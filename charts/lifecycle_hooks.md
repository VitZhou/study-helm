# 钩子

Helm提供了一个钩子机制，允许chart开发人员在发布的生命周期中的某些点进行干预。 例如，您可以使用钩子来：

- 在加载任何其他chart之前，在安装期间加载ConfigMap或Secret。
- 执行作业以在安装新chart之前备份数据库，然后在升级后执行第二个作业以恢复数据。
- 在删除发行版之前运行作业，以便在删除发行版之前优雅地停止服务。

钩子像常规模板一样工作，但它们具有特殊的注释，可以使Helm以不同的方式使用它们。 在本节中，我们介绍钩子的基本使用模式。

## 可用的钩子

定义了以下钩子:

- pre-install: 在模板渲染后执行，但在Kubernetes中创建任何资源之前执行。
- post-install: Kubernetes加载完所有资源后执行
- pre-delete: 在从Kubernetes删除任何资源的之前执行删除请求。
- post-delete: 在删除所有版本资源后执行删除请求
- pre-upgrade: 在模板渲染之后，但在将任何资源加载到Kubernetes之前（例如，在Kubernetes应用操作之前）执行升级请求。
- post-upgrade: 所有资源升级后执行升级。
- pre-rollback: 在渲染模板之后但在任何资源回滚之前执行回滚请求。
- post-rollback: 在修改所有资源后执行回滚请求。

## 钩子和Release生命周期

钩子让chart开发人员有机会在release生命周期中的战略点执行操作。 例如，helm install的生命周期。 默认情况下，生命周期如下所示：

1.  用户运行helm install foo
2. chart被加载到Tiller中
3. 经过一些验证后，Tiller渲染foo模板
4. Tiller将产生的资源加载到Kubernetes中
5. Tiller将版本名称（和其他数据）返回给客户端
6. 客户端退出

Helm为安装生命周期定义了两个钩子：pre-install和post-install。 如果foo chart的开发人员实现了两个钩子，则生命周期会如此更改：

1. 用户运行helm install foo
2. chart被加载到Tiller中
3. 经过一些验证后，Tiller渲染foo模板
4. Tiller准备执行pre-install钩子（将钩子资源加载到Kubernetes中）
5. Tiller会根据权重对钩子进行排序（默认分配权重0），并按相同权重的钩子按升序排序。
6. Tiller然后装载最低权重的钩子
7. Tiller等待，直到钩子“准备就绪”
8. Tiller将产生的资源加载到Kubernetes中。 请注意，如果设置了--wait标志，则Tiller将等待，直到所有资源都处于就绪状态，并且在准备就绪之前不会运行post-install挂钩。
9. Tiller执行后安装钩子（加载钩子资源）
10. Tiller等待，直到钩子“准备就绪”
11. Tiller将版本名称（和其他数据）返回给客户端
12. 客户端退出

等到钩子准备就绪是什么意思？ 这取决于在钩子中声明的资源。 如果资源是作业类型，则Tiller将等待作业成功完成。 如果作业失败，则发布失败。 这是一个阻塞操作，所以Helm客户端会在Job运行时暂停。

对于所有其他类型，只要Kubernetes将资源标记为加载（添加或更新），资源就被视为“就绪”。 当一个钩子声明了很多资源时，这些资源将被串行执行。 如果他们有钩子权重（见下文），他们按照权重顺序执行。 否则,排序不能保证。 （在Helm 2.3.0及之后的版本中，它们按字母顺序排序，但这种行为不具有约束力，未来可能会发生变化）。添加钩子权重被认为是一种好的做法，如果权重不重要可以设置为0。

### 钩子资源不用相应的版本进行管理

钩子创建的资源不作为Release的一部分进行跟踪或管理。 一旦Tiller验证钩子已经达到其就绪状态，它将单独离开钩子资源。

实际上，这意味着如果您在钩子中创建资源，则不能依靠helm delete来删除资源。 要销毁这些资源，您需要编写代码在pre-delete或post-delete钩子中执行此操作，或将“helm.sh/hook-delete-policy”注解添加到钩子模板文件。

## 等待钩子

钩子只是Kubernetes声明文件，在元数据部分有特殊的注解。 因为它们是模板文件，所以可以使用所有常规模板功能，包括读取.Values，.Release和.Template。

例如，存储在templates/post-install-job.yaml中的此模板声明了要在post-install运行的作业：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{.Release.Name}}"
  labels:
    heritage: {{.Release.Service | quote }}
    release: {{.Release.Name | quote }}
    chart: "{{.Chart.Name}}-{{.Chart.Version}}"
  annotations:
    # This is what defines this resource as a hook. Without this line, the
    # job is considered part of the release.
    "helm.sh/hook": post-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    metadata:
      name: "{{.Release.Name}}"
      labels:
        heritage: {{.Release.Service | quote }}
        release: {{.Release.Name | quote }}
        chart: "{{.Chart.Name}}-{{.Chart.Version}}"
    spec:
      restartPolicy: Never
      containers:
      - name: post-install-job
        image: "alpine:3.3"
        command: ["/bin/sleep","{{default "10" .Values.sleepyTime}}"]
```

下面这个annotaions使得模板成为钩子:

```yaml
  annotations:
    "helm.sh/hook": post-install
```

一个资源可以实现多个挂钩：

```yaml
annotations:
    "helm.sh/hook": post-install,post-upgrade
```

同样，可以将给定钩子的实现为不同资源的数量没有限制。 例如，我们可以将secret和config map声明为pre-install钩子。

子chart声明钩子时，也会评估这些钩子。 顶级chart无法禁用子chart所声明的钩子。

有可能为一个钩子定义一个权重，这将有助于建立一个确定性的执行顺序。 权重使用以下注解来定义：

```yaml
annotations:
    "helm.sh/hook-weight": "5"
```

钩子权重可以是正数或负数，但必须表示为字符串。 当Tiller开始执行一个特定类型的钩子的执行周期时，它将按升序对这些钩子进行排序。

还可以定义何时删除相应的钩子资源的策略。 钩子删除策略使用以下注解来定义：

```Yaml
 annotations:
    "helm.sh/hook-delete-policy": hook-succeeded
```

使用“helm.sh/hook-delete-policy”注解时，可以从“hook-succeeded”和“hook-failed”中选择其值。 值“hook-succeeded”指定Tiller应该在成功执行钩子后删除钩子，而值“hook-failed”指定Tiller应该在执行期间钩子失败时删除钩子。

