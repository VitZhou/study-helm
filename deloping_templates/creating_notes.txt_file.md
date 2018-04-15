# 创建NOTES.txt文件

在本节中，我们将看看Helm的工具，向您的chart用户提供说明。 在chart安装或chart升级结束时，Helm可以为用户打印出一大块有用的信息。 这些信息是使用模板高度定制的。

要将安装注释添加到chart，只需创建一个templates / NOTES.txt文件。 这个文件是纯文本的，但是它像一个模板一样处理，并且具有所有可用的普通模板函数和对象。

我们来创建一个简单的NOTES.txt文件：

```Yaml
Thank you for installing {{ .Chart.Name }}.

Your release is named {{ .Release.Name }}.

To learn more about the release, try:

  $ helm status {{ .Release.Name }}
  $ helm get {{ .Release.Name }}
```

现在，如果我们运行helm install ./mychart，我们将在底部看到这条消息：

```shell
RESOURCES:
==> v1/Secret
NAME                   TYPE      DATA      AGE
rude-cardinal-secret   Opaque    1         0s

==> v1/ConfigMap
NAME                      DATA      AGE
rude-cardinal-configmap   3         0s


NOTES:
Thank you for installing mychart.

Your release is named rude-cardinal.

To learn more about the release, try:

  $ helm status rude-cardinal
  $ helm get rude-cardinal
```

通过这种方式使用NOTES.txt是向用户提供有关如何使用新安装chart的详细信息的好方法。 强烈建议创建NOTES.txt文件，尽管它不是必需的。

