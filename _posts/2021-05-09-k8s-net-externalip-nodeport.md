---
title: Kubernetes网络篇——ExternalIP和NodePort
excerpt: "利用ExternalIP和NodePort，从集群外部访问Service"
categories: [tech]
tags: [lab, dummies, dummies_kubernetes, kubernetes]
toc: true
toc_sticky: true
---

注：
本文采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。

> Service的虚拟IP为Pod在集群内部的互通提供了便利，而ExternalIP和NodePort，则让我们从集群外部对Service的访问成为了可能。

![](/assets/images/lab/k8s/externalip-nodeport.png){: .align-center}

在[Kubernetes网络篇——Service网络(上)](/tech/k8s-net-service-1)和[Kubernetes网络篇——Service网络(下)](/tech/k8s-net-service-1)里，我们了解了Kubernetes的Service网络。Service不仅实现了多Pod之间的负载均衡，而且还提供了虚拟IP，使Pod在集群内可以通过虚拟IP实现相互通信，而又不用担心Pod重启导致的IP地址变化。

但是，Service的虚拟IP只有在集群内部才有效，因此也被称为Cluster IP。对于集群以外的客户端，它们是无法通过Cluster IP访问到Service的。如果我们想从集群外部对Service进行访问，那就需要借助其他手段了。

## External IP

所谓External IP，就是为Service设置一个能够在集群外访问的IP地址。只要我们能确保，访问这个IP的数据包能够从集群外路由到集群内的某个节点上，再往后，就是集群内Service的常规通信，就可以运用我们在前面介绍Service网络时掌握的知识了。

下面我们就来为test-svc设置一个External IP，看看`iptables`规则会有哪些变化。为了方便后面分析比较，在修改test-svc的配置前，我们先把`iptables`当前的规则集保存到一个文件里：
```shell
$ iptables-save >rules-0
```

然后，执行`kubectl edit`命令，动态修改test-svc的配置：
```shell
$ kubectl edit service test-svc
```

这个时候，kubectl会自动打开默认的文本编辑器，比如在我本地会打开vi，里面包含了test-svc的当前配置。我们要做的改动非常小，只要在`spec`下面增加一项`externalIPs`，填入我们预先规划好的IP地址，比如`192.168.96.10`：
```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"test-svc","namespace":"default"},"spec":{"ports":[{"port":80,"targetPort":"web-port"}],"selector":{"app":"lab-web"}}}
  creationTimestamp: "2019-06-24T03:30:08Z"
  name: test-svc
  namespace: default
  resourceVersion: "117804"
  selfLink: /api/v1/namespaces/default/services/test-svc
  uid: 5d85e807-9630-11e9-ab86-82d6af7a4ac8
spec:
  clusterIP: 10.107.169.79
  externalIPs:
  - 192.168.96.10
  ports:
  - port: 80
    protocol: TCP
    targetPort: web-port
  selector:
    app: lab-web
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```

保存退出以后，再执行`kubectl get services`，确保test-svc的配置已经成功得到了更新：
```shell
$ kubectl get services -o wide
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP     PORT(S)   AGE   SELECTOR
kubernetes   ClusterIP   10.96.0.1       <none>          443/TCP   24d   <none>
test-svc     ClusterIP   10.107.169.79   192.168.96.10   80/TCP    2d    app=lab-web
```

从输出结果里我们可以看到，test-svc除了`CLUSTER-IP`一栏里有供集群内部访问的Cluster IP外，在`EXTERNAL-IP`一栏还列出了我们刚才设置的External IP。

接下来，我们再次执行`iptables-save`，并把输出结果存成文件：
```shell
$ iptables-save >rules-1
```

