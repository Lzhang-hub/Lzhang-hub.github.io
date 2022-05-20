---
layout: post
title: k8s之容器启动过程
date: 2022-05-20
tags: k8s   
---

### 容器启动流程

#### 1、从apiserver到kubelet

- k8s中创建一个pod的过程，首先是想apiserver发送创建请求，apiserver负责写到etcd中，然后scheduler通过listandwatch监控到请求，负责将pod调度到合适的节点上并写入etcd，然后到kubelet组件，watch到bound pod，启动容器，并更新状态。

![Image](https://Lzhang-hub.github.io/images/posts/k8s/Image.png)

#### 2、从kubelet到创建容器

##### 2.1 基本概念

- CRI：

CRI：k8s定义的一组与容器运行时的接口，只要实现了这套接口的容器运行时都可以接到k8s平台上。如果容器运行时想要接到k8s平台上，就需要其按照CRI标准实现，但是有一些容器运行时自身没有实现CRI， 于是就有了 shim（垫片）， 一个 shim 的职责就是作为适配器将各种容器运行时本身的接口适配到 Kubernetes 的 CRI 接口上，其中 dockershim 就是 Kubernetes 对接 Docker 到 CRI 接口上的一个垫片实现。**它是kubelet与容器运行时的连接介质，有些容器运行时会已经支持了CRI，此时就不需要shim。**

![Image](https://Lzhang-hub.github.io/images/posts/k8s/Image-1652926490747.png)

![Image](https://Lzhang-hub.github.io/images/posts/k8s/Image-1652926495919.png)

- OCI：

OCI（开放容器标准）：创建容器时需要做一些namespace和cgroups的工作以及挂载root文件系统等操作，就是参考OCI标准，其中RunC 就是这个标准的一个实现。 这个**标准其实就是一个文档，**主要规定了容器镜像的结构、以及容器需要接收哪些操作指令，比如 create、start、stop、delete 等这些命令。 既然是标准肯定就有其他 OCI 实现，比如 Kata、gVisor 这些容器运行时都是符合 OCI 标准的。**容器运行时的一个标准，通过该文档就能实现一个容器运行时。**

- 容器运行时

一般启动容器会包含：1、拉取镜像；2、解析镜像为bundle文件；3、最后从bundle运行容器。

runc就是最后一步，runc是OCI的一种实现，OCI就是容器运行时的标准。_runc_是只是一个[命令行工具](https://cloud.tencent.com/product/cli?from=10680)，只是在rootfs 中的指定文件去运行。

到底什么是容器运行时？[参考 ](https://cloud.tencent.com/developer/article/1895805)

可以将容器运行时分成**Low-Level和High-Level容器运行时**，如图所示，每个运行时都包含了这个Low-Level到High-Level频谱的不同部分。

因此，从实际出发，通常只专注于正在运行的容器的runtime通常称为“Low-Level容器运行时”，支持更多高级功能（如镜像管理和gRPC / Web API）的运行时通常称为“High-Level容器运行时”，“High-Level容器运行时”或通常仅称为“容器运行时”，我将它们称为“High-Level容器运行时”。

- Low-Level容器运行时：容器是通过Linux nanespace和Cgroups实现的，Namespace能让你为每个容器提供虚拟化系统资源，像是文件系统和网络，Cgroups提供了限制每个容器所能使用的资源的如内存和CPU使用量的方法。在最低级别的运行时中，容器运行时负责为容器建立namespaces和cgroups,然后在其中运行命令，Low-Level容器运行时支持在容器中使用这些操作系统特性。
- High-Level容器运行时：通常情况下，开发人员想要运行一个容器不仅仅需要Low-Level容器运行时提供的这些特性，同时也需要与镜像格式、镜像管理和共享镜像相关的API接口和特性，而这些特性一般由High-Level容器运行时提供。

<u>Kubernetes 只需支持 containerd 等high-level container runtime即可。由containerd 按照OCI 规范去对接不同的low-level container runtime，比如通用的runc，安全增强的gvisor，隔离性更好的runv。</u>

![img](https://Lzhang-hub.github.io/images/posts/k8s/4eae9fb171f1e07dbf3d73be4154ae68.png)

##### 2.2 容器的创建过程

有了基本概念之后，看一下在kubelet中是如何启动容器的。

从 Docker 1.11 版本开始，Docker 容器运行就不是简单通过 Docker Daemon 来启动了，而是通过集成 containerd、runc 等多个组件来完成的。虽然 Docker Daemon 守护进程模块在不停的重构，但是基本功能和定位没有太大的变化，一直都是 CS 架构，守护进程负责和 Docker Client 端交互，并管理 Docker 镜像和容器。现在的架构中组件 containerd 就会负责集群节点上容器的生命周期管理，并向上为 Docker Daemon 提供 gRPC 接口。

所以真正启动容器是通过 containerd-shim 去调用 runc 来启动容器的，runc 启动完容器后本身会直接退出，containerd-shim 则会成为容器进程的父进程, 负责收集容器进程的状态, 上报给 containerd, 并在容器中 pid 为 1 的进程退出后接管容器中的子进程进行清理, 确保不会出现僵尸进程。

![Image](https://Lzhang-hub.github.io/images/posts/k8s/Image-1652926545531.png)



- 如果容器运行时是使用的docker，那么其创建过程会更简洁

切换到 containerd 可以消除掉中间环节，操作体验也和以前一样，但是由于直接用容器运行时调度容器，所以它们对 Docker 来说是不可见的。因此，你以前用来检查这些容器的 Docker 工具就不能使用了。 对 CRI 的适配是通过一个单独的 CRI-Containerd 进程来完成的，这是因为最开始 containerd 还会去适配其他的系统（比如 swarm），所以没有直接实现 CRI，所以这个对接工作就交给 CRI-Containerd 这个 shim 了。

![Image](https://Lzhang-hub.github.io/images/posts/k8s/Image-1652926561821.png)

然后到了 containerd 1.1 版本后就去掉了 CRI-Containerd 这个 shim，直接把适配逻辑作为插件的方式集成到了 containerd 主进程中，现在这样的调用就更加简洁了

![Image](https://Lzhang-hub.github.io/images/posts/k8s/Image-1652926574334.png)



- 兼容OCI和RCI

 Kubernetes 社区也做了一个专门用于 Kubernetes 的 CRI 运行时 CRI-O，直接兼容 CRI 和 OCI 规范。

![Image](https://Lzhang-hub.github.io/images/posts/k8s/Image-1652926596931.png)

- k8s直接使用containered

而 containerd 1.1 版本后就内置实现了 CRI，所以 Docker 也没必要再去单独实现 CRI 了，当 Kubernetes 不再内置支持开箱即用的 Docker 的以后，最好的方式当然也就是直接使用 Containerd 这种容器运行时，而且该容器运行时也已经经过了生产环境实践的，接下来我们就来学习下 Containerd 的使用。

![Image](https://Lzhang-hub.github.io/images/posts/k8s/Image-1652926639821.png)



参考：

[https://www.qikqiak.com/k8s-book/docs/15.%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5%E4%B8%8E%E7%BB%84%E4%BB%B6.html](https://www.qikqiak.com/k8s-book/docs/15.基本概念与组件.html)https://mp.weixin.qq.com/s/--t74RuFGMmTGl2IT-TFrg