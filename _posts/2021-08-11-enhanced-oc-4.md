---
title: 在团队中使用增强版OpenShift CLI
excerpt: "本文向大家介绍如何把集群访问信息分享给团队中的其他成员"
categories: [tech]
tags: [kubernetes, openshift]
toc: true
toc_sticky: true
---

> 增强版OpenShift CLI可以帮助我们高效而安全的管理大量的OpenShift集群。它是一个位于[GitHub](https://github.com/morningspace/enhanced-oc)上的开源项目。

作为增强版OpenShift CLI系列文章的第四篇，本文将向大家介绍如何把保存在本地secret store里的集群访问信息分享给团队中的其他成员。

通过把存有集群访问信息的本地secret store分享到远程，我们可以很方便的在整个团队范围内实现统一的集群管理。任何时候，当你有新的集群访问信息通过增强版oc加入到secret store，同样的访问信息将会被自动同步到其他团队成员的本地secret store里。这样，大家在不关心集群访问信息具体内容的情况下，只通过集群的别名就可以实现对同一集群的访问了。

## 分享你的Secret Store

如果你打算把保存在本地secret store里的集群访问信息分享给其他人，首先需要为secret store添加git remote信息，并将本地的secret store通过git push推送到远程：
```shell
$ gopass git remote add origin git@github.example.com:william/team-clusters.git
$ gopass git push origin master
```

当连接远程的git服务器时（比如GitHub），尽管我们可以使用HTTPS协议，但还是建议大家使用SSH进行连接。利用SSH key，我们在每次访问远程git库时，可以无需提供用户名和access token。有关如何以SSH方式连接GitHub，请参考GitHub的[相关文档](https://docs.github.com/en/github/authenticating-to-github/connecting-to-github-with-ssh)。

## 为团队成员做准备

为了让团队成员能够成功解密保存在远程secret store里的经过加密的集群访问信息，我们还需要做一点准备工作。首先，要求团队成员在各自的机器上安装好必要的工具和软件，包括：`gpg`，`git`，`gopass`，`oc`等。然后，要求大家用如下命令生成各自的密钥对：
```shell
$ gpg --full-generate-key
```

选择`RSA and RSA`作为密钥类型，keysize为2048，密钥永不过期，然后输入团队成员的用户名，邮箱地址，以及密钥对应的密码（passphrase）。

等密钥对生成完毕以后，要求大家把其中的公钥导出并发送给你。在团队成员所在的机器上执行如下命令，可以找到公钥的ID：
```shell
$ gpg --list-keys --keyid-format LONG
/root/.gnupg/pubring.gpg
------------------------
pub   2048R/93E7B0300BB9C91B 2021-03-17
uid                          Nicole <nicole@example.com>
sub   2048R/3AE8C980579D103C 2021-03-17
```

从输出结果中选取与当前团队成员的用户名及邮箱地址相匹配的公钥，复制相应的密钥ID，即`pub`所在的那一行，本例中为`93E7B0300BB9C91B`。然后用如下命令，把公钥导出到一个文件里：
```shell
$ gpg --armor --export 93E7B0300BB9C91B > nicole_pub.gpg
```

当我们接收到团队成员的公钥以后，通过如下命令将公钥导入到本地：
```shell
$ gpg --import nicole_pub.gpg
```

并将该团队成员作为`recipient`，加入到本地的secret store里：
```shell
$ gopass recipients add 93E7B0300BB9C91B
```

然后用我们自己的私钥对团队成员的公钥进行签名，并选择对其永久信任：
```shell
$ gpg --edit-key 93E7B0300BB9C91B
lsign
trust
save
```

最后，重新加一次`recipient`，以触发gopass对保存在本地secret store里的数据重新进行加密：
```shell
$ gopass recipients rm 93E7B0300BB9C91B
$ gopass recipients add 93E7B0300BB9C91B
```

上述所有步骤都会被自动同步到远程的secret store里。

## 克隆远程的Secret Store

现在，我们可以通知团队成员，让大家把远程的secret store克隆到本地了。同时，请确保这些团队成员已经以`collaborator`的身份被添加到了远程git库里。然后在每位团队成员所在的机器上执行如下命令，将远程secret store克隆到本地：
```shell
$ gopass --yes setup --remote git@github.example.com:william/team-clusters.git --alias team-clusters --name Nicole --email "nicole@example.com"
```

其中，参数`--name`和`--email`的取值和团队成员在生成各自密钥对时所使用的取值保持一致。这样，其他人就可以和我们一样，利用`oc login`命令及别名，对同一组集群进行访问了。

下图演示了整个准备工作的端到端流程：

![](/assets/images/studio/enhanced-oc/enhanced-oc-4.jpg)

## 小结

本文中，我们学习了如何把保存在本地secret store里的集群访问信息分享给团队中的其他成员。

如果你想了解更多有关增强版OpenShift CLI的信息，请阅读它的[在线文档](https://morningspace.github.io/oc/docs/)。如果你喜欢这个项目，欢迎给它[加星](https://github.com/morningspace/oc)。同时，也非常欢迎针对该项目的任何形式的贡献，比如bug报告以及代码提交。
