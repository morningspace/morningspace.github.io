---
title: Git合并那些事——认识几种Merge方法
excerpt: 介绍什么是Fast-Forward Merge，Three-Way Merge，Squash Merge
categories: [tech]
tags: [lab, dummies, dummies_git, git]
toc: true
toc_sticky: true
---

> 多人开发中的合并冲突是我们使用Git时常常会遇到的情况，小小合并门道大，讲述合并的那些事儿，晴耕 · 白话之“Git合并那些事”系列​持续连载中……

注：
本文采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/80x15.png" /></a>

![](/assets/images/lab/git/merge.png)

## 准备实验环境

本文推荐大家使用[Hello Git](https://github.com/morningspace/lab-hello-git)提供的两个Docker镜像作为实验环境：一个代表远程Git服务（`lab-git-remote`），一个代表本地Git客户端（`lab-git-local`）。这两个镜像都可以从Docker Hub上找到：
```shell
docker pull morningspace/lab-git-remote
docker pull morningspace/lab-git-local
```

有关这两个Docker镜像的具体使用方法，请见[Hello Git](https://github.com/morningspace/lab-hello-git)项目的README。本文后续讨论的所有动手环节，都将围绕这两个Docker镜像展开。

为了方便后面做实验，我们先把这两个Docker镜像启动起来。首先是远程Git服务：
```shell
docker run -it --name team-git-remote --hostname team-git-remote --net=lab -p 8080:80 morningspace/lab-git-remote
```

然后是本地Git客户端：
```shell
docker run -it --name git-local-william --hostname git-local-william --net=lab -e user_name=William -e user_email=william@example.com morningspace/lab-git-local
```

注意，这里的用户名和邮件地址是通过参数`-e`传入容器的，容器会根据传入的值，自动生成相应的Public Key。这个Public Key在容器启动时会打印到控制台。为了让Git客户端成功访问远程Git服务，我们需要从Git客户端通过SSH以root账号登录到远程Git服务（密码为passw0rd），然后把Public Key加入`/home/git/.ssh/authorized_keys`文件：
```shell
$ ssh root@team-git-remote
$ echo <your_public_key> >> /home/git/.ssh/authorized_keys
```

最后，我们来创建一个本地Git库，本文我们要讨论的所有关于合并的故事都将在这个Git库里发生：
```shell
$ git init merge-stories
Initialized empty Git repository in /root/merge-stories/.git/
$ cd merge-stories
```

## Fast-Forward Merge

首先，我们来看一下什么是Fast-Forward Merge（快进式合并）。Fast-Forward Merge是Git的各种合并方式中最容易理解的，也是较为常见的一种情况。

下面我们先往本地Git库里提交一些变更。在命令行下，按照顺序依次提交变更c1：
```shell
$ vi README
$ cat README
c1
$ git add .
$ git commit -m c1
[master (root-commit) cfbff28] c1
 1 file changed, 1 insertion(+)
 create mode 100644 README
```

和c2：
```shell
$ vi README
$ cat README 
c1 - c2
$ git commit -am c2
[master 9229adb] c2
 1 file changed, 1 insertion(+), 1 deletion(-)
```

然后，再从master分支切换到dev分支：
```shell
$ git checkout -b dev
Switched to a new branch 'dev'
```

继续提交变更c3：
```shell
$ vi README 
$ cat README 
c1 - c2
      + - c3
$ git commit -am c3
[dev c594fa9] c3
 1 file changed, 1 insertion(+)
```

和c4：
```shell
$ vi README
$ cat README 
c1 - c2
      + - c3 - c4
$ git commit -am c4
[dev 0f029c3] c4
 1 file changed, 1 insertion(+), 1 deletion(-)
```

执行上述命令后，用`git log`可以看到我们的提交历史：
```shell
$ git log --oneline --graph --all
* 0f029c3 (dev) c4
* c594fa9 c3
* 9229adb (HEAD -> master) c2
* cfbff28 c1
```

我们在两个分支上的提交历史如图所示：

![](/assets/images/lab/git/merge-stories-1.jpg){: .align-center}

现在，让我们切换回master分支：
```shell
$ git checkout master
Switched to branch 'master'
```

并把dev分支上的变更合并到master：
```shell
$ git merge dev
Updating 9229adb..0f029c3
Fast-forward
 README | 1 +
 1 file changed, 1 insertion(+)
```

可以看到输出结果中的“Fast-forward”，这表明Git在做合并时采用的是Fast-Forward Merge。此时，再观察一下Git的提交历史：
```shell
$ git log --oneline --graph --all
* 0f029c3 (HEAD -> master, dev) c4
* c594fa9 c3
* 9229adb c2
* cfbff28 c1
```

以及相应的图示：

![](/assets/images/lab/git/merge-stories-2.png)

我们会发现，由于master分支从c2开始与dev分叉以后就再也没有新的提交了，所以Git只是简单地把master的head指针向前移动到c4，合并就完成了。这就是所谓的Fast-Forward Merge。因为不涉及内容变更的比较，所以这种合并方式效率很高。Fast-Forward Merge要求参与合并的两个分支上的提交必须是“一脉相承”的父子或祖孙关系。不过它有个缺点，作为被合并的dev分支，它的提交历史在合并以后会和master分支的提交历史重合。

如果我们想在合并后保留来自被合并分支的提交历史，并显式标注出合并发生的位置，那就需要在执行合并时加上参数`--no-ff`。当然，这样也表示我们在合并时将不使用Fast-Forward Merge。为了演示这一点，让我们用git reset回退到c2：
```shell
$ git reset --hard 9229adb
HEAD is now at 9229adb c2
```

然后，再进行一次从dev到master的合并，并指定参数`--no-ff`：
```shell
$ git merge --no-ff -m c5 dev
Merge made by the 'recursive' strategy.
 README | 1 +
 1 file changed, 1 insertion(+)
```

这一次，由于没有采用Fast-Forward Merge，Git会为我们生成一个新的提交。在执行`git merge`时，我们还利用参数`-m`为这个新的提交记录指定了“名称”：c5。这里是合并之后的提交历史：
```shell
git log --oneline --graph --all
*   b735987 (HEAD -> master) c5
|\  
| * 0f029c3 (dev) c4
| * c594fa9 c3
|/  
* 9229adb c2
* cfbff28 c1
```

以及相应的图示：

![](/assets/images/lab/git/merge-stories-3.png)

实际上，当使用`--no-ff`参数进行合并时，我们的合并方式就变成了Three-Way Merge了。下面我们就来看一看，什么是Three-Way Merge。

## Three-Way Merge

前一节我们提到了，如果在合并时使用`--no-ff`参数，Git就会采用Three-Way Merge（三方合并）对两个分支进行合并。那么什么是Three-Way Merge呢？这所谓的“三方”到底是哪三方呢？

我们先通过一个例子来说明：假设William和Nicole各自都在对同一个文件进行修改。这个时候，我们要对两个人的修改进行合并了。如果只对William和Nicole各自的文件进行对比，也就是所谓的“diff”（或者也有人称之为“Two-Way Merge”），那么工具在帮我们做合并时，只知道两个文件在同一行上有差异，却没办法知道在合并后的版本里，到底该保留谁的版本，所以只能交给用户自己手工来决定。这就是普通的Merge工具所能做的。

![](/assets/images/lab/git/merge-stories-4.png)

和普通的Merge工具相比，Git最大的不同在于它记录了文件的提交历史，因此可以向前回溯文件修改前的“原件”。它在合并时不仅会看两人各自的文件内容，还会看之前的原件。通过和原来版本的对比，就可以清楚地知道，应该保留Nicole的版本，因为William的版本和原件相比并没有变化。这一过程可以由工具自动完成，而不用像Two-Way Merge那样，需要交给用户手工来决定，这就是Three-Way Merge。

![](/assets/images/lab/git/merge-stories-5.png)

前面我们提到了，在用`git merge`进行合并时，如果使用参数`--no-ff`，就会强制采用Three-Way Merge。实际上，还有一种常见的情况也会自动触发Three-Way Merge。下面我们就通过对本地Git库的操作，结合提交历史具体来看一下。接着前一节的例子，我们在master分支上再次执行git reset回退到c2：
```shell
$ git reset --hard 9229adb
HEAD is now at 9229adb c2
```

然后对README进行修改：
```shell
$ vi README 
$ cat README 
      c5
      /
c1 - c2
```

并生成新的提交：
```shell
$ git commit -am c5
[master 098be39] c5
 1 file changed, 2 insertions(+)
```

再用`git log`查看提交历史就可以看到，master分支在与dev分支分叉之后有了新的提交：
```shell
$ git log --oneline --graph --all
* 098be39 (HEAD -> master) c5
| * 0f029c3 (dev) c4
| * c594fa9 c3
|/  
* 9229adb c2
* cfbff28 c1
```

如图所示，这个时候我们就没有办法简单地通过移动指针来进行合并了。否则，master分支上分叉以后的提交就会在合并之后丢失：

![](/assets/images/lab/git/merge-stories-6.png)

这个时候，Git会自动采用Three-Way Merge方式进行合并。首先，它会在两个分支上分别找到head指针（又被称为branch tip）所对应的提交：c4和c5。然后，找到距离它们俩最近的“共同祖先”：c2（也就是前面所说的“原件”，又被称为common ancestor），然后进行Three-Way Merge。
```shell
$ git merge -m c6 dev
Auto-merging README
Merge made by the 'recursive' strategy.
 README | 1 +
 1 file changed, 1 insertion(+)
```

根据合并的结果，Git会生成一个新的快照，并创建一个新的提交指向这个快照。这个提交被称为“合并提交”（merge commit）：
```shell
$ git log --oneline --graph --all
*   9c3ee95 (HEAD -> master) c6
|\  
| * 0f029c3 (dev) c4
| * c594fa9 c3
* | 098be39 c5
|/  
* 9229adb c2
* cfbff28 c1
```

这种提交特别的地方在于，它有两个parent。如图所示，合并提交c6的parent分别是c4和c5：

![](/assets/images/lab/git/merge-stories-7.png)

## Squash Merge

接下来，我们再来看另一种Merge方式——Squash Merge。所谓Squash Merge，是指Git在做两个分支间的合并时，会把被合并分支（通常被称为topic分支）上的所有变更“压缩（squash）”成一个提交，追加到当前分支的后面，作为“合并提交”（merge commit）。从参与合并的文件变更上来说，Squash Merge和普通Merge并没有任何区别，效果完全一样。唯一的区别体现在提交历史上：正如我们前面提到的，对于普通的Merge而言，在当前分支上的合并提交通常会有两个parent；而Squash Merge却只有一个。

下面我们通过一个例子来加深理解。还是回到我们前面做实验用的本地Git库，在master分支上执行`git reset`回退到c2：
```shell
$ git reset --hard 098be39
HEAD is now at 098be39 c5
```

这个时候，master分支和dev分支的提交历史是这样的：
```shell
$ git log --oneline --graph --all
* 098be39 (HEAD -> master) c5
| * 0f029c3 (dev) c4
| * c594fa9 c3
|/  
* 9229adb c2
* cfbff28 c1
```

![](/assets/images/lab/git/merge-stories-8.png)

现在，让我们执行一次Squash Merge，把dev分支的内容合并到master分支：
```shell
$ git merge --squash dev
Auto-merging README
Squash commit -- not updating HEAD
Automatic merge went well; stopped before committing as requested
```

然后，生成新的提交：
```shell
$ git commit -m c6
[master 0b7fd35] c6
 1 file changed, 1 insertion(+)
```

再用`git log`观察提交历史：
```shell
$ git log --oneline --graph --all
* 0b7fd35 (HEAD -> master) c6
* 098be39 c5
| * 0f029c3 (dev) c4
| * c594fa9 c3
|/  
* 9229adb c2
* cfbff28 c1
```

从图上可以看到，作为合并提交的c6，的确只有一个parent，即：c5。而且，如果这个时候我们把dev分支删掉，整个提交历史就会变得非常干净。

![](/assets/images/lab/git/merge-stories-9.png)

Squash Merge不会像普通Merge那样在合并后的提交历史里保留被合并分支的分叉，被合并分支上的提交记录也不会出现在合并后的提交历史里，所有被合并分支上的变更都被“压缩”成了一个合并提交。

如果在被合并分支上，完整的提交历史里包含了很多中间提交（intermediate commit），比如：改正一个小小的拼写错误可能也会成为一个独立的提交，而我们并不希望在合并时把这些细节都反应在当前分支的提交历史里。这时，我们就可以选择Squash Merge。

另外，后面我们还会看到，如果在合并时想去除被合并分支上的那些中间提交，我们还可选择Rebase。
