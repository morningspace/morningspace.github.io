---
title: 把API Connect关进All-in-One K8S Playground里
excerpt: 把API Connect这个“庞然大物”关进All-in-One K8S Playground里需要几步呢？
categories: [tech]
tags: [lab, dummies, dummies_kubernetes, kubernetes, apic]
toc: true
toc_sticky: true
---

<!-- Place this tag in your head or just before your close body tag. -->
<script async defer src="https://buttons.github.io/buttons.js"></script>

> 众所周知，把大象关进冰箱里统共需要三步。那么把API Connect这个“庞然大物”关进All-in-One Kubernetes Playground里，到底需要几步呢？本文是开源项目[lab-k8s-playground](https://github.com/morningspace/lab-k8s-playground)的高级应用案例。如果您喜欢这个项目，欢迎点击下面的按钮，关注，加星，以及贡献代码^_^

<a class="github-button" href="https://github.com/morningspace/lab-k8s-playground/subscription" data-icon="octicon-eye" data-size="large" aria-label="Watch morningspace/lab-k8s-playground on GitHub">Watch</a> <a class="github-button" href="https://github.com/morningspace/lab-k8s-playground" data-icon="octicon-star" data-size="large" aria-label="Star morningspace/lab-k8s-playground on GitHub">Star</a> <a class="github-button" href="https://github.com/morningspace/lab-k8s-playground/fork" data-icon="octicon-repo-forked" data-size="large" aria-label="Fork morningspace/lab-k8s-playground on GitHub">Fork</a>

## API Connect

[IBM API Connect(APIC)](https://developer.ibm.com/apiconnect)是IBM开发的一款API生命周期管理软件，它的核心部分是基于Node.js开源框架[LoopBack](https://loopback.io/)构建而成的。如果你对LoopBack感兴趣，欢迎访问我的[深入浅出LoopBack](https://morningspace.github.io/lab/#%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BAloopback)系列教程。

标准的APIC安装，对于包括CPU，内存，存储在内的系统资源要求都比较高，通常需要多个虚拟机或物理机才能完成部署；其安装过程也比较复杂，通常需要几个小时甚至几天的时间才能完成安装，详情可见它的[安装文档](https://www.ibm.com/support/knowledgecenter/SSMNED_2018/mapfiles/getting_started.html)。

本文将告诉你，利用[All-in-One Kubernetes Playground](https://github.com/morningspace/lab-k8s-playground/)，只要五个步骤，就可以在单机环境下，把APIC部署到一个由多节点构成的标准[Kubernetes](https://kubernetes.io/)集群里。整个过程是完全自动化进行的，并且可以无限次重复！

## 动图演示

先上一段动图演示：

![](https://morningspace.github.io/lab-k8s-playground/docs/demo-apic.gif)

然后再来看一下具体的步骤。

## 第一步：初始化环境

首先，我们把All-in-One Kubernetes Playground的Git库克隆到本地，进入项目的根目录，执行`init`命令对Playground环境进行初始化：
```shell
$ git clone https://github.com/morningspace/lab-k8s-playground.git
$ cd lab-k8s-playground
$ ./install/launch.sh init
```

然后，打开`~/.bashrc`文件，根据需要对某些环境变量进行调整，比如：指定本机的IP地址，Kubernetes的版本，集群的节点个数等：
```shell
# The IP of your host that runs APIC
export HOST_IP=<your_host_ip>
# The Kubernetes version, default is v1.14
export K8S_VERSION=
# The number of worker nodes, must be 3
export NUM_NODES=3
```

为了在当前登录终端里让上面的改动生效，我们需要执行下面的命令重新加载`.bashrc`：
```shell
$ . ~/.bashrc
```

> 安装了APIC的Playground可以在包括Ubuntu，CentOS，RHEL在内的多个操作系统上运行。

## 第二步：准备安装包

下载APIC的安装包，并保存到`$LAB_HOME/install/.launch-cache/apic`目录下，包括安装APIC所需的全部Docker镜像，以及`apicup`可执行文件，例如：
```shell
$ ls -1 $LAB_HOME/install/.launch-cache/apic/
analytics-images-kubernetes_lts_v2018.4.1.4.tgz
apicup-linux_lts_v2018.4.1.4
idg_dk2018414.lts.nonprod.tar.gz
management-images-kubernetes_lts_v2018.4.1.4.tgz
portal-images-kubernetes_lts_v2018.4.1.4.tgz
```

> 这里，`$LAB_HOME`代表Playground项目的根目录。

> APIC的安装包可以从IBM的官方下载站点获取到，具体方法此处略。

## 第三步：修改部署配置

我们可以根据需要，在下面的文件里，对APIC各个服务所使用的主机名，以及其他相关设置进行定制：
```shell
$ vi $LAB_HOME/install/targets/apic/settings.sh
```

当在Playground里成功完成第一次APIC安装以后，我们可以把`settings.sh`里的`apic_skip_load_images`设置为`1`，这样可以跳过APIC的安装过程中“把Docker镜像加载到本地Private Registry”的步骤。因为，这一步骤在完成第一次APIC的安装以后就不需要再执行了，这可以为APIC的安装部署节省很多时间。

如果你对如何自定义APIC的配置感兴趣，可参考[附录：自定义APIC配置](#附录自定义apic配置)

## 第四步：启动APIC

执行如下命令启动Kubernetes：
```shell
$ launch default
```

> 等Kubernetes启动完以后，我们可以执行`kubectl version`来验证启动是否成功。

> 如果你是中国地区的用户，请在执行`launch`之前设置环境变量`export IS_IN_CHINA=1`，这样可以解决Kubernetes启动过程中无法从Google网站下载Docker镜像的问题。

然后，启动APIC：
```shell
$ launch apic
```

完成启动过程需要花费一定的时间，具体取决于你的系统配置。以我的虚机为例，在`apic_skip_load_images`设置为1的情况下，大概不到15分钟就完成了。

> 等APIC启动起来以后，可以执行`kubectl get pods -n apiconnect`，检查APIC的所有pod是否都已经成功的启动起来了。

如果我们想销毁当前的APIC环境，再重新启动一个新的集群，只需要执行下面这一行命令就可以了：
```shell
$ launch apic::clean kubernetes::clean registry::up kubernetes apic
```

## 第五步：暴露APIC端口

可以利用下面的命令把APIC的服务端口暴露到集群外：
```shell
$ launch apic::portforward
```

然后，我们再把主机IP和主机名的映射定义添加到本地的`/etc/hosts`文件里，这样就可以在本地的浏览器里访问APIC的UI了：
```shell
$ cat /etc/hosts
...
<your_host_ip>   cm.morningspace.com gwd.morningspace.com gw.morningspace.com padmin.morningspace.com portal.morningspace.com ac.morningspace.com apim.morningspace.com api.morningspace.com
```

## 附录：自定义APIC配置

所有APIC的配置文件都保存在`$LAB_HOME/install/targets/apic`目录下。其中，`settings.sh`文件用于配置APIC各个服务所用的主机名，以及所需内存和存储的大小。

当完成自定义以后，我们可以执行下面的命令，让APIC对我们所做的修改进行验证：
```shell
$ launch apic::validate
```

在输出结果里，大家可能会注意到，有些地方会有验证错误，比如：
```shell
data-storage-size-gb  10    ✘  data-storage-size-gb must be 200 or greater 
```

这表示，APIC的`analytics`子系统所使用的`data-storage-size-gb`必须要大于等于200GB。如果你觉得自己所指定的值，在当前环境下足以运行APIC的话，那就可以忽略这些错误。

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
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://morningspace.github.io/tech/all-in-one-k8s-playground" rel="noopener" target="_blank">All-in-One&nbsp;K8S&nbsp;Playground 中文使用指南</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">;;</span><br style="outline: none;">
&nbsp;&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(0, 201, 24);">"to understand what's behind"</span><span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">)</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://morningspace.github.io/tech/k8s-run" rel="noopener" target="_blank">Launch&nbsp;multi-node&nbsp;k8s&nbsp;cluster&nbsp;locally&nbsp;in&nbsp;one&nbsp;minute...</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://v.youku.com/v_show/id_XNDI2Mzk1NDcyMA==.html?f=52221532" rel="noopener" target="_blank">视频教程 - Kubernetes实战:&nbsp;如何1分钟内在本地启动一个多节点集群</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">;;</span><br style="outline: none;">
&nbsp;&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(0, 201, 24);">"to extend it by myself"</span><span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">)</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://morningspace.github.io/lab-k8s-playground/docs/Launch-Utility-Usage-Guide.html" rel="noopener" target="_blank">Launch&nbsp;Utility&nbsp;Usage&nbsp;Guide</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://github.com/morningspace/lab-k8s-playground/blob/master/install/targets/sample.sh" rel="noopener" target="_blank">A Sample Target for Demo Purpose</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">;;</span><br style="outline: none;">
&nbsp;&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(0, 201, 24);">"to learn more cool features"</span><span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">)</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://morningspace.github.io/lab-k8s-playground/docs/APIC-Quick-Guide.html" rel="noopener" target="_blank">Quick&nbsp;Guide&nbsp;to&nbsp;Launch&nbsp;APIC&nbsp;Playground</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://morningspace.github.io/tech/all-in-one-apic-playground" rel="noopener" target="_blank">把API Connect关进All-in-One K8S Playground里</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">;;</span><br style="outline: none;">
&nbsp;&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(0, 201, 24);">"to contribute back"</span><span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">)</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(119, 119, 119);"># Feel free to click these buttons: <a class="github-button" href="https://github.com/morningspace/lab-k8s-playground/subscription" data-icon="octicon-eye" aria-label="Watch morningspace/lab-k8s-playground on GitHub">Watch</a> <a class="github-button" href="https://github.com/morningspace/lab-k8s-playground" data-icon="octicon-star" aria-label="Star morningspace/lab-k8s-playground on GitHub">Star</a> <a class="github-button" href="https://github.com/morningspace/lab-k8s-playground/fork" data-icon="octicon-repo-forked" aria-label="Fork morningspace/lab-k8s-playground on GitHub">Fork</a></span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;git clone<span>&nbsp;</span><a href="https://github.com/morningspace/lab-k8s-playground.git" rel="noopener" target="_blank">https://github.com/morningspace/lab-k8s-playground.git</a><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">;;</span><br style="outline: none;">
&nbsp;&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(173, 0, 161);">esac</span><br style="outline: none;">
<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(31, 170, 170);">}</span></code>
</p>
