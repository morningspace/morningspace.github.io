---
title: Kubernetes网络篇——从docker0开始
excerpt: "从docker0出发，讨论network bridge的工作原理"
categories: [tech]
tags: [lab, dummies, dummies_kubernetes, kubernetes]
toc: true
toc_sticky: true
---

注：
本文采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/80x15.png" /></a>

> Linux的network bridge和network namespace是两个非常重要的概念，它们是理解Kubernetes网络工作原理的基础。本文从docker0出发，讨论network bridge的工作原理。

![](/assets/images/lab/k8s/docker-engine.png){: .align-center}

网络名字空间(network namespace)是Linux内核提供的一项非常重要的功能，它是网络虚拟化技术的基础，也是Kubernetes和以Docker为代表的容器技术在实现它们各自的网络时所依赖的基础。所以，要理解Kubernetes的网络工作原理，首先要从network namespace入手。

在Linux内核里，每个network namespace都有它自己的网络设置，比如像：网络接口(network interfaces)，路由表(routing tables)等。我们利用network namespace，可以把不同的网络设置彼此隔离开来。当运行多个Docker容器的时候，Docker会在每个容器内部创建相应的network namespace，从而实现不同容器之间的网络隔离。

但是光有隔离还不行，因为容器还要和外界进行网络联通，所以除了network namespace以外，另一个重要的概念是网桥(network bridge)。它是由Linux内核提供的一种链路层设备，用于在不同网段之间转发数据包。Docker就是利用网桥来实现容器和外界之间的通信的。默认情况下，Docker服务会在它所在的机器上创建一个名为docker0的网桥。下面我们就先从docker0入手，通过一系列动手实验，了解网桥在容器之间进行通信时所起的作用。

## 实验环境

在开始实验之前，首先确保Docker服务所在的宿主机上已经安装了`iproute2`, `bridge-utils`, `iptables`这几个依赖。因为在本文以及后续Kubernetes网络的系列文章中，我们会经常用到这些依赖包所提供的命令行工具。如果没有安装这些依赖的话，可以手动进行安装。比如，以Alpine Linux为例：
```shell
$ apk add iproute2 bridge-utils iptables
```

