---
title: Kubernetes网络篇——将CNI用于容器
excerpt: "以Docker和rkt为例，演示如何将CNI用于真正的容器系统"
categories: [tech]
tags: [lab, dummies, dummies_kubernetes, kubernetes]
toc: true
toc_sticky: true
---

注：
本文采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/80x15.png" /></a>

> CNI在真正的容器系统里是如何工作的呢？本文将利用CNI的bridge插件，分别以Docker和rkt为例，来演示它的工作原理。

## Docker与CNI

![](/assets/images/lab/k8s/docker.png){: .align-center}

在[Kubernetes网络篇——认识CNI](/tech/k8s-net-cni/)一文里，我们利用标准的CNI插件bridge，为手工创建的network namespace建立起了相应的网络环境。本文，我们来看一下如何把CNI用于真正的容器系统，首先来看一下Docker。

要让Docker使用CNI并没有想象中那么简单和直观。在[Kubernetes网络篇——从docker0开始](/tech/k8s-net-docker0/)和[模拟Docker网络](/tech/k8s-net-mimic-docker/)里我们已经看到了，利用bridge network可以实现容器和外界的网络通信。但bridge network是位于Layer-2即数据链路层的设备，只提供基于Layer-2的数据包转发，没有办法实现基于Layer-3即网络层的路由功能，以及其他一些像overlay network这样的高阶功能。这些需求要单靠Docker自己是无法全部满足的。更好的办法是开放标准接口，让第三方来实现它们自己的网络服务，这就是Docker网络的设计思路，这和CNI的出发点很类似。Docker也有很多支持各种网络功能的插件，但它们遵循的标准并不是CNI，而是另一套被称为CNM(Container Network Model)的标准。

