title: kubernetes入门-安装
date: 2017-05-16 19:55:41
tags: [技术,docker,kubernetes]
---

## 运行环境

ubuntu 14.04  
minikube version: v0.19.0   
Docker version 1.12.1   

## 安装minikube  

minikube是kubernetes的单机试玩版。

<!--more-->

**Step 1**

首先需要安装一个虚拟机软件，这里采用了[VirtualBox](https://www.virtualbox.org/wiki/Downloads)。

**Step 2**

安装kubernetes的命令行工具`kubectl`。

下载kubectl:   

```sh
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
```

配置到环境中：

```sh
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

检查是否正常运行：

```sh
kubectl cluster-info
```

**Step 3**

下载并安装minikube:

```
curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.19.0/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
```

## deployment

Deployment可以来创建和更新一个app。

![](http://oq1pehpfd.bkt.clouddn.com/blog/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-05-18%2015.49.12.png-blog)

```
kubectl run kubernetes-bootcamp --image=docker.io/jocatalin/kubernetes-bootcamp:v1 --port=8080
```

## Pod和Node 

Pod 表示了一组关系密切的容器，这些容器之间会共享一些资源，如存储资源、网络、端口等。

![](http://oq1pehpfd.bkt.clouddn.com/blog/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-05-16%2020.45.21.png-blog)

Pod运行在一个Node上，Node指一个物理机器或者一个虚拟机。Master负责管理Node，将Pod分配到Node上。`Kubelet`负责Master和Node之间的通信。Container runtime (like Docker, rkt)负责获取容器镜像和运行容器。

![](http://oq1pehpfd.bkt.clouddn.com/blog/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-05-16%2020.39.45.png-blog)

## Service

Service定义了一组pod的逻辑关系，并且设置了访问这些pod的策略。Pod拥有独立的IP，不对外暴露，Service负责将对外暴露地址。根据不同的暴露方式，Service被分为如下几类：

1. ClusterIP(默认)：在集群中暴露一个内容IP，所以只有在集群内部可以访问service。    
2. NodePort：使用NAT，暴露与集群中所选的Node的port相同的port。让外部可以通过集群IP的超集访问Service。   
3. LoadBalancer：在当前云中创建一个负载平衡器，并且给定一个外部IP给这个service。   
4. ExternalName：Exposes the Service using an arbitrary name (specified by externalName in the spec) by returning a CNAME record with the name. No proxy is used. This type requires v1.7 or higher of kube-dns.   

> A Kubernetes Service sis an abstraction layer which defines a logical set of Pods and enables external traffic exposure, load balancing and service discovery for those Pods.

![](http://oq1pehpfd.bkt.clouddn.com/blog/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-05-17%2015.17.14.png-blog)

Service使用label和selector来匹配一组pod。label是一组key/value对，可以标记对象是开发版、测试版还是发布版，还可以标记版本号等。label可以在创建是赋予，也可以后随时更改。

![](http://oq1pehpfd.bkt.clouddn.com/blog/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-05-17%2015.34.01.png-blog)

假设集群中已经有一个deployment kubernetes-bootcamp。

添加service：   
```
kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080
```

给pod打标签：   
```
kubectl label pod kubernetes-bootcamp-3271566451-ddvkt app=v1
```

通过标签获取pod    
```
kubectl get pods -l app=v1
```

删除services：   
```
kubectl delete service -l run=kubernetes-bootcamp
```

## app扩展和缩减


前面的例子中，一个app只有一个pod和一个容器。有时候，我们会根据需要，扩展和缩减一个app的规模。

![](http://oq1pehpfd.bkt.clouddn.com/blog/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-05-18%2015.43.54.png-blog)
![](http://oq1pehpfd.bkt.clouddn.com/blog/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-05-18%2015.49.48.png-blog)

将原来的deployment扩展到4个副本：   
```
kubectl scale deployments/kubernetes-bootcamp --replicas=4
```

Deployment数量发生变化：   
```
$ kubectl get deployments
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
hello-minikube        1         1         1            1           1d
kubernetes-bootcamp   4         4         4            4           1d
```

Pod数量也发生改变：   
```
$ kubectl get pods -o wide
NAME                                   READY     STATUS    RESTARTS   AGE       IP           NODE
hello-minikube-938614450-wn1r7         1/1       Running   0          1d        172.17.0.2   minikube
kubernetes-bootcamp-3271566451-2cjjl   1/1       Running   0          4m        172.17.0.6   minikube
kubernetes-bootcamp-3271566451-ddvkt   1/1       Running   0          1d        172.17.0.5   minikube
kubernetes-bootcamp-3271566451-nnwq8   1/1       Running   0          4m        172.17.0.7   minikube
kubernetes-bootcamp-3271566451-vtnzw   1/1       Running   0          4m        172.17.0.8   minikube
```

通过定义的service暴露的端口，可以访问kubernetes-bootcamp应用，可以发现请求被发送到不同的pod上，从而体现了负载均衡的实现。  
```
$ curl nodeip:31645
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-3271566451-2cjjl | v=1
$ curl nodeip:31645
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-3271566451-vtnzw | v=1
```

将app实例缩减到2个：   
```
 kubectl scale deployments/kubernetes-bootcamp --replicas=2
```

## app更新

`Rolling updates`可以增量的将pod更新到新版本，并且app不会因此中断。

![](http://oq1pehpfd.bkt.clouddn.com/blog/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-05-18%2016.59.02.png-blog)
![](http://oq1pehpfd.bkt.clouddn.com/blog/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-05-18%2016.59.16.png-blog)
![](http://oq1pehpfd.bkt.clouddn.com/blog/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-05-18%2016.59.29.png-blog)
![](http://oq1pehpfd.bkt.clouddn.com/blog/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-05-18%2016.59.41.png-blog)

在更新的时候，service会将业务负载平衡到可获得的pod上面。`Rolling updates`支持以下操作：

+ Promote an application from one environment to another (via container image updates)
+ Rollback to previous versions
+ Continuous Integration and Continuous Delivery of applications with zero downtime

下面来更新kubernetes-bootcamp应用的版本来体会一下更新的过程。

```sh
kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=docker.io/jocatalin/kubernetes-bootcamp:v2
```

可以观察到如下更新过程：

```shell
$ kubectl get pods
NAME                                   READY     STATUS              RESTARTS   AGE
hello-minikube-938614450-wn1r7         1/1       Running             0          2d
kubernetes-bootcamp-3271566451-ddvkt   1/1       Running             0          1d
kubernetes-bootcamp-3369739380-fbdtm   0/1       ContainerCreating   0          35s
kubernetes-bootcamp-3369739380-v98xt   0/1       ContainerCreating   0          35s

//一段时间后

$ kubectl get pods
NAME                                   READY     STATUS        RESTARTS   AGE
hello-minikube-938614450-wn1r7         1/1       Running       0          2d
kubernetes-bootcamp-3271566451-ddvkt   1/1       Terminating   0          2d
kubernetes-bootcamp-3369739380-fbdtm   1/1       Running       0          4m
kubernetes-bootcamp-3369739380-v98xt   1/1       Running       0          4m

//一段时间后

$ kubectl get pods
NAME                                   READY     STATUS    RESTARTS   AGE
hello-minikube-938614450-wn1r7         1/1       Running   0          2d
kubernetes-bootcamp-3369739380-fbdtm   1/1       Running   0          5m
kubernetes-bootcamp-3369739380-v98xt   1/1       Running   0          5m
```

如果新pod更新失败，则旧pod不会被全部杀死，至少会保留一个维持正常运行。docker pull jocatalin/kubernetes-bootcamp

测试更新后的效果。

首先获取service暴露的端口，31645:   
```
$ kubectl get services
NAME                  CLUSTER-IP   EXTERNAL-IP   PORT(S)          AGE
kubernetes-bootcamp   10.0.0.127   <nodes>       8080:31645/TCP   1h
```

获取节点IP：   
```
$ kubectl describe pods -l run=kubernetes-bootcamp
//...关键内容
Name:		kubernetes-bootcamp-3369739380-v98xt
Namespace:	default
Node:		minikube/192.168.99.100
//...
```

发送请求，可以看到app服务版本确实更新了：   
```
$ curl 192.168.99.100:31645
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-3369739380-fbdtm | v=2
```

回滚，恢复到更新之前。

```
kubectl rollout undo deployments/kubernetes-bootcamp
```


