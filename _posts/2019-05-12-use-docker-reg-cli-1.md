---
title: Docker Registry CLI轻松管理Docker注册表(上)
excerpt: "晴耕工坊最新开源项目发布，介绍Docker Registry CLI基本用法"
categories: [tech]
tags: [docker, shell, studio]
toc: true
toc_sticky: true
---
> 继开源项目[Elastic Shell](https://github.com/morningspace/elastic-shell)[发布](/tech/elash-and-studio)以后，[晴耕工坊](/studio)近期又有新品发布啦！今天，我们就一起来了解一下[Docker Registry CLI](https://github.com/morningspace/docker-registry-cli)这款开源小工具。

![](/assets/images/studio/docker_registry.png)

## 什么是Docker Registry CLI

我们知道，[Docker Registry（Docker注册表）](https://docs.docker.com/registry/)是一项服务，用于存储Docker Image(镜像)。例如大家所熟知的[Docker Hub](https://hub.docker.com/)，[Quay](https://quay.io/)，[Google Container Registry](https://cloud.google.com/container-registry/)，[AWS Container Registry](https://aws.amazon.com/ecr/)等，就属于Public Docker Registry。此外，我们也可以搭建属于自己的Private Docker Registry。

Docker Registry CLI是一款用Bash Shell编写的命令行工具，可以方便而灵活的对支持[HTTP API V2](https://docs.docker.com/registry/spec/api/)版本的Docker Registry进行管理，同时支持公有和私有的Docker Registry。它具备了以下几个特色：
* 批量复制：作为该工具的一大特色，利用其`cp`命令，可以从多个公有及私有Registry批量复制Image，轻松搭建起属于自己的Docker Registry；
* 基于DIND：提供基于[Docker-in-Docker(DIND)](https://github.com/jpetazzo/dind)的运行模式，支持容器内执行docker命令，所有操作对宿主机Registry缓存均不会造成任何影响；
* 强大灵活：只通过少数几个命令及其参数的组合，就完成了针对Image，Tag，Digest，Manifest在内的各种查询，以及Image的复制删除操作；
* 简单易学：由于在设计上极大程度的借鉴了Linux常用命令的语法，因而很容易上手。只要了解`cp`, `ls`, `rm`等命令，就可以很快掌握使用方法；

接下来我们就一起来看看，如何借助Docker Registry CLI快速搭建一个Private Docker Registry！我们将从中进一步体会到该工具的上述几个特点。

## 如何运行

Docker Registry CLI提供了三种运行方式。

* 在Docker容器内运行：从Docker Hub上抓取[Image](https://cloud.docker.com/u/morningspace/repository/docker/morningspace/docker-registry-cli)到本地，就可以在容器内运行Docker Registry CLI了。这是最便捷的一种运行方式，因为容器内预装了所有该工具执行的先决依赖，包括：*bash*，*curl*，*jq*，*docker*。
* 在Docker容器外运行：当然，我们也可以选择从[GitHub库](https://github.com/morningspace/docker-registry-cli)下载源码，在本地的宿主机器上直接运行。只要在运行之前，确保本机已经安装了工具所需的所有先决依赖即可。
* 利用[Docker Compose](https://docs.docker.com/compose/)：此外，我们还可以将[GitHub库](https://github.com/morningspace/docker-registry-cli)完整克隆到本地，并通过执行`docker-compose`命令来运行。本质上，这也属于容器内运行。只不过预先配置好的`docker-compose.yml`里，除了Docker Registry CLI本身外，还包含一个hostname为`mr.io`的Private Docker Registry，可以用来做实验。本文后面的演示，都将采用这一运行方式。

### 使用Docker Compose

利用Docker Compose运行Docker Registry CLI非常简单。只需要两步即可：

* 第一步：将[GitHub库](https://github.com/morningspace/docker-registry-cli)克隆到本地后执行如下命令，以同时启动Docker Registry CLI的Daemon服务及`mr.io`：

```shell
$ docker-compose up -d
Creating network "net-registry" with driver "bridge"
Creating docker-registry-cli-v0_reg-cli_1_d4de93c59ec0 ... done
Creating mr.io                                         ... done
```

* 第二步：执行如下命令，通过另一个容器连接至Docker Registry CLI的Daemon服务：

```shell
$ docker-compose exec reg-cli bash
```

进入容器后，我们就可以通过执行`reg-cli`命令运行Docker Registry CLI了！

首先，我们利用`reg-cli ls`命令查看一下`mr.io`的Catalog：

```shell
bash-4.4# reg-cli ls mr.io
mr.io
---------------------------------------
```

返回结果显示，目前`mr.io`上还没有任何Image。别急，接下来我们将通过Docker Registry CLI把来自其他Registry的Image复制到`mr.io`上！当然，我们也可以利用`docker push`命令直接往`mr.io`上推送Image。

## 复制第一个Image

向Docker Registry复制Image，需要用到`reg-cli cp`命令。利用它，可以将存储于其他Registry（包括公有和私有）的Image批量复制到目标Registry。

我们先来试一下把Docker Registry CLI自身的Image从Docker Hub上抓取下来并推送到`mr.io`。执行如下命令：

```shell
bash-4.4# reg-cli cp morningspace/docker-registry-cli mr.io
```

等待命令执行完毕后，再利用`reg-cli ls`查看`mr.io`的Catalog：

```shell
bash-4.4# reg-cli ls mr.io
mr.io
---------------------------------------
docker-registry-cli
```

恭喜你！我们的第一个Image已经成功复制到`mr.io`了！在`reg-cli ls`后面加上Image的全名，可以列出该Image位于`mr.io`上的所有Tag：

```shell
bash-4.4# reg-cli ls mr.io/docker-registry-cli
mr.io/docker-registry-cli
---------------------------------------
latest
```

目前为止，我们的`docker-registry-cli`只有一个`latest`的Tag。再用`-ld`和`-lm`选项分别查看一下`docker-registry-cli`的Digest和Manifest吧：

```shell
bash-4.4# reg-cli ls mr.io/docker-registry-cli -ld
mr.io/docker-registry-cli
---------------------------------------
sha256:d1b110ad26922d45fce7baccd572ba35eef0412a0af8354177ae742d99718992 latest
```

```shell
bash-4.4# reg-cli ls mr.io/docker-registry-cli -lm
mr.io/docker-registry-cli
---------------------------------------
{
   "schemaVersion": 1,
   "name": "docker-registry-cli",
   "tag": "latest",
   "architecture": "amd64",
   "fsLayers": [
     ...
   ],
   "history": [
     ...
   ],
   "signatures": [
     ...
   ]
}
```

### 在容器里执行docker命令

前面提到过，Docker Registry CLI的Image是基于[Docker-in-Docker(DIND)](https://github.com/jpetazzo/dind)制作的。因此，我们可以在Docker Registry CLI所在的容器内运行`docker`命令！实际上，`reg-cli cp`的功能就是通过组合多个`docker`命令的方式实现的。我们在容器里执行`docker images`命令看看：

```shell
bash-4.4# docker images
REPOSITORY                         TAG                 IMAGE ID            CREATED             SIZE
morningspace/docker-registry-cli   latest              84f9cabf011f        10 hours ago        189MB
mr.io/docker-registry-cli          latest              84f9cabf011f        10 hours ago        189MB
```

上面列出的，就是我们刚才从Docker Hub上抓取到本地的Image，以及要推送到`mr.io`的Image Tag。而这些Image或者Tag，在宿主机器上是看不到的！DIND技术的神奇之处就在于，所有在容器内进行的操作对宿主机的Registry缓存都不会产生任何影响。当我们完成工作从容器退出后，这些缓存在本地的Image和Tag都将不复存在，从而留给了宿主机一个非常干净的空间。

好了，有关Docker Registry CLI的使用就先介绍到这里了。在下一篇文章中，我们将进一步介绍如何从多个Registry批量复制Image到目标Registry，更加灵活丰富的查询功能，以及如何清理Registry上的Image。

关于Docker Registry CLI，如果你有任何疑问，或者希望通过贡献代码来增强这一工具，欢迎发送邮件至`morningspace@yahoo.com`。更多详情请访问[GitHub库](https://github.com/morningspace/docker-registry-cli)及[晴耕工坊](/studio)的主页。

---
*Have fun!*

*MorningSpace Studio*