---
layout: post
title: k8s之device-pluin
date: 2022-05-20
tags: k8s   
---

device plugin 的工作原理其实不复杂。主要有以下步骤：

- 首先 device plugin 可以通过手动或 daemonset 部署到需要的节点上。
- 为了让 Kubernetes 发现 device plugin，需要向 kubelet 的 unix socket。 进行注册，注册的信息包括 device plugin 的 unix socket，API Version，ResourceName。
- kubelet 通过 grpc 向 device plugin 调用 ListAndWatch， 获取当前节点上的资源。
- kubelet 向 api server 更新节点状态来通知资源变更。
- 用户创建 pod，请求资源并调度到节点上后，kubelet 调用 device plugin 的 Allocate 进行资源分配。

时序图如下:

![device plugins](https://Lzhang-hub.github.io/images/posts/k8s/device-plugins.svg)

在 device plugin 的实现中，最关键的两个要实现的方法是 `ListAndWatch` 和 `Allocate`。除此之外，还要注意监控 kubelet 的重启，一般是使用 `fsnotify` 类似的库监控 kubelet.sock 的重新创建事件。如果重新创建了，则认为 kubelet 是重启了，我们需要重新向 kubelet 注册 device plugin。

[https://www.myway5.com/index.php/2020/03/24/kubernetes%E5%BC%80%E5%8F%91%E7%9F%A5%E8%AF%86-device-plugin%E7%9A%84%E5%AE%9E%E7%8E%B0/](https://www.myway5.com/index.php/2020/03/24/kubernetes%E5%BC%80%E5%8F%91%E7%9F%A5%E8%AF%86-device-plugin%E7%9A%84%E5%AE%9E%E7%8E%B0/ "参考")