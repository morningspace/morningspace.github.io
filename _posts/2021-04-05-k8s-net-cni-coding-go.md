---
title: Kubernetes网络篇——自己动手写CNI插件(下)
excerpt: "用go写一个接近真实更复杂的CNI插件，加深对CNI开发的理解"
categories: [tech]
tags: [lab, dummies, dummies_kubernetes, kubernetes]
toc: true
toc_sticky: true
---

注：
本文采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/80x15.png" /></a>

> 本文，我们将用go语言，结合CNI项目提供的go工具库，来开发一个接近真实更加复杂的插件。它和CNI的标准插件bridge类似，会为我们在宿主机上创建一个bridge，并把容器连接到bridge上。

![](/assets/images/lab/k8s/cni-go.png){: .align-center}

## 第二个插件：simple-bridge

在[Kubernetes网络篇——自己动手写CNI插件(上)](/tech/k8s-net-cni-coding-shell/)一文里，我们通过一个简单的例子演示了用shell脚本写的CNI插件，并成功地和rkt实现了互动。接下来，我们再用go语言来写一个更接近真实的CNI插件。

应该说，使用go来编写CNI插件是最顺理成章的。因为，CNI的GitHub项目提供了基于go语言的工具库，再加上go社区本身就有很多工具库，开发人员可以对Linux的网络功能进行各种丰富的控制。

我们知道，通常CNI插件会在容器启动和删除的时候被调用。在容器启动时，插件会为容器分配相应的网络资源，比如：网络连接，IP地址等。在容器删除时，插件会回收之前所创建的这些网络资源。我们即将要实现的这个插实际上是一个简化版的bridge插件，它会在容器启动时为我们在宿主机上创建一个bridge，并把容器连接到bridge上，最后再为容器分配好IP地址。

## 准备工作

使用go语言开发需要准备好go的开发环境，因为等一下要用rkt来验证我们开发的插件，所以和之前一样，我会继续使用Debian环境。由于go环境的安装不是本文的关注点，所以这里就不展开了。

为了测试我们开发的插件，还需要一个网络配置文件：
```json
{
    "cniVersion": "0.4.0",
    "name": "lab-cni-mybr-net",
    "type": "simple-bridge",
    "bridge": "lab-cni-mybr",
    "ip": "10.15.30.100/24"
}
```
这个文件非常简单，除了几个标准属性，如：CNI版本号，网络名称，插件类型（simple-bridge）外，还有两个特定于插件的参数。其中，`bridge`是宿主机上的bridge名称，`ip`是veth pair位于容器一端的IP地址。我们把配置文件取名为`lab-cni-mybr-net.conf`，存到`/etc/rkt/net.d`目录下。

## 代码实现

下面我们就一起来看一下插件的代码实现。

### 主体结构

首先来看一下代码的主体结构：
```go
Package main

Import (
  ... ...
)

func cmdAdd(args *skel.CmdArgs) error {
  ... ...
}

func cmdCheck(args *skel.CmdArgs) error {
  return nil
}

func cmdDel(args *skel.CmdArgs) error {
  return nil
}

func main() {
  skel.PluginMain(cmdAdd, cmdCheck, cmdDel, version.All, bv.BuildString("simple-bridge"))
}
```

每一个go实现的CNI插件都具备如上的代码结构，除了用`import`指令把需要的依赖引入进来以外，还需要实现三个函数，分别对应CNI的三个命令：
* cmdAdd
* cmdCheck
* cmdDel

然后在`main`函数里，通过CNI工具库提供的`skel.PluginMain`函数，把这三个函数注册进来。`skel.PluginMain`的倒数第二个参数用来指定插件所支持的CNI规范的版本，这里我们选择的是所有版本；最后一个参数通过调用工具类生成一段简单的帮助文本，如果在命令行直接执行插件的可执行文件，就会看到这段文本。

另外，为了能在代码里访问到插件所需的环境变量，以及通过标准输入设备传入的配置文件，我们使用了`skel`提供的数据类型`CmdArgs`：
```go
type CmdArgs struct {
    ContainerID string
    Netns string
    IfName string
    Args string
    Path string
    StdinData []byte
}
```

可以看到，`CmdArgs`里大部分属性都分别对应相应的环境变量，最后一个属性代表通过标准输入设备传入的一个字节数组，对应于我们的网络配置文件。

我们的插件实现了CNI里的ADD命令，主要完成下面几个步骤：

* 读取网络定义的配置文件
* 配置宿主机上的bridge
* 创建veth pair，并把宿主机一端连到bridge上
* 为容器端的网络接口分配指定的IP地址

接下来，我们将针对上面几个步骤，逐一进行讲解。

### 第一步：读取配置文件

为了从配置文件里读取网络配置，我们定义了一个struct类型的数据结构。因为只关心网络配置文件里的`bridge`和`ip`，所以这里只定义了两个属性：
```go
type NetConf struct {
  BrName    string `json:"bridge"`
  IPAddress string `json:"ip"`
}
```

