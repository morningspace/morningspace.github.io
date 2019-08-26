---
title: Git合并那些事——Merge策略（上）
excerpt: 介绍各种Merge策略，包括Recursive，Resolve，Ours，以及Criss-Cross现象
categories: [tech]
tags: [lab, dummies, dummies_git, git]
toc: true
toc_sticky: true
---

> 多人开发中的合并冲突是我们使用Git时常常会遇到的情况，小小合并门道大，讲述合并的那些事儿，晴耕 · 白话之“Git合并那些事”系列​持续连载中……

注：
本文采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/80x15.png" /></a>

![](/assets/images/lab/git/merge-strategies-1.png)

## 关于Merge策略

所谓Merge策略，是指Git在执行`git merge`命令时所能选择的合并策略。默认情况下，Git在合并分支时会自动选择最合适的Merge策略，我们也可以通过参数`-s`显式指定策略。不同的策略会对合并的方式与结果产生不同的影响，比如：对参与合并的分支，其“共同祖先（common ancestor）”的选择，以及在合并过程中发生冲突时的处理方式等。

对“共同祖先”这个概念还不了解的同学，不妨看一下[《Git合并那些事儿——认识几种Merge方法》](/tech/git-merge-stories-1)一文中有关Three-Way Merge的讨论。下面我们就来介绍一下常见的几种Merge策略。

## Recursive策略

Recursive策略是Git在对两个分支进行合并时所采用的默认策略，它只适用于两个分支之间的合并。因此，对于超过两个分支的合并，需要反复地进行两两合并，才能最终完成所有分支的合并（这也是Recursive名字的由来）。本质上，Recursive就是一种Three-Way Merge。它的特点在于，如果Git在寻找共同祖先时，在参与合并的两个分支上找到了不只一个满足条件的共同祖先，它会先对共同祖先进行合并，建立临时快照。然后，把临时产生的“虚拟祖先”作为合并依据，再对分支进行合并。

如下图所示，在对两个分支上的提交A和B进行合并时，我们发现了它们有两个共同祖先，分别是：ancestor0和ancestor1。这个时候，Recursive策略会对ancestor0和ancestor1进行合并，临时创建一个虚拟祖先：ancestor2。用ancestor2与A，B一起进行Three-Way Merge。

![](/assets/images/lab/git/merge-stories-10.png)

### Criss-Cross现象

出现多个满足条件的共同祖先的现象被称为Criss-Cross现象。虽然它并不一定经常发生，但是对于那些存在时间很长，拓扑结构很复杂的分支，还是很有可能会出现的。下面我们就用一个例子来模拟一下Criss-Cross现象，并观察Recursive策略是怎么处理这种情况的。

> 提示：接下来的过程可能会有点“烧脑”，所以建议最好照着下面的步骤实际操作一遍。不过如果你的脑子足够好使的话，也可以只在大脑里做做思维模拟:-)

首先，我们新建一个本地Git库，叫做test-recursive-merge：
```shell
$ git init test-recursive-merge
Initialized empty Git repository in /root/test-recursive-merge/.git/
$ cd test-recursive-merge
```

在master分支上新建README文件：
```shell
$ vi README
$ cat README
Merge strategies include:
* Recursive
```

并建立提交记录c0：
```shell
$ git add .
$ git commit -m c0
[master (root-commit) 8937d6a] c0
 1 file changed, 2 insertions(+)
 create mode 100644 README
```

然后，我们切换到分支feature1开始特性开发：
```shell
$ git checkout -b feature1
Switched to a new branch 'feature1'
$ vi README
$ cat README
Merge strategies include:
* Recursive
* ...
```

并在上面建立提交记录c1：
```shell
$ git commit -am c1
[feature1 39b06d9] c1
 1 file changed, 1 insertion(+)
```

再回到master分支，继续在master分支上进行开发。这期间，假设我们引入了一个拼写错误，把include写成了includes：
```shell
$ git checkout master
Switched to branch 'master'
$ vi README
$ cat README
Merge strategies includes:
* Recursive
```

并建立了提交记录c2：
```shell
$ git commit -am c2
[master ff8acc9] c2
 1 file changed, 1 insertion(+), 1 deletion(-)
```

紧接着，我们又新开了一个分支feature2：
```shell
$ git checkout -b feature2
Switched to a new branch 'feature2'
```

在该分支上把feature1的修改合并了过来，并建立了提交记录c3：
```shell
$ git merge -m c3 feature1
Auto-merging README
Merge made by the 'recursive' strategy.
 README | 1 +
 1 file changed, 1 insertion(+)
$ cat README
Merge strategies includes:
1) Recursive
2) ...
```

这个时候，我们在feature2上的README既包含了在feature1上引入的正常修改，同时也包含了在master上引入的拼写错误。现在，假设我们发现了这个拼写错误。于是切换回master，改正了错误，并建立了提交记录c4：
```shell
$ git checkout master
Switched to branch 'master'
$ vi README
$ cat README
Merge strategies include:
* Recursive
$ git commit -am c4
[master c096f34] c4
 1 file changed, 1 insertion(+), 1 deletion(-)
```

