---
title: Docker Registry CLI轻松管理Docker注册表(下)
excerpt: "更多Docker Registry CLI的灵活用法，如：批量复制与删除等"
categories: [tech]
tags: [docker, shell, studio]
toc: true
toc_sticky: true
---
> 继开源项目[Elastic Shell](https://github.com/morningspace/elastic-shell)[发布](/tech/elash-and-studio)以后，[晴耕工坊](/studio)近期又有新品发布啦！今天，我们继续来了解[Docker Registry CLI](https://github.com/morningspace/docker-registry-cli)这款开源小工具。

![](/assets/images/studio/docker_registry.png)

在[上一篇文章](/tech/docker-registry-cli-1/)中，我们了解了什么是Docker Registry CLI，如何运行，以及复制与查询Image的一些基本方法。在本文中，我们将进一步介绍如何从多个Registry批量复制Image到目标Registry，更加灵活丰富的查询功能，以及如何清理Registry上的Image。

## 复制更多的Image

目前为止，我们已经成功将第一个Image复制到`mr.io`上了。接下来，让我们再多复制一些Image。`reg-cli cp`支持一次复制多个Image，只要把它们在一行命令里用空格隔开即可。例如：

```shell
bash-4.4# reg-cli cp morningspace/docker-registry-cli morningspace/elastic-shell mr.io
```

上述命令会同时将`docker-registry-cli`和`elastic-shell`复制到`mr.io`。

如果在`reg-cli ls`命令后面也加上多个Image，并以空格间隔，我们就可以查看多个Image的Tag了。例如：

```shell
bash-4.4# reg-cli ls mr.io/docker-registry-cli mr.io/elastic-shell
```

当然，也可以利用`-ld`和`-lm`选项，查看这些Image的Digest和Manifest。

### 利用.list文件批量复制

除此以外，`reg-cli cp`还支持从配置文件读取要复制的Image。将存储Image的源Registry名作为配置文件名，以`.list`为扩展名，在配置文件里指定所有要复制的Image，每行一个；然后，在执行`reg-cli cp`时指定源Registry名，Docker Registry CLI就会在当前目录下寻找同名的`.list`文件，逐行读取进行复制。

在[GitHub库](https://github.com/morningspace/docker-registry-cli)的`/samples/registries`子目录下包含有若干个可供大家参考的`.list`文件。以`morningspace.list`为例，包含了许多以`morningspace`打头的Image。它们都来自Docker Hub的[`morningspace`](https://hub.docker.com/u/morningspace)名字空间下：

```
morningspace/lab-web
morningspace/lab-lb
morningspace/lab-git-remote
morningspace/lab-git-local
morningspace/elastic-shell
morningspace/docker-registry-cli
```

在该文件所在目录下执行如下命令，就可以把上述Image批量推送到`mr.io`：

```shell
bash-4.4# reg-cli cp morningspace mr.io
```

不仅如此，`.list`文件还支持嵌套！例如，和`morningspace.list`位于同一目录下的`all.list`包含了针对另两个`.list`文件的引用，分别指向Docker Hub和Google Container Registry：

```
k8s.gcr.io
morningspace
```

执行如下命令，可以将分别来自这两个不同Registry的Image复制到`mr.io`：

```shell
bash-4.4# reg-cli cp all mr.io
```

等待命令执行完毕后，再利用`reg-cli ls`查看`mr.io`的Catalog。我们会发现，在`all.list`文件所指向的另两个`.list`文件里定义的所有Image都已经被成功复制到`mr.io`了！

```shell
bash-4.4# reg-cli ls mr.io
mr.io
---------------------------------------
coredns
docker-registry-cli
elastic-shell
etcd
hyperkube
kubernetes-dashboard-amd64
lab-git-local
lab-git-remote
lab-lb
lab-web
pause
```

同样的，我们可以通过在`reg-cli ls`命令后面加上多个Image来查看这些Image的Tag，也可以利用`-ld`和`-lm`选项，查看它们的Digest和Manifest。

我们甚至还可以查看整个`me.io`上所有Image的Tag（通过`-lt`选项），Digest和Manifest信息。例如：

```shell
bash-4.4# reg-cli ls mr.io -lt
mr.io/coredns
---------------------------------------
1.2.6

mr.io/docker-registry-cli
---------------------------------------
latest

mr.io/elastic-shell
---------------------------------------
latest
v0.3
...
```

### 最少的命令参数完成最多的功能

通过对`reg-cli ls`和`reg-cli cp`命令的使用我们可以看到，Docker Registry CLI虽然只提供了很少的命令及参数，但是通过这些命令及其参数的组合，却完成了非常丰富的功能。

接下来，我们再来看另一个命令`reg-cli rm`，该命令用于删除Registry上的Image或Tag。同样，该命令也是符合这一设计原则的。

## 清理Image

利用`reg-cli rm`命令，我们可以删除位于`mr.io`上的Image及其Tag。例如，下列命令可以删除指定Image的某个Tag：

```shell
bash-4.4# reg-cli rm mr.io/hyperkube:v1.13.0
reg-cli:(INFO) going to remove hyperkube:v1.13.0
Are you sure? [y/N] y
reg-cli:(INFO) digest: sha256:8ef688eb5bb739e7f22502e15b78eb6bd684d2957dfda7c64638cec0e9eacb3d
reg-cli:(INFO) ✔ hyperkube:v1.13.0 removed
```

根据提示，输入`y`后即可删除`mr.io/hyperkube:v1.13.0`。

同样的，`reg-cli rm`命令也可以一次删除同一Image的多个Tag，只要在Image后面跟多个Tag，并以逗号间隔即可：

```shell
bash-4.4# reg-cli rm mr.io/hyperkube:v1.13.0,v1.12.1
```

如果Image后面没有跟任何Tag，则Docker Registry CLI会试图删除该Image的所有Tag：

```shell
bash-4.4# reg-cli rm mr.io/hyperkube
```

此外，如果不希望每次删除一个Image或Tag时都显示提示信息，可以加上`-f`选项，指定强制删除：

```shell
bash-4.4# reg-cli rm mr.io/hyperkube:v1.13.0,v1.12.1 -f
```

### 严格遵循Linux命令的语法

通过前面针对`reg-cli cp`，`reg-cli ls`，及`reg-cli rm`的使用，我们可以看到，Docker Registry CLI的命令语法非常类似于Linux中`cp`，`ls`，及`rm`命令。这是在该工具设计之初就恪守的一个原则。因为遵循了这一原则，所以只要熟悉这些常用的Linux命令，用户就很容易掌握对Docker Registry CLI的使用方法。

好了，有关Docker Registry CLI的使用就介绍到这里了。

关于Docker Registry CLI，如果你有任何疑问，或者希望通过贡献代码来增强这一工具，欢迎发送邮件至`morningspace@yahoo.com`。更多详情请访问[GitHub库](https://github.com/morningspace/docker-registry-cli)及[晴耕工坊](/studio)的主页。

---
*Have fun!*

*MorningSpace Studio*