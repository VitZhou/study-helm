# 使用Helm

本指南解释了使用Helm（和Tiller）来管理Kubernetes群集上的软件包的基础知识。 它假定您已经安装了Helm客户端和Tiller服务器(通常是helm init)

###  三大理念

Chart是helm包,它包含在Kubernetes集群内部运行应用程序，工具或服务所需的所有资源定义。 可以把它想象成一个Homebrew公式，一个Apt dpkg或一个Yum RPM文件的Kubernetes等价物。

存储库是可以收集和共享Chart的地方。 这就像Perl的CPAN档案或Fedora软件包数据库，对于Kubernetes来说是软件包。

Release是发布在k8s集群中运行的chart的实例.一个chart通常可以多次安装到同一个集群中.每次安装都会创建一个新版本.例如一个mysql chart。如果你想在集群运行两个数据库实例,则可以安装该chart两次.每个实例都有自己的release,而每个release又都有属于自己的release名。

记住这些概念后，我们现在可以像这样解释Helm：

Helm将chart安装到Kubernetes中，为每个安装创建一个新版本。 要找到新的chart，您可以搜索Helm chart存储库。

###  ‘HELM SEARCH’: 查找chart

首次安装Helm时，它已预配置为与官方Kubernetes chart存储库交互。 该存储库包含许多精心策划和维护的chart。 此图库在默认情况下被命名为stable。

您可以通过运行helm search来查看哪些图表可用

```shell
$ helm search
NAME                 	VERSION 	DESCRIPTION
stable/drupal   	0.3.2   	One of the most versatile open source content m...
stable/jenkins  	0.1.0   	A Jenkins Helm chart for Kubernetes.
stable/mariadb  	0.5.1   	Chart for MariaDB
stable/mysql    	0.1.0   	Chart for MySQL
...
```

上面的命令没有过滤, helm search会将所有可用的chart列出来.你可以使用过滤器来缩小搜索范围:

```shell
$ helm search mysql
NAME               	VERSION	DESCRIPTION
stable/mysql  	0.1.0  	Chart for MySQL
stable/mariadb	0.5.1  	Chart for MariaDB
```

现在您只会看到与您的过滤器匹配的结果。

为什么mariadb在列表中？ 因为它的包描述与MySQL相关。 我们可以使用helm inspect chart来看这个chart的详情：

```shell
$ helm inspect stable/mariadb
Fetched stable/mariadb to mariadb-0.5.1.tgz
description: Chart for MariaDB
engine: gotpl
home: https://mariadb.org
keywords:
- mariadb
- mysql
- database
- sql
...
```

搜索是找到可用软件包的好方法。 一旦找到想要安装的软件包，可以使用helm install来安装它。

### 'HELM INSTALL' 安装包

要安装新软件包，请使用helm install命令。 最简单的，它只需要一个参数：chart的名称。

```shell
$ helm install stable/mariadb
Fetched stable/mariadb-0.3.0 to /Users/mattbutcher/Code/Go/src/k8s.io/helm/mariadb-0.3.0.tgz
happy-panda
Last Deployed: Wed Sep 28 12:32:28 2016
Namespace: default
Status: DEPLOYED

Resources:
==> extensions/Deployment
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
happy-panda-mariadb   1         0         0            0           1s

==> v1/Secret
NAME                     TYPE      DATA      AGE
happy-panda-mariadb   Opaque    2         1s

==> v1/Service
NAME                     CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
happy-panda-mariadb   10.0.0.70    <none>        3306/TCP   1s


Notes:
MariaDB can be accessed via port 3306 on the following DNS name from within your cluster:
happy-panda-mariadb.default.svc.cluster.local

To connect to your database run the following command:

   kubectl run happy-panda-mariadb-client --rm --tty -i --image bitnami/mariadb --command -- mysql -h happy-panda-mariadb
```

现在安装了mariadb chart。 请注意，安装chart会创建一个新版本对象。 上面的版本被命名为happy-panda。 （如果你想使用你自己的版本名称，只需在helm install上使用--name标志。）

在安装过程中，helm客户端将打印有关创建哪些资源的有用信息，该版本的状态以及是否可以或应该采取其他配置步骤。

