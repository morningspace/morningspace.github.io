---
title: Git工作流面面观——分支模型
excerpt: Git强大的分支模型，所有Git工作流的基础
categories: [tech]
tags: [lab, dummies, dummies_git, git]
toc: true
toc_sticky: true
---

注：
本文采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/80x15.png" /></a>

![](/assets/images/lab/git/workflow.png)

## Git工作流

本文所讨论的，是围绕Git的几个常见的工作流程(Workflow)。但它并不代表全部，Git对变更的管理有着非常丰富而灵活的手段，所以并不存在唯一的标准流程，或者是“包治百病”的流程。了解Git工作流的意义在于，它能给我们带来启示，让我们学会善用Git，让它能更好地服务于软件开发，在保持开发规范性的同时，还能提升开发的效率。

Git不只是单纯的版本控制工具，还是团队协作开发与软件发布管理的工具。所以，它和软件开发流程、产品或项目的发布流程都是紧密关联的，这些都会体现在工作流里面。

## 关于示例

本文我们将通过模拟示例来演示各个Git工作流的运行方式。本文推荐大家使用[Hello Git](https://github.com/morningspace/lab-hello-git)提供的两个Docker镜像作为实验环境：一个代表远程Git服务（`lab-git-remote`），一个代表本地Git客户端（`lab-git-local`）。这两个镜像都可以从Docker Hub上找到：
```shell
docker pull morningspace/lab-git-remote
docker pull morningspace/lab-git-local
```

有关这两个Docker镜像的具体使用方法，请见[Hello Git](https://github.com/morningspace/lab-hello-git)项目的README。本文后续讨论的所有动手环节，都将围绕这两个Docker镜像展开。

为了方便后面做实验，我们先把这两个Docker镜像启动起来。首先是远程Git服务：
```shell
docker run -it --name team-git-remote --hostname team-git-remote --net=lab -p 8080:80 morningspace/lab-git-remote
```

然后为两个模拟用户启动本地Git客户端，分别是William和Nicole：
```shell
docker run -it --name git-local-william --hostname git-local-william --net=lab -e user_name=William -e user_email=william@example.com morningspace/lab-git-local
docker run -it --name git-local-nicole --hostname git-local-nicole --net=lab -e user_name=Nicole -e user_email=nicole@example.com morningspace/lab-git-local
```

注意，这里的用户名和邮件地址是通过参数`-e`传入容器的，容器会根据传入的值，自动生成相应的Public Key。这个Public Key在容器启动时会打印到控制台。为了让Git客户端成功访问远程Git服务，我们需要从Git客户端通过SSH以root账号登录到远程Git服务（密码为passw0rd），然后把Public Key加入`/home/git/.ssh/authorized_keys`文件：
```shell
$ ssh root@team-git-remote
$ echo <your_public_key> >> /home/git/.ssh/authorized_keys
```

## 分支模型

Git最大的特色之一就是它的分支模型(Branching Model)，这是所有Git工作流的基础。所以，在了解工作流之前，有必要先了解一下Git的分支模型。

一个分支通常对应着从业务角度出发的一项或者多项可交付内容，比如：一系列用户故事，或者是bug修复，它让当前工作可以在一个单纯隔离的环境里进行。同时，Git也支持不同分支之间的快速切换，让我们的工作得以在多个上下文之间实现快速切换。

分支在发展到一定阶段以后，根据需要可能还要进行合并，而合并总是会有代价的。在当前分支合并到其他分支，尤其是产品主分支之前，我们要做好充分的测试，以避免合并的返工。如果真的出现返工，也要尽量降低回退的成本。比如：在合并时采用`--no-ff`参数，可以保证合并后的提交历史依然保留合并以前的“分叉”，从而在返工的时候能够清楚地找到回退的位置。

根据保留时间的长短和使用目的的不同，分支可以有不同的类型。下面我们就来认识几种常见的分支类型。分支类型的选择，要因地制宜，要和产品或项目的发布计划相适应，要满足业务上的需要。比如：如果发布频率很高，达到了一天多次，那么对于实施发布的分支，就要确保它的持续稳定性；如果频率不是那么高，那就可以考虑给分支加tag，允许分支在两个tag之间存在一定程度的不稳定。

### 短期分支

一个分支和产品主分支分开的时间越长，合并和部署的代价就越高。所以，项目里通常更推荐使用短期分支(Short-lived Branch)。这种分支，往往针对于某项特性的开发，或者是bug修复，所有在这个分支上进行的开发活动都是围绕同一主题展开的。所以，它也被称为Topic分支(Topic Branch)。无论项目的规模是大是小，Topic分支的使用都是很常见的。

![](/assets/images/lab/git/git-workflow-1.png)

Topic分支通常是由个人维护的，带有“实验”或“私有”性质。所以，它的提交历史可以带有随意性，而且根据需要是可以改写的。比如：把一些琐碎的提交记录合并成一个更大的有意义的提交。出于代码备份和协作共享的目的，这些分支也可以被推送到远程，包括：把被改写的提交历史强制推送到远程。在团队协作开发当中，通常不建议其他人直接把这种分支同步到本地进行开发。因为，提交历史的改写会导致他们本地的分支出现混乱。关于这一点，大家可以在[“Git合并的那些事儿——Rebase的烦恼”](/tech/git-merge-stories-8)一文里找到更多信息。

### 长期分支

和短期分支相对应，那些长期处于Open状态的分支，被称为长期分支(Long-Running Branch)，它们常被用于软件开发周期的不同阶段，并且会周期性地向产品的主分支合并。虽然在一个项目里开很多长期分支是不必要的，但是对于一个规模较大的复杂项目来说，长期分支也是很有帮助的。接下来我们看到的一些其他类型的分支实际上就属于长期分支。

### 集成分支

为了降低分支合并与集成的风险，作为开发过程中的一个阶段性的节点，有的开发团队会在开始新的特性开发之前，先把提交合并到集成分支(Integration Branch)。然后，再从集成分支把代码部署到集成环境里进行测试，这个过程通常可以由Pipeline自动完成。集成分支往往是长期分支。

![](/assets/images/lab/git/git-workflow-2.png)

### 环境分支

根据系统发布的不同阶段，建立与各个阶段相对应的分支，并利用Pipeline把这些分支上的代码部署到对应的运行环境里进行测试，从而分阶段地保证系统在真正进入生产环境之前的质量，这样的分支被称为环境分支(Environment Branch)。环境分支可以认为是集成分支的一种扩展应用，每一个环境分支都是一个集成分支。

![](/assets/images/lab/git/git-workflow-3.png)

对于复杂系统，这种分支在使用过程中可能会造成瓶颈。因为它把Git的工作流程和阶段性集成环境的稳定性耦合在一起了。如果集成环境出现问题，就可能会阻碍代码的提交与合并，造成待发布特性的“堆积”。

### 发布分支

发布分支(Release Branch)和系统的发布周期密切相关，所以在给分支起名的时候常常会带上发布的版本号。它的目的是为了把一组特性作为一个整体，在规定的时间里(比如：几个迭代周期内)，推送到生产环境里。所以，发布分支类似于Topic分支，是有明确主题的。它的生命周期比环境分支要短，只在需要的时候创建，而当特性被成功部署到生产环境以后，就可以销毁了。

![](/assets/images/lab/git/git-workflow-4.png)

如果希望把发布分支的职责限制在代码进入生产环境之前的最后一道检查，那么我们可以在代码合并到发布分支之前，先让它进入集成分支进行测试。为了避免多个版本并存时的维护代价，以及不同发布分支之间的同步代价，有的团队只允许一个发布分支存在。

### 版本分支

如果我们的系统不只跑在公共的生产环境里，还跑在每个客户各自的私有环境里，那么这些环境所使用的系统就有可能是不同的版本。如果要同时维护这些版本，那就需要用到版本分支(Version Branch)：针对每一个支持的版本，都要有相应的分支与之对应。

![](/assets/images/lab/git/git-workflow-5.png)
