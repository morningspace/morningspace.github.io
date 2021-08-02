---
permalink: /studio/
title: "晴耕工坊"
toc: true
toc_sticky: true
excerpt: "收录公开的源码小品"
---

## [Enhanced OpenShift CLI](https://github.com/morningspace/oc)

增强版OpenShift CLI可以帮助我们高效而安全的管理大量的OpenShift集群。

| 资源 		| 链接
| ---- 		|:----
| 源码			| [GitHub项目](https://github.com/morningspace/oc)
| 在线文档	| [Online Documentation](https://morningspace.github.io/oc/docs)
| 如何高效安全的管理OpenShift集群			| [文章](/tech/enhanced-oc-1/)
| 增强版OpenShift CLI快速入门				| [文章](/tech/enhanced-oc-2/)
| 增强版OpenShift CLI的更多功能			| [文章](/tech/enhanced-oc-3/)

## [KubeAssert](https://github.com/morningspace/kubeassert)

KubeAssert是一个kubectl插件，它用于在命令行声明针对Kubernetes资源的断言（assertion）。

| 资源 		| 链接
| ---- 		|:----
| 源码			| [GitHub项目](https://github.com/morningspace/kubeassert)
| 在线文档	| [Online Documentation](https://morningspace.github.io/kubeassert/docs)
| KubeAssert快速入门								| [文章](/tech/kubeassert-1/)
| 高效使用KubeAssert								| [文章](/tech/kubeassert-2/)
| 为KubeAssert定制断言							| [文章](/tech/kubeassert-3/)
| 集成KubeAssert与KUTTL						| [文章](/tech/kubeassert-4/)

## [KubeMacro](https://github.com/morningspace/kubemacro)

KubeMacro是一个kubectl插件，它用于把一组针对于kubectl命令或Kubernetes API的调用封装成一个命令，以便于在命令行多次执行。

| 资源 		| 链接
| ---- 		|:----
| 源码			| [GitHub项目](https://github.com/morningspace/kubemacro)
| 在线文档	| [KubeMacro Online Documentation](https://morningspace.github.io/kubemacro/docs)
| KubeMacro快速入门								| [文章](/tech/kubemacro-1/)
| KubeMacro Hub：寻找你的Kube Macro	| [文章](/tech/kubemacro-2/)
| 编写自己的Kube Macro							| [文章](/tech/kubemacro-3/)

## [Git Assist](https://github.com/morningspace/git-assist)

提供了一组命令行工具，在你日常使用Git和GitHub的过程中，当遇到一些复杂应用场景，无法简单的通过使用Git图形化工具，或一两条git命令来解决的时候，Git Assist可以帮助你处理这些情况。比如：
* 把一组文件或文件夹连同其提交历史从一个Git库复制到另一个Git库；
* 把一组文件连同其提交历史彻底从当前Git库里清除掉；
* 把Git库里所有之前用某个账号提交的历史记录批量替换成另一个账号；
* 在把Git库里的提交记录推送到远程库时，指定只推送一部分提交记录；

| 资源 		| 链接
| ---- 		|:----
| 源码			| [GitHub项目](https://github.com/morningspace/git-assist)
| 在线文档	| [Git Assist Online Documentation](https://morningspace.github.io/git-assist/docs)

## [Docker Registry CLI](https://github.com/morningspace/docker-registry-cli)

一款用Bash Shell编写的命令行工具，可以方便而灵活的对支持[HTTP API V2](https://docs.docker.com/registry/spec/api/)版本的[Docker Registry](https://docs.docker.com/registry/)进行管理，同时支持私有Docker Registry和各种公有的Docker Registry，如大家所熟知的[Docker Hub](https://hub.docker.com/)，[Quay](https://quay.io/)，[Google Container Registry](https://cloud.google.com/container-registry/)，[AWS Container Registry](https://aws.amazon.com/ecr/)等。该工具具备了以下几个特色：
* 批量复制：作为该工具的一大特色，利用其`cp`命令，可以从多个公有及私有Registry批量复制Image，轻松搭建起属于自己的Docker Registry；
* 基于DIND：提供基于[Docker-in-Docker(DIND)](https://github.com/jpetazzo/dind)的运行模式，支持容器内执行docker命令，所有操作对宿主机Registry缓存均不会造成任何影响；
* 强大灵活：只通过少数几个命令及其参数的组合，就完成了针对Image，Tag，Digest，Manifest在内的各种查询，以及Image的复制删除操作；
* 简单易学：由于在设计上极大程度的借鉴了Linux常用命令的语法，因而很容易上手。只要了解`cp`, `ls`, `rm`等命令，就可以很快掌握使用方法；

| 资源 	| 链接
| ---- 	|:----
| 源码 	| [GitHub项目](https://github.com/morningspace/docker-registry-cli)
| Docker Registry CLI轻松管理Docker注册表(上) | [文章](/tech/use-docker-reg-cli-1/)
| Docker Registry CLI轻松管理Docker注册表(下) | [文章](/tech/use-docker-reg-cli-2/)
| Docker Registry CLI Tutorial - Basic Use | [文章](/tech/reg-cli-tutorial-1/) [视频1](https://v.youku.com/v_show/id_XNDE3ODk0NTk4MA==.html?f=52175776 "优酷自频道") [视频2](https://youtu.be/j_Bd4G5aGXA "YouTube Channel")
| Docker Registry CLI Tutorial - More Use  | [文章](/tech/reg-cli-tutorial-2/) [视频1](https://v.youku.com/v_show/id_XNDE5Nzk2OTA3Mg==.html?f=52175776 "优酷自频道") [视频2](https://youtu.be/wySn3fvtmdE "YouTube Channel")

## [Elastic Shell](https://github.com/morningspace/elastic-shell)

* 一套完全由Bash Shell编写的，用于管理Elasticsearch搜索引擎的工具脚本；
* 提供针对index和snapshot的基本管理；
* 提供针对reindex和Elasticsearch集群升级的辅助自动化；
* 提供高阶辅助功能，比如：logging，dry run，auto completion等；
* 同时支持命令行与交互式两种运行模式；
* 交互模式下提供文本形态的图形化界面及菜单操作；
* 同时支持以Standalone方式运行，以及Docker容器内运行

| 资源 	| 链接
| ---- 	|:----
| 源码 	| [GitHub项目](https://github.com/morningspace/elastic-shell)
| 开源项目Elastic Shell发布暨晴耕工坊栏目上线 | [文章](/tech/elash-and-studio/)
| Elastic Shell 101: Getting Started | [文章](/tech/elash101-1/) [视频1](https://v.youku.com/v_show/id_XNDEwNjI0OTk2OA==.html?f=52133377 "优酷") [视频2](https://youtu.be/9r_RNz89SVw "YouTube")
| Elastic Shell 101: Working with Index | [文章](/tech/elash101-2/) [视频1](https://v.youku.com/v_show/id_XNDExNjc0OTU4NA==.html?f=52133377 "优酷") [视频2](https://youtu.be/nWX8miFbRPQ "YouTube")
| Elastic Shell 101: Manage Snapshot Interactively | [文章](/tech/elash101-3/) [视频1](https://v.youku.com/v_show/id_XNDEyODY2NTA3Mg==.html?f=52133377 "优酷") [视频2](https://youtu.be/_KwIkjoRQS8 "YouTube")
| Elastic Shell 101: Reindex Using Dialog | [文章](/tech/elash101-4/) [视频1](https://v.youku.com/v_show/id_XNDEzNTMwMTU5Ng==.html?f=52133377 "优酷") [视频2](https://youtu.be/ywgxY1h0PsA "YouTube")
| Elastic Shell 101: Advanced Features | [文章](/tech/elash101-5/) [视频1](https://v.youku.com/v_show/id_XNDE1NTQ1NjIwNA==.html?f=52133377 "优酷") [视频2](https://youtu.be/wc4CnChWxPE "YouTube")

## [VimRepress](https://github.com/morningspace/VimRepress)

* Vim插件，用于在Vim中编写WordPress文章，基于VimPress及VimRepress的改良版

## [WP Remote Site Sync](https://github.com/morningspace/wp-remote-site-sync)

* WordPress插件，用于在发布WordPress文章时，同步到远程WordPress站点
