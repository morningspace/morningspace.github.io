---
title: Kubernetes网络篇——Ingress
excerpt: "以Nginx为例，通过实际部署实验来认识一下Ingress"
categories: [tech]
tags: [lab, dummies, dummies_kubernetes, kubernetes]
toc: true
toc_sticky: true
---

注：
本文采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。

> 利用Ingress，我们同样可以把集群内的Service暴露到集群外，而且还可以获得更多……

![](/assets/images/lab/k8s/ingress.png){: .align-center}

在[Kubernetes网络篇——ExternalIP和NodePort](/tech/k8s-net-externalip-nodeport)一文里，我们了解了如何把集群内的Service暴露到集群外。本文要介绍的Ingress也有类似的功能，并且它结合了专业的第三方负载均衡和反向代理服务，比如像：Nginx，HAProxy，Traefik等，因此还提供了很多其他特性，比如：负载均衡，SSL，基于名称的虚拟主机(Name-based Virtual Host)服务等。

Kubernetes的Ingress，可以把集群外部的HTTP或HTTPS请求路由到集群内部的多个Service，我们只需要部署一个Ingress Controller，就能以集中的方式统一来管理一个集群内所有需要暴露的Service。具体的路由方式，则是由定义在Ingress Resource里的规则来决定的，比如：根据主机名和URL路径，把请求路由到指定的Service。

下面我们就以Nginx Ingress为例，通过实际部署实验来认识一下Ingress。

## 部署backend应用

为了方便后面做实验，我们需要在集群里部署一些测试应用，包括Pod和相应的Service。这里，我们依然沿用在[Kubernetes网络篇——Pod网络(下)](/tech/k8s-net-pod-2/)一文里使用的例子，包括一个暴露80端口的Nginx和一个暴露8080端口的Tomcat。不同的是，这次我们把两个Web服务分别部署到了各自的Pod里，而不是放在同一个Pod。它们将作为Nginx的backend，对外提供HTTP服务。

另外，我们还引入了一个默认的backend应用。当HTTP请求经过Nginx时，根据事先定义的路由规则，请求通常会匹配相应的规则，并被路由到后端应用。但如果没有匹配到任何规则，那么Nginx就会把请求发送给默认的backend应用。根据Nginx Ingress的规定，这种应用需要提供两个URL：
* `/`：返回404，并提供默认的404页面
* `/healthz`：返回200，用于提供针对该应用本身的健康检查

所以总共需要部署三个应用。我们把全部三个backend应用的Deployment和Service定义都合并到一起存成了一个文件，名叫`backends.yaml`。因为篇幅限制，加上两个Web服务的YAML片段在以前的文章里已经出现过，内容基本相似，所以这里就不再重复了。下面列举的是默认backend应用的YAML片段：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: default-http-backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: default-http-backend
  template:
    metadata:
      labels:
        app: default-http-backend
    spec:
      containers:
      - name: default-http-backend
        image: k8s.gcr.io/defaultbackend-amd64:1.5
        ports:
        - containerPort: 8080
          name: http
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: default-http-backend
spec:
  ports:
  - port: 80
    targetPort: http
  selector:
    app: default-http-backend
```

可以看到，通过`---`，我们把应用的Deployment定义和Service定义拼接在了一起。并且，我们用的是`k8s.gcr.io`提供的defaultbackend镜像。如果想加快镜像抓取的速度，也可以使用本地的私有Docker镜像注册表，比如，在[lab-kubernetes](https://github.com/morningspace/lab-kubernetes)项目里的`mr.io`。只要我们给k8s.gcr.io/defaultbackend-amd64镜像加上相应的标签，然后把它推送到`mr.io`就可以了：
```shell
$ docker tag k8s.gcr.io/defaultbackend-amd64:1.5 mr.io/defaultbackend-amd64:1.5
$ docker push mr.io/defaultbackend-amd64:1.5
```

接下来，让我们把所有backend应用部署到集群里：
```shell
$ kubectl create -f backends.yaml
deployment.apps/test-pod-1 created
deployment.apps/test-pod-2 created
deployment.apps/default-http-backend created
service/test-svc-1 created
service/test-svc-2 created
service/default-http-backend created
```

另外，我们还需要一个以Pod形式部署的lab-sleeper，做为测试用的客户端，从Pod内部访问我们的Web服务。关于lab-sleeper的镜像，请查看[Kubernetes网络篇——Service网络(上)](/tech/k8s-net-service-1/)一文。下面是执行部署的命令和返回的结果：
```shell
$ kubectl create deployment lab-sleeper --image=mr.io/lab-sleeper
deployment.apps/lab-sleeper created
```

然后，利用`kubectl`提供的命令查看一下部署的情况：
```shell
$ kubectl get deployments
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
default-http-backend   1/1     1            1           2m51s
lab-sleeper            1/1     1            1           22s
test-pod-1             1/1     1            1           2m51s
test-pod-2             1/1     1            1           2m51s

