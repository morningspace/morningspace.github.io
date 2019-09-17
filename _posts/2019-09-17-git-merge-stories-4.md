---
title: Git合并的那些事儿——当冲突发生的时候
excerpt: 讲述冲突发生时，那些你也许不曾知道的事儿
categories: [tech]
tags: [lab, dummies, dummies_git, git]
toc: true
toc_sticky: true
---

> 多人开发中的合并冲突是我们使用Git时常常会遇到的情况，小小合并门道大，讲述合并的那些事儿，晴耕 · 白话之“Git合并那些事”系列​持续连载中……

注：
本文的部分写作灵感来自于[“Pro Git book”](https://git-scm.com/book/en/v2)。感谢原作者的精彩分享。
本文采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/80x15.png" /></a>

![](/assets/images/lab/git/broken_git.png){: .align-center}

## 制造冲突

这篇文章所要讨论的，全部都是和我们进行分支合并时产生的合并冲突有关。为了通过实验加深理解，首先需要人为地制造出一个合并冲突来。下面我们就一步一步地来准备实验环境。

首先，我们新建一个本地Git库，叫做test-merge-conflict：
```shell
$ git init test-merge-conflict
Initialized empty Git repository in /root/test-merge-conflict/.git/
$ cd test-merge-conflict/
```

在master分支上新建README文件：
```shell
$ vi README
$ cat README 
When conflict happens...
* Cancel the merge
```

并建立提交记录c0：
```shell
$ git add .
$ git commit -m c0
[master (root-commit) 12de8e7] c0
 1 file changed, 2 insertions(+)
 create mode 100644 README
```

然后，创建dev分支：
```shell
$ git branch dev
```

并继续在master分支上修改README文件：
```shell
$ vi README
$ cat README 
When conflict happens...
* Cancel the merge
* Understand the conflict
```

形成新的提交记录c1：
```shell
$ git commit -am c1
[master fc3fedf] c1
 1 file changed, 1 insertion(+)
```

然后，切换到dev分支：
```shell
$ git checkout dev
Switched to branch 'dev'
```

继续修改README文件：
```shell
$ vi README 
$ cat README
When conflict happens...
* Prepare
* Cancel the merge
* ...
```

并建立提交记录c2：
```shell
$ git commit -am c2
[dev f6cd947] c2
 1 file changed, 2 insertions(+)
```

同时，再增加一个新文件.gitignore：
```shell
$ touch .gitignore
$ git add .gitignore
```

并建立提交记录c3：
```shell
$ git commit -am c3
[dev 225fb79] c3
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 .gitignore
```

现在，让我们重新回到master分支：
```shell
$ git checkout master
Switched to branch 'master'
```

然后，把dev分支合并到master分支上来：
```shell
$ git merge dev
Auto-merging README
CONFLICT (content): Merge conflict in README
Automatic merge failed; fix conflicts and then commit the result.
```

这个时候，我们发现README在合并时产生了冲突。如果查看README的内容，还可以看到Git为我们自动添加的合并冲突标记：
```shell
$ cat README 
When conflict happens...
* Prepare
* Cancel the merge
<<<<<<< HEAD
* Understand the conflict
=======
* ...
>>>>>>> dev
```

## 冲突背后


### 三份拷贝

当合并出现冲突的时候，Git会在它的index（或者说是“暂存区”）里，针对产生冲突的文件，记录下该文件的三份拷贝。分别对应于：代表“共同祖先”的提交记录（common ancestor commit，也叫merge-base commit），用数字1标识；代表当前分支上参与合并的提交记录（也叫ours commit），用数字2标识；代表被合并分支上参与合并的提交记录（也叫theris commit），用数字3标识。

可以通过`git ls-files -u`命令对这三份拷贝进行查看：
```shell
$ git ls-files -u
100644 45e560ae8081887efcfa21e0f06e8490b59297de 1	README
100644 75e3ffd958d236a08543a5ab0149c5ae262c7b32 2	README
100644 9612c0564c52f87357ff29a92b52947a7b0057d3 3	README
```

其中，第一列代表的是mode值，100644对应的是普通文件。关于mode值更详细的介绍，大家可以在[“Git解密——Tree对象和Commit对象”](/tech/inside-git-2)一文里找到。第二列代表的是这些文件在.git目录下保存成Git对象所对应的唯一键。关于Git对象，可以在[“Git解密——认识Git对象”](/tech/inside-git-1)一文里找到更多的信息。第三列就是三份拷贝各自对应的数字标识。

我们还可以用`git show`命令，结合特殊的语法，查看具体的文件内容。

比如这是“共同祖先”的版本：
```shell
$ git show :1:README > README.common
$ cat README.common
When conflict happens...
* Cancel the merge
```

这是ours的版本：
```shell
$ git show :2:README > README.ours
$ cat README.ours
When conflict happens...
* Cancel the merge
* Understand the conflict
```

这是theirs的版本：
```shell
$ git show :3:README > README.theirs
$ cat README.theirs
When conflict happens...
* Prepare
* Cancel the merge
* ...
```

不仅如此，我们还可以利用这三个版本，结合`git merge-file`命令，显式地对README文件进行一次Three-Way Merge：
```shell
$ git merge-file -p README.ours README.common README.theirs > README.merged
```

这其中，参数`-p`是为了让Git把合并结果输出到标准输出设备，然后我们再利用重定向，把结果写入README.merged文件。默认情况下，合并后的结果会直接覆盖README.ours的内容。

当然，因为我们的合并存在冲突，所以还需要做进一步的手工编辑。当冲突解决后，我们可以使用`git clean`，清除所有手工合并时产生的多余临时文件：
```shell
$ git clean -f
Removing README.common
Removing README.merged
Removing README.ours
Removing README.theirs
```

### 使用git diff

当冲突发生的时候，Git为我们提供了很多工具，可以帮助我们进一步了解冲突的具体情况。比如，我们可以利用`git diff`，对当前工作目录下等待提交的内容（也就是自动合并后的结果）和三份拷贝进行比较：
```shell
$ git diff
diff --cc README
index 75e3ffd,9612c05..0000000
--- a/README
+++ b/README
@@@ -1,3 -1,4 +1,8 @@@
  When conflict happens...
+ * Prepare
  * Cancel the merge
++<<<<<<< HEAD
 +* Understand the conflict
++=======
+ * ...
++>>>>>>> dev
```

执行`git diff`时，Git会把能够成功完成自动合并的部分略去，只列出当前遇到冲突的部分。对于冲突部分的表示方法，Git采用的是一种被称为“Combined Diff”的格式。其中，涉及冲突的每一行最前面会多出两列来。第一列告诉我们，当前工作目录里的内容是否和ours版本有差异；第二列告诉我们，当前工作目录里的内容是否和theirs版本有差异。如果是新增的，就用“+”表示；如果是删除的，就用“-”表示。

所以，在上面的例子里：`* Prepare`和`* ...`都是来自theirs版本，相对于ours版本而言新增的内容；`* Understand the conflict`则刚好相反；`<<<<<<< HEAD`，`=======`，`>>>>>>> dev`都是Git自动为我们加入的合并冲突标记，对于ours和theirs而言都是新增的内容。

另外，如果我们只想和当前分支合并前的内容进行比较，也可以使用`git diff --ours`；和被合并分支上的内容进行比较，可以使用`git diff --theirs`；和作为两个分支的共同祖先进行比较，则可以使用`git diff --base`。

### 使用git log

为了快速掌握合并冲突是由那些改动引起的，我们也可以利用`git log`查看提交历史。它可以帮助我们把参与合并的两个分支上，所有和当前这次合并相关的提交记录都列出来：
```shell
$ git log --oneline --left-right HEAD...MERGE_HEAD
> 225fb79 (dev) c3
> f6cd947 c2
< fc3fedf (HEAD -> master) c1
```

这里，`MERGE_HEAD`指向的是我们的被合并分支。和它相对应的是`ORIG_HEAD`，指向的是当前分支。因为在我们的例子里它等价于`HEAD`，所以我们就直接使用了`HEAD`。对于`HEAD...MERGE_HEAD`这种写法，我们用到了一种被称为“Triple Dot”的特殊语法。它指定的是，两个分支上的所有提交记录，但不包含这两个分支的共同祖先。例如，在我们的例子里，c1，c2，c3都是满足条件的。而c0则不满足条件，因为它是两个分支的共同祖先：

![](/assets/images/lab/git/merge-stories-15.png)

上面的命令里，我们还使用了`--left-right`参数。它可以让Git标记出，影响合并结果的那些提交记录分别来自于哪个分支（左边还是右边）。来自左侧的提交记录会标记为`<`，在我们的例子里就是HEAD所指向的master分支；来自右侧的提交记录则会标记为`>`，在我们的例子里就是MERGE_HEAD所指向的被合并分支dev。

另外，如果我们把参数换成`--merge`，那么Git只会显示参与合并的两个分支上，直接造成当前文件冲突的那些提交记录：
```shell
$ git log --oneline --left-right --merge
> f6cd947 c2
< fc3fedf (HEAD -> master) c1
```

这里我们并没有看到提交记录c3，因为c3没有涉及对README文件的修改。

### 使用git checkout

如果我们把存在冲突的文件改乱了，想重新开始；或者，如果我们只想从其他分支上合并个别文件，我们可以使用`git checkout`。结合`--conflict`参数，它可以把存在冲突的文件，或者我们希望合并的文件重新checkout出来，并重新加上代表合并冲突的标记：
```shell
$ git checkout --conflict=diff3 README
```

参数`--conflict`的取值可以是`merge`或者`diff3`。默认情况下使用`merge`，Git会在文件里发生冲突的地方把代表ours和theirs的内容都显示出来。如果使用了`diff3`，Git还会把代表merge base的部分也显示出来：
```shell
$ cat README
When conflict happens...
* Prepare
* Cancel the merge
<<<<<<< ours
* Understand the conflict
||||||| base
=======
* ...
>>>>>>> theirs
```

其中，从`|||||||`开始到`=======`结束的部分就是代表merge base的部分。

另外，`git checkout`也可以带`--ours`或`--theirs`参数。这对于处理二进制文件的合并冲突很有用，因为二进制文件做文本比较没有意义，我们在合并时通常只要简单地选择其中一个分支上的内容就可以了。Git会为我们快速地选择参与合并的两个分支中的其中一方，而根本不需要做对比。
