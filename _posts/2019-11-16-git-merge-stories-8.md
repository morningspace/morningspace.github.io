---
title: Git合并那些事——Rebase的烦恼
excerpt: 通过一个例子来演示Rebase使用不当带来的麻烦
categories: [tech]
tags: [lab, dummies, dummies_git, git]
toc: true
toc_sticky: true
---

注：
本文的部分写作灵感来自于[“Pro Git book”](https://git-scm.com/book/en/v2)。感谢原作者的精彩分享。
本文采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/80x15.png" /></a>

## 烦恼在哪里

通过[“Git合并那些事儿——神奇的Rebase”](/tech/git-merge-stories-6/)一文我们明白了，Rebase本质上就是丢弃已有的提交记录，创建新的提交记录，从内容上看是完全一样的，但提交历史却改变了。

对提交历史的修改是需要谨慎的。如果我们已经把提交推送到远程，而其他人又从远程下载了我们的提交，并在这个基础上工作。这个时候，如果我们又利用Rebase修改了提交记录，并再次推送到远程，那么其他人就不得不重新合并他们之前的工作了。

## 烦恼的前奏

对于Rebase使用不当所造成的后果，下面让我们通过一个例子来演示整个过程，从而加深对它的理解。

本文推荐大家使用[Hello Git](https://github.com/morningspace/lab-hello-git)提供的两个Docker镜像作为实验环境：一个代表远程Git服务（`lab-git-remote`），一个代表本地Git客户端（`lab-git-local`）。

关于实验环境的准备，请同学们查看“[Git合并那些事儿——认识几种Merge方法](/tech/git-merge-stories-6)”一文的“准备实验环境”部分。按照文章里介绍的步骤一步步做，最终我们会得到一个远程Git服务和用户William的本地Git客户端。由于这个例子里，我们会涉及到两个人的协作开发，所以还需要启动另一个本地Git客户端，用户名为Nicole：
```shell
$ docker run -it --name git-local-nicole --hostname git-local-nicole --net=lab -e user_name=Nicole -e user_email=nicole@example.com morningspace/lab-git-local
```

假设William和Nicole都在开发test-rebase项目。首先，William在本地创建了test-rebase项目，并建立了提交记录c0：
```shell
$ git init test-rebase
Initialized empty Git repository in /root/test-rebase/.git/
$ cd test-rebase/

$ vi README
$ cat README
The magic rebase
* When to use?

$ git add .
$ git commit -m c0
[master (root-commit) f14301e] c0
 1 file changed, 2 insertions(+)
 create mode 100644 README
```

然后，他登录到远程Git服务，创建了test-rebase的远程Git库：
```shell
$ ssh git@my-git-remote
git> create test-rebase
Initialized empty Git repository in /home/git/test-rebase.git/
git> list
hello-git.git
test-rebase.git
git> exit
```

把本地库和远程库关联起来的同时，又把本地的c0推送到了远程：
```shell
$ git remote add origin git@my-git-remote:~/test-rebase.git

$ git push origin master
Counting objects: 3, done.
Writing objects: 100% (3/3), 232 bytes | 21.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0)
To my-git-remote:~/test-rebase.git
 * [new branch]      master -> master
```

之后又继续在本地工作，并建立了提交记录c1：
```shell
$ vi README
$ cat README
The magic rebase
* What is it?
* When to use?

$ git commit -am c1
[master 9a5bdd2] c1
 1 file changed, 1 insertion(+)
```

另一方面，Nicole也把test-rebase从远程克隆到了本地：
```shell
$ git clone git@my-git-remote:~/test-rebase.git
Cloning into 'test-rebase'...
remote: Counting objects: 3, done.
remote: Total 3 (delta 0), reused 0 (delta 0)
Receiving objects: 100% (3/3), done.
```

并在本地新建了一个dev分支后，继续在master上工作，并产生了提交记录c2：
```shell
$ git branch dev

$ touch .gitignore
$ git add .gitignore
$ git commit -m c2
[master f0f4430] c2
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 .gitignore
```

然后切换到dev分支上工作，产生提交记录c3：
```shell
$ git checkout dev
Switched to branch 'dev'

$ touch VERSION
$ git add VERSION
$ git commit -m c3
[dev 9aff71a] c3
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 VERSION
```

再切换回master分支，合并dev分支，产生合并提交c4：
```shell
$ git checkout master
Switched to branch 'master'
Your branch is ahead of 'origin/master' by 1 commit.
  (use "git push" to publish your local commits)

$ git merge -m c4 dev
Merge made by the 'recursive' strategy.
 VERSION | 0
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 VERSION
```

然后删除dev分支，把本地提交推送到远程：
``` shell
$ git branch -d dev
Deleted branch dev (was 9aff71a).

$ git push origin master
Counting objects: 7, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (6/6), done.
Writing objects: 100% (7/7), 619 bytes | 47.00 KiB/s, done.
Total 7 (delta 2), reused 0 (delta 0)
To my-git-remote:~/test-rebase.git
   f14301e..05ca470  master -> master
```

这时候远程Git库的提交历史是这样的：

![](/assets/images/lab/git/merge-stories-25.png)

而William在他本地和远程进行同步以后：
```shell
$ git pull origin master
From my-git-remote:~/test-rebase
 * branch            master     -> FETCH_HEAD
Merge made by the 'recursive' strategy.
 .gitignore | 0
 VERSION    | 0
 2 files changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 .gitignore
 create mode 100644 VERSION
```

本地的提交历史就变成下面这样了：
```shell
$ git log --oneline --graph
*   1b9384a (HEAD -> master) c5
|\  
| *   05ca470 (origin/master) c4
| |\  
| | * 9aff71a c3
| * | f0f4430 c2
| |/  
* | 9a5bdd2 c1
|/  
* f14301e c0
```

其中，c5是同步远程修改以后自动形成的合并提交：

![](/assets/images/lab/git/merge-stories-26.png)

## 烦恼的开始

这个时候，Nicole开始在她本地进行Rebase了。她首先重建（恢复）了dev分支，并让dev分支的head指针指向c3（9aff71a）
```shell
$ git branch dev 9aff71a
```

然后让master分支的head指针退回到上次合并之前的提交记录c2：
```shell
$ git reset --hard HEAD^
HEAD is now at f0f4430 c2
$ git log --oneline --graph --all
*   05ca470 (origin/master, origin/HEAD) c4
|\  
| * 9aff71a (dev) c3
* | f0f4430 (HEAD -> master) c2
|/  
* f14301e c0
```

并针对master和dev进行Rebase操作：
```shell
$ git rebase master dev
First, rewinding head to replay your work on top of it...
Applying: c3

$ git checkout master
Switched to branch 'master'
Your branch is behind 'origin/master' by 2 commits, and can be fast-forwarded.
  (use "git pull" to update your local branch)

$ git merge dev
Updating f0f4430..604a471
Fast-forward
 VERSION | 0
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 VERSION
```

完成Rebase以后，再次清除多余的dev分支：
```shell
$ git branch -d dev
Deleted branch dev (was 604a471).
```

这个时候，Nicole本地的提交历史是这样的：
```shell
$ git log --oneline --graph --all
* 604a471 (HEAD -> master) c3
| *   05ca470 (origin/master, origin/HEAD) c4
| |\  
|/ /  
| * 9aff71a c3
* | f0f4430 c2
|/  
* f14301e c0
```

注意，这里出现了两个c3。其中，SHA-1值为`604a471`的c3是Rebase完成后新建的提交记录。而SHA-1值为`9aff71a`的c3则是保存在远程库以及William的本地库里的原来的c3。

![](/assets/images/lab/git/merge-stories-27.png)

现在，Nicole利用`git push`加上`--force`参数，把她本地的修改强制推送到了远程：
```shell
$ git push --force origin master
Counting objects: 2, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (2/2), 270 bytes | 24.00 KiB/s, done.
Total 2 (delta 0), reused 0 (delta 0)
To my-git-remote:~/test-rebase.git
 + 05ca470...604a471 master -> master (forced update)
```

这个时候再看远程库的提交历史，就会变成这个样子：
```shell
$ git log --oneline --graph --all
* 604a471 (HEAD -> master, origin/master, origin/HEAD) c3
* f0f4430 c2
* f14301e c0
```

![](/assets/images/lab/git/merge-stories-28.png)

如果这个时候William把远程库的改动同步到本地：
```shell
$ git pull origin master
remote: Counting objects: 2, done.
remote: Compressing objects: 100% (2/2), done.
remote: Total 2 (delta 0), reused 0 (delta 0)
Unpacking objects: 100% (2/2), done.
From my-git-remote:~/test-rebase
 * branch            master     -> FETCH_HEAD
 + 05ca470...604a471 master     -> origin/master  (forced update)
Merge made by the 'recursive' strategy.
```

那么，本地的提交历史就会变成这样：
```shell
$ git log --oneline --graph
*   aac5943 (HEAD -> master) c6
|\  
| * 604a471 (origin/master) c3
* |   1b9384a c5
|\ \  
| * \   05ca470 c4
| |\ \  
| | |/  
| |/|   
| | * 9aff71a c3
| * | f0f4430 c2
| |/  
* | 9a5bdd2 c1
|/  
* f14301e c0
```

其中，c6是同步远程修改以后自动形成的合并提交：

![](/assets/images/lab/git/merge-stories-29.png)

于是，在本地的提交历史里，我们会看到两个c3。它们的作者，创建日期，备注信息是完全相同的。如果William再把目前的本地修改推送到远程：
```shell
$ git push origin master
Counting objects: 9, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (7/7), done.
Writing objects: 100% (9/9), 1.00 KiB | 44.00 KiB/s, done.
Total 9 (delta 1), reused 0 (delta 0)
To my-git-remote:~/test-rebase.git
   604a471..aac5943  master -> master
```

那么，这种混乱而复杂的提交历史又会出现在远程库里。如果Nicole再次从远程同步变更到她本地：
```shell
$ git pull origin master
remote: Counting objects: 9, done.
remote: Compressing objects: 100% (7/7), done.
remote: Total 9 (delta 1), reused 0 (delta 0)
Unpacking objects: 100% (9/9), done.
From my-git-remote:~/test-rebase
 * branch            master     -> FETCH_HEAD
   604a471..aac5943  master     -> origin/master
Updating 604a471..aac5943
Fast-forward
 README | 1 +
 1 file changed, 1 insertion(+)
```

就会发现原来经过Rebase之后变得很干净的本地提交历史，一下子多了很多复杂的分叉，并且还出现了两个一摸一样的提交记录（c3）：
```shell
$ git log --oneline --graph --all
*   aac5943 (HEAD -> master, origin/master, origin/HEAD) c6
|\  
| * 604a471 c3
* |   1b9384a c5
|\ \  
| * \   05ca470 c4
| |\ \  
| | |/  
| |/|   
| | * 9aff71a c3
| * | f0f4430 c2
| |/  
* | 9a5bdd2 c1
|/  
* f14301e c0
```

![](/assets/images/lab/git/merge-stories-30.png)

## 烦恼的解决

本地的提交历史被别人的Rebase操作“污染”了。当这种情况发生时，我们所面临的最大挑战，是如何把“真正需要”的那部分提交历史找出来，去掉那些因为分支合并而形成的提交记录，还有那些像c3一样的重复提交。Git为我们提供了自动完成这一工作的方法，那就是在执行`git pull`时加上`--rebase`参数。或者，也可以先执行`git fetch`把改动从远程下载到本地，然后再在本地执行`git rebase origin/master`，两者的效果是一样的。

接下来，我们就通过实验来进一步理解`git pull --rebase`是怎么工作的。还是接着上面的实验，首先，让我们把Nicole的本地环境恢复到执行`git pull`以前的状态：
```shell
$ git reset --hard 604a471
HEAD is now at 604a471 c3

$ git log --oneline --graph
* 604a471 (HEAD -> master) c3
* f0f4430 c2
* f14301e c0
```

然后再把这个提交历史强制推送到远程，从而让远程Git库也恢复到之前的状态，和Nicole本地的提交历史保持一致：
```shell
$ git push --force origin master
Total 0 (delta 0), reused 0 (delta 0)
To my-git-remote:~/test-rebase.git
 + aac5943...604a471 master -> master (forced update)
 
$ git log --oneline --graph --all
* 604a471 (HEAD -> master, origin/master, origin/HEAD) c3
* f0f4430 c2
* f14301e c0
```

另一方面，我们把William的本地库也恢复到最近一次远程同步之前的状态：
```shell
$ git reset --hard 1b9384a
HEAD is now at 1b9384a c5
```

注意这个时候William本地的提交历史：
```shell
$ git log --oneline --graph      
*   1b9384a (HEAD -> master) c5
|\  
| *   05ca470 c4
| |\  
| | * 9aff71a c3
| * | f0f4430 c2
| |/  
* | 9a5bdd2 c1
|/  
* f14301e c0
```

现在，执行`git pull --rebase`：
```shell
$ git pull --rebase origin master
From my-git-remote:~/test-rebase
 * branch            master     -> FETCH_HEAD
First, rewinding head to replay your work on top of it...
Applying: c1
```

在`git pull`执行的过程中，Git会做这么几件事情：
* 找出所有只在William本地出现，而不在远程（或Nicole本地）出现的提交记录，结果是：c1，c3，c4，c5；
* 从中去掉所有合并提交（因为这些提交只负责合并，它们所包含的变更在其他提交记录里已经有了），结果是：c1，c3；
* 再从中去掉那些内容重复的提交记录，如：c3，最后剩下：c1；
* 把所有剩余的提交记录（即：c1）在origin/master的最新提交后面重新“回放”一遍；

对于上面的第三步，Git之所以能做到，是因为每个提交记录除了包含有作为唯一键的SHA-1值以外，还有一个叫做“patch-id”的东西，它是Git根据该提交记录所引入的变更计算其校验和（checksum）得到的。如果两个提交的变更内容一摸一样（比如：c3），那它们的patch-id就是一样的。

经过这样的处理以后，再来看William本地的提交历史：
```shell
$ git log --oneline --graph
* 2665bc4 (HEAD -> master) c1
* 604a471 (origin/master) c3
* f0f4430 c2
* f14301e c0
```

我们发现，提交历史又变干净了！和远程库的提交历史相比，我们只多了一个提交记录：c1，其他提交记录，由于对项目的内容本身都没有引入真正的变更，所以都被Git去掉了。这样的提交历史才是我们真正需要的！

![](/assets/images/lab/git/merge-stories-31.png)
