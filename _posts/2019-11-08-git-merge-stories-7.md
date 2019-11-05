---
title: Git合并那些事——交互式Rebase
excerpt: 介绍更多关于Rebase的玩法
categories: [tech]
tags: [lab, dummies, dummies_git, git]
toc: true
toc_sticky: true
---

> 多人开发中的合并冲突是我们使用Git时常常会遇到的情况，小小合并门道大，讲述合并的那些事儿，晴耕 · 白话之“Git合并那些事”系列​持续连载中……

注：
本文采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/80x15.png" /></a>

![](/assets/images/lab/git/rebase-2.png){: .align-center}

本文的主要目的是搜集和介绍更多有关Rebase的有趣用法。关于Rebase的基本使用，同学们可以在[“Git合并那些事儿——神奇的Rebase”](/tech/git-merge-stories-6)一文里找到。

## 问题由来

前段时间在往某个开源项目提交代码时，遇到了一个问题。平时我在公网和公司内部分别用的是两个不同的Git账号，用户名和邮箱都不一样，它们分别对应于公网的GitHub和公司内部的GitHub Enterprise。但是，有一天因为一时疏忽，我在往公网GitHub上提交代码时误用了公司内部的账号。而且，提交记录已经被推送到了远程。

对于已经提交到本地Git库甚至被推送到远程的提交记录，如何修改它们的信息呢？网上搜了一下，发现还真有办法解决。方法就是利用`git rebase -i`，结合`git commit --amend`。

## 交互式Rebase

通常我们执行`git rebase`命令时，整个Rebase的过程都是自动进行的。但是，当我们指定了`-i`参数以后，Rebase就进入了交互模式。于是，我们就可以“为所欲为”了;-) 

在[“Git合并那些事儿——神奇的Rebase”](/tech/git-merge-stories-6)一文里，我们已经知道Rebase可以影响提交历史。实际上，我们在交互模式下可以做更多事情。纯手工，想怎么玩儿都行！比如：合并多个提交记录，丢弃某些提交记录，修改提交记录的信息等。

## 一个例子

下面通过一个例子来演示这个过程，假设我们的提交历史是这样的：
```shell
$ git log --format='%h %s %an %ae %d'
2665bc4 c3 Nicole nicole@example.com  (HEAD -> master)
604a471 c2 Nicole nicole@example.com  (origin/master)
f0f4430 c1 Nicole nicole@example.com 
f14301e c0 William william@example.com 
```

如果我们希望修改从第一条提交记录c0开始的所有提交记录，可以在执行Rebase时指定`--root`参数：
```shell
$ git rebase -i --root
```

如果希望修改从某条提交记录开始往后的剩余提交记录，那么就在执行Rebase时指定这条提交记录的SHA-1值。比如，假设我们想修改c2和c3的作者名称和邮箱地址，也就是从c1往后的剩余提交记录，那就指定c1的SHA-1值，即：f0f4430：
```shell
$ git rebase -i f0f4430
```

这个时候，Git会自动打开默认的文本编辑器，比如在我本地使用的是`vim`。在文本编辑器的开始几行会列出所有满足条件的提交记录，每个提交记录对应一行。除了提交记录的SHA-1值和备注信息外，还有一列代表要对这条提交记录实施的操作。
```shell
pick 604a471 c2
pick 2665bc4 c3
```

下面的表格列出了所有可供选择的操作：

| 操作 	| 说明
| ---- 	|:----
| pick 	| 采用该提交（默认行为）
| reword| 采用该提交，但要求修改提交记录的备注
| edit	| 采用该提交，但要求修改提交记录的信息，如：作者名称，邮箱地址等
| squash| 采用该提交，但它会被并入前一条提交
| fixup	| 类似“squash”，但是会丢弃这条提交记录的日志信息
| exec	| 执行指定的shell脚本或命令
| drop	| 丢弃该提交

针对我们要解决的问题，edit是最合适的选择。所以，我们把pick改成edit，然后保存并退出：
```shell
edit 604a471 c2
edit 2665bc4 c3
```

于是，整个Rebase的过程就开始了。这个时候，Git会自动停在第一个满足条件的提交记录处：
```shell
Stopped at 604a471...  c2
You can amend the commit now, with

  git commit --amend 

Once you are satisfied with your changes, run

  git rebase --continue
```

当Git提示要修改提交记录时，我们就执行下面的`git commit --amend`命令，对当前提交记录进行修改：
```shell
$ git commit --amend --author="Murphy <murphy@example.com>"
[detached HEAD f10bd4b] c3
 Author: Murphy <murphy@example.com>
 Date: Thu May 16 00:45:41 2019 +0000
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 VERSION
```

修改完成后执行`git rebase --continue`，告诉Git跳到下一个待处理的提交记录处：
```shell
$ git rebase --continue
Stopped at 2665bc4...  c3
You can amend the commit now, with

  git commit --amend 

Once you are satisfied with your changes, run

  git rebase --continue
```

反复执行上面的操作，直到所有提交记录都处理完了为止：
```shell
$ git rebase --continue
Successfully rebased and updated refs/heads/master.
```

这个时候，整个Rebase的过程就结束了。现在，我们再来看一下提交历史：
```shell
$ git log --format='%h %s %an %ae %d'
5498264 c3 Murphy murphy@example.com  (HEAD -> master)
551e6a4 c2 Murphy murphy@example.com 
f0f4430 c1 Nicole nicole@example.com 
f14301e c0 William william@example.com
```

就会发现，c2和c3对应的作者信息果然更新了。而且，仔细观察我们会发现，c2和c3的SHA-1值都和之前不一样了。这也是Rebase的特点，它实际上会把旧的提交记录删掉，重新建立新的提交记录。

对于提交记录还没有被推送到远程的本地库而言，到这里为止，问题就已经得到解决了。如果已经推送到了远程，那么我们还需要多执行一步`git pull`，加上`--force`参数，把本地修改强制推送到远程：
```shell
$ git push -f origin master
Counting objects: 5, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (4/4), done.
Writing objects: 100% (5/5), 582 bytes | 38.00 KiB/s, done.
Total 5 (delta 0), reused 0 (delta 0)
To my-git-remote:~/test-rebase.git
 + 604a471...5498264 master -> master (forced update)
```

最后，如果Rebase过程中发现做错了，想撤销或者终止Rebase，也很简单。任何时候，只要在执行`git rebase`命令时使用`--abort`参数就可以了：
```shell
$ git rebase --abort
```