然后对前后两份`iptables`规则集进行对比，从中我们会看到下面几点变化：
```shell
⎢ -A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000
⎢ -A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -m mark --mark 0x4000/0x4000 -j MASQUERADE
⎢ ...
① -A KUBE-SERVICES -d 192.168.96.10/32 -p tcp -m comment --comment "default/test-svc: external IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
② -A KUBE-SERVICES -d 192.168.96.10/32 -p tcp -m comment --comment "default/test-svc: external IP" -m tcp --dport 80 -m physdev ! --physdev-is-in -m addrtype ! --src-type LOCAL -j KUBE-SVC-W3OX4ZP4Y24AQZNW
③ -A KUBE-SERVICES -d 192.168.96.10/32 -p tcp -m comment --comment "default/test-svc: external IP" -m tcp --dport 80 -m addrtype --dst-type LOCAL -j KUBE-SVC-W3OX4ZP4Y24AQZNW
⎢ ...
⎢ -A KUBE-SVC-W3OX4ZP4Y24AQZNW -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-E2HMOHPUOGTHZJEP
⎢ -A KUBE-SVC-W3OX4ZP4Y24AQZNW -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-WFXGQBTRL5EC2R2Y
⎢ -A KUBE-SVC-W3OX4ZP4Y24AQZNW -j KUBE-SEP-EEXR7SABLH35O4XP
```

其中，行①到行③是新增的规则，都是匹配目标地址为`192.168.96.10`（也就是我们设置的External IP）以及端口号为80的数据包的。

* 行①处的规则会跳转到KUBE-MARK-MASQ，也就是对数据包的源地址进行地址伪装。关于地址伪装，可以在[Kubernetes网络篇——Service网络(下)](/tech/k8s-net-service-2)一文里找到更多细节，稍后我还会解释之所以要地址伪装的原因；
* 在KUBE-MARK-MASQ为数据包设好地址伪装的标记以后，又会跳回行②和行③，继续匹配后面的规则；
* 行②处的规则表示，如果数据包不是从bridge接口进来的(`! --physdev-is-in`)，同时源地址也不是LOCAL类型的(`! --src-type LOCAL`)，则匹配该规则，并跳转到KUBE-SVC-W3OX4ZP4Y24AQZNW链。这里，不从bridge接口进来，就说明访问Service的数据包一定不是来自Pod的；源地址不是LOCAL类型，则说明数据包一定不是由集群里的主机发起的；
* 行③处的规则表示，如果数据包的目标地址是LOCAL类型的(`--dst-type LOCAL`)，则匹配该规则，并跳转到KUBE-SVC-W3OX4ZP4Y24AQZNW链。这说明我们为Service指定的External IP本身就是一个local地址，也就是某个本机网卡的IP地址；

这两条规则实际上是要把数据包的来源限定在集群以外，也就是External IP要解决的典型场景。当跳转到KUBE-SVC-W3OX4ZP4Y24AQZNW链以后，再往后就和普通的集群内数据包处理逻辑完全一样了，这里就不再啰嗦了。

### 地址伪装的作用

这里再解释一下地址伪装的重要性，我们先来看如果没有地址伪装会怎么样。

如果没有地址伪装，那么数据包在传送过程中的源地址就始终会是最初发送方的IP地址，也就是位于集群外的发起对Service请求的那个客户端。数据包在经过集群中的某个节点以后，最终会到达test-pod。而test-pod在返回结果的时候，会试图直接把返回的数据包发往集群外的那个客户端。但是，客户端在收到返回的数据包以后会立刻把包丢弃，因为包里的源地址（test-pod的IP地址）和它发送时指定的目标地址（test-svc的External IP）是不相符的。

![](/assets/images/lab/k8s/externalip-1.png){: .align-center}

如果使用了地址伪装，那么数据包在经过集群中的某个节点时，会把源地址替换成该节点的IP地址。这样一来，在test-pod看来，和它打交道的就是这个节点。所以，返回结果的时候，也会把数据包发往该节点。然后，数据包的源地址在这个节点上会再次被反向还原成Service的External IP，也就是客户端最初发起请求时所使用的那个目标地址。最终，数据包将成功到达客户端。

![](/assets/images/lab/k8s/externalip-2.png){: .align-center}

## NodePort

NodePort是另一种暴露Service的方法。它把Service通过端口号暴露到集群中的节点主机上，这也是为什么它被称为NodePort的原因。这样一来，通过访问主机上的某个端口号，我们就可以访问到Service了。

下面我们就来为test-svc配置一个NodePort。还是执行`kubectl edit`命令：
```shell
$ kubectl edit service test-svc
```