$ kubectl get pods -o wide
NAME                                    READY   STATUS    RESTARTS   AGE   IP           NODE          NOMINATED NODE   READINESS GATES
default-http-backend-6464987848-dwsmk   1/1     Running   0          3m8s   10.244.3.2   kube-node-2   <none>           <none>
lab-sleeper-7ff95f64d7-mwpdj            1/1     Running   0          39s    10.244.2.4   kube-node-1   <none>           <none>
test-pod-1-64c6878596-hzjxl             1/1     Running   0          3m8s   10.244.3.3   kube-node-2   <none>           <none>
test-pod-2-785db554c6-vrn9w             1/1     Running   0          3m8s   10.244.2.3   kube-node-1   <none>           <none>

$ kubectl get services -o wide
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE   SELECTOR
default-http-backend   ClusterIP   10.98.39.188     <none>        80/TCP    3m24s   app=default-http-backend
kubernetes             ClusterIP   10.96.0.1        <none>        443/TCP   25d     <none>
test-svc-1             ClusterIP   10.108.0.250     <none>        80/TCP    3m25s   app=lab-web-1
test-svc-2             ClusterIP   10.103.200.191   <none>        80/TCP    3m25s   app=lab-web-2
```

可以看到，所有Pod都已经成功启动，相应的Service也都成功创建出来了。

在继续下面的实验之前，我们还可以用lab-sleeper测一下部署的这些backend应用。比如：从lab-sleeper内部访问test-svc-1，应该由Nginx返回响应：
```shell
$ kubectl exec -it lab-sleeper-7ff95f64d7-mwpdj curl http://test-svc-1
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
```

访问test-svc-2，则应该由Tomcat返回响应：
```shell
$ kubectl exec -it lab-sleeper-7ff95f64d7-mwpdj curl http://test-svc-2
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="UTF-8" />
        <title>Apache Tomcat/8.5.42</title>
        <link href="favicon.ico" rel="icon" type="image/x-icon" />
        <link href="favicon.ico" rel="shortcut icon" type="image/x-icon" />
        <link href="tomcat.css" rel="stylesheet" type="text/css" />
    </head>
```

如果要是访问default-http-backend，则应该返回默认的404响应：
```shell
$ kubectl exec -it lab-sleeper-7ff95f64d7-mwpdj curl http://default-http-backend
default backend - 404
```

## 部署Ingress Controller

接下来，我们要部署Nginx Ingress Controller了。本质上，Nginx Ingress Controller就是一个以Pod形式部署在Kubernetes集群里的Nginx的Daemon。它通过监控Kubernetes API的`/ingresses`服务来监控Ingress Resource的变化，并相应地更新Nginx的配置。下面是它的部署YAML片段：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-ingress
  template:
    metadata:
      labels:
        app: nginx-ingress
    spec:
      serviceAccountName: nginx-ingress-serviceaccount
      containers:
        - name: nginx-ingress-controller
          image: mr.io/kubernetes-ingress-controller/nginx-ingress-controller:0.24.1
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --default-backend-service=$(POD_NAMESPACE)/default-http-backend
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
          livenessProbe:
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            timeoutSeconds: 10
          readinessProbe:
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            timeoutSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: http
  selector:
    app: nginx-ingress
```

这里，我们用的是`quay.io`上的nginx-ingress-controller镜像，并且把它推送到了本地的私有Docker注册表：
```shell
$ docker tag quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.24.1 mr.io/kubernetes-ingress-controller/nginx-ingress-controller:0.24.1
$ docker push mr.io/kubernetes-ingress-controller/nginx-ingress-controller
```

在定义Service的时候，我们是以NodePort的形式把Ingress Controller暴露到集群外的。因为我们的实验环境是在本地，不像AWS，Azure，GCP等云服务环境，有现成的LoadBalancer服务，所以使用NodePort是比较合理的。稍后，我们就可以通过访问集群节点的IP地址和端口号来访问Nginx Ingress了。

