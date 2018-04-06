# 同步你的存储库

> 此示例专门用于提供chart存储库的Google Cloud Storage（GCS）存储。

## 先决条件

- 安装[gsutil](https://cloud.google.com/storage/docs/gsutil)工具。*我们严重依赖于gsutil rsync功能*
- 请务必访问helm二进制文件
- _可选：我们建议您在GCS存储上设置[对象版本控制](https://cloud.google.com/storage/docs/gsutil/addlhelp/ObjectVersioningandConcurrencyControl#top_of_page)，以防意外删除某些内容._

## 设置本地chart存储库目录

像我们在chart[存储库指南中](./repository.md)一样创建一个本地目录，并将打包的chart放入该目录中。

例如:

```shell
$ mkdir fantastic-charts
$ mv alpine-0.1.0.tgz fantastic-charts/
```

## 生成更新的INDEX.YAML

使用helm通过将远程存储库的目录路径和URL传递到`helm repo index`命令来生成更新的index.yaml文件，如下所示：

```shell
$ helm repo index fantastic-charts/ --url https://fantastic-charts.storage.googleapis.com
```

这将生成一个更新的index.yaml文件并放置在`fantastic-charts/`目录中。

## 同步您的本地和远程chart存储库

通过运行将目录的内容上传到您的GCS存储，`scripts/sync-repo.sh`并传入本地目录名称和GCS存储名称。

```shell
$ pwd
/Users/funuser/go/src/github.com/kubernetes/helm
$ scripts/sync-repo.sh fantastic-charts/ fantastic-charts
Getting ready to sync your local directory (fantastic-charts/) to a remote repository at gs://fantastic-charts
Verifying Prerequisites....
Thumbs up! Looks like you have gsutil. Let's continue.
Building synchronization state...
Starting synchronization
Would copy file://fantastic-charts/alpine-0.1.0.tgz to gs://fantastic-charts/alpine-0.1.0.tgz
Would copy file://fantastic-charts/index.yaml to gs://fantastic-charts/index.yaml
Are you sure you would like to continue with these changes?? [y/N]} y
Building synchronization state...
Starting synchronization
Copying file://fantastic-charts/alpine-0.1.0.tgz [Content-Type=application/x-tar]...
Uploading   gs://fantastic-charts/alpine-0.1.0.tgz:              740 B/740 B
Copying file://fantastic-charts/index.yaml [Content-Type=application/octet-stream]...
Uploading   gs://fantastic-charts/index.yaml:                    347 B/347 B
Congratulations your remote chart repository now matches the contents of fantastic-charts/
```

## 更新您的chart存储库

您需要保留chart存储库内容的本地副本，或者`gsutil rsync`将远程chart存储库的内容复制到本地目录。

```shell
$ gsutil rsync -d -n gs://bucket-name local-dir/    # the -n flag does a dry run
Building synchronization state...
Starting synchronization
Would copy gs://bucket-name/alpine-0.1.0.tgz to file://local-dir/alpine-0.1.0.tgz
Would copy gs://bucket-name/index.yaml to file://local-dir/index.yaml

$ gsutil rsync -d gs://bucket-name local-dir/       # performs the copy actions
Building synchronization state...
Starting synchronization
Copying gs://bucket-name/alpine-0.1.0.tgz...
Downloading file://local-dir/alpine-0.1.0.tgz:                        740 B/740 B
Copying gs://bucket-name/index.yaml...
Downloading file://local-dir/index.yaml:                              346 B/346 B
```

有用的链接：*有关[gsutil rsync的](https://cloud.google.com/storage/docs/gsutil/commands/rsync#description)文档 * [Chart Repository指南](https://docs.helm.sh/developing_charts/#developing_charts) *有关Google云存储中[对象版本控制和并发控制的](https://cloud.google.com/storage/docs/gsutil/addlhelp/ObjectVersioningandConcurrencyControl#overview)文档

