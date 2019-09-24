---
title: Git合并那些事——撤销合并
excerpt: 讲述冲各种撤销合并的方法
categories: [tech]
tags: [lab, dummies, dummies_git, git]
toc: true
toc_sticky: true
---

> 多人开发中的合并冲突是我们使用Git时常常会遇到的情况，小小合并门道大，讲述合并的那些事儿，晴耕 · 白话之“Git合并那些事”系列​持续连载中……

注：
本文的部分写作灵感来自于[“Pro Git book”](https://git-scm.com/book/en/v2)。感谢原作者的精彩分享。
本文采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/80x15.png" /></a>

![](/assets/images/lab/git/merge-revert.jpg){: .align-center}

## 撤销合并

当冲突发生时，我们有多种选择，这其中当然也包括撤销合并。比如，我们可以利用`git merge --abort`，恢复或回退到执行合并以前的状态。或者，如果我们合并后的提交还停留在本地Git库，没有被推送到远程，我们还可以利用`git reset --hard HEAD`命令，恢复到当前分支的最近一次提交。Git在接到这个命令以后，会按照下面的步骤一步步进行撤销：
* 把当前分支的HEAD指针移动到合并前的提交记录上；
* 把暂存区恢复成HEAD所指向的版本；
* 把工作目录恢复成和暂存区保持一致；

## 使用git revert

另外，还有一种撤销合并的办法，就是利用`git revert`命令。和`git merge --abort`，以及`git reset`不同，它是在合并提交生成以后使用的。这种方法的好处是，它可以在恢复以前某个提交记录的同时，还能保留从那以后的所有提交历史。其实现思路是，在合并提交的后面再创建一个新的提交记录，在这个提交记录里撤销掉从某个前序提交记录开始往后的所有更改。

我们来看一个例子，接着[“Git合并那些事儿——当冲突发生时”](/tech/git-merge-stories-4/)我们所使用的实验环境，假设我们手工解决了合并冲突，并建立了一个新的合并提交M：
```shell
$ cat README
When conflict happens...
* Prepare
* Cancel the merge
* Understand the conflict
* ...
$ git commit -am M
[master 01a84b6] M
```

这个时候的提交历史是这样的：
```shell
$ git log --oneline --all --graph
*   01a84b6 (HEAD -> master) M
|\  
| * 225fb79 (dev) c3
| * f6cd947 c2
* | fc3fedf c1
|/  
* 12de8e7 c0
```

现在，我们执行`git revert`，把README的内容恢复到合并之前的ours版本，其实就是提交记录c1对应的版本：
```shell
$ git revert -m 1 HEAD
[master 931c6f9] Revert "M"
 2 files changed, 2 deletions(-)
 delete mode 100644 .gitignore
```

这里，`-m 1`表示当前分支应该保留哪个版本（1代表ours）。

这个时候的提交记录历史变成了这个样子：
```shell
$ git log --oneline --all --graph
* 931c6f9 (HEAD -> master) Revert "M"
*   01a84b6 M
|\  
| * 225fb79 (dev) c3
| * f6cd947 c2
* | fc3fedf c1
|/  
* 12de8e7 c0
```

![](/assets/images/lab/git/merge-stories-16.png)

观察README的内容我们会发现，就好像合并从来没有发生过一样。但是合并的过程的确被记录在提交历史里了。因为Git在提交历史里保留了合并提交，所以当我们尝试再次合并时，Git会决绝：
```shell
$ git merge dev
Already up-to-date.
```

### git revert的问题

有趣的是，如果这个时候我们又在dev分支上做了新的改动，当再次进行合并时，我们会发现Git只会合并提交记录Revert "M"之后引入的变更。

接下来，我们通过实验来进一步加深理解。首先我们切换到dev分支：

```shell
$ git checkout dev
Switched to branch 'dev'
```

在上面做些改动：
```shell
$ vi VERSION
$ cat VERSION
1.0
```

然后生成一个新的提交记录c4：
```shell
$ git add VERSION
$ git commit -m c4
[dev 060ab8a] c4
 1 file changed, 1 insertion(+)
 create mode 100644 VERSION
```

这个时候，再切换回master分支：
```shell
$ git checkout master
Switched to branch 'master'
```

合并dev分支上的修改：
```shell
$ git merge -m c5 dev
Merge made by the 'recursive' strategy.
 VERSION | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 VERSION
```

这个时候，我们会发现在dev分支上新加的VERSION文件的确被成功合并了。但是，README文件依然保持着原来的样子，也就是提交记录c1对应的版本：
```shell
$ ls
README	VERSION
$ cat README 
When conflict happens...
* Cancel the merge
* Understand the conflict
```

这个时候的提交历史是这样的：
```shell
$ git log --oneline --all --graph
*   8243ea6 (HEAD -> master) c5
|\  
| * 060ab8a (dev) c4
* | 931c6f9 Revert "M"
* |   01a84b6 M
|\ \  
| |/  
| * 225fb79 c3
| * f6cd947 c2
* | fc3fedf c1
|/  
* 12de8e7 c0
```

### 撤销上一次撤销

如果我们希望再次从dev分支合并README的内容，可以再次执行`git revert`命令，让它恢复到合并提交所对应的版本，也就是把`Revert "M"`对应的提交记录撤销掉。

首先，让我们回退到合并以前的状态：
```shell
$ git reset --hard HEAD^
HEAD is now at 931c6f9 Revert "M"
```

然后，执行`git revert`命令，撤销掉提交记录Revert "M"：
```shell
$ git revert 931c6f9
[master 929008c] Revert "Revert "M""
 2 files changed, 2 insertions(+)
 create mode 100644 .gitignore
```

再次查看提交历史：
```shell
$ git log --oneline --all --graph
* 929008c (HEAD -> master) Revert "Revert "M""
* 931c6f9 Revert "M"
*   01a84b6 M
|\  
* | fc3fedf c1
| | * 060ab8a (dev) c4
| |/  
| * 225fb79 c3
| * f6cd947 c2
|/  
* 12de8e7 c0
```

如图所示，虽然我们在Revert "M"后面多了一个新的提交记录Revert "Revert "M""：

![](/assets/images/lab/git/merge-stories-17.png)

但是查看README的文件内容会发现，它已经恢复到合并提交M的状态。也就是说，已经包含了上一次合并时来自dev分支的修改了：
```shell
$ cat README 
When conflict happens...
* Prepare
* Cancel the merge
* Understand the conflict
* ...
```

在这个基础上，我们再进行新的合并，把dev分支上的新变更合并到master分支：
```shell
$ git merge -m c5 dev
Merge made by the 'recursive' strategy.
 VERSION | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 VERSION
```

最后来看一下提交历史：
```shell
$ git log --oneline --all --graph
*   7084e12 (HEAD -> master) c5
|\  
| * 060ab8a (dev) c4
* | 929008c Revert "Revert "M""
* | 931c6f9 Revert "M"
* |   01a84b6 M
|\ \  
| |/  
| * 225fb79 c3
| * f6cd947 c2
* | fc3fedf c1
|/  
* 12de8e7 c0
```

![](/assets/images/lab/git/merge-stories-18.png)