执行下面的命令把Nginx Ingress Controller部署到集群里：
```shell
$ kubectl create -f ingress-controller.yaml
deployment.apps/nginx-ingress created
service/nginx-ingress created
```

然后查看部署的情况：
```shell
$ kubectl get deployments nginx-ingress
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
nginx-ingress   1/1     1            1           5m42s

$ kubectl get pods -o wide
NAME                                    READY   STATUS    RESTARTS   AGE     IP            NODE          NOMINATED NODE   READINESS GATES
default-http-backend-6464987848-dwsmk   1/1     Running   0          4h20m   10.244.3.2    kube-node-2   <none>           <none>
lab-sleeper-7ff95f64d7-mwpdj            1/1     Running   0          4h18m   10.244.2.4    kube-node-1   <none>           <none>
nginx-ingress-6877758467-vrbzc          1/1     Running   0          5m55s   10.244.3.10   kube-node-2   <none>           <none>
test-pod-1-64c6878596-hzjxl             1/1     Running   0          4h20m   10.244.3.3    kube-node-2   <none>           <none>
test-pod-2-785db554c6-vrn9w             1/1     Running   0          4h20m   10.244.2.3    kube-node-1   <none>           <none>

$ kubectl get services nginx-ingress -o wide
NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE     SELECTOR
nginx-ingress   NodePort   10.108.175.68   <none>        80:32222/TCP   7m54s   app=nginx-ingress
```

确认部署成功以后，可以尝试在lab-sleeper内部访问Nginx Ingress服务。从上面的输出结果里可以看到，Ingress的Cluster IP是`10.244.3.10`：
```shell
$ kubectl exec -it lab-sleeper-7ff95f64d7-mwpdj curl http://10.244.3.10
default backend - 404
```

我们也可以在任何一个集群节点上通过IP地址和端口号访问Nginx Ingress。比如这里我们访问的是master节点，端口号可以从前面的输出结果里找到：
```shell
$ curl http://10.192.0.2:32222
default backend - 404
```

### 关于RBAC