然后，我们在函数`loadNetConf`里实现了读取配置文件的逻辑：
```go
func loadNetConf(bytes []byte) (*NetConf, error) {
  n := &NetConf{}
  if err := json.Unmarshal(bytes, n); err != nil {
    return nil, err
  }
  return n, nil
}
```

这里，我们先创建了一个NetConf类型的变量`n`，然后调用`json.Unmarshal`函数，把来自标准输入设备的字节数据读入了这个变量。

### 第二步：配置bridge

我们把对bridge的配置封装到了`setupBridge`函数里，它的输入参数就是前面调用`loadNetConf`的返回结果，也就是我们的网络配置信息。下面是函数的实现逻辑，这里主要利用了`netlink`包对bridge进行操作：
```go
func setupBridge(n *NetConf) (*netlink.Bridge, error) {
  br := &netlink.Bridge{                          ①
    LinkAttrs: netlink.LinkAttrs{
      Name: n.BrName,
      MTU:  1500,
      TxQLen: -1,
    },
  }

  err := netlink.LinkAdd(br)                      ②
  if err != nil && err != syscall.EEXIST {
    return nil, err
  }

  br, err = bridgeByName(n.BrName)                ③
  if err != nil {
    return nil, err
  }

  if err := netlink.LinkSetUp(br); err != nil {   ④
    return nil, err
  }

  return br, nil
}
```

行①：创建`netlink.Bridge`对象，传入在配置文件里定义的`bridge`属性；

行②：调用`netlink.LinkAdd`函数，传入`netlink.Bridge`对象，把bridge创建出来，相当于执行`ip link add`命令。如果创建失败时返回`syscall.EEXIST`，表示这个bridge已经存在了，这种情况我们认为是正常的；

行③：调用`bridgeByName`函数，重新读取一次新建的bridge，用于验证其是否被正确创建出来了；

行④：如果一切正常，则调用`netlink.LinkSetUp`函数，把bridge启动起来，相当于执行`ip link set ... up`命令；

下面是`bridgeByName`函数的实现逻辑：
```go
func bridgeByName(name string) (*netlink.Bridge, error) {
  l, err := netlink.LinkByName(name)  ①
  if err != nil {
    return nil, err
  }
  br, ok := l.(*netlink.Bridge)       ②
  if !ok {
    return nil, fmt.Errorf("%q already exists but is not a bridge", name)
  }
  return br, nil
}
```

行①：调用`netlink.LinkByName`函数，传入bridge的名称，重新读取一次新建的bridge；

行②：尝试把读取出来的对象转换成`netlink.Bridge`类型，如果失败，就表示读出来的对象不是一个bridge，于是返回错误；

### 第三步：配置veth pair

创建veth pair的逻辑被封装到了`setupVeth`函数里。它包含如下几个参数：
* netns：容器对应的network namespace，类型为`NetNS`，属于CNI提供的工具包`ns`里的一个数据结构；
* br：位于宿主机上的bridge；
* ifName：veth pair位于容器一端的网络接口名；
* ipAddress：容器端网络接口的IP地址；

下面来看一下函数的实现逻辑：
```go
func setupVeth(netns ns.NetNS, br *netlink.Bridge, ifName string, ipAddress string) error {
  hostIface := &current.Interface{}                           ①

  err := netns.Do(func(hostNS ns.NetNS) error {               ②
    hostVeth, containerVeth, err := 
      ip.SetupVeth(ifName, 1500, hostNS)                      ③
    if err != nil {
      return err
    }
    hostIface.Name = hostVeth.Name                            ④

    return nil
  })
  if err != nil {
    return err
  }

  hostVeth, err := netlink.LinkByName(hostIface.Name)         ⑤
  if err != nil {
    return err
  }

  if err := netlink.LinkSetMaster(hostVeth, br); err != nil { ⑥
    return err
  }

  return nil
}
```

行①：这里用到了CNI提供的`types/current`包，`Interface`是一个struct类型的数据结构，对应一个网络接口。我们用它来保存veth pair位于宿主机一端的网络接口信息，以方便后面的调用；

行②：这里的`netns`就是容器的network namespace，对应类型为`NetNS`。 `NetNS`有一个特殊的函数`Do`，它接受一个函数闭包作为参数，而我们设置veth pair的逻辑就在这个闭包里；

对veth pair的配置是在容器的network namespace，即`netns`下进行的。在go里，每个操作系统线程都可能对应不同的network namespace，但由于go线程调度机制的特点，在我们的代码逻辑开始执行的时候，并不能够保证当前的network namespace就一定是容器所对应的那个，而`Do`函数就是用来解决这一问题的：

它通过调用`runtime.LockOSThread()`函数，锁定了执行闭包的当前线程。Do函数在执行闭包之前，会先保存当前线程的network namespace，也就是宿主机对应的namespace，并作为参数传入闭包。然后在闭包执行结束时，把当前network namespace恢复成宿主机对应的namespace。

行③：这里我们调用了`SetupVeth`函数，该函数位于CNI提供的`ip`包里。利用它，我们在容器里创建了veth pair，把容器端的网络接口名设置为`ifName`，并把宿主机一端的网络接口“移入”了宿主机的namespace里，即:`hostNS`；