Helm不会等到所有资源都在运行之后才运行。 许多chart需要大小超过600M的Docker映像，并且可能需要很长时间才能安装到群集中。

要跟踪release状态或重新读取配置信息，可以使用helm  status：

```shell
$ helm status happy-panda
Last Deployed: Wed Sep 28 12:32:28 2016
Namespace: default
Status: DEPLOYED

Resources:
==> v1/Service
NAME                     CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
happy-panda-mariadb   10.0.0.70    <none>        3306/TCP   4m

==> extensions/Deployment
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
happy-panda-mariadb   1         1         1            1           4m

==> v1/Secret
NAME                     TYPE      DATA      AGE
happy-panda-mariadb   Opaque    2         4m


Notes:
MariaDB can be accessed via port 3306 on the following DNS name from within your cluster:
happy-panda-mariadb.default.svc.cluster.local

To connect to your database run the following command:

   kubectl run happy-panda-mariadb-client --rm --tty -i --image bitnami/mariadb --command -- mysql -h happy-panda-mariadb
```

以上显示了你的release的当前状态。

#### 在安装之前自定义chart

上面的方式安装只能使用默认配置,但是很多时候你需要自定义chart的配置来安装.

要查看chart的可配置选项,你可以使用helm inspect value:

```shell
helm inspect values stable/mariadb
Fetched stable/mariadb-0.3.0.tgz to /Users/mattbutcher/Code/Go/src/k8s.io/helm/mariadb-0.3.0.tgz
## Bitnami MariaDB image version
## ref: https://hub.docker.com/r/bitnami/mariadb/tags/
##
## Default: none
imageTag: 10.1.14-r3

## Specify a imagePullPolicy
## Default to 'Always' if imageTag is 'latest', else set to 'IfNotPresent'
## ref: http://kubernetes.io/docs/user-guide/images/#pre-pulling-images
##
# imagePullPolicy:

## Specify password for root user
## ref: https://github.com/bitnami/bitnami-docker-mariadb/blob/master/README.md#setting-the-root-password-on-first-run
##
# mariadbRootPassword:

## Create a database user
## ref: https://github.com/bitnami/bitnami-docker-mariadb/blob/master/README.md#creating-a-database-user-on-first-run
##
# mariadbUser:
# mariadbPassword:

## Create a database
## ref: https://github.com/bitnami/bitnami-docker-mariadb/blob/master/README.md#creating-a-database-on-first-run
##
# mariadbDatabase:
```

然后，您可以在YAML格式的文件中覆盖这些设置，然后在安装过程中传递该文件。

```shel
$ echo '{mariadbUser: user0, mariadbDatabase: user0db}' > config.yaml
$ helm install -f config.yaml stable/mariadb
```

以上将创建一个名为user0的默认MariaDB用户，并授予此用户对新创建的user0db数据库的访问权限，但将接受该chart的所有其他默认值。

在安装过程中有两种方式传递配置数据：

- —values(或者-f): 用覆盖指定一个YAML文件。 这可以指定多次，最右边的文件优先级更高
- —set: 在命令行上指定覆盖

如果两者都被使用，—set和—values值合并,并且—set的值有更高的优先级。 使用--set指定的覆盖将保存在配置chart中。 对于给定版本的helm，可以查看已经被设置为--set的值获取值<release-name>。 通过使用指定的--reset-values运行helm upgrade，可以清除已设置的值。

##### —set的格式和限制

--set选项使用零个或多个name/value对。 最简单的就是这样使用：--set name = value。 YAML的等价物是：

```yaml
name: value
```

多个值由,分隔。 所以--set a = b，c = d变成：

```yam
a: b
c: d
```

支持更复杂的表达式。 例如，--set outer.inner = value被翻译为：

```yaml
outer:
  inner: value
```

列表可以通过在{和}中包含值来表示。 例如，--set name = {a，b，c}转换为：

```yaml
name:
  - a
  - b
  - c
```

从Helm 2.5.0开始，可以使用数组索引语法访问列表项。 例如，--set servers [0] .port = 80变成：

