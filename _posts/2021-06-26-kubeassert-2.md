---
title: 高效使用KubeAssert
excerpt: "本文将和大家分享一些有关KubeAssert使用的技巧"
categories: [tech]
tags: [kubernetes]
toc: true
toc_sticky: true
---

> KubeAssert是一个kubectl插件，用于在命令行声明针对Kubernetes资源的断言（assertion）。KubeAssert是一个位于[GitHub](https://github.com/morningspace/kubeassert)上的开源项目。

![](/assets/images/studio/kubeassert/kubeassert-2.png)

作为KubeAssert系列文章的第二篇，本文将和大家分享一些有关KubeAssert使用的技巧。

## 使用多个Label及Field Selector

当使用像`exist`或`not-exist`这样的断言，对Kubernetes资源进行验证时，我们可以利用label selector或field selector对返回的查询结果进行过滤。例如，为了验证foo namespace下存在标签`app`值为`echo`的pod，且处于`Running`状态，我们可以采用下面的断言，并同时指定label selector和field selector：
```shell
kubectl assert exist pods \
  -l app=echo --field-selector status.phase=Running -n foo
```

我们还可以在同一个断言里根据需要同时指定多个label及field selector。例如，为了验证位于指定namespace下的处于`Running`状态的pod同时满足多个标签及其取值，我们可以这样声明断言：
```shell
kubectl assert exist pods -l app=echo -l component=proxy \
  --field-selector metadata.namespace==foo \
  --field-selector status.phase=Running \
  --all-namespaces
```

或者，我们也可以用一个`-l`或`--field-selector`参数后跟一个逗号分隔的列表，达到同样的效果。例如，下面的断言和之前的断言效果一样，但看起来更加紧凑：
```shell
kubectl assert exist pods -l app=echo,component=proxy \
  --field-selector metadata.namespace==foo,status.phase=Running \
  --all-namespaces
```

## 使用增强的Field Selector

当使用`exist`或`not-exist`断言对Kubernetes资源进行验证时，尽管我们可以通过指定field selector对查询结果进行过滤，但是这种field selector的功能是非常受限的。这是因为`exist`和`not-exist`在底层都是利用`kubectl get`来实现针对资源的查询的，并且它们都是直接用的由kubectl提供的原生`--field-selector`参数。但是根据Kubernetes官方文档，这种通过字段进行过滤的操作是在服务器端完成的，而且Kubernetes在服务器端根据资源类型的不同，只支持非常有限的字段查询能力。举个例子，当对pod声明断言时，我们可以使用field selector针对`status`下的某些字段进行过滤查询。但是同样的过滤条件对于deployment却不适用：
```shell
kubectl assert exist deployments -l app=echo --field-selector status.replicas=1
ASSERT deployments matching label criteria 'app=echo' and field criteria 'status.replicas=1' should exist.
Error from server (BadRequest): Unable to find "extensions/v1beta1, Resource=deployments" that match label selector "app=echo", field selector "status.replicas=1": "status.replicas" is not a known field selector: only "metadata.name", "metadata.namespace"
ASSERT FAIL Error getting resource(s).
```

由于这个原因，KubeAssert提供了两个额外的断言，`exist-enhanced`和`not-exist-enhanced`，它们提供了和`exist`以及`not-exist`同样的功能，但是对field selector做了增强。因此，我们可以将上述针对deployment的断言改写如下：
```shell
kubectl assert exist-enhanced deployments -l app=echo --field-selector status.replicas=1
ASSERT deployments matching label criteria 'app=echo' and field criteria 'status.replicas=1' should exist.
INFO   Found 1 resource(s).
NAME   NAMESPACE   COL0
echo   default     1
ASSERT PASS
```

原生的field selector支持`=`，`==`，以及`!=`操作符，其中`=`和`==`含义相同。而经过增强的field selector除了支持这些操作符以外，甚至还支持正则表达式匹配符`=~`。这使得我们在定义field selector时变得更加灵活和强大。下面是几个例子：

验证foo namespace下的service account是否包含指定的secret：
```shell
kubectl assert exist-enhanced serviceaccounts --field-selector 'secrets[*].name=~my-secret' -n foo
```

验证某个custom resource的status下面，至少有一个condition，其type的取值为`Deployed`：
```shell
kubectl assert exist-enhanced MyResources --field-selector 'status.conditions[*].type=~Deployed'
```

验证某个custom resource的所有实例名都是以`foo`，`bar`，或`baz`打头的：
```shell
kubectl assert exist-enhanced MyResource --field-selector metadata.name=~'foo.*|bar.*|baz.*'
```

## 验证Pod的状态

尽管原则上我们可以利用`exist`，`not-exist`，`exist-enhanced`，以及`not-exist-enhanced`对pod的状态进行验证，但是要在一条命令里用这些断言完成针对pod的某些验证是非常复杂的。

为了方便起见，KubeAssert为我们提供了几个专门针对pod的断言，它们都是以`pod-`打头的。用它们对pod的状态进行验证，可以变得更加高效：
* `pod-ready`用于验证pod的就绪状态；
* `pod-restarts`用于验证pod的重启次数；
* `pod-not-terminating`用于验证pod是否处于terminating状态；

这里有几个例子。

验证某个namespace下或所有namespace下的所有pod都处于就绪状态：
```shell
kubectl assert pod-ready pods -n foo
kubectl assert pod-ready pods --all-namespaces
```

验证某个namespace下或所有namespace下都不存在任何处于terminating状态的pod：
```shell
kubectl assert pod-not-terminating -n foo
kubectl assert pod-not-terminating --all-namespaces
```

验证某个namespace下或所有namespace下的pod，其重启次数都小于指定数值：
```shell
kubectl assert pod-restarts -lt 10 -n foo
kubectl assert pod-restarts -lt 10 --all-namespaces
```

## 检查处于Terminating状态的对象

虽然`pod-not-terminating`可以被用来检查pod是否处于terminating状态，但并非只有pod才有可能遇到这种情况。如果想检查除pod以外的其他Kubernetes资源是否也存在这种情况，可以使用`exist-enhanced`，`not-exist-enhanced`，或者编写自己的断言。

例如，假如要判断某个custom resource在任何namespace下都不存在处于terminating状态的对象，我们可以检查它是否有元数据`deletionTimestamp`，且`status.phase`字段为`Running`。当一个资源被删除时，Kubernetes会自动为其加上`deletionTimestamp`元数据。如果一个资源被删除，但同时还保持运行状态，那它就很可能是一个陷入terminating状态的对象：
```shell
kubectl assert not-exist-enhanced MyResources --field-selector metadata.deletionTimestamp!='<none>',status.phase==Running --all-namespaces
```

再来看一个例子，为了判断集群里没有处于terminating状态的namespace，我们可以检查namespace是否同时具备`deletionTimestamp`元数据和finalizer。如果满足这一条件，就说明该namespace很有可能陷入了terminating状态，因为只要namespace有finalizer存在，Kubernetes就不会主动把它删除。
```shell
kubectl assert not-exist-enhanced namespace --field-selector metadata.deletionTimestamp!='<none>',spec.finalizers[*]!='<none>'
```

## 小结

如你所见，利用各种不同的断言以及相应的参数组合，我们可以非常灵活的对Kubernetes资源进行检验。在下一篇文章里，我将和大家分享，当现有断言无法满足需求的时候，如何通过编写自己的断言来扩展KubeAssert的能力。

如果你想了解更多有关KubeAssert的信息，请阅读它的[在线文档](https://morningspace.github.io/kubeassert/docs/)。如果你喜欢这个项目，欢迎给它[加星](https://github.com/morningspace/kubeassert)。同时，也非常欢迎针对该项目的任何形式的贡献，比如bug报告以及代码提交。