或者，也可以使用[lab-k8s](https://github.com/morningspace/lab-k8s)为我们提供的名为morningspace/lab-dind的Docker镜像，里面预装了所有的依赖。在撰写这一系列文章的时候，笔者就是利用该镜像在一台Mac笔记本上完成的所有实验。

由于目前Mac上运行的Docker Desktop依然是把Docker放在一个虚拟机上跑的，而这个虚拟机通常情况下又不太容易直接访问到，所以如果我们想查看虚拟机上的网络设置会比较困难。这就是为什么笔者建议我们在做实验的时候使用lab-dind镜像的原因。该镜像是一个标准的Alpine Linux，进入容器以后，还可以执行各种docker命令，进行我们的实验。比如在容器内部启动另一个容器，这和在物理机上的操作体验是完全一样的。这就是所谓的dind(Docker-in-Docker)技术。虽然严格地说，这种操作方式和跑在物理机上的Linux，以及相应的Docker环境还是有一定的差异的，但这并不妨碍我们做实验。

如果我们已经把[lab-k8s](https://github.com/morningspace/lab-k8s)项目客隆到了本地，那么就可以直接在项目的根目录下运行`docker-compose`命令来启动lab-dind容器：
```shell
$ docker-compose up -d dind
Creating lab-k8s_dind_1_9791018c4641 ... done
```

等容器成功启动起来以后，我们再通过如下命令进入到容器内部：
```shell
$ docker-compose exec dind bash
bash-4.4#
```

接下来，就可以开始进行各种实验了。本文从这里开始，包括这一系列的后续文章，里面提到的所有实验环节都假设大家是在lab-dind容器里进行的。

## 什么是Docker0

现在我们来看一下Docker的默认网桥docker0。首先，我们利用`brctl`命令，列出当前系统的所有网桥：
```shell
$ brctl show
bridge name	bridge id		STP enabled	interfaces
docker0		8000.024277d8e553	no
```

在我们的实验环境里，目前就只有docker0这一个网桥。执行`ifconfig`，我们还可以列出docker0的详细信息：
```shell
$ ifconfig docker0
docker0   Link encap:Ethernet  HWaddr 02:42:77:D8:E5:53  
          inet addr:172.17.0.1  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

从输出结果里，我们可以看到，docker0的IP地址为172.17.0.1，子网掩码为255.255.0.0。

前面我们提到过，docker0是由Docker服务在启动时自动创建出来的以太网桥。默认情况下，所有Docker容器都会连接到docker0，然后再通过这个网桥来实现容器和外界之间的通信。那么，Docker具体是怎么做到这一点的呢？我们先用`docker network ls`命令来看一下Docker默认提供的几种网络：
```shell
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
d307261937e9        bridge              bridge              local
9a4e6a25b62e        host                host                local
53cf3a3b2e4f        none                null                local
```

我们知道，Docker容器在启动时，如果没有显式指定加入任何网络，就会默认加入到名为bridge的网络。而这个bridge网络就是基于docker0实现的。

除此以外，这里的host和none是有特殊含义的。其中，加入host网络的容器，可以实现和Docker daemon守护进程（也就是Docker服务）所在的宿主机网络环境进行直接通信；而none网络，则表示容器在启动时不带任何网络设备。

## 启动第一个容器

现在，我们就来看一下Docker容器在加入bridge网络的过程中，容器以及宿主机网络设置的变化情况。首先，我们通过`docker network inspect`命令，看一下bridge网络目前的状态：
```shell
$ docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "d307261937e987e9a0d46279c2033824920167f31f1b0371a9f7dfc52b9e55ca",
        "Scope": "local",
        "Driver": "bridge",
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16"
                }
            ]
        },
        "Containers": {},
        ... ...
    }
]
```

我们注意到，`Subnet`字段的值为172.17.0.0/16。这表明了，bridge网络位于172.17.0.0网段，子网掩码为255.255.0.0。这和前面`ifconfig`命令看到的结果保持一致，docker0的IP地址刚好位于这一网段内。

对`172.17.0.0/16`这种写法感到陌生的同学，可以在网上搜一下CIDR。这是一种被称为“无类域间路由”的标记方法，其英文全称为Classless Inter-Domain Routing（简称CIDR）。这个名字听起来有点唬人，不过没关系，我们只要记住，“/”前面的部分代表当前网段的网络ID，“/”后面的部分代表子网掩码中连续1的个数。比如，在我们的例子里，16就表示连续16个1，对应的子网掩码就是：`11111111.11111111.00000000.00000000`，十进制表示就是：`255.255.0.0`。

另外，我们还注意到，这里的`Containers`字段目前是空的。接下来，我们启动一个新的Docker容器：
```shell
$ docker run -dit --name busybox1 busybox sh
Unable to find image 'busybox:latest' locally
latest: Pulling from library/busybox
ff5eadacfa0b: Pull complete 
Digest: sha256:c888d69b73b5b444c2b0bd70da28c3da102b0aeb327f3a297626e2558def327f
Status: Downloaded newer image for busybox:latest
4710242fd42dc97b8f36470ceb8a29c32979a60f00cccc8d55edcab04216d6d3
```

可以看到，Docker为我们在当前宿主机上启动了一个名为busybox1的容器。这个时候，再查看bridge网络：
```shell
$ docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "d307261937e987e9a0d46279c2033824920167f31f1b0371a9f7dfc52b9e55ca",
        "Scope": "local",
        "Driver": "bridge",
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16"
                }
            ]
        },
        "Containers": {
            "4710242fd42dc97b8f36470ceb8a29c32979a60f00cccc8d55edcab04216d6d3": {
                "Name": "busybox1",
                "EndpointID": "2c82c71f3dc5b34e283c7f72c300912ce0f0e11890e7570c0b72bc748a5c1184",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        },
        ... ...
    }
]
```

就会发现，`Containers`字段里出现了busybox1，也就是我们刚刚启动的那个容器。这说明busybox1容器已经成功地加入到了我们的bridge网络里。如果这个时候查看docker0网桥：
```shell
$ brctl show
bridge name	bridge id		STP enabled	interfaces
docker0		8000.024277d8e553	no		vethe657f66
```

就会发现，和一开始执行`brctl show`的输出结果相比，`interfaces`字段的位置多了一个名叫vethe657f66的网络接口。实际上，这是一种虚拟以太网设备(Virtual Ethernet Device，简称veth)。确切地说，这不是一个设备，而是一对设备，所以也被称为“veth pair”。它包含两个总是成对出现的网络接口，分别连接不同的network namespace。一端的网络接口接收到数据以后，就会立刻传送给另一端，从而在两个network namespace之间建立起了一个“通道”，实现了彼此之间的网络连通。通常，这一对接口本身并不会被分配IP地址。在我们的例子里，这个veth pair的一端位于容器busybox1里，另一端则位于宿主机上，也就是这里的vethe657f66。执行`ip addr show`命令查看该接口的详细信息：
```shell
$ ip addr show vethe657f66
6: vethe657f66@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
    link/ether 1a:e1:8a:b5:d3:93 brd ff:ff:ff:ff:ff:ff link-netnsid 1
