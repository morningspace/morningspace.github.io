---
title: 极速搭建Kubernetes集群的新方案！
excerpt: "开源项目[kubeadm-dind-cluster](https://github.com/kubernetes-sigs/kubeadm-dind-cluster)与[lab-k8s-playground](https://github.com/morningspace/lab-k8s-playground)的引介"
categories: [tech, lab]
tags: [news, lab, kubernetes]
toc: true
toc_sticky: true
---

> 注意：本文的配套视频教程，可以在[YouTube](https://www.youtube.com/watch?v=0uVdF3Inv48&list=PLVQM6jLkNkfqHgd0aX7TnjioOiQrqsXIa)或[优酷](https://v.youku.com/v_show/id_XNDI2Mzk1NDcyMA==.html?f=52221532)上找到。

如何在一分钟内快速在本机搭建起一个多节点的Kubernetes集群环境，甚至是在断网的情况下？如何做到在不同的Kubernetes版本之间随意切换，甚至是基于Kubernetes源代码仓库的最新版本？晴耕小筑向你推荐一个有趣的解决方案，它就是开源项目[kubeadm-dind-cluster](https://github.com/kubernetes-sigs/kubeadm-dind-cluster)。更多详情请查看“[Launch multi-node Kubernetes cluster locally in one minute, and more…](/tech/k8s-run/)”一文。

![](/assets/images/lab/k8s/kubeadm-dind-cluster-1.png){: .align-center}

## kubeadm-dind-cluster开源项目

kubeadm-dind-cluster是Kubernetes社区下的一个子项目，利用kubeadm结合dind(Docker-in-Docker)技术，它用Docker容器模拟了Kubernetes集群中的物理节点，然后让原本跑在集群里的Pod跑在了Docker容器里，从而实现了在单机环境里搭建多节点集群的效果。

利用kubeadm-dind-cluster，我们可以很轻松地在本地进行基于多节点集群的Kubernetes开发，部署，测试，以及学习和实验。同时，我们也可以把它集成到CI/CD环境里，实现接近生产环境下的快速部署，测试，以及验证。

## 更多优化技巧，尽在晴耕小筑

而且，晴耕小筑也积极参与了kubeadm-dind-cluster项目的开发，对它进行了各种优化改进，“[Launch multi-node Kubernetes cluster locally in one minute, and more…](/tech/k8s-run/)”一文介绍的大多数内容就是对该项目的优化，并由晴耕小筑以源码方式贡献给了kubeadm-dind-cluster。比如：
* 加速Kubernetes集群的启动速度而启用私有Docker注册表；
* 允许指定Kubernetes Dashboard的任意一个版本进行安装；
* 允许跳过Kubernetes Dashboard的安装以加快集群的启动；

![](/assets/images/lab/k8s/kubeadm-dind-cluster-2.png)

## lab-k8s-playground实验项目

为了更好地演示kubeadm-dind-cluster的用法，充分发挥它的各种功能，包括晴耕小筑贡献的优化手段，晴耕实验室发布的实验项目[lab-k8s-playground](https://github.com/morningspace/lab-k8s-playground)也于同期和大家见面了。在阅读本文的同时，大家可以把lab-k8s-playground克隆到本地，它提供了对kubeadm-dind-cluster运行脚本的封装，以帮助大家更好的使用该工具。

```shell
$ git clone https://github.com/morningspace/lab-k8s-playground.git
```

未来，lab-k8s-playground还将会有更多惊喜和大家见面，欢迎各位关注，加星，以及分享！
