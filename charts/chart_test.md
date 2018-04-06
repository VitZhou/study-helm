# Chart测试

一个chart包含许多一起工作的Kubernetes资源和组件。 作为chart作者，您可能需要编写一些测试来验证chart在安装时是否按预期工作。 这些测试还有助于chart消费者了解您的chart应该做什么。

helm chart中的测试位于templates /目录下，并且是一个pod定义，它指定一个包含给定命令的容器来运行。 容器应该成功退出（退出0），以便测试被认为是成功的。 这个pod定义必须包含一个helm测试钩子注解：helm.sh/hooks：test-success或helm.sh/hooks：test-failure。

示例测试： - 验证来自values.yaml文件的配置是否已正确注入。 - 确保您的用户名和密码正常工作 - 确保不正确的用户名和密码不起作用 - 声明您的服务已启动并正确进行负载均衡 - 等等

您可以使用命令helm test <RELEASE_NAME>在发布版本中运行Helm中的预定义测试。 对于chart使用者来说，这是一种很好的方式来检查他们发布的chart（或应用程序）是否按预期工作。

## helm的测试钩子

在Helm中，有两个测试钩子：test-success和test-fail

test-success表示测试pod应该成功完成。 换句话说，容器中的容器应该退出0.test-fail是一种声明测试pod不能成功完成的方式。 如果容器中的容器未退出0，则表示成功。

## 测试例子

下面是一个示例mariadb图表中helm测试pod定义的示例：

```yaml
mariadb/
  Chart.yaml
  README.md
  values.yaml
  charts/
  templates/
  templates/tests/test-mariadb-connection.yaml
```

在wordpress/templates/tests/test-mariadb-connection.yaml：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-credentials-test"
  annotations:
    "helm.sh/hook": test-success
spec:
  containers:
  - name: {{ .Release.Name }}-credentials-test
    image: {{ .Values.image }}
    env:
      - name: MARIADB_HOST
        value: {{ template "mariadb.fullname" . }}
      - name: MARIADB_PORT
        value: "3306"
      - name: WORDPRESS_DATABASE_NAME
        value: {{ default "" .Values.mariadb.mariadbDatabase | quote }}
      - name: WORDPRESS_DATABASE_USER
        value: {{ default "" .Values.mariadb.mariadbUser | quote }}
      - name: WORDPRESS_DATABASE_PASSWORD
        valueFrom:
          secretKeyRef:
            name: {{ template "mariadb.fullname" . }}
            key: mariadb-password
    command: ["sh", "-c", "mysql --host=$MARIADB_HOST --port=$MARIADB_PORT --user=$WORDPRESS_DATABASE_USER --password=$WORDPRESS_DATABASE_PASSWORD"]
  restartPolicy: Never
```

## 在Release版上运行测试套件的步骤

1. $ helm install mariadb

   ```shell
   NAME:   quirky-walrus
   LAST DEPLOYED: Mon Feb 13 13:50:43 2017
   NAMESPACE: default
   STATUS: DEPLOYED
   ```

2. $ helm test quirky-walrus

   ```Shell
   RUNNING: quirky-walrus-credentials-test
   SUCCESS: quirky-walrus-credentials-test
   ```

> - 您可以在单个yaml文件中定义尽可能多的测试，也可以在`templates/`目录中的多个yaml文件中进行分布
> - 欢迎您将测试套件嵌入到一个`tests/`目录下，`<chart-name>/templates/tests/`以便更好地隔离