```

可以看到，vethe657f66后面跟着一个后缀@if5。其中的数字5，代表了作为这个veth pair的另一端（即位于busybox1容器里的那个网络接口）在对应的network namespace里所有网络接口中的位置序号。也就是每当我们执行`ip addr show`命令的时候，输出结果里每个网络接口前面的那个数字。如果我们在busybox1里执行`ip addr show`：
```shell
$ docker exec busybox1 ip addr show
... ...
5: eth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

就可以看到，eth0的序号正是5，和宿主机里vethe657f66后面的@if5是一致的。与此同时，busybox1里eth0的后缀是@if6。对照前面vethe657f66的输出结果，它的序号正是6。这说明，宿主机里的vethe657f66和busybox1里的eth0构成了一对veth pair。利用这个“通道”，我们的busybox1就可以实现和外界的通信了。

## 启动另一个容器

接下来，我们再来启动另一个容器：
```shell
$ docker run -dit --name busybox2 busybox sh
fa6c607330b5cf06753e86e89b0fa7c9620e7187a4905a67c07499fdc477d4c2
```

然后，查看bridge网络：
```shell
$ docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "d307261937e987e9a0d46279c2033824920167f31f1b0371a9f7dfc52b9e55ca",
        "Scope": "local",
        "Driver": "bridge",
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16"
                }
            ]
        },
        "Containers": {
            "4710242fd42dc97b8f36470ceb8a29c32979a60f00cccc8d55edcab04216d6d3": {
                "Name": "busybox1",
                "EndpointID": "2c82c71f3dc5b34e283c7f72c300912ce0f0e11890e7570c0b72bc748a5c1184",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            },
            "fa6c607330b5cf06753e86e89b0fa7c9620e7187a4905a67c07499fdc477d4c2": {
                "Name": "busybox2",
                "EndpointID": "2980a88338acb0b07ea4deb1e5a25d0264ece9fb16c5a8386469e853a101684a",
                "MacAddress": "02:42:ac:11:00:03",
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            }
        },
        ... ...
    }
]
```

可以看到，每次启动新的容器，默认总是会加入到这个bridge网络里的。现在的`Containers`字段，已经包含两个容器了，分别是busybox1和busybox2。我们还可以继续查看docker0网桥，以及veth pair分别在宿主机和容器里的网络接口信息，这里就不再赘述了。

## 验证网络连通性

这个时候，如果我们分别在宿主机和两个容器里执行ping命令，会发现它们三者是彼此连通的。比如，在宿主机里可以ping通容器：
```shell
$ ping 172.17.0.2 -c 3
PING 172.17.0.2 (172.17.0.2): 56 data bytes
64 bytes from 172.17.0.2: seq=0 ttl=64 time=0.125 ms
64 bytes from 172.17.0.2: seq=1 ttl=64 time=0.175 ms
64 bytes from 172.17.0.2: seq=2 ttl=64 time=0.194 ms

--- 172.17.0.2 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.125/0.164/0.194 ms
```

在容器里可以ping通宿主机：
```shell
$ docker exec busybox1 ping 172.17.0.1 -c 3
PING 172.17.0.1 (172.17.0.1): 56 data bytes
64 bytes from 172.17.0.1: seq=0 ttl=64 time=0.090 ms
64 bytes from 172.17.0.1: seq=1 ttl=64 time=0.147 ms
64 bytes from 172.17.0.1: seq=2 ttl=64 time=0.162 ms

--- 172.17.0.1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.090/0.133/0.162 ms
```

也可以ping通其他容器：
```shell
$ docker exec busybox1 ping 172.17.0.3 -c 3
PING 172.17.0.3 (172.17.0.3): 56 data bytes
64 bytes from 172.17.0.3: seq=0 ttl=64 time=0.513 ms
64 bytes from 172.17.0.3: seq=1 ttl=64 time=0.533 ms
64 bytes from 172.17.0.3: seq=2 ttl=64 time=0.238 ms

--- 172.17.0.3 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.238/0.428/0.533 ms
```

## IPTables和`-p`参数

如果数据包里包含的目标端口号是8080，就把该数据包转发到目标容器：修改目标地址为容器的IP地址，修改目标端口号为容器内服务的端口号。
