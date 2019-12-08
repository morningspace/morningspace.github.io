---
title: OpenShift v4的填坑之旅
excerpt: 记录All-in-One K8S Playground支持OpenShift v4沿途踩过的坑
categories: [tech]
tags: [lab, dummies, dummies_kubernetes, kubernetes, openshift]
toc: true
toc_sticky: true
---

<!-- Place this tag in your head or just before your close body tag. -->
<script async defer src="https://buttons.github.io/buttons.js"></script>

> 开源项目[lab-k8s-playground](https://github.com/morningspace/lab-k8s-playground) v1.3近期发布了，这次为大家带来的，是All-in-One K8S Playground针对OpenShift最新版本v4的支持！本文总结了在支持OpenShift v4过程中遇到的一系列问题。如果您喜欢这个项目，欢迎点击下面的按钮，关注，加星，以及贡献代码^_^

<a class="github-button" href="https://github.com/morningspace/lab-k8s-playground/subscription" data-icon="octicon-eye" data-size="large" aria-label="Watch morningspace/lab-k8s-playground on GitHub">Watch</a> <a class="github-button" href="https://github.com/morningspace/lab-k8s-playground" data-icon="octicon-star" data-size="large" aria-label="Star morningspace/lab-k8s-playground on GitHub">Star</a> <a class="github-button" href="https://github.com/morningspace/lab-k8s-playground/fork" data-icon="octicon-repo-forked" data-size="large" aria-label="Fork morningspace/lab-k8s-playground on GitHub">Fork</a>

## OpenShift

![](/assets/images/lab/k8s/openshift.png){: .align-center}

[OpenShift](https://www.openshift.com/)是Red Hat开发的构筑于Kubernetes之上的企业级容器应用平台，为企业应用与服务的开发，管理，以及部署提供了全面的支持，是当前炙手可热的一款Kubernetes发行版本。其社区版称为[OKD](https://www.okd.io/)（OpenShift Kubernetes Distribution的缩写，原名Origin）。

OpenShift v4是目前OpenShift的最新版本，和v3相比，新版本从功能，架构，到实现，都有非常巨大的变化，当然这也包括安装方式的变化。从v4开始，OpenShift不再支持`oc cluster up`方式的单节点集群部署，而是开发了专门的[Installer](https://github.com/openshift/installer)，并在此基础上提供了一种在本机部署的单节点集群安装方式，叫做[CodeReady Containers](https://github.com/code-ready/crc)（简称为crc）。

在[All-in-One K8S Playground支持OpenShift v4](/tech/all-in-one-openshift-playground/)一文中，我们已经了解了，如何利用[All-in-One Kubernetes Playground](https://github.com/morningspace/lab-k8s-playground/)，在单机环境下部署一个OpenShift v4的单节点集群，以及如何在上面部署[Istio](https://istio.io)。

本文将告诉你，All-in-One K8S Playground在支持OpenShift v4过程中遇到的一些问题，以及相应的解决办法。这其中，大部分问题的解决办法都已经通过脚本的形式固化在了All-in-One K8S Playground里，这也是该Playground的价值所在！

## 坑#1：OpenShift的版本选择

目前的crc，我们是没有办法自主选择OpenShift版本的。当下载了某个crc的build以后，我们只能使用和这个crc的build相绑定的OpenShift。并且，由于kubelet与集群中master节点通信所用的证书是有期限的（目前是30天），所以某个新出的crc build，在证书过期之后就无法再用它来启动集群了。此时，我们就需要更换新的crc build了。这是目前crc在使用方面最大的问题之一。

有关这一问题的详细讨论，参见：

| 网络资源
| ----
| [Enabling OpenShift 4 Clusters to Stop and Resume Cluster VMs](https://blog.openshift.com/enabling-openshift-4-clusters-to-stop-and-resume-cluster-vms/)
| [TLS errors due to expired kubelet certificates after node was shutdown](https://bugzilla.redhat.com/show_bug.cgi?id=1693951)
| [Retrigger certificate rotation/generation for internal cluster communication](https://github.com/code-ready/crc/issues/11)
| [Can CRC support specifying particular OpenShift version](https://github.com/code-ready/crc/issues/697)

## 坑#2：远程访问OpenShift

这是目前限制crc使用的另一个主要问题。由于crc是在宿主机上跑着的一个虚拟机里运行OpenShift的，它所使用的网络是一个只在当前宿主机范围内可见的本地局域网，所以通常我们只能在本机对OpenShift集群，以及集群里的应用进行访问。如果希望从另一台机器上远程访问这个集群，目前可行的一种办法是使用代理服务器。

> 在Playground里，我们预装了nginx，通过它可以远程访问OpenShift集群。详细用法请见[All-in-One K8S Playground支持OpenShift v4](/tech/all-in-one-openshift-v4-playground)。

并且，对于OpenShift v4的Web Console，由于crc限定了其访问的域名必须是`console-openshift-console.apps-crc.testing`，所以除了使用代理外，我们还需要在`/etc/hosts`里加上从集群所在机器的IP到固定域名的映射。

关于这一点，原理上和crc类似的[MiniShift](https://github.com/minishift/minishift/)，也有同样的[问题](https://github.com/minishift/minishift/issues/1287)。

有关这一问题的详细讨论，参见：

| 网络资源
| ----
| [Can I connect to minishift from another pc/laptop?](https://github.com/minishift/minishift/issues/1287)
| [How to access CRC or apps deployed on CRC remotely](https://github.com/code-ready/crc/issues/705)

## 坑#3：Persistent Volume的使用

如前所述，crc不是直接在宿主机上运行OpenShift的，而是在宿主机上的一个虚机里运行OpenShift。所以，对于Persistent Volume的使用需要注意。crc为使用者预先提供了hostPath类型的Persistent Volume，位于虚机内的`/mnt/pv-data`目录下。通过kubectl命令可以看到这些PV：
```shell
$ kubectl get pv
```

如果这些PV不能满足你的需求，也许你还需要进入虚机内部进行某些操作，比如：为这些PV所对应的目录赋足够的权限，或者创建新的PV目录。虽然，直接进入虚机进行操作是不被官方推荐的；不过，如果你勇于探索，敢于尝鲜的话，也可以试一下下面的命令：
```shell
$ ssh -i ~/.crc/machines/crc/id_rsa core@$(crc ip)
```

利用保存在当前用户主目录下`.crc`子目录的`id_rsa`，我们可以通过SSH的方式以用户`core`登录虚机。

相比而言，[MiniShift](https://github.com/minishift/minishift/)在这方面提供了更加方便的功能，它提供了专门的命令: `minishift ssh`，可以很方便地登录到虚机里。[这里](https://developers.redhat.com/blog/2017/04/05/adding-persistent-storage-to-minishift-cdk-3-in-minutes/)有一篇文章就是介绍如何进入MiniShift虚机，手工初始化Persistent Volume对应的目录的。

有关这一问题的详细讨论，参见：

| 网络资源
| ----
| [Pods unable to write to persistent volumes](https://github.com/minishift/minishift/issues/856)
| [How does CRC handle persistent volume e.g. local or hostPath](https://github.com/code-ready/crc/issues/728)

## 坑#4：CNI插件的安装和配置路径

在往OpenShift v4上部署Istio以及Istio的示例应用Bookinfo时，部署过程一切都很顺利，唯独在查看Jaeger UI时，发现只有`istio-ingressgateway`的trace数据，而没有其他几个Bookinfo应用的数据。这说明，跑在Bookinfo的那几个Pod里的sidecar proxy，并没有把trace数据成功地发送出来。

以往，Istio都是通过`proxy-init`容器，在Pod的初始化阶段，为Pod配置好相应的`iptables`，从而把所有进入Pod的网络流量都重新定向到sidecar proxy，实现了对网络流量的监控。这种方法的一个缺点是，它要求部署Pod的用户或者Service Account必须具有足够的权限（NET_ADMIN），而这对于有严格安全要求的用户而言可能是不允许的。

不过，OpenShift从v4开始用`nftables`代替了`iptables`。所以，我们无法再用`proxy-init`配置Pod的`iptables`了，取而代之的是Istio的[CNI插件](https://github.com/istio/cni)。并且，使用插件不需要NET_ADMIN权限，更加方便。关于部署插件的具体方法，参见[Istio的文档](https://istio.io/docs/setup/additional-setup/cni/)。我们的Playground在针对OpenShift v4部署Istio时，就是用的CNI插件。

通过SSH登录crc所在的虚机后，查看`/etc/crio/crio.conf`文件：
```shell
# network_dir is is where CNI network configuration`
# files are stored.  Note this default is changed from the RPM.
network_dir = "/etc/kubernetes/cni/net.d/"

# plugin_dir is is where CNI plugin binaries are stored.
# Note this default is changed from the RPM.
plugin_dir = "/var/lib/cni/bin"
```

会发现，CNI插件的二进制文件所在路径，以及配置文件所在路径用的都不是[CRI-O](https://github.com/cri-o/cri-o)的默认值。CRI-O的默认值分别是：
```shell
network_dir = "/etc/cni/net.d/"
plugin_dirs = "/opt/cni/bin/"
```

> OpenShift v4使用CRI-O做为容器运行时，完全取代了Docker。其所使用的操作系统，也统一换成了[CoreOS](https://coreos.com/)。

而默认情况下，Istio在安装CNI插件时，会使用默认路径进行安装，从而导致CRI-O没有成功检测到Istio的CNI插件，使得重定向到sidecar的路由没有被成功建立起来。这就可以解释为什么在Jager UI上看不到Bookinfo那几个应用的trace数据了。解决的方法，是在使用Helm部署Istio的CNI插件时，通过参数告诉Helm，安装CNI插件的正确路径。

> 这个问题的解决方法，已经被加入到Playground里安装Istio所用的脚本里面了。

至于OpenShift为什么要调整CRI-O有关CNI插件的安装和配置路径，可以参考这两个PR：[kubelet: enable crio runtime](https://github.com/openshift/installer/pull/235)和[use default cni paths](https://github.com/openshift/installer/pull/250)。主要的原因是，OpenShift有自己的CNI插件，而CRI-O默认也提供了一系列插件，两者在容器系统对网络进行初始化时会发生冲突。为了完全接管网络，OpenShift就修改了默认路径，让安装和配置路径都指向了它自己的CNI插件。

有关这一问题的详细讨论，参见：

| 网络资源
| ----
| [Only see istio-ingressgateway traces ... when deploy Istio 1.3.2 on Openshift ...](https://github.com/istio/istio/issues/17795)
| [Where is the default CNI plugin bin and conf folders overridden](https://github.com/code-ready/crc/issues/721)

## 坑#5：MacOS上/etc/hosts的权限

在MacOS上启动OpenShift v4时，crc会修改`/etc/hosts`文件，以实现针对集群的网络配置。通常，在MacOS上只有`root`权限才有权修改这个文件，而我们日常所用的Mac登录账号都是`non-root`权限的。所以在修改`/etc/hosts`之前，crc还会修改`/etc/hosts`文件的访问权限，把默认的访问权限：
```shell
$ ls -l /etc/hosts
-rw-r--r--  1 root  wheel  2094 Oct 23 14:41 /etc/hosts
```
改成：
```shell
$ ls -l /etc/hosts
-rw-------  1 morningspace  wheel  2094 Oct 23 13:51 /etc/hosts
```

这里，`morningspace`是我登录Mac所用的账号。否则，在启动OpenShift时会报下面的错：
```shell
FATA /etc/hosts is not readable/writable by the current user 
```

> 在Linux环境下，由于crc是用NetworkManager来实现网络配置的，所以不存在修改`/etc/hosts`的问题

不过，目前crc对`/etc/hosts`的权限修改是不会自己Undo的。比如，在关闭并清除OpenShift集群的时候，crc不会把文件权限恢复到原来的默认状态。

> 在Playground里，我们会在执行`launch kubernetes::clean`时，用脚本恢复`/etc/hosts`文件的默认权限。

有关这一问题的详细讨论，参见：

| 网络资源
| ----
| [macOS FATA /etc/hosts is not readable/writable by the current user](https://github.com/code-ready/crc/issues/618)
| [Does crc undo the file ownership & mode changes made to /etc/hosts ...?](https://github.com/code-ready/crc/issues/683)
| [RFE: crc cleanup](https://github.com/code-ready/crc/issues/507)

## 更多参考资料

读到这里，相信你对开源项目[lab-k8s-playground](https://github.com/morningspace/lab-k8s-playground)及其All-in-One Playground的应用应该有了更全面的了解了。那么，还有哪些参考资料你可以访问呢？那就请快速读一下下面这段Shell脚本编写的“代码”吧😊 相信一定难不倒你！

选择你感兴趣的分支，点击相应分支下列出的文档资源。最后，如果您喜欢这个项目，欢迎点击本文开头处（或下列代码注释里）的按钮，关注，加星，以及贡献代码^_^

<p style="background-color: #111; padding: 1em; text-align: left; font-size: 20px">
<code style="outline: none; box-sizing: border-box; font-family: Monaco, Consolas, &quot;Lucida Console&quot;, monospace; background-color: #111">
<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(173, 0, 161);">function<span>&nbsp;</span></span>where_to_start<span>&nbsp;</span><span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(31, 170, 170);">{</span><br style="outline: none;">
&nbsp;&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(173, 0, 161);">case</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(255, 0, 134);">$I_want</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(173, 0, 161);">in</span><br style="outline: none;">
&nbsp;&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(0, 201, 24);">"to learn what it provides in general"</span><span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">)</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://morningspace.github.io/lab-k8s-playground/docs/All-in-One-Playground-Overview.html" rel="noopener" target="_blank">All-in-One&nbsp;K8S&nbsp;Playground&nbsp;Overview</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">;;</span><br style="outline: none;">
&nbsp;&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(0, 201, 24);">"to get started quickly"</span><span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">)</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://morningspace.github.io/lab-k8s-playground/docs/All-in-One-Playground-Usage-Guide.html" rel="noopener" target="_blank">All-in-One&nbsp;K8S&nbsp;Playground&nbsp;Usage&nbsp;Guide</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="http://localhost:4000/tech/all-in-one-k8s-playground" rel="noopener" target="_blank">All-in-One&nbsp;K8S&nbsp;Playground 中文使用指南</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">;;</span><br style="outline: none;">
&nbsp;&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(0, 201, 24);">"to understand what's behind"</span><span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">)</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://morningspace.github.io/tech/k8s-run" rel="noopener" target="_blank">Launch&nbsp;multi-node&nbsp;k8s&nbsp;cluster&nbsp;locally&nbsp;in&nbsp;one&nbsp;min...</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://v.youku.com/v_show/id_XNDI2Mzk1NDcyMA==.html?f=52221532" rel="noopener" target="_blank">Kubernetes实战视频:&nbsp;如何1分钟内在本地启动一个多节点集群</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://morningspace.github.io/tech/openshift-v3-trap-and-pitfalls" rel="noopener" target="_blank">OpenShift v3的填坑之旅</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://morningspace.github.io/tech/openshift-v4-trap-and-pitfalls" rel="noopener" target="_blank">OpenShift v4的填坑之旅</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">;;</span><br style="outline: none;">
&nbsp;&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(0, 201, 24);">"to extend it by myself"</span><span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">)</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://morningspace.github.io/lab-k8s-playground/docs/Launch-Utility-Usage-Guide.html" rel="noopener" target="_blank">Launch&nbsp;Utility&nbsp;Usage&nbsp;Guide</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://github.com/morningspace/lab-k8s-playground/blob/master/install/targets/sample.sh" rel="noopener" target="_blank">A Sample Target for Demo Purpose</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">;;</span><br style="outline: none;">
&nbsp;&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(0, 201, 24);">"to learn more cool features"</span><span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">)</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://morningspace.github.io/lab-k8s-playground/docs/APIC-Quick-Guide.html" rel="noopener" target="_blank">Quick&nbsp;Guide&nbsp;to&nbsp;Launch&nbsp;APIC&nbsp;Playground</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://morningspace.github.io/tech/all-in-one-apic-playground/" rel="noopener" target="_blank">把API Connect关进All-in-One K8S Playground里</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://morningspace.github.io/tech/all-in-one-openshift-playground/" rel="noopener" target="_blank">All-in-One K8S Playground新增OpenShift支持</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://morningspace.github.io/tech/all-in-one-openshift-v4-playground/" rel="noopener" target="_blank">All-in-One K8S Playground支持OpenShift v4</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">;;</span><br style="outline: none;">
&nbsp;&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(0, 201, 24);">"to contribute back"</span><span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">)</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(119, 119, 119);"># Feel free to click these buttons: <a class="github-button" href="https://github.com/morningspace/lab-k8s-playground/subscription" data-icon="octicon-eye" aria-label="Watch morningspace/lab-k8s-playground on GitHub">Watch</a> <a class="github-button" href="https://github.com/morningspace/lab-k8s-playground" data-icon="octicon-star" aria-label="Star morningspace/lab-k8s-playground on GitHub">Star</a> <a class="github-button" href="https://github.com/morningspace/lab-k8s-playground/fork" data-icon="octicon-repo-forked" aria-label="Fork morningspace/lab-k8s-playground on GitHub">Fork</a></span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;git clone<span>&nbsp;</span><a href="https://github.com/morningspace/lab-k8s-playground.git" rel="noopener" target="_blank">https://github.com/morningspace/lab-k8s-playground.git</a><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">;;</span><br style="outline: none;">
&nbsp;&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(173, 0, 161);">esac</span><br style="outline: none;">
<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(31, 170, 170);">}</span></code>
</p>