```yaml
servers:
  - port: 80
```

可以通过这种方式设置多个值。 line --set servers [0] .port = 80，servers [0] .host = example变成：

```yaml
servers:
  - port: 80
    host: example
```

有时你需要在你的--set行中使用特殊字符。 您可以使用反斜杠来转义字符; --set name = value1 \，value2将变为：

```yaml
name: "value1,value2"
```

同样，您也可以转义点序列，当图表使用toYaml函数解析注释，标签和节点选择器时可能会派上用场。 --set nodeSelector。“kubernetes \ .io / role”= master的语法变为：

```she
nodeSelector:
  kubernetes.io/role: master
```

深度嵌套的数据结构可能难以用--set表达。 鼓励chart设计者在设计values.yaml文件的格式时考虑--set用法。

#### 更多install方法

helm install命令可以从几个来源安装：

- char存储库(如之前我们所见)
- 本地图char存档（helm install foo-0.1.1.tgz）
- 解压后的chart目录（helm install path/to/foo）
- 一个完整的URL(helm install https://example.com/charts/foo-1.2.3.tgz)

###'HELM UPGRADE' 和'HELM ROLLBACK': 解压后的chat目录（helm install path/to/foo）升级release和失败恢复

当新版本的chart release时，或者当您想要更改release版的配置时，可以使用helm upgrade命令。

升级需要现有版本并根据您提供的信息进行升级。 因为Kubernetes chart可能很大而且很复杂，所以Helm会尝试执行最不具有侵入性的升级。 它只会更新自上次发布以来发生更改的内容。

```she
$ helm upgrade -f panda.yaml happy-panda stable/mariadb
Fetched stable/mariadb-0.3.0.tgz to /Users/mattbutcher/Code/Go/src/k8s.io/helm/mariadb-0.3.0.tgz
happy-panda has been upgraded. Happy Helming!
Last Deployed: Wed Sep 28 12:47:54 2016
Namespace: default
Status: DEPLOYED
...
```

在上面的例子中，happy-panda版本升级了同样的chart，但是新增了一个YAML文件：

```yaml
mariadbUser: user1
```

我们可以使用helm get values来查看新设置是否生效。

```shell
$ helm get values happy-panda
mariadbUser: user1
```

helm get命令是查看集群中的发行版的有用工具。 正如我们上面所看到的，它表明我们从panda.yaml获得的新值已部署到群集中。

现在，如果某个版本在发布过程中没有按计划进行，可以使用helm rollback [RELEASE]\[REVISION]轻松回滚到以前的版本。

```shell
$ helm rollback happy-panda 1
```

上述回滚我们的happ-panda到它的第一个发行版本。 发布版本是增量修订。 每次发生安装，升级或回滚时，修订号都会加1。第一个修订号始终为1.我们可以使用helm history [RELEASE]查看某个版本的修订号。

### INSALL/UPGRADE/ROLLBACK的help选项

在INSTALL/UPGRADE/ROLLBACK期间，您可以指定几个其他有用的选项来定制Helm的行为。 请注意，这不是cli标志的完整列表。 要查看所有标志的描述，只需运行helm <command> --help。

- —timeout: 等待Kubernetes命令完成的数值（以秒为单位）默认为300（5分钟）
- —wait: 等待所有Pod都处于就绪状态，PVC绑定，Deployments在就绪状态下具有最小（Desired minus maxUnavailable）Pod，并且在将release版标记为成功之前，服务具有IP地址（以及Ingress，如果是LoadBalancer）。 它将等待 —timeout值。 如果达到超时，则释放将被标记为FAILED。 注意：在Deployment的副本设置为1且maxUnavailable未设置为0的情况下，作为滚动更新策略的一部分，--wait将返回就绪状态，因为它已满足就绪状态下的最小Pod。
- --no-hooks:  这会跳过该命令的运行钩子
- --recreate-pods(仅在upgrade或rollback命令下有效): 该标志将导致所有的pod被重新创建（除了属于deployments的pod） 

### 'HELM DELETE':删除一个release

当需要从群集中卸载或删除发行版时，请使用helm delete命令：

```shell
$ helm delete happy-panda
```

这将从集群中删除该版本。 您可以使用helm list命令查看所有当前部署的版本：

```shell
$ helm list
NAME           	VERSION	UPDATED                        	STATUS         	CHART
inky-cat       	1      	Wed Sep 28 12:59:46 2016       	DEPLOYED       	alpine-0.1.0
```

从上面的输出中，我们可以看到happy-pand版本被删除。

然而，Helm总是保留记录发生了什么。 需要查看已删除的版本？ helm list --deleted显示了这些，并且helm list --all显示所有的版本（已删除并且当前部署，以及失败的版本）：

```shell
⇒  helm list --all
NAME           	VERSION	UPDATED                        	STATUS         	CHART
happy-panda   	2      	Wed Sep 28 12:47:54 2016       	DELETED        	mariadb-0.3.0
inky-cat       	1      	Wed Sep 28 12:59:46 2016       	DEPLOYED       	alpine-0.1.0
kindred-angelf 	2      	Tue Sep 27 16:16:10 2016       	DELETED        	alpine-0.1.0
```

由于Helm保留已删除版本的记录，因此不能重新使用版本名称。 （如果您确实需要重新使用发行版名称，则可以使用--replace标志，但它只会重新使用现有版本并替换其资源。）

请注意，因为版本以这种方式保存，所以可以回滚已删除的资源并重新激活它。

### 'HELM REPO': 使用存储库

到目前为止，我们只从稳定的存储库安装chart。 但是你可以配置helm来使用其他仓库。 Helm在helm repo命令下提供了一些存储库工具。

你可以看到使用helm repo list配置了哪些仓库：

```shell
$ helm repo list
NAME           	URL
stable         	https://kubernetes-charts.storage.googleapis.com
local          	http://localhost:8879/charts
mumoshu        	https://mumoshu.github.io/charts
```

新的仓库可以添加helm repo add:

```shell
$ helm repo add dev https://example.com/dev-charts
```

由于chart存储库经常更改，您可以随时通过运行helm repo update来确保您的Helm客户端是最新的。

### 创建你自己的charts

Chat开发指南解释了如何开发自己的图表。 但是您可以通过使用helm create命令快速入门：

```shell
$ helm create deis-workflow
Creating deis-workflow
```

现在在./deis-workflow中有一个chart。 您可以编辑它并创建自己的模板。

在您编辑chart时，您可以通过运行helm lint来验证其格式是否良好。

当将char打包分发时，您可以运行helm package命令：

```shell
$ helm package deis-workflow
deis-workflow-0.1.0.tgz
```

现在可以通过helm install轻松安装该chart：

```shell
$ helm install ./deis-workflow-0.1.0.tgz
...
```

可以将已归档的图表加载到chart存储库中。 请参阅chart存储库服务器的文档以了解如何上传。

注意：稳定存储库在[Kubernetes Charts GitHub存储库](https://github.com/kubernetes/charts)上进行管理。 该项目接受chart源代码，并且（在审计后）为您打包。

### TILLER，NAMESPACES和RBAC

在某些情况下，您可能希望将Tiller的范围或将多个Tillers部署到单个群集。 以下是在这些情况下运营的一些最佳做法。

1. Tiller可以安装到任何命名空间(namespace)。 默认情况下，它安装在kube-system中。 您可以运行多个Tillers，只要它们各自在自己的名称空间中运行。
2. 限制Tiller只能安装到特定的名称空间和/或资源类型由Kubernetes RBAC角色和角色绑定控制。 通过helm init --service-account <NAME>配置Helm时，您可以向Tiller添加服务帐户。 你可以在这里找到更多的信息。
3. 版本名称是唯一的PER TILLER实例。
4. chart应该只包含存在于单个命名空间中的资源。
5. 不建议将多个Tillers配置为在相同的命名空间中管理资源。

### 结论

本章介绍了helm客户端的基本使用模式，包括search，install，upgrade和delete。 它还涵盖了有用的实用程序命令，如helm status，helm get和helm repo。

有关这些命令的更多信息，请查看Helm的内置帮助：helm help。

在下一章中，我们将看看开发图表的过程。