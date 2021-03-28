---
title: Kubernetes网络篇——自己动手写CNI插件(上)
excerpt: "用shell写一个简单的CNI插件，理解CNI和容器生命周期的关系"
categories: [tech]
tags: [lab, dummies, dummies_kubernetes, kubernetes]
toc: true
toc_sticky: true
---

注：
本文采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/80x15.png" /></a>

> 有没有想过自己写一个CNI插件呢？也许大多数时候我们都没必要自己开发插件。不过，出于学习的目的，或者为了排查错误，也许你会读到别人写的插件。这个时候，如果事先对CNI插件编程有一定了解，那就会事半功倍。

![](/assets/images/lab/k8s/cni-shell.png){: .align-center}

## CNI插件编程

如果看过这一系列的前面两篇文章，大家一定知道，CNI插件和遵循CNI规范的容器系统之间是通过配置文件加环境变量，并借助标准输入输出设备来进行通信的，每个CNI插件都是一个标准的可执行文件。正是因为这样的设计，决定了CNI插件可以用任何编程语言来实现，只要按照CNI的标准接口，我们的插件就可以适用于任何支持CNI的容器系统。

本文，我们将使用shell语言来实现一个非常简单的CNI插件，用日志输出的方式来演示CNI所提供的命令和容器生命周期之间的关系。在下一篇文章里，我们将使用go语言，结合CNI项目提供的go工具库，开发一个接近真实的更复杂一些的插件。它和CNI的标准插件bridge类似，会为我们在宿主机上创建一个bridge，并把容器连接到bridge上。

## 第一个插件：logger

接下来，我们就用shell语言来开发一个非常简单的CNI插件：logger。它并不对网络配置做真正的修改，而只是在容器从启动到停止的过程中，把容器系统传给它的信息输出到终端显示器。

## 代码实现

下面一起来看一下代码：

```text
#!/bin/bash                                 ①

log () {
  >&2 echo "$1"                             ②
}

log "Command: $CNI_COMMAND"                 ③
log "Container Id: $CNI_CONTAINERID"
log "Path to Network Namespace: $CNI_NETNS"
log "Network Interface: $CNI_IFNAME"
log "Path to CNI Plugins: $CNI_PATH"
log "Network Configuration:"                ④
while IFS='' read line
do
  log "$line"
done < /dev/stdin                           ⑤

if [[ $CNI_COMMAND == "ADD" ]]; then
  echo "{}"                                 ⑥
fi
```

可以看到，用shell写一个最简单的CNI插件不需要几行代码。下面我们就来解释一下这段代码：

行①：这里指明了，我们所采用的shell语言为bash shell。

行②：这是我们用来向终端显示器输出日志的逻辑。由于CNI规范定义了，容器系统是通过标准输入(stdin)和输出设备(stdout)和插件进行交互的，因此我们通过`>&2`，把日志输出重定向到了标准错误输出设备(stderr)。这里的`2`对应的就是stderr的设备号（stdout的设备号为`1`）。这样，我们的日志输出结果就不会干扰容器系统和插件的正常通信了。

行③：从这一行开始，我们陆续把一系列由容器系统通过环境变量传给插件的信息输出成了日志，比如：执行的CNI命令，容器Id，network namespace路径等。

行④：从这一行开始，我们输出的是通过行⑤的标准输入设备(stdin)传入插件的JSON格式的网络配置文件。代码通过一个循环，逐行读取配置文件，并输出成日志。

行⑥：可以看到，我们在这一行直接向标准输出设备(stdout)打印了一对空的大括号。这主要是因为，CNI规范里规定了，对于ADD操作，要求插件在返回之前把执行结果以JSON格式打印到标准输出设备。这里，我们为了演示，只是简单的输出了一个“{}”。

## 代码运行

