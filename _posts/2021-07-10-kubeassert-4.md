---
title: 集成KubeAssert与KUTTL
excerpt: "本文向大家介绍如何集成KubeAssert与[KUTTL](https://kuttl.dev/)，让Kubernetes自动化测试如虎添翼"
categories: [tech]
tags: [kubernetes]
toc: true
toc_sticky: true
---

> KubeAssert是一个kubectl插件，用于在命令行声明针对Kubernetes资源的断言（assertion）。KubeAssert是一个位于[GitHub](https://github.com/morningspace/kubeassert)上的开源项目。

![](/assets/images/studio/kubeassert/kubeassert-4.png)

作为KubeAssert系列文章的第四篇，本文将向大家介绍，如何把KubeAssert和[KUTTL](https://kuttl.dev/)集成在一起，让针对Kubernetes集群的自动化测试如虎添翼。

## 关于KUTTL

KUbernetes Test TooL (KUTTL)是一个Kubernetes的自动化测试工具，它利用YAML语法，提供了以声明式手段对Kubernetes进行测试的方法。KUTTL能在测试准备阶段动态注入即将要被测试的operator，并允许以YAML文件的形式对测试用例进行描述。测试过程中出现的断言，通常也是一系列YAML文件，用来验证Kubernetes资源的状态字段是否满足预期条件。此外，KUTTL也可以被用来实现Kubernetes集群的自动化部署。获取有关KUTTL的更多信息，可以访问它的[网站](https://kuttl.dev/)。

## 结合KubeAssert与KUTTL

在KUTTL里，测试的断言是用YAML语言来定义的。我们可以用YAML语法来描述某个Kubernetes对象是否匹配指定的对象名称，或者相应的状态字段。如果我们为对象指定了预期的名称，KUTTL就会在集群中查找匹配该条件的Kubernetes对象。例如，假设我们定义的断言如下所示：
```yaml
apiVersion: v1
kind: Pod
metadata:
 name: my-pod
status:
 phase: Successful
```

那么KUTTL就会在运行测试用例的namespace里寻找名为`my-pod`的pod资源，并等待其字段`status.phase`的值变成`Successful`。

然而，KUTTL的断言表达能力非常有限。例如，默认情况下，KUTTL是无法用这种YAML格式的断言来验证pod的重启次数小于某个期望数值的，它也无法验证没有pod处于terminating状态这样的情况。这就是KubeAssert的用武之地了！

幸运的是，从v0.9.0开始，KUTTLE允许用户在声明断言时指定命令行命令或脚本文件。这就给了我们把KubeAssert和KUTTL集成在一起的机会，从而允许我们能写出更加灵活而强大的断言来。

## 用KubeAssert编写KUTTL断言

接下来，让我们以KUTTL官网上的“[Writing Your First Test](https://kuttl.dev/docs/kuttl-test-harness.html#writing-your-first-test)”为例，看一下我们是如何把它改造成用KubeAssert来编写断言的。

### 创建测试用例

首先，我们来创建一个`tests/e2e`目录，作为存放所有测试用例的根目录，并在该目录下创建`example-test`子目录，代表对应的测试用例：
```shell
mkdir -p tests/e2e/example-test
```

接下来，在`tests/e2e/example-test/`目录下创建测试步骤`00-install.yaml`，用于创建一个名为`example-deployment`的deployment：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

然后，创建测试断言`tests/e2e/example-test/00-assert.yaml`：
```yaml
apiVersion: kuttl.dev/v1beta1
kind: TestAssert
commands:
- command: kubectl assert exist-enhanced deployment example-deployment -n $NAMESPACE --field-selector status.readyReplicas=3
```

此处，我们使用了TestAssert，并通过`command`定义调用了KubeAssert。判断如果`example-deployment`的`status.readyReplicas`字段取值为3，则认为测试步骤执行完毕。请注意，我们在这里使用了环境变量`$NAMESPACE`。这是KUTTL提供的，用来指明当前测试所在的namespace。

### 编写第二个测试步骤

在第二个测试步骤里，我们把deployment的replica数从3增加到了4。这是在`tests/e2e/example-test/01-scale.yaml`中定义的：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-deployment
spec:
  replicas: 4
```

为此，我们在`tests/e2e/example-test/01-assert.yaml`中利用KubeAssert定义了相应的断言：
```yaml
apiVersion: kuttl.dev/v1beta1
kind: TestAssert
commands:
- command: kubectl assert exist-enhanced deployment example-deployment -n $NAMESPACE --field-selector status.readyReplicas=4
```

此处，我们所声明的断言几乎和前一个测试步骤里定义的一摸一样。唯一的区别是，我们把`status.readyReplicas`的期望值改成了4。

执行该测试用例并验证测试是否通过：
```shell
kubectl kuttl test - start-kind=true ./tests/e2e/
```

有关这一示例的更多信息，请访问位于KUTTL网站上的[原文](https://kuttl.dev/docs/kuttl-test-harness.html#writing-your-first-test)。

## 小结

如你所见，把KubeAssert与KUTTL集成在一起非常的容易。本文中，我们只演示了KubeAssert的几个非常基本的断言功能。根据实际需要，我们还可以在编写KUTTL的测试用例时定义更加复杂的KubeAssert断言。

如果你想了解更多有关KubeAssert的信息，请阅读它的[在线文档](https://morningspace.github.io/kubeassert/docs/)。如果你喜欢这个项目，欢迎给它[加星](https://github.com/morningspace/kubeassert)。同时，也非常欢迎针对该项目的任何形式的贡献，比如bug报告以及代码提交。
