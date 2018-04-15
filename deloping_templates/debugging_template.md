# Debugging模板

调试模板可能会很棘手，因为模板在Tiller服务器而不是Helm客户端上呈现。 然后渲染的模板被发送到Kubernetes API服务器，由于格式化以外的原因，这些服务器可能会拒绝YAML文件。

有几个命令可以帮助您进行调试。

- helm lint是您的工具，用于验证您的chart是否遵循最佳实践
- helm install --dry-run --debug：我们已经看到了这个技巧。 这是让服务器渲染你的模板，然后返回结果清单文件的好方法。
- helm get manifest：这是查看服务器上安装的模板的好方法。

当你的YAML没有解析，但你想看看产生了什么时，检索YAML的一个简单方法是注释掉模板中的问题部分，然后重新运行helm install --dry-run --debug：

```yaml
apiVersion: v1
# some: problem section
# {{ .Values.foo | quote }}
```

以上内容将被完整呈现并返回。

```yaml
apiVersion: v1
# some: problem section
#  "bar"
```

这提供了一种快速查看生成内容的方式，而不会阻止YAML分析错误。

