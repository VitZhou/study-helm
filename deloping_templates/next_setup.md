# Wrapping Up

本指南旨在为您提供chat开发人员对如何使用Helm模板语言的深入了解。 本指南着重介绍模板开发的技术方面。

但是当谈到chart的实际日常开发时，本指南还没有涉及很多东西。 以下是一些有用的指向其他文档的指南，这些指南将帮助您创建新chart：

- [Kubernetes Charts](https://github.com/kubernetes/charts)项目是chart不可缺少的来源。 该项目也是图表开发中最佳实践的标准。
- “[Kubernetes用户指南](http://kubernetes.io/docs/user-guide/)”提供了可以使用的各种资源类型的详细示例，从ConfigMaps和Secrets到DaemonSet和Deployments。
- [Helm Charts](../charts/charts.md)指南介绍了使用图表的工作流程。
- [Helm Chart Hooks](../chart/lifecycle_hooks.md)指南解释了如何创建生命周期钩子。
- [Helm Chart技巧和窍门](../charts/tips_and_tricks.md)文章提供了一些写chart的有用技巧。
- [Sprig文档](https://github.com/Masterminds/sprig)记录了六十多个模板函数。
- [Go模板文档](./quik_start.md)详细解释了模板语法。
- [Schelm工具](https://github.com/databus23/schelm)是用于调试chart的好助手工具。

有时候，问几个问题并从有经验的开发人员那里获得答案会更容易。 最好的地方是在Kubernetes #Helm Slack chanel：

- [Kubernetes Slack](https://slack.k8s.io/):helm

最后，如果您在本文档中发现错误或遗漏，想要推荐一些新内容或希望参与，请访问[Helm项目](https://github.com/kubernetes/helm)。

