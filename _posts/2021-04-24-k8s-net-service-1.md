---
title: Kubernetes网络篇——Service网络(上)
excerpt: "学习Service的定义，部署和访问，认识Kubernetes的DNS服务"
categories: [tech]
tags: [lab, dummies, dummies_kubernetes, kubernetes]
toc: true
toc_sticky: true
---

注：
本文采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。

> 作为Kubernetes最基本的执行单元，Pod是Kubernetes网络的基础。但它也是有局限的，我们访问Pod用的是IP，当Pod出问题，或因重启而发生IP地址变化时，原来的IP就失效了，这时就需要Service了。

![](/assets/images/lab/k8s/service-1.png){: .align-center}

## Pod和Service

在[Kubernetes网络篇——Pod网络(上)](/tech/k8s-net-pod-1/)和[Kubernetes网络篇——Pod网络(下)](/tech/k8s-net-pod-2/)里，我们已经了解了Pod网络的工作原理。如果集群里有一个Pod想访问另一个Pod里的服务，那它就需要知道这个Pod的IP地址。这种做法的问题在于，它没有办法适应动态变化的情况：Pod可能会因为故障而被重启，而重启之后的Pod，其IP地址以及所在的节点，都有可能会改变。

在Kubernetes里，我们是通过Service来解决这一问题的。Kubernetes允许我们为Pod添加标签（label），并通过标签来对Pod进行分组。在定义Service时，可以通过声明`selector`把Service和一组Pod关联起来。每个Service都有一个属于自己的虚拟IP（Virtual IP，简称VIP）作为前端（front-end），它可以和后端（back-end）的一组Pod相关联，并提供负载均衡的能力。

下面，我们通过对一个Service的部署，来学习访问Service的几种方法。

## 定义Service

