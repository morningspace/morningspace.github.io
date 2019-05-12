---
permalink: /studio/
title: "晴耕工坊"
toc: true
toc_sticky: true
excerpt: "收录公开的源码小品"
---

## [Docker Registry CLI](https://github.com/morningspace/docker-registry-cli)

一款用Bash Shell编写的命令行工具，可以方便而灵活的对支持[HTTP API V2](https://docs.docker.com/registry/spec/api/)版本的[Docker Registry](https://docs.docker.com/registry/)进行管理，同时支持私有Docker Registry和各种公有的Docker Registry，如大家所熟知的[Docker Hub](https://hub.docker.com/)，[Quay](https://quay.io/)，[Google Container Registry](https://cloud.google.com/container-registry/)，[AWS Container Registry](https://aws.amazon.com/ecr/)等。该工具具备了以下几个特色：
* 批量复制：作为该工具的一大特色，利用其`cp`命令，可以从多个公有及私有Registry批量复制Image，轻松搭建起属于自己的Docker Registry；
* 基于DIND：提供基于[Docker-in-Docker(DIND)](https://github.com/jpetazzo/dind)的运行模式，支持容器内执行docker命令，所有操作对宿主机Registry缓存均不会造成任何影响；
* 强大灵活：只通过少数几个命令及其参数的组合，就完成了针对Image，Tag，Digest，Manifest在内的各种查询，以及Image的复制删除操作；
* 简单易学：由于在设计上极大程度的借鉴了Linux常用命令的语法，因而很容易上手。只要了解`cp`, `ls`, `rm`等命令，就可以很快掌握使用方法；

| 资源 	| 链接
| ---- 	|:----
| 源码 | [GitHub项目](https://github.com/morningspace/docker-registry-cli)
| Docker Registry CLI轻松管理Docker注册表(上) | [文章](/tech/use-docker-reg-cli-1/)
| Docker Registry CLI轻松管理Docker注册表(下) | 文章
| Docker Registry CLI Tutorial - Basic Use    | 文章 视频1 视频2
| Docker Registry CLI Tutorial - More Use | 文章 视频1 视频2

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
| 源码 | [GitHub项目](https://github.com/morningspace/elastic-shell)
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