如果我们此时把feature1上做的修改合并到master上：
```shell
$ git merge -m c5 feature1
Merge made by the 'recursive' strategy.
 README | 1 +
 1 file changed, 1 insertion(+)
$ cat README
Merge strategies include:
* Recursive
* ...
```

然后用`git log`查看提交历史，就会发现，这个时候传说中的“Criss-Cross现象”出现了：
```shell
$ git log --graph --oneline --all
*   1725ff2 (HEAD -> master) c5
|\  
* | c096f34 c4
| | *   26dfc71 (feature2) c3
| | |\  
| |/ /  
|/| /   
| |/    
| * 39b06d9 (feature1) c1
* | ff8acc9 c2
|/  
* 8937d6a c0
```

如图所示，c3和c5的共同祖先有两个，它们分别是c1和c2。

![](/assets/images/lab/git/merge-stories-11.png)

这个时候，如果我们执行`git merge-base`命令，并传入master和feature2，即：寻找分支master和feature2的最佳共同祖先，则会得到两个结果，分别对应c1（39b06d9）和c2（ff8acc9）：
```shell
$ git merge-base --all master feature2
39b06d96f6d584d2a7d5b2caff21dc75d5d5075f
ff8acc923e2a4d7d09a8d42236a190d47ff38286
```

如果我们查看master和feature1，或者feature1和feature2的最佳共同祖先，则只会得到一个结果，对应c1（39b06d9）：
```shell
$ git merge-base --all master feature1
39b06d96f6d584d2a7d5b2caff21dc75d5d5075f
$ git merge-base --all feature1 feature2
39b06d96f6d584d2a7d5b2caff21dc75d5d5075f
```

当遇到这种情况时，如果我们想把feature2上的修改合并到master，Git会自动先合并c1和c2，为它们创建虚拟祖先，然后再对两个分支进行合并：
```shell
$ git merge -m c6 feature2
Already up to date!
Merge made by the 'recursive' strategy.
```

执行git log查看提交历史：
```shell
$ git log --graph --oneline --all
*   0935c24 (HEAD -> master) c6
|\  
| *   26dfc71 (feature2) c3
| |\  
* | \   1725ff2 c5
|\ \ \  
| | |/  
| |/|   
| * | 39b06d9 (feature1) c1
* | | c096f34 c4
| |/  
|/|   
* | ff8acc9 c2
|/  
* 8937d6a c0
```

如图所示，会看得更清楚些，c3和c5最终合并后生成了c6：

![](/assets/images/lab/git/merge-stories-12.png)

### Ours和Theirs参数

在处理合并时，和其他某些Merge策略一样，Recursive策略通常会尽量自动完成合并。如果在合并过程中发现冲突，Git会在被合并的文件里插入冲突标记（merge conflict markers），并标记当前文件存在冲突，然后交由人工来处理。

不过，我们也可以通过指定参数告诉Git，当发生冲突时自动选择或丢弃其中一个分支上的修改。比如，假设我们要把分支B合并到分支A。如果指定参数`-Xours`，则表明丢弃分支B上的修改，保留当前分支A上的内容；指定参数`-Xtheirs`则刚好相反。

这里要注意的是，这两个参数只在发生冲突时起作用。而正常情况下，即没有发生冲突时，Git还是会帮我们自动完成合并的。关于这一点，在后面谈到Ours策略时可以做一个对比，两者在概念上比较容易混淆。

下面我们来做个实验加深理解。接着前面的例子，我们切换到feature1分支：
```shell
$ git checkout feature1
Switched to branch 'feature1'
```

对README文件做些修改：
```shell
$ vi README
$ git diff README 
diff --git a/README b/README
index f725440..1115275 100644
--- a/README
+++ b/README
@@ -1,3 +1,3 @@
-Merge strategies include:
+Git merge strategies include:
 * Recursive
-* ...
+* and so on...
```

并建立提交记录c7：
```shell
$ git commit -am c7
[feature1 9232cf1] c7
 1 file changed, 2 insertions(+), 2 deletions(-)
```

然后再切换回master分支：
```shell
$ git checkout master
Switched to branch 'master'
```

继续对README做修改：
```shell
$ vi README 
$ git diff README 
diff --git a/README b/README
index f725440..9855509 100644
--- a/README
+++ b/README
@@ -1,3 +1,3 @@
 Merge strategies include:
 * Recursive
-* ...
+* etc.
```

并建立提交记录c8：
```shell
$ git commit -am c8
[master 306f03a] c8
 1 file changed, 1 insertion(+), 1 deletion(-)
```

