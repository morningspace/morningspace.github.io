---
title: Git工作流面面观——集中式工作流
excerpt: 最为基本的一种Git工作流，适合习惯传统版本控制方式的团队
categories: [tech]
tags: [lab, dummies, dummies_git, git]
toc: true
toc_sticky: true
---

注：
本文采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/80x15.png" /></a>

![](/assets/images/lab/git/workflow.png)

## 名词解释

所谓集中式工作流(Centralized Workflow)，是指所有人共享一个Git库，且只有一个分支(即master)，所有人的变更都统一提交到这个Git库的master上。

这种工作流程非常适合那些从其他传统版本控制工具（如：SVN）迁移过来的团队，因为这就是他们以前熟悉的工作方式。它允许每个人在本地有一个属于自己的隔离环境，能够独立工作，然后在需要的时候把本地的工作同步到远程的共享库。

当然，这样做并没有完全发挥出Git在分布式协作方面的优势。而且，随着团队规模的扩大，大家在维护共享库时产生冲突的机会也会逐渐增多，从而会使共享库成为整个团队开发的瓶颈。

尽管如此，集中式工作流仍然是很重要的一种工作流，因为它是其他Git工作流的基础。本质上讲，其他工作流都是它的某种“衍生”。接下来，我们就通过一个模拟示例来演示一下集中式工作流的运行方式。

## 如何工作？

第一步，在服务器上建立一个远程Git库，供所有人使用：
```shell
$ ssh git@my-git-remote
git> create test-centralized
Initialized empty Git repository in /home/git/test-centralized.git/
git> exit
Connection to my-git-remote closed.
```

第二步，所有人把远程库克隆到本地：
```shell
$ git clone git@my-git-remote:~/test-centralized.git
Cloning into 'test-centralized'...
warning: You appear to have cloned an empty repository.
cd test-centralized/
```

第三步，开发人员在本地进行日常开发，把变更提交到本地Git库。比如，William在开发feature-1：
```shell
$ vi README
$ cat README     
Git workflows
* Centralized workflow

$ git add README 
$ git commit -m 'working on feature-1'
[master (root-commit) b550c19] working on feature-1
 1 file changed, 2 insertions(+)
 create mode 100644 README
```

Nicole在开发feature-2：
```shell
$ vi README
$ cat README 
Git workflows
* ...

$ git add README 
$ git commit -m 'working on feature-2'
[master (root-commit) dac348b] working on feature-2
 1 file changed, 2 insertions(+)
 create mode 100644 README
```

第四步，开发人员把本地master分支上的变更推送到远程。比如，William完成工作以后，把变更成功推送到了远程：
```shell
$ git push
Counting objects: 3, done.
Writing objects: 100% (3/3), 243 bytes | 30.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0)
To my-git-remote:~/test-centralized.git
 * [new branch]      master -> master
```

这个时候，Nicole也完成工作了，当她尝试推送时却被Git拒绝了：
```shell
$ git push
To my-git-remote:~/test-centralized.git
 ! [rejected]        master -> master (fetch first)
error: failed to push some refs to 'git@my-git-remote:~/test-centralized.git'
hint: Updates were rejected because the remote contains work that you do
hint: not have locally. This is usually caused by another repository pushing
hint: to the same ref. You may want to first integrate the remote changes
hint: (e.g., 'git pull ...') before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.
```

这是由于在Nicole推送之前，远程库里已经有了William推送的变更，而这些变更还没有出现在Nicole本地。为了避免后面推送的变更覆盖前面的，Git拒绝了Nicole的请求。

第五步，Nicole需要先利用`git pull`命令抓取远程库的最新变更，然后集成到本地。集成的时候，最好使用rebase的方式（使用参数`--rebase`），把本地生成的提交逐个“追加”到远程库最新提交的后面，而不是把所有工作都放到一个大的合并提交里面（不使用参数`--rebase`的时候）。这样做的好处是，可以保证整个项目的提交历史始终是线性的，易于理解。
```shell
$ git pull --rebase
warning: no common commits
remote: Counting objects: 3, done.
remote: Total 3 (delta 0), reused 0 (delta 0)
Unpacking objects: 100% (3/3), done.
From my-git-remote:~/test-centralized
 * [new branch]      master     -> origin/master
First, rewinding head to replay your work on top of it...
Applying: working on feature-2
Using index info to reconstruct a base tree...
Falling back to patching base and 3-way merge...
Auto-merging README
CONFLICT (add/add): Merge conflict in README
error: Failed to merge in the changes.
Patch failed at 0001 working on feature-2
Use 'git am --show-current-patch' to see the failed patch

Resolve all conflicts manually, mark them as resolved with
"git add/rm <conflicted_files>", then run "git rebase --continue".
You can instead skip this commit: run "git rebase --skip".
To abort and get back to the state before "git rebase", run "git rebase --abort".
```

如果集成的时候遇到了冲突，比如在我们这个例子里，那就需要先解决冲突：
```shell
$ cat README 
Git workflows
<<<<<<< HEAD
* Centralized workflow
=======
* ...
>>>>>>> working on feature-2

$ vi README 
$ cat README 
Git workflows
* Centralized workflow
* ...
$ git add README 
$ git rebase --continue
Applying: working on feature-2
```

在这种情况下，使用rebase的方式进行集成，可以把冲突的范围局限在单个提交里，而不是所有提交都揉在一起。而且，这样做还可以保持每个提交记录都专注于自己的事情，当出现问题时更容易定位引入问题的地方，需要回退提交历史的时候也可以让回退的影响降到最小。解决完合并冲突以后，本地的提交历史变成下面这样了：
```shell
$ git log --oneline
3f92cbb (HEAD -> master) working on feature-2
b550c19 (origin/master) working on feature-1
```

第六步，Nicole再次尝试推送她的修改，这一回终于成功了。
```shell
$ git push
Counting objects: 3, done.
Writing objects: 100% (3/3), 280 bytes | 28.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0)
To my-git-remote:~/test-centralized.git
   b550c19..3f92cbb  master -> master
```
