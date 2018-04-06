# Helm插件

Helm 2.1.0引入了客户端Helm插件的概念。 插件是一个可以通过helm CLI访问的工具，但它不是内置Helm代码库的一部分。

现有的插件可以在相关部分找到或者通过搜索[Github](https://github.com/search?q=topic%3Ahelm-plugin&type=Repositories)。

本指南介绍了如何使用和创建插件。

## 概述

helm插件是与Helm无缝集成的附加工具。 它们提供了一种扩展Helm核心功能集的方法，但不需要将每个新功能都写入Go并添加到核心工具中。

helm的插件拥有如下特性:

- 可以在Helm安装中添加和删除它们，而不会影响核心Helm工具。
- 它们可以用任何编程语言编写。
- 他们与Helm整合，并会出现在helm help和其他地方。

helm的插件安装在 $(helm home)/plugins目录.

Helm插件model部分建模在Git的插件model上。 为此，您有时可能会听到称为porcelain层的chart，插件就是*plumbing*。 这是提示Helm提供用户体验和顶级处理逻辑的简略方式，而插件则是执行所需操作的“detail work”。

## 安装插件

插件使用\$ helm plugin install <path | url>命令安装。 您可以将路径传递到本地文件系统上或远程VCS回购的URL。 helm plugin install命令克隆或复制插件的路径/ URL到$(helm home)/plugin

```shell
$ helm plugin install https://github.com/technosophos/helm-template
```

如果你有一个插件tar包，只需将插件解压到$(helm home)/plugins目录。

您也可以通过发布helm插件直接从url安装tarball:helm plugin install http://domain/path/to/plugin.tar.gz

## 构建插件

在很多方面，插件类似于chart。 每个插件都有一个顶级目录，然后是一个plugin.yaml文件。

```shell
$(helm home)/plugins/
  |- keybase/
      |
      |- plugin.yaml
      |- keybase.sh
```

在上面的例子中，keybase插件包含在名为keybase的目录中。 它有两个文件：plugin.yaml（必需）和一个可执行脚本，keybase.sh（可选）。

插件的核心是一个简单的名为plugin.yaml的YAML文件。 这是一个插件的YAML，它增加了对Keybase操作的支持：

```yam
name: "keybase"
version: "0.1.0"
usage: "Integrate Keybase.io tools with Helm"
description: |-
  This plugin provides Keybase services to Helm.
ignoreFlags: false
useTunnel: false
command: "$HELM_PLUGIN_DIR/keybase.sh"
```

Name是插件的名称。 当Helm执行它的插件时，这是它将使用的名称（例如，helm NAME将调用这个插件）。

name应该与目录名称匹配。 意味着在我们上面的示例中，name为keybase的插件应该包含在名为keybase的目录中。

name的限制:

- 不能复制现有的helm任何顶层命令
- 必须为限制ASCII字符: a-z, A-Z, 0-9, `_` and `-`

Version是插件的SemVer 2版本。 usage和description都用于生成命令的帮助文本。

ignoreFlags开关告诉Helm不要将标志传递给插件。 所以如果用helm myplugin --foo和ignoreFlags：true调用一个插件，那么--foo将被默默丢弃。

useTunnel开关指示该插件需要一个到Tiller的隧道。 只要插件需要与Tiller交互，就应该设置为true。 它会导致Helm打开一个隧道，然后将$ TILLER_HOST设置为该隧道的正确本地地址。 但是不用担心：如果Helm由于Tiller在本地运行而检测到隧道不是必需的，它不会创建隧道。

最后，最重要的是，command是这个插件在调用时执行的命令。 在执行插件之前插入环境变量。 上面的模式说明了插件程序所在位置的首选方式。

有一些使用插件命令的策略：

- 如果插件包含可执行文件，则command的可执行文件应打包在插件目录中。
- 该command:行将在执行前展开任何环境变量。 $ HELM_PLUGIN_DIR将指向插件目录。
- 该命令本身不在shell中执行。 所以你不能在一个shell脚本上运行。
- Helm将大量配置注入到环境变量中。 查看环境变量以查看可用信息。
- Helm对插件的语言没有任何假设。 你可以用你喜欢的任何方式来写。
- 命令负责实现-h和--help的特定帮助文本。 Helm将使用helm help和helm help myplugin的用法和描述，但不会处理helm myplugin --help。

## 下载插件

默认情况下，Helm可以使用HTTP/S获取charts。 从Helm 2.4.0开始，插件可以具有从任意源下载charts的特殊功能。

插件应在plugin.yaml文件（顶层）中声明这个特殊功能：

```yaml
downloaders:
- command: "bin/mydownloader"
  protocols:
  - "myprotocol"
  - "myprotocols"
```

如果安装了这样的插件，Helm可以通过调用该命令使用指定的协议方案与存储库进行交互。 特殊的仓库应该类似于常规仓库：helm repo add favorite myprotocol://example.com/ 为了发现并缓存可用chart的列表特殊仓库的规则与普通仓库的规则相同：Helm必须能够下载index.yaml文件。

定义的命令将使用以下方案调用：command certFile keyFile caFile full-URL。 SSL凭证来自回购定义，存储在$ HELM_HOME/repository/repositories.yaml中。 预计下载器插件将原始内容转储到stdout(标准输入)并在stderr(标准输出)上报告错误。

## 环境变量

当Helm执行插件时，它将外部环境传递给插件，并且还会注入一些其他环境变量。

如果插件设置在外部环境中，则会为插件设置像KUBECONFIG这样的变量。

以下变量保证设置：

- HELM_PLUGIN: 插件的目录
- HELM_PLUGIN_NAME: 插件的名称，由helm调用。 所以helm myplug将有短名myplug。
- HELM_PLUGIN_DIR: 包含该插件的目录
- HELM_BIN: helm命令的路径（由用户执行）
- HELM_HOME: helm的主目录
- HELM_PATH_*: 重要Helm文件和目录的路径存储在以HELM_PATH为前缀的环境变量中。
- TILLER_HOST:  连接到Tiller的host:port。 如果创建隧道，则会指向隧道的本地端点。 否则，它将指向$ HELM_HOST，--host或默认主机（根据Helm的优先级规则）。

尽管可能设置了HELM_HOST，但不能保证它会指向正确的Tiller实例。 这样做是为了允许插件开发者在插件本身需要手动配置连接时以其原始状态访问HELM_HOST。

## 关于USETUNNEL需要注意的点

如果插件指定useTunnel:true，Helm将执行以下操作（按顺序）：

1. 解析全局标志和环境
2. 创建隧道
3. 设置TILLER_HOST
4. 执行插件
5. 关闭隧道

一旦命令返回，隧道即被删除。 因此，例如，一个命令不能后台进程，并假定该进程将能够使用该隧道。

## 关注标志划分需要注意的点

当执行插件时，Helm会解析全局标志以供自己使用。 其中一些标志不会传递给插件。

- —debug:如果指定了这个，$ HELM_DEBUG被设置为1
- —home:这被转换为$ HELM_HOME
- —host:这被转换为$ HELM_HOST
- --kube-context: 简单的丢弃。 如果您的插件使用useTunnel，则用于为您设置隧道。

插件应显示帮助文本，然后退出-h和--help。 在所有其他情况下，插件可以根据需要使用标志。