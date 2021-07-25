---
title: 增强版OpenShift CLI快速入门
excerpt: "本文向大家介绍增强版oc的安装及其用法演示"
categories: [tech]
tags: [kubernetes, openshift]
toc: true
toc_sticky: true
---

> 增强版OpenShift CLI可以帮助我们高效而安全的管理大量的OpenShift集群。它是一个位于[GitHub](https://github.com/morningspace/enhanced-oc)上的开源项目。

作为增强版OpenShift CLI系列文章的第二篇，本文将向大家介绍如何安装增强版oc，并为大家演示它的一些基本用法。

## 安装增强版OpenShift CLI

为了让增强版OpenShift CLI能够正常工作，我们在安装增强版OpenShift CLI之前，首先需要确保其所依赖的软件已经安装就绪，这其中包括：
* gpg：用于数据加密和解密
* git：用于存储和共享加密数据
* gopass：用于加密数据的管理，运行于gpg和git之上
* oc：RedHat官方提供的OpenShift CLI

根据你所使用的操作系统，从下列命令中选择正确的命令，安装git与gpg：
```shell
# MacOS
$ brew install gnupg2 git
# RHEL & CentOS
$ brew install gnupg2 git
# Ubuntu & Debian
$ apt-get install gnupg2 git
```

关于gopass的安装，推荐到gopass在GitHub上的[releases页面](https://github.com/gopasspw/gopass/releases)下载并安装最新的版本。根据你所使用的操作系统，选择相应的压缩包，将之下载到本地并解压，然后从中找出gopass的可执行文件，放入`$PATH`能访问的路径下。

关于oc的安装，请参考[OpenShift的官方文档](https://docs.openshift.com/container-platform/latest/cli_reference/openshift_cli/getting-started-cli.html#installing-openshift-cli)。

当增强版OpenShift CLI的所有依赖软件安装完毕以后，请从GitHub上下载增强版oc的Shell脚本：
```shell
$ curl -OL https://raw.githubusercontent.com/morningspace/oc/master/oc.sh
```

如果你用的是Bash，请在用户主目录下找到Shell初始化文件`.bashrc`，并将下列代码加入其中。如果你用的是其他Shell，比如Zsh，则相应的Shell初始化文件会有所不同。
```shell
[[ -f /path/to/oc.sh ]] && source /path/to/oc.sh
```

随后，我们打开一个新的终端窗口，并运行`oc`命令，以验证安装是否成功。如果一切正常，我们会看到，在oc命令输出的正常帮助信息前面，会有几行提示信息，代表这是一个增强版的OpenShift CLI。

## 设置环境

gopass是依赖于gpg对数据进行加密和解密的。在我们开始使用增强版oc之前，我们必须先创建一组密钥对：
```shell
$ gpg --full-generate-key
```

选择`RSA and RSA`作为密钥类型，keysize为2048，密钥永不过期，然后输入你的用户名，邮箱地址，以及密钥对应的密码（passphrase）。

此外，我们还应该把下列代码加入位于用户主目录下的Shell初始化文件内：
```shell
export GPG_TTY=$(tty)
```

然后运行gopass命令初始化secret store：
```shell
$ gopass init
```

根据提示，选择之前生成的密钥对中的私钥，用它对存储于secret store中的数据进行加密，随后输入你的邮箱地址用于git的配置。

## 首次登录集群

第一次登录某个集群时，我们将使用`oc login`命令：
```shell
$ oc login
Server [https://localhost:8443]: https://api.cluster-foo.example.com:6443
Username [kubeadmin]:
Password:
Context alias [api-cluster-foo-example-com-6443]: cluster-foo
Login successful.

You have access to 59 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default".
Save context 'cluster-foo' into secret store...
Context saved successfully.
```

根据提示，输入集群的服务器地址，登录用户名，以及密码。这些都是原有oc命令所支持的功能。在这之后，增强版oc会提示我们为即将访问的集群指定一个便于记忆的别名。这个别名相当于一个”助记符“，不仅对应于所访问的集群，还包括所使用的登录账号等信息。稍后我们会看到，增强版oc是如何利用这个别名，实现在多个集群之间的快速切换的。

当然，就像原有oc命令所支持的那样，我们也可以把上述输入的信息作为命令行参数，直接放在`oc login`命令的后面。不过，为了避免登录信息通过命令行执行历史暴露出来，还是建议大家不要这样做。

## 通过别名登录集群

当集群的登录会话超时以后，我们就需要用`oc login`命令再次进行登录。不过和原有oc命令不同，这一次我们无需再重新输入全部登录信息了，只要通过`-c`参数告诉oc命令我们要访问的集群别名即可：
```shell
$ oc login -c cluster-foo
Read context 'cluster-foo' from secret store...
Context loaded successfully.
Login successful.

You have access to 59 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default".
```

这是因为，之前我们所提供的有关该集群的全部访问信息，连同对应的别名，都已经被保存在gopass的secret store里了。因此，只要我们提供别名信息，增强版oc就会根据别名找到对应的集群访问信息，并自动登录集群。

## 小结

本文中，我们学习了增强版OpenShift CLI的安装，及其基本使用方法。在下一篇文章里，我将和大家分享有关这一工具所提供的更多有意思的功能。

如果你想了解更多有关增强版OpenShift CLI的信息，请阅读它的[在线文档](https://morningspace.github.io/oc/docs/)。如果你喜欢这个项目，欢迎给它[加星](https://github.com/morningspace/oc)。同时，也非常欢迎针对该项目的任何形式的贡献，比如bug报告以及代码提交。
