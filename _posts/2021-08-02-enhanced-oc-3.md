---
title: 增强版OpenShift CLI的更多功能
excerpt: "本文和大家分享有关增强版oc所提供的更多有意思的功能"
categories: [tech]
tags: [kubernetes, openshift]
toc: true
toc_sticky: true
---

> 增强版OpenShift CLI可以帮助我们高效而安全的管理大量的OpenShift集群。它是一个位于[GitHub](https://github.com/morningspace/enhanced-oc)上的开源项目。

作为增强版OpenShift CLI系列文章的第三篇，本文将和大家分享有关这一工具所提供的更多有意思的功能。

## 层级化管理集群

在一个实际项目里，同时管理多个集群是一件很常见的事。使用增强版oc，通过指定别名可以实现在多个集群之间的快速切换，这样做即安全又高效。不过，随着集群数量的增加，管理成本还是会逐步上升。针对这种情况，一种更为有效的方法，是将这些集群以层级化的方式进行管理。

增强版oc底层依赖于gopass，而gopass天生就具备层级化管理加密数据的能力。这是因为，gopass在存储加密数据时，把一组加密数据以一个独立文件的形式保存在secret store里，而彼此相关的一组文件可以根据需要存放在同一目录下面。例如，假设有三个集群，我们可以采用`path/to/your/context`这样的格式来为集群定义别名。
```shell
$ oc login -s https://api.foo.example.com -c dev-env/cluster-foo
$ oc login -s https://api.bar.example.com -c dev-env/cluster-bar
$ oc login -s https://api.baz.example.com -c dev-env/cluster-baz
```

这样，集群的访问信息就会以层级化的方式保存在secret store里，相应的目录结构与我们定义别名时的格式保持一致。这一点可以通过执行`gopass ls`命令清楚直观的展现出来：
```shell
$ gopass ls
gopass
└── dev-env
    ├── cluster-foo
    ├── cluster-bar
    └── cluster-baz
```

当我们在这些集群之间切换时，只要给出`path/to/your/context`别名即可：
```shell
$ oc login -c dev-env/cluster-foo
$ oc login -c dev-env/cluster-bar
$ oc login -c dev-env/cluster-baz
```

## 在多个集群中选择

以层级化方式对集群进行管理允许我们能够非常高效的管理大量集群，我们可以把集群按照不同的用途进行分类。不过，由于别名里包含了层级结构，输入完整的别名就会变得有点冗长。

增强版oc允许我们在输入集群的别名时只输入名字的一部分。例如，如果我们把所有开发用的集群环境都放到`dev-env`类别下，相应的集群别名均以`dev-env`打头。那么，当我们指定别名的时候，只要输入完整名字的几个首字母，比如：`de`，`dev`，`dev-`，`dev-env`，就会有多个满足输入条件的集群返回回来。这些集群会以有序列表的形式依次排列，要想选中其中一个集群登录，只要输入集群前面的数字即可：
```shell
$ oc login -c dev
1) dev-env/cluster-bar
2) dev-env/cluster-baz
3) dev-env/cluster-foo
#? 1
Read context 'dev-env/cluster-bar' from secret store...
Context loaded successfully.
Login successful.

You have access to 59 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default".
```

## 模糊查找

增强版oc允许我们在输入集群的别名时只输入名字的一部分，从而进一步提高了在多个集群间切换的效率。不仅如此，如果我们安装了[fzf](https://github.com/junegunn/fzf)，一个通用的支持模糊查找与过滤的命令行工具，就可以在指定别名时利用typeahead实现模糊查找的功能了。增强版oc与fzf做了无缝集成，不需要额外配置，它会自动监测fzf存在与否，并开启相应的模糊查找功能。有关fzf的安装，请参考其[文档](https://github.com/junegunn/fzf#installation)。

![](/assets/images/studio/enhanced-oc/enhanced-oc-2.gif)

## 定制Shell提示

我们还可以安装[kube-ps1](https://github.com/jonmosco/kube-ps1)。这是一个脚本程序，允许把当前集群的上下文信息，包括当前所在的namespace，在命令行的Shell提示里显示出来。当我们需要同时管理多个集群，并在这些集群之间来回切换的时候，这一功能就会非常有用。通过查看命令行的Shell提示信息，我们可以快速获知当前操作的是哪个集群，从而避免了误操作。

增强版oc和kube-ps1做了无缝集成，不需要额外配置，它会自动监测kube-ps1存在与否，并对kube-ps1控制的Shell提示进行定制，将完整的集群上下文名称替换为相应的集群别名，通常而言，后者要比前者更短更易于记忆。有关kube-ps1的安装，请参考其[文档](https://github.com/jonmosco/kube-ps1#installing)。

![](/assets/images/studio/enhanced-oc/enhanced-oc-3.gif)

## 小结

本文中，我们学习了有关增强版oc的更多有趣的功能，比如：以层级化方式对集群进行管理；通过只输入别名的一部分，实现多个集群间的快速切换；以及通过对Shell提示的定制，显示当前操作的集群。在下一篇文章里，我将向大家介绍增强版oc的另一个重要的功能：如何将集群访问信息分享给团队中的其他成员。

如果你想了解更多有关增强版OpenShift CLI的信息，请阅读它的[在线文档](https://morningspace.github.io/oc/docs/)。如果你喜欢这个项目，欢迎给它[加星](https://github.com/morningspace/oc)。同时，也非常欢迎针对该项目的任何形式的贡献，比如bug报告以及代码提交。
