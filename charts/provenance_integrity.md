# Helm的出处和诚信

Helm拥有可帮助chart用户验证包装的完整性和源码的出处工具。 使用基于PKI，GnuPG和备受尊敬的软件包管理器的行业标准工具，Helm可以生成并验证签名文件。

> 版本2.0.0-alpha.4引入了一个验证chart真实性的系统。 虽然我们预计文件格式或证明算法的算法不会发生重大变化，但这部分Helm在2.0.0-RC1发布之前不会被视为冻结。 [issure983](https://github.com/kubernetes/helm/issues/983)可以找到有关此功能的原始计划。

## 概览

通过将chart与出处记录进行比较来建立完整性。 出处记录存储在provenance文件中，并与打包chart一起存储。 例如，如果chart名称为myapp-1.2.3.tgz，则其出处文件将为myapp-1.2.3.tgz.prov。

Provenance文件在打包时生成（helm package --sign ...），可以通过多个命令进行检查，值得注意的是helm install --verify。

## workflow

本节描述了有效使用provenance数据的潜在工作流程。

先决条件：

- 二进制（非ASCII铠装）格式的有效PGP密钥对
- helm命令行工具
- GnuPG命令行工具（可选）
- Keybase命令行工具（可选）

> 如果您的PGP私钥具有密码，则系统会提示您输入该密码以查看支持--sign选项的任何命令。

## 跟之前一样创建一个新chart

```shell
$ helm create mychart
Creating mychart
```

准备打包后，将--sign标志添加到helm包。 另外，指定签名密钥已知的名称和包含相应私钥的密钥环：

```shell
$ helm package --sign --key 'helm signing key' --keyring path/to/keyring.secret mychart
```

提示：对于GnuPG用户，您的密钥环在〜/.gnupg/secring.gpg中。 您可以使用gpg --list-secret-keys列出您拥有的密钥。

在这一点上，你应该看到mychart-0.1.0.tgz和mychart-0.1.0.tgz.prov。 这两个文件最终应该上传到您想要的chart存储库。

你可以使用helm verify验证你的chart:

```shell
$ helm verify mychart-0.1.0.tgz
```

验证失败如下所示：

```shell
$ helm verify topchart-0.1.0.tgz
Error: sha256 sum does not match for topchart-0.1.0.tgz: "sha256:1939fbf7c1023d2f6b865d137bbb600e0c42061c3235528b1e8c82f4450c12a7" != "sha256:5a391a90de56778dd3274e47d789a2c84e0e106e1a37ef8cfa51fd60ac9e623a"
```

要在安装过程中进行验证，请使用--verify标志。

```shell
$ helm install --verify mychart-0.1.0.tgz
```

如果密钥环（包含与签名chart关联的公钥）不在默认位置，则可能需要使用--keyring PATH指向密钥环，如helm软件包示例中所示。

如果验证失败，安装将在chart被推到Tiller之前中止。

## 使用Keybase.io凭据

该[Keybase.io](https://keybase.io/)服务可以很容易建立信任链的密码身份。密钥库凭证可用于对chart进行签名。

先决条件：

- 已配置的Keybase.io帐户
- GnuPG在本地安装
- 该`keybase`CLI本地安装

### 签署软件包

第一步是将keybase密钥导入到本地GnuPG密钥环中：

```shell
$ keybase pgp export -s | gpg --import
```

这会将您的Keybase密钥转换为OpenPGP格式，然后将其本地导入到您的`~/.gnupg/secring.gpg`文件中。

您可以通过运行进行仔细检查`gpg --list-secret-keys`。

```shell
$ gpg --list-secret-keys                                                                                                       1 ↵
/Users/mattbutcher/.gnupg/secring.gpg
-------------------------------------
sec   2048R/1FC18762 2016-07-25
uid                  technosophos (keybase.io/technosophos) <technosophos@keybase.io>
ssb   2048R/D125E546 2016-07-25
```

请注意，您的密钥将有一个标识符字符串：

```shell
technosophos (keybase.io/technosophos) <technosophos@keybase.io>
```

这是你的钥匙的全名。

接下来，您可以使用打包和签名chart`helm package`。确保至少使用该名称字符串的一部分`--key`。

```shell
$ helm package --sign --key technosophos --keyring ~/.gnupg/secring.gpg mychart
```

结果，该`package`命令应该生成一个`.tgz`文件和一个`.tgz.prov` 文件。

### 验证软件包

您还可以使用类似的技术来验证由其他人的Keybase密钥签名的chart。假设您想验证签名的软件包`keybase.io/technosophos`。为此，请使用该`keybase`工具：

```shell
$ keybase follow technosophos
$ keybase pgp pull
```

上面的第一条命令跟踪用户`technosophos`。接下来`keybase pgp pull` ，将您关注的所有帐户的OpenPGP密钥下载到您的GnuPG密钥环（`~/.gnupg/pubring.gpg`）中。

此时，您现在可以使用`helm verify`或带有`--verify` 标志的任何命令：

```shell
$ helm verify somechart-1.2.3.tgz
```

### chart可能无法验证的原因

这些是失败的常见原因。

- prov文件丢失或损坏。这表明某些内容配置错误或原始维护人员未创建出处文件。
- 用于签署文件的密钥不在您的钥匙环中。这表明签名chart的实体不是您已经表示信任的人员。
- prov文件的验证失败。这表明chart或来源数据有问题。
- 源文件中的文件哈希与存档文件的哈希不匹配。这表明档案已被篡改。

如果验证失败，则有理由怀疑该软件包。

## PROVENANCE文件

PROVENANCE文件包含chart的YAML文件以及几条验证信息。PROVENANCE文件被设计为自动生成。

添加了以下几个provenance数据：

- 包含chart文件（Chart.yaml）可以让人员和工具轻松查看chart内容。
- 包括chart包（.tgz文件）的签名（SHA256，就像Docker）一样，可以用来验证chart包的完整性。
- 整个机构使用PGP使用的算法进行签名（参见[ [http://keybase.io](http://keybase.io/) ]，这是一种使加密签名和验证变得容易的新方法）。

这样的组合给了用户以下保证：

- 包本身没有被篡改（校验和包tgz）。
- 已知发布此包的实体（通过GnuPG / PGP签名）。

该文件的格式如下所示：

```yam
-----BEGIN PGP SIGNED MESSAGE-----
name: nginx
description: The nginx web server as a replication controller and service pair.
version: 0.5.1
keywords:
  - https
  - http
  - web server
  - proxy
source:
- https://github.com/foo/bar
home: http://nginx.com

...
files:
        nginx-0.5.1.tgz: “sha256:9f5270f50fc842cfcb717f817e95178f”
-----BEGIN PGP SIGNATURE-----
Version: GnuPG v1.4.9 (GNU/Linux)

iEYEARECAAYFAkjilUEACgQkB01zfu119ZnHuQCdGCcg2YxF3XFscJLS4lzHlvte
WkQAmQGHuuoLEJuKhRNo+Wy7mhE7u1YG
=eifq
-----END PGP SIGNATURE-----
```

请注意，YAML部分包含两个文档（由...分隔`...\n`）。首先是Chart.yaml。第二个是校验和，一个到SHA-256摘要的文件名map（显示的值是假的/被截断的）

签名块是一个标准的PGP签名，它提供了[防篡改功能](http://www.rossde.com/PGP/pgp_signatures.html)。

## chart 库

Chart库用作Helm chart的集中集合。

Chart存储库必须能够通过特定的请求通过HTTP为源代码文件提供服务，并且必须使它们在与chart相同的URI路径下可用。

例如，如果软件包的基本URL是`https://example.com/charts/mychart-1.2.3.tgz源码文件（如果存在）必须可以在`https://example.com/charts/mychart-1.2.3.tgz.prov`。

从最终用户的角度来看，`helm install --verify myrepo/mychart-1.2.3` 应该导致无需额外的用户配置或操作即可下载chart和prov文件。

## 建立权威和真实性

在处理信任链系统时，能够建立签名者的权威很重要。或者，简单地说，上述系统取决于您相信签名人员的事实。这反过来又意味着你需要信任签名者的公钥。

Kubernetes Helm的设计决策之一是Helm项目不会将自己插入信任链中作为必要的成员。我们不希望成为所有chart签名者的“证书颁发机构”。相反，我们强烈支持分散模式，这是我们选择OpenPGP作为基础技术的原因之一。所以说到建立权威时，我们已经在Helm 2.0.0中对这个步骤进行了或多或少的未定义。

但是，对于那些有兴趣使用出处(provenance)系统的人，我们有一些建议和建议：

- 该[Keybase](https://keybase.io/)平台提供了可靠信息的公开集中存放。
  - 您可以使用Keybase存储您的密钥或获取其他公钥。
  - Keybase也有很多可用的文档
  - 虽然我们还没有对它进行测试，但Keybase的“安全网站”功能可用于服务Helm chart。
- 该[官员Kubernetes chart](https://keybase.io/)项目正在设法解决这个问题的官方chart库。
  - 这里有一个很长的问题，[详细介绍了当前的想法](https://github.com/kubernetes/charts/issues/23)。
  - 基本思想是官方的“chart评论者”用她或他的钥匙签名chart，然后将得到的出处文件上传到chartchar存储库。
  - 关于有效签名密钥列表可能包含在`index.yaml`存储库文件中的想法已经有了一些工作。

最后，信任链是Helm的一个发展特征，一些社区成员已经提出了将OSI模型的一部分用于签名。这是赫尔姆团队的一个开放性调查。如果你有兴趣，请继续。

