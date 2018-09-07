---
title: Git极简教程
categories: [blog, tech]
tags: [git]
---

摘自廖雪峰的[Git教程](http://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)。

## 安装配置

安装完成后的配置：
```shell
$ git config --global user.name "Your Name"
$ git config --global user.email "email@example.com"
```
加上--global针对当前用户起作用，如不加，只针对当前仓库起作用。当前用户的Git配置文件放在用户主目录下的.gitconfig中，每个仓库的Git配置文件放在.git/config中。

创建本地版本库：
```shell
$ git init
```
### 其他配置

让Git显示颜色：
```shell
$ git config --global color.ui true
```
忽略某些文件时，需要编写.gitignore，并放到版本库里，参考[这里](https://github.com/github/gitignore)。

定义命令别名：
```shell
$ git config --global alias.st status
$ git config --global alias.co checkout
$ git config --global alias.ci commit
$ git config --global alias.br branch
$ git config --global alias.unstage 'reset HEAD'
$ git config --global alias.last 'log -1'
$ git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"
```

## Github配置

### SSH Key配置
在用户主目录下执行如下命令以生成SSH Key：
```shell
$ ssh-keygen -t rsa -C "youremail@example.com"
```
登录GitHub添加SSH Key，粘贴id_rsa.pub文件的内容。

### 库配置

登录GitHub创建新的库。

将本地库与远程库关联：
```shell
$ git remote add origin git@server-name:path/repo-name.git
```
### 推送及获取

把本地库的所有内容推送到远程库：
```shell
$ git push -u origin master
```
下次，只要每次本地做了提交，可以执行如下命令：
```shell
$ git push origin master
```
克隆仓库：
```shell
$ git clone git@server-name:path/repo-name.git
```
## 常用命令

### 基本命令

添加一个或多个文件：
```shell
$ git add <file>
```
提交更改：
```shell
$ git commit -m "<message>"
```
查看工作区状态：
```shell
$ git status
```
查看修改内容：
```shell
$ git diff <file>
```
查看提交历史：
```shell
$ git log [--pretty=oneline]
```
查看命令历史：
```shell
$ git reflog
```
删除文件：
```shell
$ git rm <file>
```
### 恢复版本

恢复到指定版本：
```shell
$ git reset --hard <commit_id>
$ git reset --hard HEAD^|HEAD^^|...|HEAD~100
```
丢弃工作区修改：
```shell
$ git checkout -- <file>
```
丢弃暂存区修改：
```shell
$ git reset HEAD <file>
```
### 管理库

关联远程库：
```shell
$ git remote add origin git@server-name:path/repo-name.git
```
第一次推送master分支的所有内容：
```shell
$ git push -u origin master
```
首次推送后的推送：
```shell
$ git push origin master
```
克隆仓库：
```shell
$ git clone git@server-name:path/repo-name.git
```
查看远程库：
```shell
$ git remote [-v]
```
抓取分支：
```shell
$ git pull
```
## 分支

### 基本命令

创建分支：
```shell
$ git checkout -b <branch>
```
创建远程分支到本地：
```shell
$ git checkout -b <branch> origin/<branch>
```
查看当前分支：
```shell
$ git branch
```
切换回master分支：
```shell
$ git checkout master
```
合并别的分支到当前分支：
```shell
$ git merge <branch>
```
删除分支：
```shell
$ git branch -d <branch>
```
强行删除分支：
```shell
$ git branch -D <branch>
```
查看分支合并情况：
```shell
$ git log --graph --pretty=oneline --abbrev-commit
```
合并分支时禁用fastforward模式：
```shell
$ git merge --no-ff -m "<comment>" <branch>
```
指定本地分支与远程分支的链接：
```shell
$ git branch --set-upstream <branch> origin/<branch>
```
### 暂存工作区

保存当前工作现场：
```shell
$ git stash
```
列出保存的工作现场：
```shell
$ git stash list
```
恢复工作现场：
```shell
$ git stash apply stash@{n}
$ git stash drop stash@{n}
```
或，
```shell
$ git stash pop
```
## 标签

建立标签：
```shell
$ git tag <tag> [<commit id>]
```
建立带说明的标签：
```shell
$ git tag -a <tag> -m <comment> [<commit id>]
```
查看所有标签：
```shell
$ git tag
```
查看标签信息：
```shell
$ git show <tag>
```
删除标签：
```shell
$ git tag -d <tag>
```
推送标签到远程
```shell
$ git push origin <tag>
```
推送所有尚未被推送的本地标签
```shell
$ git push origin --tags
```
删除远程标签（先删除本地标签）：
```shell
$ git push origin :refs/tags/<tag>
```
## 搭建Git服务器

安装Git：
```shell
$ sudo apt-get install git
```
创建一个git用户：
```shell
$ sudo adduser git
```
把所有用户id_rsa.pub中的公钥导入到/home/git/.ssh/authorized_keys文件里，一行一个

选定一个目录<path>，在该目录下初始化Git仓库：
```shell
$ sudo git init --bare <repo>.git
```
把owner改成git：
```shell
$ sudo chown -R git:git <repo>.git
```
禁用shell登录，编辑/etc/passwd，将：
```shell
    git:x:1001:1001:,,,:/home/git:/bin/bash
```
改成：
```shell
    git:x:1001:1001:,,,:/home/git:/usr/bin/git-shell
```
克隆远程仓库：
```shell
$ git clone git@<server>:/<path>/<repo>.git
```
## 其他资源

* [Git Cheat Sheet](https://www.git-tower.com/blog/git-cheat-sheet)
* [Git官网](http://git-scm.com/)