最后，我们把feature1上的修改合并到master：
```shell
$ git merge feature1
Auto-merging README
CONFLICT (content): Merge conflict in README
Automatic merge failed; fix conflicts and then commit the result.
$ cat README 
Git merge strategies include:
* Recursive
<<<<<<< HEAD
* etc.
=======
* and so on...
>>>>>>> feature1
```

可以看到，对于第一行，我们在feature1上的修改已经成功的被自动合并到master；而对于最后一行，由于我们在feature1和master上都做了修改。所以，Git无法进行自动合并，于是插入合并冲突标记，交由人工来处理。

我们可以看一下，假如指定`-Xours`参数会有什么效果。再次执行git merge之前，先使用`--abort`参数撤销当前合并：
```shell
$ git merge --abort
$ cat README 
Merge strategies include:
* Recursive
* etc.
```

然后执行`git merge`并指定`-Xours`参数：
```shell
$ git merge -Xours -m c9 feature1
Auto-merging README
Merge made by the 'recursive' strategy.
 README | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
```

利用git diff对比合并后的提交记录c9和之前的c8可以看到：对于第一行，因为不存在冲突，Git还是会帮我们自动完成合并；对于最后一行，由于存在合并冲突，而且我们选择的是保留当前分支（即：master）的版本，所以合并后的内容和上一个版本是一摸一样的：
```shell
$ git diff HEAD^ HEAD
diff --git a/README b/README
index 9855509..c090928 100644
--- a/README
+++ b/README
@@ -1,3 +1,3 @@
-Merge strategies include:
+Git merge strategies include:
 * Recursive
 * etc.
```

假如指定`-Xtheirs`参数呢？让我们先回退到上次合并前的提交记录c8：
```shell
$ git reset --hard HEAD^
HEAD is now at 306f03a c8
```

然后再执行合并，并指定`-Xtheirs`参数：
```shell
$ git merge -Xtheirs -m c9 feature1
Auto-merging README
Merge made by the 'recursive' strategy.
 README | 4 ++--
 1 file changed, 2 insertions(+), 2 deletions(-)
```

这时的结果则刚好相反，合并产生冲突的部分保留了feature1分支的版本：
```shell
$ git diff HEAD^ HEAD 
diff --git a/README b/README
index 9855509..1115275 100644
--- a/README
+++ b/README
@@ -1,3 +1,3 @@
-Merge strategies include:
+Git merge strategies include:
 * Recursive
-* etc.
+* and so on...
```

## Resolve策略

和Recursive策略类似，Resolve策略是另一种解决两个分支间合并问题的策略，同样也是采用的Three-Way Merge算法。关于它的介绍，网络上资料不多。只知道它是在Recursive策略出现之前，Git合并时所采用的默认策略。和Recursive策略不同的是，它在处理Criss-Cross情况时，会在多个满足条件的共同祖先里选取其中一个作为合并的基础（称为Merge Base）。在某些情况下，如果使用Recursive策略作为默认策略进行合并时遇到了问题，也可以尝试选择Resolve策略。

## Ours策略

前面提到了，如果在使用Recursive策略时指定`-Xours`参数，那么当发生冲突时，Git会选择丢弃来自被合并分支的修改，而保留被当前分支上的原有修改。这种情况只在有冲突时才会发生，如果没有冲突，Git还是会帮我们自动完成合并的。

与之不同的是，Ours策略无论有没有冲突发生，都会毫不犹豫的丢弃来自被合并分支的修改，完整保留当前分支上的修改。所以，对于Ours策略而言，实质上根本就没有做任何真正意义上的合并，或者说做的是假合并（fake merge）。不过，从提交历史上看，Git依然会创建一个新的合并提交（merge commit），并让它的parent分别指向参与合并的两个分支上的提交记录。

接着前面的例子，我们继续通过实验来加深理解。先回退到上次合并前的提交记录c8：
```shell
$ git reset --hard HEAD^
HEAD is now at 306f03a c8
```

在当前分支master上执行`git merge`，并指定Ours作为合并策略：
```shell
$ git merge -s ours -m c9 feature1
Merge made by the 'ours' strategy.
```

查看提交历史会发现，master分支上多了一个提交记录c9：
```shell
$ git log --oneline --graph --all
*   e9d35fd (HEAD -> master) c9
|\  
| * 9232cf1 (feature1) c7
* | 306f03a c8
* |   0935c24 c6
|\ \  
| |/  
|/|   
| *   26dfc71 (feature2) c3
| |\  
* | \   1725ff2 c5
|\ \ \  
| | |/  
| |/|   
| * | 39b06d9 c1
* | | c096f34 c4
| |/  
|/|   
* | ff8acc9 c2
|/  
* 8937d6a c0
```

但对比合并前后的提交记录：c8和c9，却没有发现任何差异，两者内容是完全一样的：
```shell
$ git diff HEAD^ HEAD
$
```

OK，今天就到这里了。在接下来的一篇文章里，我们将继续学习Git的另外几种合并策略，比如Octopus策略，还有Subtree策略。我们下次再见！