不过，Docker并不是唯一的容器技术，还有一些其他的竞争者，比如像rkt，就是支持CNI的。一般认为，CNI的接口更加简单，所以它被认可和采纳的范围更广一些。另外，以Kubernetes为代表的一些容器集群与编排技术，也不希望绑定到某个特定的容器技术上，所以就会倾向于接受程度更广的CNI。关于这一点，Kubernetes的网站上有[一篇文章](https://kubernetes.io/blog/2016/01/why-kubernetes-doesnt-use-libnetwork/)，专门提到了它之所以没有选择CNM的原因。

## 让Docker使用CNI

虽然Docker有自己的网络插件机制，但我们还是可以让它和CNI工作在一起的。为了做到这一点，可以在启动容器时指定`--net=none`，告诉Docker不要带任何网络设备，下面我们就来演示一下。

这次我们选择以nginx为例，把它当作一个基本的Web服务器。默认的nginx镜像不包含`ip`命令，为了方便后面的实验，我们以nginx作为base镜像，在它的基础上安装了iproute2。把下面的Dockerfile保存到当前目录：
```docker
FROM nginx

RUN apt-get update && \
    apt-get install --no-install-recommends --no-install-suggests -y \
    iproute2 && \
    rm -rf /var/lib/apt/lists/*
```

然后执行下面的命令构建Docker镜像：
```shell
$ docker build -t morningspace/lab-web .
```

并启动相应的容器，不带任何网络：
```shell
$ docker run --name lab-web --net=none -d morningspace/lab-web
2296af8b515dcda2683deb4e95935427ffbce9a5e13332e33c52230fadc1aec0
```

等容器启动以后，查看它的网络设置：
```shell
$ docker exec lab-web ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1
    link/ipip 0.0.0.0 brd 0.0.0.0
3: ip6tnl0@NONE: <NOARP> mtu 1452 qdisc noop state DOWN group default qlen 1
    link/tunnel6 :: brd ::
```

我们会发现，这个时候容器里只有一个loopback和两个默认创建的网络接口，并没有带IP地址的以太网接口，所以容器也就没有和宿主机的bridge连接起来。

### 配置CNI插件

接下来，我们要利用CNI把这个Docker容器连接到宿主机的bridge上。首先为CNI插件创建配置文件：
```shell
$ cat > lab-br1.conf <<"EOF"
{
    "cniVersion": "0.4.0",
    "name": "lab-br1",
    "type": "bridge",
    "bridge": "lab-br1",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "subnet": "10.15.20.0/24",
        "routes": [
            { "dst": "0.0.0.0/0" }
        ]
    }
}
EOF
```

关于文件里的每一项配置，这里就不再一一解释了，大家可以在[Kubernetes网络篇——认识CNI](/tech/k8s-net-cni/)一文里找到详细的说明。

### 运行CNI插件

接下来，我们要运行CNI插件为容器创建网络了。之前曾经提到过，在运行CNI插件时，我们需要通过环境变量CNI_CONTAINERID和CNI_NETNS告诉插件容器的Id和对应network namespace的路径。

为了拿到这两个数据，我们可以执行如下命令，对容器的元数据进行查询：
```shell
$ docker inspect lab-web | grep -E 'SandboxKey|Id'
        "Id": "2296af8b515dcda2683deb4e95935427ffbce9a5e13332e33c52230fadc1aec0",
            "SandboxKey": "/var/run/docker/netns/801b81aea4e1",
```

这里，属性Id的取值，对应的就是容器的Id，SandboxKey的取值就是network namespace的路径。有了这两个值，我们就可以运行CNI插件了。此处，我们确保当前位于/root/test-cni/plugins目录下：
```shell
$ CNI_COMMAND=ADD CNI_CONTAINERID=2296af8b515dcda2683deb4e95935427ffbce9a5e13332e33c52230fadc1aec0 CNI_NETNS=/var/run/docker/netns/801b81aea4e1 CNI_IFNAME=eth0 CNI_PATH=`pwd` ./bridge <../conf/lab-br1.conf
{
    "cniVersion": "0.4.0",
    "interfaces": [
        {
            "name": "lab-br1",
            "mac": "52:ba:e2:62:01:1e"
        },
        {
            "name": "vethd5747d99",
            "mac": "2e:42:d8:e3:c9:11"
        },
        {
            "name": "eth0",
            "mac": "26:3f:a5:1f:c8:27",
            "sandbox": "/var/run/docker/netns/801b81aea4e1"
        }
    ],
    "ips": [
        {
            "version": "4",
            "interface": 2,
            "address": "10.15.20.5/24",
            "gateway": "10.15.20.1"
        }
    ],
    "routes": [
        {
            "dst": "0.0.0.0/0"
        }
    ],
    "dns": {}
}
```

### 验证网络

接下来，让我们来验证一下CNI为我们配置的网络环境。首先查看一下宿主机的网络接口：
```shell
$ ip addr show
... ...
9: lab-br1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 52:ba:e2:62:01:1e brd ff:ff:ff:ff:ff:ff
    inet 10.15.20.1/24 brd 10.15.20.255 scope global lab-br1
       valid_lft forever preferred_lft forever
21: vethd5747d99@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master lab-br1 state UP group default 
    link/ether 2e:42:d8:e3:c9:11 brd ff:ff:ff:ff:ff:ff link-netnsid 2
... ...
```

可以看到，输出结果里和原来相比多了lab-br1和veth pair在宿主机一端的接口vethd5747d99@if5。并且，CNI为lab-br1自动分配的IP地址`10.15.20.1`，正好位于我们指定的网段`10.15.20.0/24`范围内。

再来看一下容器里的网络接口：
```shell
$ docker exec lab-web ip addr show
... ...
5: eth0@if21: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 26:3f:a5:1f:c8:27 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.15.20.5/24 brd 10.15.20.255 scope global eth0
       valid_lft forever preferred_lft forever
```

这个时候，和原来相比除了默认的几个网络接口以外，还多了一个eth0，并且CNI为它分配的IP地址`10.15.20.5`，也在我们指定的网段内。

接下来再看一下容器的路由规则：
```shell
$ docker exec lab-web ip route
default via 10.15.20.1 dev eth0 
10.15.20.0/24 dev eth0 proto kernel scope link src 10.15.20.5 
```

可以看到，所有来自`10.15.20.5`，也就是当前容器的数据包，都将直接发往`10.15.20.0/24`网段。对于其他情况，则选择默认路由规则，数据包将被发往位于宿主机上的网桥。

最后，我们来验证一下从宿主机到容器的网络连通性：
```shell
$ curl http://10.15.20.5
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
... ...
</body>
</html>
```

通过上面的实验我们看到了，利用CNI插件为Docker容器设置网络环境和为network namespace设置网络环境没有什么差别。我们只要告诉插件，目标网络的配置信息，以及容器的Id和对应network namespace的路径，剩下的事情就可以交给插件自己来完成了。

## 关于rkt

![](/assets/images/lab/k8s/rkt.png){: .align-center}

[rkt](https://coreos.com/rkt/)是2014年底由[CoreOS](https://coreos.com/)推出的。作为Docker容器技术的竞争者之一，它的主要特点包括：
* 以pod而非容器为基本执行单元，也就是所谓的“pod-native”。这里pod的概念和Kubernetes里的pod是类似的，即：在一个共享的上下文里执行的一个或多个容器化应用，这也是它和Docker的主要区别之一；
* rkt和Docker相比的另一个显著不同在于，它不像Docker那样有一个集中的Daemon来负责所有容器的生命周期管理，这样可以更好的和Linux的一些所谓的init系统相兼容，比如像：`systemd`，`upstart`等；
* 与Docker不同，rkt是直接原生支持CNI的，因此它不用像Docker那样，需要在启动容器时，通过特殊参数禁掉默认的网络设置逻辑以后，再手工执行CNI插件；
* rkt还兼容Docker的镜像格式，它可以直接从Docker的注册表里抓取镜像并运行容器；

如果说本文前半部分用Docker来演示CNI的使用看起来有点“不太正规”，那么后半部分作为原生支持CNI的rkt，通过它来演示CNI应该会更具有代表性，也更符合实际情况。下面我们就来看一下，rkt是怎么和CNI结合在一起工作的。

## rkt结合CNI


### 关于实验环境

由于rkt是和Docker一样的容器运行环境，因此针对rkt的实验，就不太适合在`morningspace/lab-dind`这样的Docker容器里进行了。对于这一部分实验，我是在本地MacOS上的一个虚机环境里进行的，我所使用的Linux发行版本是debian/jessie64。它是跑在VirtualBox上，并通过Vagrant来管理的。关于如何在MacOS上利用VirtualBox和Vagrant来跑Debian，因为不是本文关注的焦点，所以这里就不展开了。

> 笔者还真的尝试过在基于CentOS的Docker容器里跑rkt，而且貌似差不多要成功了。不过后来觉得太tricky了，加上时间关系，最后还是选择了放弃。过程中遇到的“坑”主要是：在容器里运行overlay文件系统的冲突，以及privileged权限的缺失问题。

### 安装rkt

在Debian上安装rkt很简单，rkt的发布版本包含了.deb的安装包，我们只要在它的GitHub库里选择希望安装的版本，然后把它下载到本地进行安装就可以了，比如：
```shell
$ wget https://github.com/rkt/rkt/releases/download/v1.30.0/rkt_1.30.0-1_amd64.deb
$ wget https://github.com/rkt/rkt/releases/download/v1.30.0/rkt_1.30.0-1_amd64.deb.asc
$ gpg --keyserver keys.gnupg.net --recv-key 18AD5014C99EF7E3BA5F6CE950BDD3E0FC8A365E
$ gpg --verify rkt_1.30.0-1_amd64.deb.asc
$ sudo dpkg -i rkt_1.30.0-1_amd64.deb
```

当安装完毕以后，可以执行下面的命令来验证安装是否成功：
```shell
$ rkt version
rkt Version: 1.30.0
appc Version: 0.8.11
Go Version: go1.8.3
Go OS/Arch: linux/amd64
Features: -TPM +SDJOURNAL
```

### 配置CNI插件

rkt是借助CNI来配置容器的网络接口的。因此，我们也需要为CNI插件提供相应的配置文件。由于rkt原生支持CNI，所以我们不需要像Docker那样手工执行CNI插件，而是把配置文件放到特定目录下，默认位于`/etc/rkt/net.d`，rkt会在容器启动的时候自动扫描这个目录下的配置文件。

为方便后面的操作，我们先把当前用户切换到root，并创建`/etc/rkt/net.d`目录：
```shell
$ su
Password:
$ mkdir /etc/rkt/net.d
```

然后在该目录下创建配置文件：
```shell
$ cd /etc/rkt/net.d
$ cat > lab-rkt-br.conf <<"EOF"
{
    "cniVersion": "0.4.0",
    "name": "lab-rkt-br",
    "type": "bridge",
    "bridge": "lab-rkt-br",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "subnet": "10.15.20.0/16",
        "routes": [
            { "dst": "0.0.0.0/0" }
        ]
    }
}
EOF
```

### 启动容器

执行下面的命令启动一个容器：
```shell
$ rkt run --insecure-options=image --interactive --net=lab-rkt-br docker://busybox
Downloading sha256:8e674ad76dc [===============================] 763 KB / 763 KB
run: disabling overlay support: "overlay entry not present in /proc/filesystems"
/ #
```

其中，`--net`参数告诉rkt，容器在启动时要加入的网络；`--interactive`参数告诉rkt，容器在启动以后进入交互模式；另外，我们选择了位于[Docker Hub](https://hub.docker.com/)上的Docker镜像busybox。为此，我们需要在镜像名前面加上`docker://`。同时，因为Docker镜像不像rkt那样支持签名验证(signature verification)，所以还要加上`--insecure-options=image`参数。

### 验证网络

因为执行完`rkt run`以后，就进入到busybox容器里面了，所以我们可以在容器里执行`ip`命令，查看网络接口的变化情况：
```
/ # ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
3: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue 
    link/ether b2:e1:43:77:64:85 brd ff:ff:ff:ff:ff:ff
    inet 10.15.20.2/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::b0e1:43ff:fe77:6485/64 scope link 
       valid_lft forever preferred_lft forever
```

可以看到，这里的eth0对应的就是veth pair在容器内的那一端，它的IP地址是`10.15.20.2`，位于我们之前定义的网段`10.15.20.0/16`范围内。

紧接着，我们再通过按键`Ctrl + ]]]`，即：按住`Ctrl`的同时连续按三次`]`，从容器的交互模式里退出并回到宿主机。然后进入`/var/lib/cni/networks`目录，也就是我们的`host-local`插件保存网络配置信息的地方：
```shell
$ cd /var/lib/cni/networks/
$ ls
lab-rkt-br

$ cd lab-rkt-br/
$ ls
10.15.20.2

$ cat 10.15.20.2 
0310ac0a-08f8-4d84-8b65-1669bb9c5f19
```
可以看到，`networks`目录下的子目录名就是我们之前定义的网络名`lab-rkt-br`。这个目录下包含的文件`10.15.20.2`，其名称就是我们为容器分配的IP地址，文件内容则是容器的Id。

我们可以通过执行`rkt list --full`查看当前运行的所有容器，并返回完整的容器Id：
```shell
$ rkt list --full
UUID					APP	IMAGE NAME					IMAGE ID		STATE	CREATED					STARTED					NETWORKS
0310ac0a-08f8-4d84-8b65-1669bb9c5f19	busybox	registry-1.docker.io/library/busybox:latest	sha512-bea5a3990a66	running	2019-06-17 12:49:40.58 +0000 GMT	2019-06-17 12:49:40.784 +0000 GMT	lab-rkt-br:ip4=10.15.20.2
```

在UUID一栏所显示的值，正是我们在`10.15.20.2`文件里看到的值。
