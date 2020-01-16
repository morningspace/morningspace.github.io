---
title: Git工作流面面观——特性分支工作流
excerpt: 非常重要的一种Git工作流，充分发挥分支模型的优势
categories: [tech]
tags: [lab, dummies, dummies_git, git]
toc: true
toc_sticky: true
---

注：
本文采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/80x15.png" /></a>

![](/assets/images/lab/git/workflow.png)

## 名词解释

所谓特性分支工作流(Feature Branching)，是指当开发新特性或者进行bug修复的时候，开发人员从master分支出发，建立新的分支，把开发工作放到这些专门的分支上独立进行。

特性分支工作流作为集中式工作流的一种扩展，依然沿用了统一远程Git库，但是它充分发挥了Git分支模型的优势，让多人协作开发同一个项目变得更加方便。在这种工作模式下，每个特性的开发都有自己的分支，对项目的主干（即master分支）不会构成任何影响。在特性分支上的变更，只有经过确认以后才会被合并到master分支，这样就保证了master分支的稳定性。

从特性分支往master分支合并的时候，我们还可以通过创建Pull Request来发起讨论，允许其他人对涉及变更的代码进行复查，给出反馈意见。虽然Git本身并不直接提供Pull Request功能，但是现在很多Git服务解决方案，比如像GitHub，Bitbucket，GitLab等，都有相应的功能，使用起来非常方便。

此外，特性分支工作流是一种非常重要的工作流，后面我们在学习更高层次的工作流时会发现，它们都是以特性分支工作流为基础的。

## 如何工作

第一步，在服务器上建立一个远程Git库，供所有人使用：
```shell
$ ssh git@my-git-remote
git> create test-feature-branching
Initialized empty Git repository in /home/git/test-feature-branching.git/
git> exit
Connection to my-git-remote closed.
```

第二步，William把远程库克隆到本地：
```shell
$ git clone git@my-git-remote:~/test-feature-branching.git
Cloning into 'test-feature-branching'...
warning: You appear to have cloned an empty repository.
$ cd test-feature-branching/
```

生成了项目的初始提交：
```shell
$ touch README
$ git add README
$ git commit -m 'initial commit'
[master (root-commit) 2d6529c] initial commit
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 README
```

并把它推送到了远程：
```shell
$ git push
Counting objects: 3, done.
Writing objects: 100% (3/3), 207 bytes | 29.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0)
To my-git-remote:~/test-feature-branching.git
 * [new branch]      master -> master
```

第三步，Nicole也把远程库克隆到了本地：
```shell
$ git clone git@my-git-remote:~/test-feature-branching.git
Cloning into 'test-feature-branching'...
warning: You appear to have cloned an empty repository.
$ cd test-feature-branching/
```

建立了一个新的分支：
```shell
$ git checkout -b new-feature
Switched to a new branch 'new-feature'
```

并在新分支上开始了新特性的开发：
```shell
$ vi README
$ cat README 
Git workflows
* Feature branching
```

第四步，Nicole在本地建立了一系列提交以后：
```shell
$ git add README 
$ git commit -m 'working on new-feature'
[new-feature 30dc189] working on new-feature
 1 file changed, 2 insertions(+)
```

把特性分支推送到远程，作为备份，也作为和其他人协作共享的基础：
```shell
$ git push origin new-feature
Counting objects: 3, done.
Writing objects: 100% (3/3), 278 bytes | 46.00 KiB/s, done.
Total 3 (delta 0), reused 2 (delta 0)
To my-git-remote:~/test-feature-branching.git
 * [new branch]      new-feature -> new-feature
```

因为是在远程库新建了一个和本地同名的new-feature分支，所以它对master分支不会产生任何影响。

第五步，Nicole的开发工作完成了，在变更被合并到master分支以前，她请William对变更涉及的代码进行复查。由于我们使用的是纯命令行的Git实验环境，所以并不直接提供Pull Request的功能。不过，Git命令行也为我们提供了其他功能，可以模拟这一过程。比如，我们可以让Nicole生成一个包含当前变更的补丁(Patch)：
```shell
$ git format-patch master --stdout > new-feature.patch
```

然后发给William，等William拿到这个补丁以后，就可以检查里面的修改：
```shell
$ cat new-feature.patch 
From 30dc189d391f41f97ef12255b1061880ed75ca23 Mon Sep 17 00:00:00 2001
From: Nicole <nicole@example.com>
Date: Wed, 29 May 2019 03:47:38 +0000
Subject: [PATCH] working on new-feature

---
 README | 2 ++
 1 file changed, 2 insertions(+)

diff --git a/README b/README
index e69de29..e3e052a 100644
--- a/README
+++ b/README
@@ -0,0 +1,2 @@
+Git workflows
+* Feature branching
-- 
2.17.1
```

并把它集成到本地的当前工作目录下，进行验证：
```shell
$ git apply new-feature.patch
```

第六步，William和Nicole之间经过反复的讨论，最后终于达成一致可以合并到master分支了。于是，William再次把补丁集成到本地，并通过`--signoff`参数进行签收：
```shell
$ git am --signoff < new-feature.patch
Applying: working on new-feature
```

上述命令执行完以后会在William本地形成新的提交记录。因为使用了参数`--signoff`，所以在提交记录里会有“Signed-off-by”这一行：
```shell
$ git log          
commit b3138e88926c2b6176d7a45fb67ce7b959ae69f9 (HEAD -> master)
Author: Nicole <nicole@example.com>
Date:   Wed May 29 03:47:38 2019 +0000

    working on new-feature
    
    Signed-off-by: William <william@example.com>

commit 2d6529cb9c190db73b3dd8473477446edd2a7790 (origin/master)
Author: William <william@example.com>
Date:   Wed May 29 03:43:04 2019 +0000

    initial commit
```

最后，我们把提交记录推送到远程的master分支：
```shell
$ git push
Counting objects: 3, done.
Writing objects: 100% (3/3), 314 bytes | 26.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0)
To my-git-remote:~/test-feature-branching.git
   2d6529c..b3138e8  master -> master
```
