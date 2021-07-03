---
title: 为KubeAssert定制断言
excerpt: "本文向大家介绍如何编写自己的断言来扩展KubeAssert的能力"
categories: [tech]
tags: [kubernetes]
toc: true
toc_sticky: true
---

> KubeAssert是一个kubectl插件，用于在命令行声明针对Kubernetes资源的断言（assertion）。KubeAssert是一个位于[GitHub](https://github.com/morningspace/kubeassert)上的开源项目。

![](/assets/images/studio/kubeassert/kubeassert-3.png)

作为KubeAssert系列文章的第三篇，本文将向大家介绍，当现有断言无法满足需求的时候，如何通过编写自己的断言来扩展KubeAssert的能力。

## 自己动手写断言

虽然KubeAssert默认提供的断言可以支持Kubernetes集群日常操作的很多场景，但是，也有可能这些断言并不能满足你的特定需求。KubeAssert支持定制断言，它允许你自己动手编写断言，然后让KubeAssert加载并运行。

作为一个例子，本文我们将编写一个自定义断言，用来验证kubeconfig里是否包含指定的集群。其中，集群的名称是在该断言被执行的时候通过命令行参数传入的。

## 创建Shell脚本文件

首先，让我们创建一个shell脚本文件，取名为`custom-assertions.sh`，并将它放到`$HOME/.kubeassert`目录下。这里是KubeAssert加载所有断言的地方。每当KubeAssert启动时，它会寻找该目录下的所有`.sh`文件，并作为断言加载到内存。

一个脚本文件可以包含多个断言，每个断言都是通过shell函数来实现的。本例中，我们只有一个断言，对应函数名是`cluster`，目前函数体还是空的：
```shell
#!/bin/bash
function cluster {
 :
}
```

为了测试断言，我们可以调用`kubectl assert`命令，并指定函数名`cluster`，作为该断言的名字：
```shell
kubectl assert cluster
ASSERT PASS
```

正常情况下，我们会看到命令能够正确返回，只是返回结果是空的。接下来，让我们为函数加入具体的实现逻辑。

## 实现断言的逻辑

为了实现一个断言，通常我们需要做下面这几件事情：
* 验证输入参数：在本例中，就是由用户通过命令行输入的集群名。如果验证失败，比如用户没有指定集群名，可以调用函数`logger::error`输出错误信息，然后以非零值立即返回并退出断言。
* 打印断言信息：通过调用函数`logger::assert`，我们可以在命令行打印出一条断言信息，以便告诉用户这个断言即将要做的事情。
* 执行断言逻辑：作为断言的主体部分，此处通常会调用一系列kubectl命令或Kubernetes API，例如：调用`kubectl config`命令，从kubeconfig中查询集群信息。
* 验证断言结果：通过对kubectl或Kubernetes API的返回结果进行解析，判断断言成功与否。如果断言失败，则调用函数`logger::fail`输出一条断言失败的信息。如果成功，则不需要做任何事情，或者也可以根据需要调用`logger::info`函数打印一些日志信息。

将下列逻辑加入`cluster`函数：
```shell
#!/bin/bash
function cluster {
  # Validate input arguments
  [[ -z $1 ]] && logger::error "You must specify a cluster name." && exit 1
  # Print assertion message
  logger::assert "Cluster with name $1 should be included in kubeconfig."
  # Run some kubectl commands
  kubectl config get-clusters
  # Validate results
  if cat $HOME/.kubeassert/result.txt | grep -q ^$1$; then
    # Print normal logs
    logger::info "Found $1 in kubeconfig."
  else
    # Print failure message
    logger::fail "$1 not found."
  fi
}
```

可能你已经注意到了，函数中对于断言结果的验证，是通过解析一个位置处于`$HOME/.kubeassert/`目录下的名为`result.txt`的文件实现的。KubeAssert把调用kubectl的返回结果都存入了该文件。

现在，让我们试一下执行该断言时不指定集群名的情况：
```shell
kubectl assert cluster
ERROR You must specify a cluster name.
```

结果报错，并提示调用断言时必须指定集群名。再试一下指定一个kubeconfig中不存在的集群名：
```shell
kubectl assert cluster kind
ASSERT Cluster with name kind should be included in kubeconfig.
ASSERT FAIL kind not found.
```

结果显示断言失败，并给出了原因。指定一个kubeconfig中存在的集群名：
```shell
kubectl assert cluster kind-foo
ASSERT Cluster with name kind-foo should be included in kubeconfig.
INFO Found kind-foo in kubeconfig.
ASSERT PASS
```

这次结果显示断言成功了。

如果断言执行过程中遇到了问题，需要进行排查，我们也可以在执行断言时加上`-v`参数，以打开详细日志输出的开关。这样，我们就可以看到断言中涉及了哪些kubectl调用，以及它们各自的返回结果是什么。
```shell
kubectl assert cluster kind-foo -v
ASSERT Cluster with name kind-foo should be included in kubeconfig.
INFO kubectl config get-clusters
NAME
kind-foo
kind-bar
INFO Found kind-foo in kubeconfig.
ASSERT PASS
```

## 为断言添加注释

到目前为止，我们已经实现了断言的全部逻辑。不过，还有一件事情我们需要做。当执行`kubectl assert`并带着`-h/--help`参数或者不带任何参数时，为了能够让我们的断言出现在已安装断言的列表里，我们还需要为断言添加一个特别的注释。

此外，当我们执行`kubectl assert <assertion>`命令并带着`-h/--help`参数时，为了能够打印输出针对该断言的帮助信息，我们还需要提前准备好相应的帮助信息。这些信息也是通过断言注释添加的。

为断言添加的注释必须以`##`开头，并以`##`结尾，两者之间包含了若干个字段，每个字段的名字都是以`@`打头的。下面是一个注释模版，包含了每个字段的详细解释：
```shell
##
# @Name: <Input your single-line assertion name here>
# @Description: <Input your single-line assertion description here>
# @Usage: <Input your single-line assertion usage information here>
# @Options:
# <Input help information for all your options started from here>
# <It supports multiple lines>
# @Examples:
# <Input detailed information for all examples started from here>
# <It supports multiple lines>
##
```

对于`Options`字段，KubeAssert提供了几个预定义变量可供我们使用。如果我们的断言支持某些KubeAssert预定义的通用Option，那么在定义`Options`字段时，我们就可以引用这些预定义变量作为占位符。在输出实际的帮助信息时，KubeAssert会把这些占位符替换成真正的内容。
* ${GLOBAL_OPTIONS}：该变量代表了适用于所有断言的全局Option，比如：用于输出帮助信息的`-h/--help`。
* ${SELECT_OPTIONS}：该变量代表了用于对资源进行过滤的Option，比如：用于标签过滤的`-l/--selector`，以及用于指定namespace的`-n/--namespace`。
* ${OP_VAL_OPTIONS}：该变量代表了比较运算符，例如：`-eq`，`-lt`，`-gt`，`-ge`，`-le`。

通过使用这些变量，我们可以确保所有断言的帮助信息里涉及这些Option的描述都是一致的。

下面是针对`cluster`断言的注释内容：
```shell
##
# @Name: cluster
# @Description: Assert specified cluster included in kubeconfig
# @Usage: kubectl assert cluster (NAME) [options]
# @Options:
# ${GLOBAL_OPTIONS}
# @Examples:
# # To assert a cluster is included in kubeconfig
# kubectl assert cluster kind-foo
##
function cluster {
 ...
}
```

为了验证，我们可以执行如下命令，看一下我们的断言是否已经出现在已安装断言的列表中了：
```shell
kubectl assert --help
```

如果一切正常，我们会在返回结果中找到`cluster`。执行如下命令，打印输出断言的帮助信息：
```shell
kubectl assert cluster --help
```

我们会看到如下输出结果，这正是我们在注释里定义的内容：
```shell
Assert cluster with specified name included in kubeconfig file
Usage: kubectl assert cluster (NAME) [options]
Options:
  -h, --help: Print the help information.
  -v, --verbose: Enable the verbose log.
  -V, --version: Print the version information.
```

## 小结

断言cluster的最终版本如下：
```shell
#!/bin/bash
##
# @Name: cluster
# @Description: Assert specified cluster included in kubeconfig
# @Usage: kubectl assert cluster (NAME) [options]
# @Options:
#   ${GLOBAL_OPTIONS}
# @Examples:
#   # To assert a cluster is included in kubeconfig
#   kubectl assert cluster kind-foo
##
function cluster {
  # Validate input arguments
  [[ -z $1 ]] && logger::error "You must specify a cluster name." && exit 1
  # Print assertion message
  logger::assert "Cluster with name $1 should be included in kubeconfig."
  # Run some kubectl commands
  kubectl config get-clusters
  # Validate results
  if cat $HOME/.kubeassert/result.txt | grep -q ^$1$; then
    # Print normal logs
    logger::info "Found $1 in kubeconfig."
  else
    # Print failure message
    logger::fail "$1 not found."
  fi
}
```

如你所见，为KubeAssert定制断言是非常容易的。在下一篇文章里，我将和大家分享，如何把KubeAssert和[KUTTL](https://kuttl.dev/)集成在一起，让针对Kubernetes集群的自动化测试如虎添翼。

如果你想了解更多有关KubeAssert的信息，请阅读它的[在线文档](https://morningspace.github.io/kubeassert/docs/)。如果你喜欢这个项目，欢迎给它[加星](https://github.com/morningspace/kubeassert)。同时，也非常欢迎针对该项目的任何形式的贡献，比如bug报告以及代码提交。
