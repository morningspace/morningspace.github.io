---
title: KubeAssert快速入门
excerpt: "本文将告诉你什么是KubeAssert，如何安装并运行KubeAssert"
categories: [tech]
tags: [kubernetes]
toc: true
toc_sticky: true
---

> KubeAssert是一个kubectl插件，用于在命令行声明针对Kubernetes资源的断言（assertion）。KubeAssert是一个位于[GitHub](https://github.com/morningspace/kubeassert)上的开源项目。

![](/assets/images/studio/kubeassert/kubeassert-1.png)

作为KubeAssert系列文章的第一篇，本文将告诉你什么是KubeAssert，如何安装并运行KubeAssert。

## 什么是KubeAssert

KubeAssert是一个kubectl插件，它提供了一组断言（assertion），可以用于在命令行对Kubernetes资源进行判断。利用KubeAssert，可以帮助我们检查和分析Kubernetes集群的状态，集群中资源的部署情况。例如，我们可以通过KubeAssert来判断：
* 某个资源是否存在于当前集群中；
* 某个资源的状态是否满足指定条件；
* 某个资源的实例数是否小于或等于指定数值；

使用KubeAssert声明针对Kubernetes资源的断言和我们在代码里用xUnit编写单元测试中的断言非常类似。将这样的一组断言放到一个脚本文件里，可以作为针对Kubernetes集群的自动化测试的一部分，和CI/CD pipeline进行集成。

## 安装KubeAssert

由于KubeAssert本质上是由脚本实现的，所以它可以作为一个独立的命令在命令行执行。但同时，它也可以被安装成一个kubectl插件，这样我们就可以像执行普通的`kubectl <command>`命令那样执行KubeAssert了。

KubeAssert目前已被提交至[krew](https://krew.sigs.k8s.io/)。作为Kubernetes社区官方的kubectl插件管理工具，krew提供了[krew-index](https://sigs.k8s.io/krew-index)，用于在社区范围内集中发布与分享kubectl插件。因此，安装KubeAssert最简单的方式就是使用`krew`：
```shell
krew install assert
```

如果没有安装krew，也没关系。我们可以从GitHub库将其下载到本地，并将文件设置为可执行：
```shell
curl -L https://raw.githubusercontent.com/morningspace/kubeassert/master/kubectl-assert.sh -o kubectl-assert
chmod +x kubectl-assert
```

把脚本文件放到任何`PATH`环境变量能够找到的地方，比如：
```shell
mv ./kubectl-assert /usr/local/bin
```

然后，我们就可以像执行普通的kubectl命令那样调用KubeAssert了：
```shell
kubectl assert
```

执行上述命令，将会返回KubeAssert的帮助信息，以及一个KubeAssert当前支持的断言清单。如果在执行`kubectl assert`命令时指定了断言的名字以及`--help`参数，就会得到更多有关雨如何使用相应断言的详细信息。例如：
```shell
kubectl assert exist --help
```
## 使用KubeAssert

以下给出的是目前为止KubeAssert默认提供的所有断言：

| Assertion             | Description
|:----------------------|:----------------------
| exist                 | Assert resource should exist.
| not-exist             | Assert resource should not exist.
| exist-enhanced        | Assert resource should exist using enhanced field selector.
| not-exist-enhanced    | Assert resource should not exist using enhanced field selector.
| num                   | Assert the number of resource should match specified criteria.
| pod-ready             | Assert pod should be ready.
| pod-not-terminating   | Assert pod should not keep terminating.
| pod-restarts          | Assert pod restarts should match specified criteria.
| apiservice-available  | Assert API service should be available.

这里给出了一些例子，演示了如何利用这些断言来帮助我们对Kubernetes集群中的资源进行检查。

检查foo namespace下是否存在任何满足标签`app`取值为`echo`的pod：
```
kubectl assert exist pods -l app=echo -n foo
```

检查是否所有namespace下的pod都处于就绪（ready）状态：
```
kubectl assert pod-ready --all-namespaces
```

检查pod的重启次数小于指定数值：
```
kubectl assert restarts pods -n foo -lt 10
```

## 小结

如你所见，KubeAssert的使用非常简单。在下一篇文章里，我将和大家分享更多有关KubeAssert使用的技巧。

如果你想了解更多有关KubeAssert的信息，请阅读它的[在线文档](https://morningspace.github.io/kubeassert/docs/)。如果你喜欢这个项目，欢迎给它[加星](https://github.com/morningspace/kubeassert)。同时，也非常欢迎针对该项目的任何形式的贡献，比如bug报告以及代码提交。