这里，我们继续沿用之前基于[kubeadm-dind-cluster](https://github.com/kubernetes-sigs/kubeadm-dind-cluster)的实验环境，有关如何使用kubeadm-dind-cluster，以及笔者对它所做的优化，可以参考Kuberntes系列的热身篇[Launch multi-node Kubernetes cluster locally in one minute, and more…](/tech/k8s-run/)一文。假定大家在阅读前面有关Pod网络的文章时，已经在实验环境里部署过`test-pod`这个Pod了，接下来我们为Pod定义一个Service：
```yaml
kind: Service
apiVersion: v1
metadata:
  name: test-svc
spec:
  selector:
    app: lab-web
  ports:
    - port: 80
      targetPort: web-port
```

这里，Service的名字叫`test-svc`。通过`selector`，我们定义了所有`app`标签为`lab-web`的Pod和`test-svc`相关联。除此以外，我们还对端口号做了定义：
* `port`：定义了作为Service前端的虚拟IP所监听的端口号；
* `targetPort`：指明了与Service关联的后端Pod所监听的端口号。

在定义`targetPort`的时候，我们没有直接引用端口号的具体值，而是用了端口名称。如果大家留意前面的文章，我们在为`test-pod`定义端口的时候，除了指定端口号的具体值以外，还为它指定了一个名称。这样，即使端口号发生变化，只要名称不变，其他引用或依赖这个端口的地方就都不用改变；

我们把上述内容以文件形式保存到实验环境里master节点上的`/root/test-k8s-net`目录下，并取名`test-svc.yaml`。然后执行如下命令进行部署：
```shell
$ kubectl create -f test-svc.yaml
service "test-svc" created
```

通过执行`kubectl get services`命令可以看到，我们的Service已经成功部署到集群里了，它的虚拟IP地址为`10.98.114.171`：
```shell
$ kubectl get services
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   22d
test-svc     ClusterIP   10.107.169.79   <none>        80/TCP    14s
```

## 从节点访问

我们可以在集群里的任何一个节点上执行`curl`命令对这个Service进行访问。比如在master节点上：
```shell
$ curl http://10.107.169.79
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
... ...
```

## 从Pod访问

我们也可以通过集群里的其他Pod访问这个Service。这里我们使用`mr.io/lab-sleeper`镜像生成的Pod进行测试：
```dockerfile
FROM centos

RUN yum -y update && yum -y install iproute bind-utils && yum clean all

CMD ["/bin/bash", "-c", "echo Sleeping...; trap : TERM INT; sleep 365d & wait"]
```

为了方便在本地做测试的时候，能够让集群里的节点快速拉取镜像，我们把镜像在本地构建出来以后，推送到了本地的私有Docker注册表`mr.io`上：
```shell
$ docker build -t mr.io/lab-sleeper .
$ docker push mr.io/lab-sleeper
```

然后利用`kubectl run`命令，把`mr.io/lab-sleeper`镜像部署到集群里：
```shell
$ kubectl create deployment lab-sleeper --image=mr.io/lab-sleeper
deployment.apps/lab-sleeper created

$ kubectl get pods -o wide
NAME                           READY   STATUS    RESTARTS   AGE     IP           NODE          NOMINATED NODE   READINESS GATES
lab-sleeper-7ff95f64d7-6p7bh   1/1     Running   0          28s     10.244.3.5   kube-node-2   <none>           <none>
test-pod-9dd7d4f7b-56znc       1/1     Running   0          3h52m   10.244.2.4   kube-node-1   <none>           <none>
```

现在，我们可以在lab-sleeper对应的Pod里，通过Service的虚拟IP地址，访问lab-web对应的Pod了：
```shell
$ kubectl exec lab-sleeper-7ff95f64d7-6p7bh curl http://10.107.169.79
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0<!DOCTYPE html>
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
... ...
```

## 通过DNS访问

我们还可以借助Kubernetes提供的DNS服务——kube-dns，通过Service名称来访问Service。这样，我们就不需要知道Service的IP地址了，因为IP地址是可以由kube-dns通过Service名称解析得到的。
```shell
$ kubectl exec lab-sleeper-7ff95f64d7-6p7bh curl http://test-svc     
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   612  100   612    0     0  27818      0 --:--:-- --:--:-- --:--:-- 27818
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
... ...
```

### kube-dns

和普通应用一样，Kubernetes的DNS服务也是通过Pod + Service的方式部署到集群里的，只不过它是位于kube-system名字空间下的。所有Kubernetes提供的系统级服务都被部署在这个名字空间下。
```shell
$ kubectl get pods -n kube-system
NAMESPACE     NAME                                  READY     STATUS    RESTARTS   AGE
coredns-fb8b8dccf-nggnj               1/1     Running   0          47m
etcd-kube-master                      1/1     Running   3          46m
kube-apiserver-kube-master            1/1     Running   3          46m
kube-controller-manager-kube-master   1/1     Running   3          46m
kube-proxy-f2d74                      1/1     Running   5          22d
kube-proxy-px4g4                      1/1     Running   5          22d
kube-proxy-qkvwj                      1/1     Running   5          22d
kube-scheduler-kube-master            1/1     Running   3          46m
```

从Kubernetes v1.12开始，CoreDNS替代了原来的Kube-DNS成为Kubernetes推荐的默认DNS方案，目的是为了解决Kube-DNS在性能，安全性，以及稳定性方面的缺陷。但为了保持兼容，Service名称还是沿用的kube-dns。因为我本地的实验环境选择了v1.13版本的Kubernetes，所以我们看到kube-system里部署的Pod是coredns，但Service却是kube-dns：
```shell
$ kubectl get services -n kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   22d
```

当然，你也可以选择其他版本，[kubeadm-dind-cluster](https://github.com/kubernetes-sigs/kubeadm-dind-cluster)支持多个Kubernetes版本的快速部署，具体方法参见Kuberntes系列的热身篇[Launch multi-node Kubernetes cluster locally in one minute, and more…](/tech/k8s-run/)一文。

### resolv.conf

当我们对Service有创建，更新，或删除操作的时候，kube-dns会监听到来自Kubernetes API的事件，并根据需要更新DNS记录。每个Pod的容器里都有一个文件叫`/etc/resolv.conf`，其中包含了DNS的配置信息，比如：
```shell
$ kubectl exec lab-sleeper-7ff95f64d7-6p7bh cat /etc/resolv.conf     
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

kubelet会负责修改这个文件，把里面的nameserver设置成kube dns的cluster IP，在我们的例子是`10.96.0.10`，并更新搜索域名(search domain)，利用搜索域名，我们可以在用名称访问Pod时使用更短形式的主机名(hostname)。

kube-dns服务的Cluster IP和搜索域名，都可以在启动kubelet时通过参数来指定。比如，如果在我们的实验环境里查看kubelet的启动参数：
```shell
$ ps axuw | grep cluster-dns
root       351  4.7  1.4 925132 119008 ?       Ssl  02:54   2:27 /k8s/hyperkube kubelet --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --cgroup-driver=cgroupfs --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.1 --pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true --network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin --cluster-dns=10.96.0.10 --cluster-domain=cluster.local --eviction-hard=memory.available<100Mi,nodefs.available<100Mi,nodefs.inodesFree<1000 --fail-swap-on=false --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --feature-gates=DynamicKubeletConfig=true --v=4
root     22649  0.0  0.0  11108   944 pts/0    S+   03:46   0:00 grep cluster-dns
```

就会发现，里面有和DNS相关的参数：
* `--cluster-dns`，DNS服务的Cluster IP，我们的例子里是`10.96.0.10`；
* `--cluster-domain`，集群使用的域名，我们的例子里是`cluster.local`；

### 使用nslookup

把`resolv.conf`文件里定义的搜索域名作为后缀，我们在访问Service的时候就不用给出完整的域名，而是只要提供Service的名称就可以了。

比如，当我们用`nslookup`命令查询test-svc服务的域名和IP地址时，可以直接使用Service的名称`test-svc`作为输入参数：
```shell
$ kubectl exec lab-sleeper-7ff95f64d7-6p7bh nslookup test-svc
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	test-svc.default.svc.cluster.local
Address: 10.107.169.79
```

返回结果里包含了test-svc的完整域名和IP地址。其中：
* `default`代表名字空间，我们的test-svc是部署在`default`下的；
* `svc`代表Service；
* `cluster.local`就是前面`--cluster-domain`参数定义的值；

这样一来，跑在容器里的应用程序就可以通过`test-svc`解析出对应的IP地址了。

下图给出了test-pod以及test-svc的网络拓扑。需要注意的是，和Pod以及容器不同，Service在Kubernetes里更多的是逻辑上的概念，所以它在网络拓扑里并没有物理实体存在。

![](/assets/images/lab/k8s/test-svc-1.png)
