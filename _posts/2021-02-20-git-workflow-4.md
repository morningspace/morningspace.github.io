---
title: Git工作流面面观——Gitflow工作流
excerpt: 广泛应用的一种Git工作流，适合管理有固定发布周期的大项目
categories: [tech]
tags: [lab, dummies, dummies_git, git]
toc: true
toc_sticky: true
---

注：
本文采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/80x15.png" /></a>

![](/assets/images/lab/git/workflow.png)

## 名词解释

Gitflow工作流(Gitflow Workflow)是2010年由Vincent Driessen在他的一篇[博客](https://nvie.com/posts/a-successful-git-branching-model)里提出来的。它定义了一整套完善的基于Git分支模型的框架，结合了版本发布的研发流程，适合管理具有固定发布周期的大型项目。

和特性分支工作流相比，Gitflow工作流并没有引入任何新的概念。不同的地方在于，它强化了对Git分支模型的使用，结合产品或项目发布周期的特定需求，定义了各种不同类型的分支，每一种分支都有它自己特定的职责，并且分支之间什么时候、以什么样的方式交互，也都有相应的规则。下面我们就具体来看一下。

### Master分支

Master分支作为唯一一个正式对外发布的分支，是所有分支里最稳定的。这是因为，只有经过了严格审核和测试，并且在当前发布计划里的特性，才会被合并到master分支。当某个版本发布的时候，我们通常还会为master分支加上带有相应版本号的tag。

### Develop分支

Develop分支是根据master分支创建出来的，它作为一种集成分支(Integration Branch)，是专门用来集成开发完成的各种特性的。Develop分支通常具有更加详细和完整的提交历史，包括一些很细节的提交记录。而master分支则因为是面向版本发布的，所以它的提交历史会略去这些细节，显得比较精简。

### Feature分支

Feature分支是根据develop分支创建出来的，Gitflow工作流里的每个新特性都有自己的feature分支，这一点和特性分支工作流是一样的。这些分支除了在开发人员的本地存在以外，也可以被推送到共享的远程Git库，作为工作备份，以及与其他人协同工作的基础。当特性开发结束以后，这些分支上的工作会被合并到develop分支。但feature分支从来不会直接和master分支打交道。

### Release分支

当积累了足够多的已完成特性，或者预定的系统发布周期临近的时候，我们就会从develop分支创建出一个release分支，专门用来做和当前版本发布有关的工作。Release分支一旦开出来以后，就不允许再有新的特性被加入到这个分支了，只有bug修复或者文档编辑之类的工作才允许进入该分支。

Release分支上的内容最终会被合并到master分支，等版本发布的时候，我们通常还会为master分支加上带有相应版本号的tag。同时，release分支也会被合并到develop分支。在release分支活跃其间，develop分支也一直处于Open状态。Release分支上的内容代表当前版本在发布之前的准备工作，develop分支上的内容则代表下一个版本的开发工作，两者是可以并行展开的。

### Hotfix分支

Hotfix分支不从是develop分支创建出来的，而是直接根据master分支创建得到的，其目的是为了给运行在生产环境中的系统快速提供补丁，同时确保不会给正在其他上分支进行的工作造成影响。当hotfix分支上的工作完成以后，可以合并到master分支和develop分支，以及当前的release分支。如果有版本的更新，也可以为master分支打上相应的tag。

![](/assets/images/lab/git/git-workflow-6.png)

上面提到的所有分支，从稳定性的角度来说，每一种分支都处在各自不同的层次。如果当前分支的代码达到了更加稳定的水平，那它就可以向更稳定的分支进行合并了。

## 如何工作？

Vincent Driessen不仅定义了Gitflow的工作流程，还提供了一个相应的命令行工具[git-flow](https://github.com/nvie/gitflow)，可以简化我们在执行Gitflow工作流时，对每一个分支的各种繁琐的操作。它实际上是对Git命令行的封装。接下来，我们就利用这个工具演示一下整个Gitflow工作流。

如果你使用的是[Hello Git](https://github.com/morningspace/lab-hello-git)的实验环境，那么git-flow已经提前安装好了，否则需要自己手动安装。git-flow的安装非常简单，比如在Ubuntu环境下，只要执行下面的命令就可以了：
```shell
$ apt-get update
$ apt-get install git-flow
```

第一步，在服务器上建立一个远程Git库，供所有人使用：
```shell
$ ssh git@my-git-remote
git> create test-gitflow
Initialized empty Git repository in /home/git/test-gitflow.git/
git> exit
Connection to my-git-remote closed.
```

第二步，William把远程库克隆到本地：
```shell
$ git clone git@my-git-remote:~/test-gitflow.git
Cloning into 'test-gitflow'...
warning: You appear to have cloned an empty repository.
cd test-gitflow/
```

并生成项目的初始提交：
```shell
$ touch README
$ git add README
$ git commit -m 'Initial commit'
[master (root-commit) db64af2] Initial commit
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 README
```

然后把它推送到远程：
```shell
$ git push
Counting objects: 3, done.
Writing objects: 100% (3/3), 208 bytes | 17.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0)
To my-git-remote:~/test-gitflow.git
 * [new branch]      master -> master
```

第三步，William在他本地的工作目录下执行`git flow init`命令，对本地库进行初始化：
```shell
$ git flow init

Which branch should be used for bringing forth production releases?
   - master
Branch name for production releases: [master] 
Branch name for "next release" development: [develop] 

How to name your supporting branch prefixes?
Feature branches? [feature/] 
Bugfix branches? [bugfix/] 
Release branches? [release/] 
Hotfix branches? [hotfix/] 
Support branches? [support/] 
Version tag prefix? [] 
Hooks and filters directory? [/root/test-gitflow/.git/hooks] 
```

git-flow的初始化过程包含一系列问答环节，主要涉及各个分支的名称以及名称前缀的配置。如果我们选择默认配置，就可以一直按回车键直到命令执行完毕。这个时候，我们会发现本地除了master分支以外，还会多出一个develop分支。并且，git-flow已经替我们把当前分支切换到了develop分支：
```shell
$ git branch
* develop
  master
```

第四步，William开始新特性xyz的开发，执行`git flow feature start`命令，并传入特性的名称：
```shell
$ git flow feature start xyz
Switched to a new branch 'feature/xyz'

Summary of actions:
- A new branch 'feature/xyz' was created, based on 'develop'
- You are now on branch 'feature/xyz'

Now, start committing on your feature. When done, use:

     git flow feature finish xyz
```

git-flow会自动为我们从develop分支创建出一个名叫feature/xyz的分支，并把当前分支切换到该分支。

现在，William可以在feature/xyz分支上进行新特性的开发，并生成相应的提交记录了：
```shell
$ vi README
$ cat README 
Git workflows
* Gitflow workflow

$ git commit -am 'Working on feature/xyz'
[feature/xyz 48d92a9] Working on feature/xyz
 1 file changed, 2 insertions(+)
```

然后再把提交记录推送到远程，可以作为当前工作的备份，也可以让其他人访问到他的工作，还可以通过Pull Request发起讨论：
```shell
$ git push origin feature/xyz
Counting objects: 3, done.
Writing objects: 100% (3/3), 269 bytes | 24.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0)
To my-git-remote:~/test-gitflow.git
 * [new branch]      feature/xyz -> feature/xyz
```

第五步，Nicole接到了一个hotfix的任务，她和William一样，也把远程库克隆到本地，并利用`git flow init`命令对本地库进行了初始化，然后执行`git flow hotfix start`命令，并传入hotfix的名称：
```shell
$ git flow hotfix start 123
Switched to a new branch 'hotfix/123'

Summary of actions:
- A new branch 'hotfix/123' was created, based on 'master'
- You are now on branch 'hotfix/123'

Follow-up actions:
- Start committing your hot fixes
- Bump the version number now!
- When done, run:

     git flow hotfix finish '123'
```

git-flow会自动为我们直接从master分支创建出一个名叫hotfix/123的分支，并把当前分支切换到该分支。

现在，Nicole可以在hotfix/123分支上进行hotfix的开发，并生成相应的提交记录了：
```shell
$ touch LICENSE
$ git add LICENSE
$ git commit -m 'Working on hotfix/123'
[hotfix/123 1a70748] Working on hotfix/123
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 LICENSE
```

然后再把提交记录推送到远程，可以作为当前工作的备份，也可以让其他人访问到她的工作，还可以通过Pull Request发起讨论：
```shell
$ git push origin hotfix/123
Counting objects: 2, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (2/2), 245 bytes | 27.00 KiB/s, done.
Total 2 (delta 0), reused 0 (delta 0)
To my-git-remote:~/test-gitflow.git
 * [new branch]      hotfix/123 -> hotfix/123
```

当hotfix的开发工作结束以后，再执行`git flow hotfix finish`命令，并传入hotfix的名称：
```shell
$ git flow hotfix finish 123
Switched to branch 'master'
Your branch is up to date with 'origin/master'.
Merge made by the 'recursive' strategy.
 LICENSE | 0
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 LICENSE
Switched to branch 'develop'
Merge made by the 'recursive' strategy.
 LICENSE | 0
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 LICENSE
To my-git-remote:~/test-gitflow.git
 - [deleted]         hotfix/123
Deleted branch hotfix/123 (was 1a70748).

Summary of actions:
- Hotfix branch 'hotfix/123' has been merged into 'master'
- The hotfix was tagged '123'
- Hotfix tag '123' has been back-merged into 'develop'
- Hotfix branch 'hotfix/123' has been locally deleted; it has been remotely deleted from 'origin'
- You are now on branch 'develop'
```

git-flow会自动为我们把hotfix/123分支上的工作合并到master分支和develop分支，并给master分支加上hotfix的tag，然后再把hotfix分支分别从本地库和远程库里删掉，最后再把当前分支切换到develop。

第六步，William的新特性开发工作完成了，执行`git flow feature finish`命令，并传入特性的名称：
```shell
$ git flow feature finish xyz           
Switched to branch 'develop'
Updating db64af2..48d92a9
Fast-forward
 README | 2 ++
 1 file changed, 2 insertions(+)
To my-git-remote:~/test-gitflow.git
 - [deleted]         feature/xyz
Deleted branch feature/xyz (was 48d92a9).

Summary of actions:
- The feature branch 'feature/xyz' was merged into 'develop'
- Feature branch 'feature/xyz' has been locally deleted; it has been remotely deleted from 'origin'
- You are now on branch 'develop'
```

git-flow会自动为我们把分支feature/xyz上的工作合并到develop分支，并把该分支分别从本地库和远程库里删掉，然后再把当前分支切换到develop。

第七步，当前版本abc即将要发布了，William执行`git flow release start`命令，并传入版本号：
```shell
$ git flow release start abc
Switched to a new branch 'release/abc'

Summary of actions:
- A new branch 'release/abc' was created, based on 'develop'
- You are now on branch 'release/abc'

Follow-up actions:
- Bump the version number now!
- Start committing last-minute fixes in preparing your release
- When done, run:

     git flow release finish 'abc'
```

git-flow会根据develop分支自动为我们创建出一个名叫release/abc的分支，专门用于发布准备，然后把当前分支切换到该分支。release/abc分支一旦创建，就不能再往里面添加新特性了，但可以继续做一些bug修复的工作：
```shell
$ vi README 
$ cat README 
Git workflows
* Gitflow workflow
* ...
```

生成相应的提交记录：
```shell
$ git commit -am 'Working on release/abc'
[release/abc 79f1e5e] Working on release/abc
 1 file changed, 1 insertion(+)
```

然后推送到远程：
```shell
$ git push origin release/abc
Counting objects: 6, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (6/6), 510 bytes | 19.00 KiB/s, done.
Total 6 (delta 0), reused 0 (delta 0)
To my-git-remote:~/test-gitflow.git
 * [new branch]      release/abc -> release/abc
```

最后，发布工作结束，执行`git flow release`命令，并传入版本号：
```shell
$ git flow release finish abc
Switched to branch 'master'
Your branch is up to date with 'origin/master'.
Merge made by the 'recursive' strategy.
 README | 3 +++
 1 file changed, 3 insertions(+)
Already on 'master'
Your branch is ahead of 'origin/master' by 3 commits.
  (use "git push" to publish your local commits)
Switched to branch 'develop'
Merge made by the 'recursive' strategy.
 README | 1 +
 1 file changed, 1 insertion(+)
To my-git-remote:~/test-gitflow.git
 - [deleted]         release/abc
Deleted branch release/abc (was 79f1e5e).

Summary of actions:
- Release branch 'release/abc' has been merged into 'master'
- The release was tagged 'abc'
- Release tag 'abc' has been back-merged into 'develop'
- Release branch 'release/abc' has been locally deleted; it has been remotely deleted from 'origin'
- You are now on branch 'develop'
```

git-flow会自动为我们把release/abc分支上的工作合并到master分支和develop分支，并在master分支上为当前版本加上tag。然后把release分支分别从本地库和远程库里删掉，最后再切换回develop分支，一个版本发布周期就结束了。
