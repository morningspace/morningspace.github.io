---
title: Git合并那些事——神奇的Rebase
excerpt: 介绍Rebase的基本概念，各种玩法，以及什么时候用到它
categories: [tech]
tags: [lab, dummies, dummies_git, git]
toc: true
toc_sticky: true
---

注：
本文的部分写作灵感来自于[“Pro Git book”](https://git-scm.com/book/en/v2)。感谢原作者的精彩分享。
本文采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/80x15.png" /></a>

![](/assets/images/lab/git/rebase.png){: .align-center}

## 什么是Rebase？

在Git里，要想把一个分支上的改动合并到另一个分支上，通常有两种做法：一种是Merge，另一种是Rebase。

其中，Merge是分支合并时比较直观的一种方式。它的基本思路是：在参与合并的两个分支上找到各自的最新提交，以及这两个分支的公共祖先，对这三个提交进行“三方合并”（Three-Way-Merge）。然后，根据合并后的结果生成新的提交，而两个分支上的提交则会成为新建提交的parent。有关Merge的更多内容，可以从[Git合并那些事儿——认识几种Merge方法](/tech/git-merge-stories-1/)一文中找到。

而Rebase，乍一看则不是那么容易理解。简单地说，Rebase就是把一个分支上的所有提交，在另一个分支上按照同样的顺序重新“回放”（replay）一遍。下面我们用一个具体的例子来体会一下。

首先，我们新建一个本地Git库，叫做test-rebase：
```shell
$ git init test-rebase
Initialized empty Git repository in /root/test-rebase/.git/
$ cd test-rebase/
```

在master分支上新建README文件：
```shell
$ vi README
$ cat README
The magic rebase
* When to use?
```

并建立提交记录c0：
```shell
$ git add .
$ git commit -m c0
[master (root-commit) f14301e] c0
 1 file changed, 2 insertions(+)
 create mode 100644 README
```

然后创建分支dev：
```shell
$ git branch dev
```

并继续在master分支上修改README文件：
```shell
$ vi README
$ cat README
The magic rebase
* What is it?
* When to use?
```

建立提交记录c1：
```shell
$ git commit -am c1
[master 8b8ca49] c1
 1 file changed, 1 insertion(+)
```

然后再切换到dev分支：
```shell
$ git checkout dev
Switched to branch 'dev'
```

在dev分支上接着修改README文件：
```shell
$ vi README
$ cat README
The magic rebase
* When to use?
* More use
```

并建立提交记录c2：
```shell
$ git commit -am c2
[dev 6528967] c2
 1 file changed, 1 insertion(+)
```

这个时候的提交记录是这样的：
```shell
$ git log --oneline --all --graph
* 6528967 (HEAD -> dev) c2
| * 8b8ca49 (master) c1
|/  
* f14301e c0
```

![](/assets/images/lab/git/merge-stories-19.png)

现在，我们在dev分支上执行rebase：
```shell
$ git rebase master
First, rewinding head to replay your work on top of it...
Applying: c2
Using index info to reconstruct a base tree...
M	README
Falling back to patching base and 3-way merge...
Auto-merging README
```

Git在执行rebase的过程中，首先会找到dev分支和master分支各自的最新提交c1和c2；然后从c1和c2开始向前回溯，找到它们在提交历史上的“共同祖先”c0；把dev分支上从c0往后的每个提交记录和c0进行对比，并把对比结果存到临时文件里；然后重置dev分支的提交历史，使它和master分支的提交历史保持一致，即：原本只出现在master分支上的提交记录c1，现在也出现在了dev分支上；在这个基础上，然后再逐一追加只在dev分支上出现的提交记录，比如：c2，整个过程才算结束。我们来看一下提交历史：
```shell
$ git log --oneline --all --graph
* 32abc1c (HEAD -> dev) c2
* 8b8ca49 (master) c1
* f14301e c0
```

如图所示，原本的“分叉”被“拉直”了。由于c1现在也出现在dev分支上，并且成了c2的parent，这就好像我们为c2重新设立了“起点”：c2原来是从c0出发后到达的，现在变成从c1出发后到达了。Rebase的本意就是重新设立起点的意思。

![](/assets/images/lab/git/merge-stories-20.png)

这里还有一点要注意，如果我们仔细观察提交记录c2，会发现它的唯一键（SHA-1值）在Rebase前后发生了改变，从`6528967`变到了`32abc1c`。虽然都叫c2，对应的快照也一摸一样，但是它们的确是两个不同的提交记录。

现在，让我们回到master分支：
```shell
$ git checkout master
Switched to branch 'master'
```

把dev分支合并到master：
```shell
$ git merge dev
Updating 8b8ca49..32abc1c
Fast-forward
 README | 1 +
 1 file changed, 1 insertion(+)
```

因为不存在分叉，所以Git可以采用Fast-Forward方式进行合并，效率非常高。关于Fast-Forward Merge，同学们可以从[Git合并那些事儿——认识几种Merge方法](/tech/git-merge-stories-1/)一文中找到更多解释。

最终的提交历史是这个样子的：
```shell
$ git log --oneline --all --graph
* 32abc1c (HEAD -> master, dev) c2
* 8b8ca49 c1
* f14301e c0
```

![](/assets/images/lab/git/merge-stories-21.png)

利用Rebase得到的提交记录c2所指向的快照，和利用Merge得到的快照，在内容上是完全一样的。所以，从最终结果来看两者并没有什么差别。但是，Rebase让提交历史变得更加干净了。它把原来分叉了的提交历史进行了“线性化”处理（称为线性历史，linear history）。就好像所有工作都是线性挨个儿进行的，从来都没有分过叉。

## 更多玩法

关于Rebase的使用，还有更多玩法，这里我们就来举一个例子。接着上面的实验环境，让我们回退到分叉的状态，也就是Rebase之前的状态：
```shell
$ git log --oneline --all --graph
* 55642be (HEAD -> master) c1
| * 0bd4f42 (dev) c2
|/  
* f14301e c0
```

并切换到dev分支：
```shell
$ git checkout dev
Switched to branch 'dev'
```

这个时候，我们决定从dev分支新建另一个分支出来，叫：bug-fix，专门用于修复bug：
```shell
$ git branch bug-fix
```

然后继续在dev分支上工作，并创建了新的提交记录c3：
```shell
$ touch .gitignore
$ git add .gitignore
$ git commit -m c3
[dev ae216dc] c3
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 .gitignore
```

而且，我们同时还在bug-fix分支上工作，并创建了新的提交记录c4：
```shell
$ git checkout bug-fix
Switched to branch 'bug-fix'
$ touch VERSION
$ git add VERSION 
$ git commit -m c4
[bug-fix 0cfd41c] c4
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 VERSION
```

目前的提交历史是这样的：
```shell
$ git log --oneline --all --graph
* 0cfd41c (HEAD -> bug-fix) c4
| * ae216dc (dev) c3
|/  
* 0bd4f42 c2
| * 55642be (master) c1
|/  
* f14301e c0
```

![](/assets/images/lab/git/merge-stories-22.png)

现在，假设我们想把bug-fix分支上的修改合并到master分支，但是又不希望把dev分支上的修改带到master分支上。也就是说，我们只想把bug-fix分支上的c4合并到master上；而至于c2，虽然它也在bug-fix分支上，但它同时还属于dev分支，仍然在开发当中，所以不希望被合并到master上。为了达到这个效果，我们可以利用`git rebase`命令，结合`--onto`参数：
```shell
$ git rebase --onto master dev bug-fix
First, rewinding head to replay your work on top of it...
Applying: c4
```

这行命令的意思是，告诉Git：找到bug-fix分支和dev分支的共同祖先c2；在dev分支上找到从c2开始往后的所有提交记录，即：c4；然后在bug-fix分支上重新“回放”这些提交记录，就好像是直接在master分支上工作的那样。相当于把c4的“起点”从原来的c2重新设置成了c1。

然后，我们再切换回master分支，执行一次Fast-Forward Merge，就可以把bug-fix分支上的变更合并到master上了。
```shell
$ git checkout master
Switched to branch 'master'
$ git merge bug-fix
Updating 55642be..c284ee9
Fast-forward
 VERSION | 0
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 VERSION
```

最终的提交历史是这样的：
```shell
$ git log --oneline --all --graph
* c284ee9 (HEAD -> bug-fix, master) c4
* 55642be c1
| * ae216dc (dev) c3
| * 0bd4f42 c2
|/  
* f14301e c0
```

![](/assets/images/lab/git/merge-stories-23.png)

假设现在我们要把dev分支上的变更也合并到master上了，同样利用Rebase。前面我们执行`git rebase`命令时，需要先切换到dev分支，等执行完Rebase以后还要再回到master分支，然后进行最后的合并。实际上，可以不用这么麻烦，只要我们按照下面的语法执行`git rebase`命令就行：
```shell
git rebase <basebranch> <topicbranch>
```

其中，`basebranch`就是作为基准的那个分支，而`topicbranch`则是被合并的那个分支。在我们的例子里，`basebranch`就是master，而`topicbranch`则是dev。如果`topicbranch`没有提供，也就是我们前面执行`git rebase`时所使用的那种格式，那么Git会把当前所在分支作为`topicbranch`。这也就是为什么我们之前要先切换到被合并分支的原因。现在，我们在master分支上执行`git rebase`，并指定dev分支为`topicbranch`：
```shell
$ git rebase master dev
First, rewinding head to replay your work on top of it...
Applying: c2
Using index info to reconstruct a base tree...
M	README
Falling back to patching base and 3-way merge...
Auto-merging README
Applying: c3
```

这个时候的提交历史是这样的：
```shell
$ git log --oneline --all --graph
* 2fbf3be (HEAD -> dev) c3
* 73bdd74 c2
* c284ee9 (master, bug-fix) c4
* 55642be c1
* f14301e c0
```

最后，我们再回到master分支，对dev分支进行合并：
```shell
$ git checkout master
Switched to branch 'master'
$ git merge dev
Updating c284ee9..2fbf3be
Fast-forward
 .gitignore | 0
 README     | 1 +
 2 files changed, 1 insertion(+)
 create mode 100644 .gitignore
```

最后的提交历史是这样的：
```shell
* 2fbf3be (HEAD -> master, dev) c3
* 73bdd74 c2
* c284ee9 (bug-fix) c4
* 55642be c1
* f14301e c0
```

这实际上相当于把dev分支上的提交记录c2和c3在master分支上做了“回放”。

![](/assets/images/lab/git/merge-stories-24.png)

最后，等到所有开发都完成以后，我们把多余的分支dev和bug-fix删除掉：
```shell
$ git branch -d dev
Deleted branch dev (was 2fbf3be).
$ git branch -d bug-fix
Deleted branch bug-fix (was c284ee9).
```

再看整个提交历史，就变成一条非常干净的直线了：
```shell
$ git log --oneline --graph
* 2fbf3be (HEAD -> master) c3
* 73bdd74 c2
* c284ee9 c4
* 55642be c1
* f14301e c0
```

## 何时使用

关于Git的提交历史，有两种观点。

* 一种观点认为，它如实的记录了人们的实际操作，所以不应该被“篡改”；
* 另一种观点认为，多人协作开发的时候，提交历史的作用是把项目如何演化的过程展示给其他人看，所以没必要把全部细节都暴露出来。比如，像一些很细微的修改记录，类似整个提交历史中的某些中间状态，这些提交记录是可以去掉的。

后一种观点就是人们通常使用Rebase修改提交历史的原因或依据。

关于什么时候可以使用Rebase的问题，一般的建议是：

如果你还没有把提交推送到远程Git库，Rebase是没问题的。因为，所有改动，包括对提交历史的改动都只发生在本地，不会对别人造成影响。而且，那样做还有助于在把提交推送到远程之前，让提交历史变得更加清晰。这个在开源项目的开发里是很常见的。比如：假设你以Contributor的身份在为某个开源项目贡献代码。一般我们会从原项目那里fork一份出来形成自己的Git库，然后在自己的Git库上新开一个分支，在分支上进行开发。要提交修改的时候，先把修改Rebase到远程的origin/master上（自己的Git库），然后再以Pull Request的方式提交到原项目，让这个项目的Maintainer来Review。如果我们使用了Rebase，那么Maintainer在合并修改的时候就会很轻松，他不需要做任何复杂的集成，只要简单地做一个Fast-Forward Merge就可以了。

但是，如果你的改动已经被推送到了远程Git库，并且其他人已经开始在这个基础上继续他们的工作时，那就最好不要轻易做Rebase了。后面，我还会专门用一篇文章来说明，在那种情况下做Rebase会有什么后果。以及，当问题发生时我们应该怎么处理。