接下来，我们用rkt作为容器运行时环境来验证我们logger插件。关于在rkt里运行CNI插件的详细情况，大家可以参考[Kubernetes网络篇——将CNI用于容](/tech/k8s-net-cni-docker-rkt/)一文的后半部分。

根据rkt的文档，它会在特定路径下搜索CNI插件的可执行文件，比如：`/usr/lib/rkt/plugins/net`，`/etc/rkt/net.d`，后者也是rkt读取网络配置文件的路径。我们把上面的代码以文件形式保存到`/etc/rkt/net.d`目录下，取名为logger，并为其赋予可执行权限：
```shell
$ chmod +x ./logger
```

然后再在同一目录下创建网络配置文件，取名`lab-test-net-1.conf`：
```shell
$ cat lab-test-net-1.conf
{
  "cniVersion": "0.4.0",
  "name": "lab-test-net-1",   ①
  "type": "logger"            ②
}
```

这里：

行①指定了我们所定义的网络名称。当我们用rkt命令启动容器时，务必要保证`--net`参数所指定的网络名称和这里定义的名称是保持一致的，否则rkt会报错；

行②指定了我们要调用的CNI插件的名称，它和我们的插件可执行文件名是保持一致的，rkt会根据这里的配置，在`/etc/rkt/net.d`目录下寻找并执行我们编写的插件；

现在，我们来执行下面的命令，启动一个busybox容器：
```shell
$ rkt run --insecure-options=image --interactive --net=lab-test-net-1 docker://busybox
run: disabling overlay support: "overlay entry not present in /proc/filesystems"
Command: ADD
Container Id: 1ebcb360-bcf6-4c3f-b38e-a8826eac7f9b
Path to Network Namespace: /var/run/netns/cni-d59164ee-3cc1-0a62-a4ec-b486b3028dae
Network Interface: eth0
Path to CNI Plugins: /etc/rkt/net.d:/usr/lib/rkt/plugins/net:stage1/rootfs/usr/lib/rkt/plugins/net
Network Configuration:
{
  "cniVersion": "0.4.0",
  "name": "lab-test-net-1",
  "type": "logger"
}
```

从输出结果中可以看到，我们的logger插件被成功调用了。输出的Command值显示，rkt在启动容器的时候会调用CNI插件的ADD命令。同时，它还会把刚刚创建的busybox容器的Id，对应network namespace的路径，以及默认的网络接口名称(eth0)等信息，也都传进来。最后是从标准输入设备读到的网络配置信息，和我们之前定义的完全一致。

如果这个时候输入`ip addr show`查看网络接口：
```
/ # ip addr sh
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
```

可以看到只有一个默认的loopback网络接口，并没有eth0，那是因为我们的脚本只是简单地显示了日志，并没有实际创建网络接口。当然，如果你有兴趣的话，也可以对这个插件进行改造，让它具备更加丰富的功能。比如：利用`ip link add`命令为容器创建真正的网络接口。

接下来，我们输入`Ctrl + ]]]`从容器退回到宿主机。然后执行`rkt rm`并传入容器Id，把容器删除掉。这里的容器Id，可以从前面执行ADD命令时的输出结果里找到。并且，我们只要从完整的Id值里取前面几位就可以了：
```shell
$ rkt rm 1ebcb360
Command: DEL
Container Id: 1ebcb360-bcf6-4c3f-b38e-a8826eac7f9b
Path to Network Namespace: /var/run/netns/cni-d59164ee-3cc1-0a62-a4ec-b486b3028dae
Network Interface: eth0
Path to CNI Plugins: /etc/rkt/net.d:/usr/lib/rkt/plugins/net:stage1/rootfs/usr/lib/rkt/plugins/net
Network Configuration:
{
  "cniVersion": "0.4.0",
  "name": "lab-test-net-1",
  "type": "logger"
}
"1ebcb360-bcf6-4c3f-b38e-a8826eac7f9b"
```

可以看到，我们的logger插件再一次被调用了，这次执行的是DEL命令。