按照下面的内容对test-svc的配置进行修改：
```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"test-svc","namespace":"default"},"spec":{"ports":[{"port":80,"targetPort":"web-port"}],"selector":{"app":"lab-web"}}}
  creationTimestamp: "2019-06-24T03:30:08Z"
  name: test-svc
  namespace: default
  resourceVersion: "3832"
  selfLink: /api/v1/namespaces/default/services/test-svc
  uid: 5d85e807-9630-11e9-ab86-82d6af7a4ac8
spec:
  clusterIP: 10.107.169.79
  ports:
  - port: 80
    protocol: TCP
    targetPort: web-port
  selector:
    app: lab-web
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```

这里，我们去掉了之前的`externalIPs`；并且，把`type`属性从`ClusterIP`改成了`NodePort`。保存退出以后，再执行`kubectl get services`，确保test-svc的配置已经成功得到了更新：
```shell
$ kubectl get services -o wide
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE   SELECTOR
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        24d   <none>
test-svc     NodePort    10.107.169.79   <none>        80:30454/TCP   47h   app=lab-web
```

可以看到，`EXTERNAL-IP`一栏现在变成了`<none>`，而`PORTS`栏和原来相比则有了变化。NodePort的这种处理方式和Docker在bridge network模式下对外暴露容器端口号的方式很像。这里，冒号前面的数字就是Service在容器内部的端口号；冒号后面的数字则是Service在节点主机上暴露出来的一个随机端口号，也就是NodePort。

另外，我们还注意到`CLUSTER-IP`一栏和之前保持一致。这表明，NodePort是工作在ClusterIP的基础上的。当我们为Service配置NodePort时，Kubernetes依然会为Service配置ClusterIP。因此，NodePort不会阻止Pod从集群内部通过Cluster IP对Service进行访问。

再来看一下，`iptables`规则方面的变化情况：
```shell
② -A KUBE-NODEPORTS -p tcp -m comment --comment "default/test-svc:" -m tcp --dport 30454 -j KUBE-MARK-MASQ
③ -A KUBE-NODEPORTS -p tcp -m comment --comment "default/test-svc:" -m tcp --dport 30454 -j KUBE-SVC-W3OX4ZP4Y24AQZNW
⎢ ...
① -A KUBE-SERVICES -m comment --comment "kubernetes service nodeports; NOTE: this must be the last rule in this chain" -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS
⎢ ...
⎢ -A KUBE-SVC-W3OX4ZP4Y24AQZNW -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-E2HMOHPUOGTHZJEP
⎢ -A KUBE-SVC-W3OX4ZP4Y24AQZNW -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-WFXGQBTRL5EC2R2Y
⎢ -A KUBE-SVC-W3OX4ZP4Y24AQZNW -j KUBE-SEP-EEXR7SABLH35O4XP
```

* 和原来相比，行①处的规则是一直存在于`iptables`的规则集里面的。即使没有定义NodePort，它也存在，只是那时还没有KUBE-NODEPORTS链。这条规则的意思是，当数据包的目标地址是LOCAL类型的时候（`--dst-type LOCAL`），则匹配该规则，并跳转到KUBE-NODEPORTS链。这表明我们是在集群里的节点主机上向Service发起访问的；
* 行②和行③处的规则是新加入的，表示当我们访问的端口号是30454时，会跳转到KUBE-SVC-W3OX4ZP4Y24AQZNW链，从那再往后就是正常的负载均衡逻辑了；

### 集成外部负载均衡

因为集群里的每个节点上都有大体相近的`iptables`规则集，所以在集群中的任何一个节点上向NodePort端口号发送请求，都可以访问到我们的test-svc。

基于这个原因，我们也可以把Kubernetes集群与外部的负载均衡服务进行集成，把集群中的所有节点都作为负载均衡服务连接的后端服务。然后再配上相应的健康检查(Health Check)，比如监听集群节点的某个端口。当集群中有节点出现故障而无法访问时，可以由外部的负载均衡服务自动进行调度。

## 小结

ExternalIP和NodePort都是为了将Service暴露到Kubernetes集群之外，从而让外部的客户端也能访问到集群内部的Service。其中，

* ExternalIP为Service提供了一个对外可见的IP地址；
* NodePort则通过端口号直接把Service暴露到了集群节点上，通过访问节点IP和端口号，就可以访问到Service；

从`iptables`规则的角度来看，ExternalIP和NodePort都不过是原有Service基础上的规则叠加。在理解了Service网络的工作原理之后，再去理解ExternalIP和NodePort是非常容易的。