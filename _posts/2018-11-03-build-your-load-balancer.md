---
title: 自己动手玩转负载均衡
excerpt: "手把手，全方位教会你如何在本机搭建高可用的负载均衡集群服务！"
categories: [tech]
tags: [haproxy, keepalived, loadbalancing, training]
toc: true
toc_sticky: true
---

> 透过4段系列视频，手把手，全方位教会你如何搭建高可用的负载均衡服务
>
> 利用Docker镜像，提供身临其境的实验环境，所有实验内容只需一台机器即可轻松搞定

![](/assets/images/lab/lb.png)

今天向大家推荐的，是[“晴耕实验室”](/lab)新近推出的教学实验项目——[HAProxy+Keepalived:自己动手玩转负载均衡](/lab/#haproxykeepalived自己动手玩转负载均衡)。该项目计划将分四期推出系列教学视频，向大家介绍如何利用HAProxy结合Keepalived搭建具有高可用性的负载均衡服务。

## 我能学到什么？

期间，各位将随着教程一步步学会：
* 如何通过不同途径安装HAProxy
* 利用HAProxy实现基本的负载均衡
* 如何配置HAProxy，让其支持SSL
* 为HAProxy配备健康检查（Health Check）
* 利用Keepalived为HAProxy保驾护航，解决单点故障
* 如何对负载均衡服务进行调试

由于这一教程重点强调的是动手实践，而非理论知识，因此，对于希望能够透过实际操作来领会负载均衡服务搭建过程的同学而言，应该是一份相当不错的福利！强烈建议各位可以边看视频，边做实验。等看完全部视频以后，一个具备高可用性的负载均衡服务也就应运而生了！

## 全部实验只需一台机器！

由于通常负载均衡会涉及到多台服务器之间的协作，因此需要多台机器才能完成一个完整的实验。不过由于使用了Docker技术，现在我们可以在本机上安装并完成教学实验项目中所涉及的全部实验了！

这其中包括：
* 多台基于nginx的Web服务器，作为提供业务功能的后端服务
* 多台负载均衡服务器，同时安装了HAProxy和Keepalived

所有这些，都被包装成了Docker容器，透过在本机构筑的虚拟网络，实现相互连接！

届时，我们将通过[GitHub库](https://github.com/morningspace/lab-load-balancing)，将该实验环境所需的Docker镜像对应的Dockerfile以及相关的配置示例，连同说明文档，以开放源码的形式公布出来。欢迎各位关注该[GitHub库](https://github.com/morningspace/lab-load-balancing)。

在本系列教学视频的最后部分，我们还将专门推出一集约30分钟的视频，向各位介绍：如何利用这套基于Docker镜像的实验环境，进行HAProxy和Keepalived的相关实验。是不是很期待呢？

## 关于题目

本教学实验项目的全称为“HAProxy+Keepalived:自己动手玩转负载均衡”，英文名为“HAProxy + Keepalived: Build Your Load Balancer in 30 Minutes!”。

如前所述，因为全部教程内容都是着眼于动手实践的，因此取名“自己动手玩转”应该是再贴切不过了。

而至于“in 30 minutes”，笔者在制作视频的过程中注意到，根据规划的几个议题所制作出来的视频，恰好都在30分钟以内，所以就以此为名，也算是与题目相呼应了:-)

Have fun!