Kubernetes的RBAC功能从1.8开始就已经基本稳定了，所以很多Kubernetes的部署环境都默认开启了RBAC。我们的实验环境——[kubeadm-dind-cluster](https://github.com/kubernetes-sigs/kubeadm-dind-cluster)也不例外。

要开启RBAC功能，就需要在Kubernetes启动API Server的时候带上参数`--authorization-mode=RBAC`。如果好奇的话，我们也可以登录到master节点上，在`/etc/kubernetes/manifests`目录下寻找一个叫`kube-apiserver.yaml`的文件，这是API Server的部署文件：
```shell
$ cat /etc/kubernetes/manifests/kube-apiserver.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=10.192.0.2
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
...
```

其中，启动kube-apiserver的命令行参数里就有`--authorization-mode`，可以看到我们的集群环境是开启了RBAC的。

对于这种情况，我们需要为Nginx Ingress Controller配置相应的：
* ServiceAccount
* ClusterRole
* Role
* RoleBinding
* ClusterRoleBinding

否则，启动Nginx Ingress Controller时就可能会失败。关于如何定义RBAC的相关配置，大家可以参考[lab-kubernetes](https://github.com/morningspace/lab-kubernetes)里的示例，这里就不展开了。

## 定义Ingress Resource

接下来，我们要为Nginx Ingress定义Ingress Resource了，也就是为Nginx定义路由规则。前面我们已经提到过，在Ingress Resource里定义的规则会在运行期被Nginx Ingress Controller动态检测到，并转成Nginx的配置信息。
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: kubernetes.lab
    http:
      paths:
      - path: /svc-1
        backend:
          serviceName: test-svc-1
          servicePort: http
      - path: /svc-2
        backend:
          serviceName: test-svc-2
          servicePort: http
```

这里的规则很简单，当请求到达Ingress时，如果主机名为`kubernetes.lab`，且URL路径为`/svc-1`，则导向test-svc-1；如果URL路径为`/svc-2`，则导向test-svc-2。同时，通过`rewrite-target`注解，我们把URL重写为`/`。

现在，让我们先把Ingress Resource部署到集群里：
```shell
$ kubectl create -f ingress.yaml
ingress.extensions/nginx-ingress created
```

然后查看部署的情况：
```shell
$ kubectl get ingress
NAME            HOSTS            ADDRESS   PORTS   AGE
nginx-ingress   kubernetes.lab             80      14s
```

下面，我们就可以来验证，刚才定义的路由规则是否已经生效了：
```shell
$ curl -H host:kubernetes.lab http://10.192.0.2:32222/svc-1
$ curl -H host:kubernetes.lab http://10.192.0.2:32222/svc-2
```

这里，我们是直接从master节点上向Ingress服务发送请求的。因为没有为集群节点配DNS，所以我们用了`-H`参数，把主机名作为HTTP头传给Ingress。当然，也可以在节点本地的hosts里加上主机名和IP地址的映射，这样就不需要HTTP头了：
```shell
$ cat /etc/hosts
... ...
10.192.0.2	kubernetes.lab
```

### Ingress和Pod

虽然我们在定义Ingress Resource的时候用的是Service，但Ingress Controller实际是直接和Pod打交道的。因为Kubernetes知道每个Service包含哪些Pod，所以Ingress Controller只要查询Kubernetes的API Server，就能得到Service当前的所有Pod。

因为Ingress是直接对Pod而非Service进行负载均衡，所以它可以实现比如像Session保持这样的功能，即：来自同一个客户端的前后两次请求最终送达的是同一个Pod。

## 修改Ingress Resource

我们可以利用`kubectl edit ingress`对Ingress Resource里的路由规则动态进行修改。Nginx Ingress Controller会定期监控Kubernetes的API Server，一旦Ingress Resource有更新，Controller就会重新生成Nginx的配置，并重新加载Nginx。比如，我们可以为Ingress配上虚拟主机：
* 当主机名为`svc-1.kubernetes.lab`时，导向test-svc-1；
* 当主机名为`svc-2.kubernetes.lab`时，导向test-svc-2；

```yaml
spec:
  rules:
  - host: kubernetes.lab
    http:
      paths:
      - path: /svc-1
        backend:
          serviceName: test-svc-1
          servicePort: http
      - path: /svc-2
        backend:
          serviceName: test-svc-2
          servicePort: http
  - host: svc-1.kubernetes.lab
    http:
      paths:
      - backend:
          serviceName: test-svc-1
          servicePort: http
  - host: svc-2.kubernetes.lab
    http:
      paths:
      - backend:
          serviceName: test-svc-2
          servicePort: http
```

如果我们观察Nginx Ingress Controller的日志，就可以看到，当修改Ingress后，就会有类似下面这样的日志产生：
```shell
$ kubectl logs nginx-ingress-6877758467-vrbzc
... ...
I0627 14:56:41.872772       7 controller.go:170] Configuration changes detected, backend reload required.
I0627 14:56:41.875866       7 event.go:209] Event(v1.ObjectReference{Kind:"Ingress", Namespace:"default", Name:"nginx-ingress", UID:"c6070246-98eb-11e9-b7fa-0ee56cf2fc99", APIVersion:"extensions/v1beta1", ResourceVersion:"16248", FieldPath:""}): type: 'Normal' reason: 'CREATE' Ingress default/nginx-ingress
I0627 14:56:42.033294       7 controller.go:188] Backend successfully reloaded.
```

我们再来验证一下：
```shell
$ curl -H host:svc-1.kubernetes.lab http://10.192.0.2:32222
$ curl -H host:svc-2.kubernetes.lab http://10.192.0.2:32222
```

## 小结

和ExternalIP以及NodePort一样，Ingress可以把Kubernetes集群内的Service暴露到集群外，但它还提供了更多丰富的功能，比如负载均衡，SSL，虚拟主机等。Ingress对负载均衡的配置不涉及对Service的修改，所以它是和Service解耦的，并且它以集中的方式统一配置路由规则，非常便于管理。

有很多Ingress Controller可供选择，除了本文介绍的Nginx之外，常见的还有Traefik，HAProxy，Isitio，Kong等。

当然，功能更加丰富的同时，Ingress的部署也相对更加复杂一些。前面我们已经看到了，部署涉及Ingress Resource，Ingress Controller的Deployment和Service。另外，除了针对Ingress Resource进行路由规则的配置外，如果我们还想对Ingress Controller本身的功能进行配置，还需要配置ConfigMap，以及Secret（比如要开启SSL功能）。
