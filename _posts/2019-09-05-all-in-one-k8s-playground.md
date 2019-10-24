---
title: All-in-One K8S Playground中文使用指南
excerpt: 开源项目[lab-k8s-playground](https://github.com/morningspace/lab-k8s-playground)全新升级，一键搞定k8s部署
categories: [tech]
tags: [lab, dummies, dummies_kubernetes, kubernetes]
toc: true
toc_sticky: true
---

<!-- Place this tag in your head or just before your close body tag. -->
<script async defer src="https://buttons.github.io/buttons.js"></script>

> 本文是开源项目[lab-k8s-playground](https://github.com/morningspace/lab-k8s-playground)及其All-in-One Playground的中文使用指南！如果您喜欢这个项目，欢迎点击下面的按钮，关注，加星，以及贡献代码^_^

<a class="github-button" href="https://github.com/morningspace/lab-k8s-playground/subscription" data-icon="octicon-eye" data-size="large" aria-label="Watch morningspace/lab-k8s-playground on GitHub">Watch</a> <a class="github-button" href="https://github.com/morningspace/lab-k8s-playground" data-icon="octicon-star" data-size="large" aria-label="Star morningspace/lab-k8s-playground on GitHub">Star</a> <a class="github-button" href="https://github.com/morningspace/lab-k8s-playground/fork" data-icon="octicon-repo-forked" data-size="large" aria-label="Fork morningspace/lab-k8s-playground on GitHub">Fork</a>

在[Launch multi-node Kubernetes cluster locally in one minute and more](https://morningspace.github.io/tech/k8s-run/)一文里，我们提到过开源项目[lab-k8s-playground](https://github.com/morningspace/lab-k8s-playground)。它提供了一种用于在单机环境下部署多节点Kubernetes集群的方法，并在此基础上进行了多种优化，以加速集群的启动过程。有了这样一套环境，我们就可以高效地在本地进行各种和Kubernetes有关的学习，开发，部署，测试，以及演示了；或者，也可以把它放到CI/CD的Pipeline上，快速启动起一个用于测试的环境来。

现在，该项目又有了全新升级，那就是“All-in-One Playground”🎉🎉🎉利用它，我们可以一键搞定Kubernetes部署：通过把部署集群之前要做的很多复杂的准备工作固化在一套脚本里，我们就不用再关心集群部署的优化细节了。只要一行命令，就能完成集群部署。不仅如此，这个Playground环境里还预装了很多常用的Kubernetes命令行工具，并支持Istio等应用的追加部署。所有这些功能，都可以一键式搞定，并且可以反复卸载和安装！

是不是有点迫不及待地想体验一下这个Playground了呢？下面我们就一起来了解一下这个All-in-One Kubernetes Playground吧！

## 我都等不及啦！

首先，我们把lab-k8s-playground项目克隆到本地，并进入该项目的根目录：
```shell
$ git https://github.com/morningspace/lab-k8s-playground.git
```

### 在Vagrant Box里启动

启动Playground非常简单，如果你的机器安装了Vagrant Box，那么只要一条命令就行了：
```shell
$ vagrant up
```

> 整个启动过程大约需要一杯咖啡 ☕️ 的时间。具体取决于你的系统配置，以及分配给虚机的资源大小，还有网络环境等。以我的笔记本为例，8核CPU，16G内存，网络畅通的情况下，大概15分钟内完成。

在这个过程中，脚本会为我们完成所有的脏活累活，包括：
* 自动安装docker，docker compose，kubectrl，helm，以及Kubernetes的一些日常使用的命令行工具；
* 自动搭建Private Registry，用于保存集群部署用到的所有Docker镜像。利用Private Registry可以免去通过网络访问远程Public Registry的需要，从而极大地提高了部署的速度。并且，因为全部镜像都在本地，所以还可以实现离线安装，非常适合单机的demo环境。
* 最后，它还会为我们部署好一个默认由三个节点构成的Kubernetes集群，即：一个master节点加两个worker节点。这个节点数目是可以定制的，不仅如此，就连Kubernetes所用的版本也是可以指定的。关于怎么定制Playground，后面还会讲解。

### 在宿主机上启动

除了Vagrant Box，Playground也支持直接在宿主机上启动。只要在项目的根目录下执行如下命令，对环境进行初始化：
```shell
$ ./install/launch.sh init
```

因为这条命令会修改当前用户Home目录下的`.bashrc`，所以建议执行完毕后重新登录一次终端，以便在新的用户Session里重新加载`.bashrc`。或者，你也可以执行下面的命令，直接在当前Session里主动加载`.bashrc`：
```shell
$ . ~/.bashrc
```

另外，`.bashrc`里还有一些和Playground相关的环境变量，你可以根据自己的实际需求对它们进行修改。如果不修改也没关系，脚本会为它们指定默认值：
```shell
# The IP of your host, default is 127.0.0.1
export HOST_IP=
# The Kubernetes version, default is v1.14
export K8S_VERSION=
# The number of worker nodes, default is 2
export NUM_NODES=
```

最后，我们执行下面的命令，把Kubernetes集群启动起来：
```shell
$ launch default
```

> 也许你已经注意到，这里我们用的是`launch`，而不是带着一串路径的`launch.sh`。当执行完`init`命令对Playground进行初始化，并确保`.bashrc`被重新加载以后，我们就可以直接使用`launch`命令了。你可以在任意目录下执行`launch`，而不必非得在项目的根目录下。

### Vagrant Box还是宿主机？

如果是跑在笔记本或者本地机器上，并且你希望机器始终保持干净，那么使用Vagrant Box应该是一个不错的选择。否则地话，不妨直接在宿主机上安装，因为这样可以省去在box里安装操作系统的时间，所以启动速度会更快。

如果是跑在虚机里，那也可以不用Vagrant Box，因为没有必要让一个虚机（即box）跑在另一个虚机里。

### 我该选择哪种操作系统？

Playground支持多种操作系统，包括：
* Ubuntu
* CentOS
* RHEL
* MacOS

如果你选择Vagrant Box启动Playground，尽管Vagrantfile里用的是Ubuntu，但你也可以改成其他的操作系统。

## 我该怎么访问呢？

环境准备好了，该怎么访问呢？如果是在Vagrant Box里启动的集群，我们可以利用`vagrant ssh`命令进入虚机环境。如果是在宿主机上启动的集群，那什么也不用做，就可以直接访问集群了。比如，用`kubectl`对集群进行管理，或者用一下预装的那些Kubernetes常用命令行工具。

不仅如此，Playground还提供了一个Web终端，可以在浏览器里模拟普通终端，执行命令行命令。只要在浏览器地址栏输入[https://192.168.56.100:4200](https://192.168.56.100:4200)就可以访问到Web终端。根据提示输入用户名和密码，如果你用的是Vagrant Box，那么用户名密码分别是`vagrant`和`vagrant`。这里的IP地址对应的就是运行Playground的那台机器的地址，如果你用的是Vagrant Box，则对应虚机的IP地址，默认值为`192.168.56.100`。当然，它也是可以定制的。

### 动图演示

在下面这个GIF动图里，大家可以看到我是如何分别利用普通终端和Web终端展示各种Kubernetes命令行工具的使用的，是不是很酷啊😊（点击图片可放大观看哦）

[![](https://github.com/morningspace/lab-k8s-playground/raw/master/docs/demo-tools.gif)](https://github.com/morningspace/lab-k8s-playground/raw/master/docs/demo-tools.gif)

另外，对于基于Vagrant Box建立起来的Playground环境，如果想暂停手头和Playground相关的工作，可以执行`vagrant suspend`命令，把当前的虚机环境停掉。

要想再次恢复虚机环境，则执行`vagrant resume`命令。和第一次启动虚机环境不同，再次启动会变得非常快。这是因为它只是简单地把虚机环境恢复，而不需要重新初始化和安装所需要的软件了。

最后，如果不想保留当前的虚机环境，比如想重新启动一个配置和原来不一样的新环境，可以执行`vagrant destroy`命令，这会把当前的虚机彻底清除掉。

## 我还想装点别的……

默认启动起来的Playground，里面只包含了一个Kubernetes集群，和一系列命令行工具。如果我们想在这个基础上继续部署安装更多其他的应用该怎么办呢？这就需要用到`Launch Utility`了。

`Launch Utility`是[晴耕实验室](https://morningspace.github.io/lab/)专门为这个Playground开发的一套命令行工具。它的目的是为了简化基于Kubernetes的各种应用的部署安装。所有应用，都只需要清一色地执行一条`launch <target>`命令，就可以自动完成部署安装。其中的`<target>`，是对所有部署或安装任务的一种抽象和封装。

实际上，我们在前面已经用到过`Launch Utility`了，比如在对Playground环境进行初始化的时候。包括Vagrant Box在启动的过程中有很多环节，比如像：各种命令行工具的安装，Private Registry的搭建，Kubernetes集群的启动，背后都是通过执行`Launch Utility`完成的，它们都有各自对应的`<target>`可以“launch”。

### 安装Istio

下面，我们以安装[Istio](https://istio.io)，及Istio的演示程序[Bookinfo](https://istio.io/docs/examples/bookinfo)为例，熟悉一下`Launch Utility`的使用。

当Playground启动起来以后，我们通过普通终端或Web终端进入命令行。首先，确保几个Private Registry都已经启动起来了：
```shell
$ docker ps
CONTAINER ID  NAMES                 IMAGE       COMMAND                 CREATED       STATUS        PORTS
6e48d7ef4eb5  quay.io               registry:2  "/entrypoint.sh /etc…"  4 minutes ago Up 4 minutes  5000/tcp
58432b935674  gcr.io                registry:2  "/entrypoint.sh /etc…"  4 minutes ago Up 4 minutes  5000/tcp
8e26e4f744e3  registry-1.docker.io  registry:2  "/entrypoint.sh /etc…"  4 minutes ago Up 4 minutes  5000/tcp
8d3ede4e6c7b  k8s.gcr.io            registry:2  "/entrypoint.sh /etc…"  4 minutes ago Up 4 minutes  5000/tcp
0ca0b10a2d87  mr.io                 registry:2  "/entrypoint.sh /etc…"  4 minutes ago Up 4 minutes  5000/tcp, 0.0.0.0:5000->80/tcp
```

从输出结果里我们可以看到，前面几个代表的都是Public Registry，分别对应了`gcr.io`，`k8s.gcr.io`，`registry-1.docker.io`和`quay.io`。最后的`mr.io`不代表任何Public Registry，我们可以用它来存放一些私有的Docker镜像。如果Private Registry没有启动，我们可以利用`launch`命令把它们启动起来：
```shell
$ launch registry
```

然后，确保Kubernetes集群是正常启动着的：
```shell
$ kgno
NAME          STATUS   ROLES    AGE     VERSION
kube-master   Ready    master   2d19h   v1.14.4
kube-node-1   Ready    <none>   2d19h   v1.14.4
kube-node-2   Ready    <none>   2d19h   v1.14.4
```

如果集群没有启动，同样也可以利用`launch`命令把它启动起来：
```shell
$ launch kubernetes
```

如果两者都没启动，还可以在`launch`后面分别跟上`registry`和`kubernetes`，在一条命令里分别把Private Registry和Kubernetes集群依次启动起来：
```shell
$ launch registry kubernetes
```

最后，Istio的安装，也同样用的`launch`命令。我们只需要执行如下命令，就可以完成Istio的自动安装：
```shell
$ launch istio
```

`launch`命令会在部署完Istio后，持续等待直到Istio的所有Pod都成功启动起来以后才结束。这个过程可能会持续大概一到几分钟的时间。

紧接着，我们再利用`launch`命令把Bookinfo也部署到集群里：
```shell
$ launch istio-bookinfo
```

同样地，`launch`命令会在部署完Bookinfo以后监控Bookinfo的Pod，直到所有Pod成功启动才退出。

等Istio和Bookinfo都启动起来以后，我们再利用`launch`命令，通过端口转发的方式把它们各自的应用端口都暴露出来。这里，我们用的是`istio`和`istio-bookinfo`这两个target提供的`portforward`命令。调用target提供的命令，可以通过在指定`<target名称>`后面跟`::<命令名称>`的方式来实现。例如：
* `istio::portforwarding`
* `istio-bookinfo::portforwarding`

同样地，我们可以把这两个target的命令串接起来，在一个命令行里执行：
```shell
$ launch istio::portforward istio-bookinfo::portforward 
Targets to be launched: [istio::portforward istio-bookinfo::portforward]
####################################
# Launch target istio...
####################################
» Forwarding service/grafana 3000:3000...
Done. Please check /home/vagrant/lab-k8s-playground/install/logs/pfwd-grafana.log
» Forwarding service/kiali 20001:20001...
Done. Please check /home/vagrant/lab-k8s-playground/install/logs/pfwd-kiali.log
» Forwarding pod/jaeger 15032:16686...
Done. Please check /home/vagrant/lab-k8s-playground/install/logs/pfwd-jaeger.log
» Forwarding pod/prometheus 9090:9090...
Done. Please check /home/vagrant/lab-k8s-playground/install/logs/pfwd-prometheus.log
####################################
# Launch target istio-bookinfo...
####################################
» Forwarding service/istio-ingressgateway 31380:80...
Done. Please check /home/vagrant/lab-k8s-playground/install/logs/pfwd-istio-ingressgateway.log
Total elapsed time: 2 seconds
```

这样，我们就能在本机浏览器里对它们进行访问了。利用`launch endpoints`命令，我们可以看到当前集群里可供访问的所有endpoint：
```shell
$ launch endpoints
Targets to be launched: [endpoints]                                                                                                          
####################################                                                                                                         
# Launch target endpoints...                                                                                                                 
####################################                                                                                                         
» common endpoints...
✔ Web terminal: https://192.168.56.100:4200                                                                                                  
✔    Dashboard: http://192.168.56.100:32768/api/v1/namespaces/kube-system/services/http:kubernetes-dashboard:/proxy                          
» istio endpoints...
✔        Grafana: http://192.168.56.100:3000                                                                                                 
✔          Kiali: http://192.168.56.100:20001                                                                                                
✔         Jaeger: http://192.168.56.100:15032                                                                                                
✔     Prometheus: http://192.168.56.100:9090                                                                                                 
✔ Istio Bookinfo: http://192.168.56.100:31380/productpage                                                                                    
Total elapsed time: 1 seconds                                                                                                                
```

输出结果最左侧显示的，是每个endpoint当前的连通状态，`✔`表示当前可访问，`✗`则表示当前不可访问。如果是在Web终端里执行上述命令，你就可以直接点击每个endpoint后面跟着的URL链接，在新打开的浏览器窗口里访问各个应用，非常方便！

怎么样，`Launch Utility`的使用是不是很简单啊😊 “简单易用，风格一致”，是开发`Launch Utility`始终坚持的设计原则。后面，我们还将看到有关`Launch Utility`的更多玩法！

> Launch Utility支持Auto-Completion，如果你想知道当前都支持哪些target，只要在命令行输入`launch<空格>`，然后按两次Tab，就可以列出全部target。如果想知道当前target都支持哪些命令，则在target名称后面输入`::`，然后按两次Tab，就可以列出target的全部命令。

### 动图演示

在下面这个GIF动图里，大家可以看到在Istio和Bookinfo成功部署以后，我是如何在浏览器里访问各个不同应用的，包括：Grafana，Kiali，Jaeger，以及Kubernete Dashboard等（点击图片可放大观看哦）

[![](https://github.com/morningspace/lab-k8s-playground/raw/master/docs/demo-apps.gif)](https://github.com/morningspace/lab-k8s-playground/raw/master/docs/demo-apps.gif)

### 利用快照恢复集群

随着更多的应用被部署到当前的集群里，我们还可以通过下面的命令为当前集群建立一份快照：
```shell
$ launch kubernetes::snapshot
```

在这一过程中，脚本会搜集并保存当前集群的各种信息。这样一来，即使后面因为某些原因我们把集群“玩”坏了也没关系。只要“销毁”当前集群：
```shell
$ launch kubernetes::down
```

再根据先前保存的快照重启一个新的就可以了：
```shell
$ launch kubernetes::up
```

通过快照恢复得到的集群，会包含创建快照之前部署的所有应用。并且集群启动的速度会很快，因为它不需要再从无到有的部署先前那些应用了。

## 容器的镜像有更新了怎么办？

前面我们已经看到了，利用Private Registry，我们可以把部署集群需要的所有Docker镜像都保存到Private Registry里。这样，就可以免去通过网络访问远程Public Registry的需要，从而极大地提高了部署的速度。

那么，假如有些保存在Private Registry里的Docker镜像有版本更新了，或者我们有新的Docker镜像需要放到Private Registry里，对于一个已经启动起来的Playground，我们该怎么对Private Registry进行更新，从而反映Docker镜像的这些变化呢？要完成这一工作，依然离不开`Launch Utility`！用到的命令就是我们前面见到过的：
```shell
$ launch registry
```

每当Private Registry需要更新的时候，我们都可以通过执行这条命令来完成。即使虚机环境已经启动起来了，这条命令也是可以被反复执行的。我们只需要在执行命令之前，找到lab-k8s-playground项目的`install/targets/`目录，并在该目录下找到`images.list`文件进行编辑，修改里面Docker镜像的配置信息就可以了，比如修改已有镜像的版本，或添加新的镜像。`launch registry`命令在执行时会读取这个文件，根据里面的信息对Private Registry进行更新。

## 我能和别人共享Registry吗？

在基于Vagrant Box的Playground环境里，Private Regsitry是安装在虚机环境里面的，所以当我们利用`vagrant destroy`把虚机销毁以后，下次再启动新的虚机时，这个初始化过程又会重新执行一遍。在整个虚机环境的启动过程中，Private Registry的初始化是比较耗时的一个环节。如果我们不希望每次启动一个新的虚机环境都要初始化一遍Private Registry，我们也可以把这个环节“搬到”虚机外面来执行。

这种做法还有一个好处，就是能实现一套Private Registry在多个Playground之间共享。比如，在一个团队里面，我们可以搭建起一套团队共享的Private Registry，然后让每个开发人员的本机都指向这个Private Registry。

具体的做法还是执行`launch registry`命令，只是这一次我们是在虚机外面执行这条命令。耐心等待一段时间之后，Private Registry就在本地启动起来了。这个时候再通过Web终端或普通终端进入虚机，执行如下命令，把当前正在虚机里运行的所有Private Registry都停掉：
```shell
$ launch registry::down
```

然后再利用如下命令，在虚机内部为每个Private Registry启动相应的代理：
```shell
$ launch registry-proxy
```

如果我们为Playground里的代理所提供的Private Registry不是跑在本地宿主机上，而是远程的某台服务器，那么在运行`registry-proxy`命令之前，需要通过环境变量`REGISTRY_REMOTE`告诉脚本，远程服务器的IP地址：
```shell
$ export REGISTRY_REMOTE=<your_team_registry>
$ launch registry-proxy
```

最后，我们再次执行`launch`命令，重新部署一次Kubernetes集群，看看集群是否能成功部署：
```shell
$ launch kubernetes
```

## 怎么在离线模式下使用虚机？

前面我们提到过，Private Registry实际上是把部署集群所需的Docker镜像都保存到了本地，从而免去通过网络访问远程Public Registry抓取镜像的需要。这样做除了能够提高部署速度以外，还有另一个效果，那就是我们可以实现在没有网络连接的情况下运行Kubernetes集群，从而实现真正的“All-in-One”，非常适合demo环境的搭建。

要实现这一点，除了用到前面的`launch registry`命令外，我们还可以通过下面这条命令，把前面提到的用于模拟Docker Hub的`registry-1.docker.io`通过443端口暴露到集群外部：
```shell
$ launch registry::docker.io
```

完成这一步之后，我们就可以彻底把网络切断了！

### 动图演示

在下面这个GIF动图里，大家可以看到我是如何在没有网络连接的状态下部署Kubernetes集群并安装Helm的！（点击图片可放大观看哦）

[![](https://github.com/morningspace/lab-k8s-playground/raw/master/docs/demo-offline.gif)](https://github.com/morningspace/lab-k8s-playground/raw/master/docs/demo-offline.gif)

## 还有哪些地方我可以定制的？

前面我们已经看到了，即使Playground环境已经启动了，我们依然可以利用强大而灵活的`Launch Utility`根据需要对环境进行各种定制，比如：有选择性的部署安装某些应用，或者对Private Registry进行更新等等。

### Vagrantfile

除此以外，针对基于Vagrant Box的Playground环境，我们还可以通过位于项目根目录下的`Vagrantfile`文件对其进行定制：
```ruby
# set number of vcpus
cpus = '6'
# set amount of memory allocated by vm
memory = '8192'

# targets to be run, can be customized as your need
targets = "init default helm tools"

# set Kubernetes version, supported versions: v1.12, v1.13, v1.14, v1.15
k8s_version = "v1.14"
# set number of worker nodes
num_nodes = 2
# set host ip of the box
host_ip = '192.168.56.100'

# special optimization for users in China, 1 or 0
is_in_china = 0
# set https proxy
https_proxy = ""
```

比如，我们可以调节分配给虚机环境的CPU和内存资源的大小，以及虚机的IP地址，还可以指定集群环境所使用的Kubernetes版本，节点的数目，以及所要安装和部署的target。

### 针对中国用户的优化

另外，对于中国地区的用户，lab-k8s-playground还专门设计了一个配置项，叫做`is_in_china`。其目的是为了解决Kubernetes集群部署过程中，由于区域网络限制而无法访问某些国外网站，如：Google的Container Registry，Helm的默认Repository等，而导致集群部署失败的问题。

注意，大部分配置项在虚机里面和外面都是有效和可用的。如果是在虚机外面使用，只要把配置项的名称转换成全大写形式，然后以环境变量的方式定义就可以了。比如：
```shell
$ NUM_NODES=3 IS_IN_CHINA=1 launch registry kubernetes helm
```

这条命令告诉`launch.sh`，启动的集群要包含3个工作节点，且针对中国地区的用户进行优化。

### 扩展Launch Utility

最后，`Launch Utility`本身也是支持扩展的。如果我们希望用`launch`命令部署lab-k8s-playground项目目前还不支持的应用，包括我们自己的应用，那就定义一个你自己的target吧！自定义target非常简单，除了需要会一点Shell编程的技能以外，剩下的就是快速阅读一下“[Launch Utility Usage Guide](https://morningspace.github.io/lab-k8s-playground/docs/Launch-Utility-Usage-Guide.html)”这篇文档，或者学习一下 [install/targets/sample.sh](https://github.com/morningspace/lab-k8s-playground/blob/master/install/targets/sample.sh) 这个参考实现，然后你就可以动手改造`Launch Utility`啦！

## 还有哪些参考资料我能访问？

读到这里，相信你对开源项目[lab-k8s-playground](https://github.com/morningspace/lab-k8s-playground)及其All-in-One Playground的使用应该有了比较全面的了解了。那么，还有哪些参考资料你可以访问呢？那就请快速读一下下面这段Shell脚本编写的“代码”吧😊 相信一定难不倒你！

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
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://v.youku.com/v_show/id_XNDI2Mzk1NDcyMA==.html?f=52221532" rel="noopener" target="_blank">视频教程 Kubernetes实战:&nbsp;如何1分钟内在本地启动一个多节点集群</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://morningspace.github.io/tech/openshift-v3-trap-and-pitfalls" rel="noopener" target="_blank">OpenShift v3的填坑之旅</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">;;</span><br style="outline: none;">
&nbsp;&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(0, 201, 24);">"to extend it by myself"</span><span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">)</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://morningspace.github.io/lab-k8s-playground/docs/Launch-Utility-Usage-Guide.html" rel="noopener" target="_blank">Launch&nbsp;Utility&nbsp;Usage&nbsp;Guide</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://github.com/morningspace/lab-k8s-playground/blob/master/install/targets/sample.sh" rel="noopener" target="_blank">A Sample Target for Demo Purpose</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">;;</span><br style="outline: none;">
&nbsp;&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(0, 201, 24);">"to learn more cool features"</span><span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">)</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://morningspace.github.io/lab-k8s-playground/docs/APIC-Quick-Guide.html" rel="noopener" target="_blank">Quick&nbsp;Guide&nbsp;to&nbsp;Launch&nbsp;APIC&nbsp;Playground</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://morningspace.github.io/tech/all-in-one-apic-playground" rel="noopener" target="_blank">把API Connect关进All-in-One K8S Playground里</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">cat</span>&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(55, 119, 230);">"<a href="https://morningspace.github.io/tech/all-in-one-openshift-playground/" rel="noopener" target="_blank">All-in-One K8S Playground新增OpenShift支持</a>"</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">;;</span><br style="outline: none;">
&nbsp;&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(0, 201, 24);">"to contribute back"</span><span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">)</span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(119, 119, 119);"># Feel free to click these buttons: <a class="github-button" href="https://github.com/morningspace/lab-k8s-playground/subscription" data-icon="octicon-eye" aria-label="Watch morningspace/lab-k8s-playground on GitHub">Watch</a> <a class="github-button" href="https://github.com/morningspace/lab-k8s-playground" data-icon="octicon-star" aria-label="Star morningspace/lab-k8s-playground on GitHub">Star</a> <a class="github-button" href="https://github.com/morningspace/lab-k8s-playground/fork" data-icon="octicon-repo-forked" aria-label="Fork morningspace/lab-k8s-playground on GitHub">Fork</a></span><br style="outline: none;">
&nbsp;&nbsp; &nbsp;git clone<span>&nbsp;</span><a href="https://github.com/morningspace/lab-k8s-playground.git" rel="noopener" target="_blank">https://github.com/morningspace/lab-k8s-playground.git</a><br style="outline: none;">
&nbsp;&nbsp; &nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s;">;;</span><br style="outline: none;">
&nbsp;&nbsp;<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(173, 0, 161);">esac</span><br style="outline: none;">
<span style="outline: none; box-sizing: border-box; transition: all 0.2s ease-in-out 0s; color: rgb(31, 170, 170);">}</span></code>
</p>
