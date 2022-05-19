---
layout: post
title: k8s之服务发现
date: 2022-05-16
tags: k8s   
---

#### 服务注册：

首先，在创建一个service的时候会发生什么事情？k8s集群会生成那些东西？

##### 1、服务注册：

k8s集群中使用DNS作为服务注册表，每一个service都会注册到集群的DNS中。流程如下：

![service-registration.png](https://github.com/Lzhang-hub/Lzhang-hub.github.io/tree/master/images/posts/k8s/service-registration.png)

1. 向 API Server 用 POST 方式提交一个新的 Service 定义；
2. 这个请求需要经过认证、鉴权以及其它的准入策略检查过程之后才会放行；
3. Service 得到一个 `ClusterIP`（虚拟 IP 地址），并保存到集群数据仓库；
4. 在集群范围内传播 Service 配置；
5. 集群 DNS 服务得知该 Service 的创建，据此创建必要的 DNS A 记录。

最后就创建一个**DNS A记录，它是从service名称映射到clusterIP的域名记录**。这样就能被集群中其他pod发现了。

##### 2、endpoint

集群中会有一个endpoint controller，它会根据service中的Label Selector创建一个与service对应的endpoint，endpoint中会保存符合条件的**可用pod列表**，包含pod ip信息。

![image-20220519182621294](https://github.com/Lzhang-hub/Lzhang-hub.github.io/tree/master/images/posts/k8s/image-20220519182621294.png)

##### 3、kube-proxy

集群中每个节点中都有kube-proxy，注意它并不是一个普通的代理，主要作用是生成iptables。

在上面的service创建了clusterIP和endpoint之后，kube-proxy的控制器会通过api server 检测到这种变化，会根据发送到该节点上的ClusterIP 以及 Endpoints 对象去创建 **iptables 或者 IPVS 规则**，这个规则的作用就是告诉节点，如果有来自ClusterIP 的流量，那么你就要捕获它并转发endpoint中对应的podip。

##### 4、总结

创建新的 Service 对象时，会得到一个虚拟 IP，被称为 ClusterIP。服务名及其 ClusterIP 被自动注册到集群 DNS 中，并且会创建相关的 Endpoints 对象用于保存符合标签条件的健康 Pod 的列表，Service 对象会向列表中的 Pod 转发流量。

与此同时集群中所有节点都会配置相应的 iptables/IPVS 规则，监听目标为 ClusterIP 的流量并转发给真实的 Pod IP。

![service register](https://github.com/Lzhang-hub/Lzhang-hub.github.io/tree/master/images/posts/k8s/registeration-flow.png)

#### 服务发现

##### 1、整个流程

好了，在service注册之后，已经具备了服务发现的条件了，下面就将整个流程串起来

podA 通过service找pod B，**注意**每个 Pod 中的每个容器的 `/etc/resolv.conf` 文件都被配置为使用集群 DNS 进行解析。

podA通过podB 的service name，在DNS中能解析到B的clusterIP，然后就会想这个clusterIP发送流量了，但是clusterIP所在的service网络中**没有路由**，因为没有路由，所有容器把发现这种地址的流量都发送到了缺省网关（名为 `CBR0` 的网桥）。这些流量会被转发给 Pod 所在节点的网卡上。节点的网络栈也同样没有路由能到达 Service Network，所以只能发送到自己的缺省网关。路由到节点缺省网关的数据包会通过 Node **内核**。**（这里发生了变化）**

前面可以知道kube-proxy在每个节点上生成了clusterIP到pod的iptables 规则，那么在内核在处理目标为 Service 网络的数据包时，就会发现这个规则，并且按照该**规则修改数据包的header，把目标IP改成endpoint中的可用IP。**至此实现了服务发现。

##### 2、总结

一个 Pod 需要用 Service 连接其它 Pod。首先向集群 DNS 发出查询，把 Service 名称解析为 ClusterIP，然后把流量发送给位于 Service 网络的 ClusterIP 上。然而没有到 Service 网络的路由，所以 Pod 把流量发送给它的缺省网关。这一行为导致流量被转发给 Pod 所在节点的网卡，然后是节点的缺省网关。这个操作中，节点的内核修改了数据包 Header 中的目标 IP，使其转向健康的 Pod。





![discovery](../images/discovery-flow.png)