行④：`ip.SetupVeth`的返回结果里包含了宿主机一端的网络接口信息。这里，我们把接口名称保存到了前面行①处定义的`hostIface`里，以方便后面使用；

行⑤：通过调用`netlink.LinkByName`函数，并传入在行④处获取到的宿主机一端的网络接口名，我们拿到了宿主机一端的网络接口对象`hostVeth`；

行⑥：通过调用`netlink.LinkSetMaster`函数，并传入`hostVeth`，我们把veth pair的宿主机一端连到了bridge上；

### 第四步：分配IP地址

下面我们来为容器端的网络接口分配IP地址。这部分逻辑，被放在了传入`netns.Do`的那个闭包里，`ip.SetupVeth`调用的后面：
```shell
  ipv4Addr, ipv4Net, err := net.ParseCIDR(ipAddress)  ①
  if err != nil {
    return err
  }
  ipv4Net.IP = ipv4Addr
  addr := &netlink.Addr{IPNet: ipv4Net}               ②

  link, err := netlink.LinkByName(containerVeth.Name) ③
  if err != nil {
    return err
  }

  if err = netlink.AddrAdd(link, addr); err != nil {  ④
    return err
  }
```

行①：通过调用`net.ParseCIDR`函数，我们对传入`setupVeth`的`ipAddress`进行解析，并生成`ipv4Addr`和`ipv4Net`对象，分别对应IP地址和所在网段。这里的`ipAddress`就是前面在网络配置文件里定义的`ip`属性；

行②：根据前面的解析结果，构造一个`netlink.Addr`对象，对应容器端网络接口的IP地址；

行③：调用`netlink.LinkByName`函数，传入`containerVeth.Name`，获得容器端的网络接口对象。这里的`containerVeth`是前面调用`ip.SetupVeth`时返回的；

行④：调用`netlink.AddrAdd`函数，分别传入行③处获得的容器端网络接口和行②处构造的IP地址对象，完成IP地址的真正分配；

### 把所有代码串起来

前面我们讲了每一步的实现逻辑，接下来就可以把这些逻辑在`cmdAdd`函数里串起来了：
```go
func cmdAdd(args *skel.CmdArgs) error {
  n, err := loadNetConf(args.StdinData)     ①
  if err != nil {
    return err
  }

  br, err := setupBridge(n)                 ②
  if err != nil {
    return err
  }

  netns, err := ns.GetNS(args.Netns)        ③
  if err != nil {
    return err
  }
  defer netns.Close()

  if err := setupVeth(netns, br,            ④
    args.IfName, n.IPAddress); err != nil {
    return err
  }

  result := current.Result{}
  return result.Print()
}
```

可以看到，这里的行①，②，④分别对应的就是我们前面提到的四个步骤。而行③是为调用`setupVeth`所做的准备，它利用`ns.GetNS`函数获取到当前容器的network namespace，并作为参数传入了`setupVeth`里。

最后，前面已经讨论过，为了确保线程不会在我们对network namespace进行操作期间被切换，从而导致namespace被切换，我们在`init`函数里加入了对`runtime.LockOSThread()`的调用，`init`函数会在`main`函数执行之前被自动调用：
```go
func init() {
  runtime.LockOSThread()
}
```

## 代码运行

在运行插件以前，我们需要先把代码编译成可执行文件。为了成功编译，可以在项目所在目录下调用`go get`命令，把相关的依赖下载下来：
```shell
$ go get -d -v ./...
```

然后执行`go build`把插件代码编译成二进制文件：
```shell
$ go build simple-bridge.go
````

在当前目录下找到编译生成的可执行文件`simple-bridge`，然后把它拷贝到`/etc/rkt/net.d`目录下。再执行下面命令启动一个新的容器，并指定`--net`参数的值为`lab-cni-mybr-net`：
```shell
$ rkt run --insecure-options=image --interactive --net=lab-cni-mybr-net docker://busybox
```

如果一切正常，我们会进入到busybox容器内部，执行`ip addr show`命令：
```
/ # ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
3: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue 
    link/ether 86:c8:3c:a7:23:23 brd ff:ff:ff:ff:ff:ff
    inet 10.15.30.100/24 brd 10.15.30.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::84c8:3cff:fea7:2323/64 scope link 
       valid_lft forever preferred_lft forever
```

可以看到，作为veth pair在容器一端的网络接口`eth0`，它的IP地址已经被成功设置成我们指定的值了(`10.15.30.100`)。

输入`Ctrl + ]]]`退回到宿主机后，再执行`ip addr show`：
```shell
$ ip addr show
... ...
5: lab-cni-mybr: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 0a:67:c4:8f:93:12 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::6437:83ff:fe38:507b/64 scope link 
       valid_lft forever preferred_lft forever
8: veth95dc17da: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master lab-cni-mybr state UP group default 
    link/ether 0a:67:c4:8f:93:12 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::867:c4ff:fe8f:9312/64 scope link 
       valid_lft forever preferred_lft forever
```

可以看到，宿主机上多了一个名为lab-cni-mybr的bridge，还有作为veth pair在宿主机一端的网络接口veth95dc17da。
