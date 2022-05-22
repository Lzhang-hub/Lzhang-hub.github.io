---
layout: post
title: k8s之基本组件
date: 2022-05-22
tags: k8s   
---

## 一、基础概念

### 1、声明式API和命令式API

​	2.1 命令式API

  命令式API命令机器如何去做，不管你想要什么，它都会按照你的命令去实现，但是最后出来的结果是不是你想要的东西就要看你的代码的能力。此种场景下计算机不具备“智能”。

​	2.2 声明式API

  声明式API是告诉机器你想要什么，让机器想出如何去做。这种场景的前提是机器要具备一定的”智能“。

### 2、定制资源定义（CRD）

CRD是 Kubernetes（v1.7+）为提高可扩展性，让开发者去自定义资源的一种方式。CRD 资源可以动态注册到集群中，注册完毕后，用户可以通过 kubectl 来创建访问这个自定义的资源对象，类似于操作 Pod 一样。不过需要注意的是 CRD 仅仅是资源的定义而已，需要一个 Controller 去监听 CRD 的各种事件来添加自定义的业务逻辑。

`注意`：如果只是对CRD资源本身进行CRUD操作的话，不需要controller也是可以的，相当于只有数据存入到etcd中，而没有对这个数据进行相关操作而已。

CRD定义之后会保存到etcd中，之后如果出现对该自定义资源对象的相关操作，kube_apiserver就能识别出来了。

如：deployment类型的资源，创建一个pod时，apiserver就会在etcd中查找是否存在该资源，如果存在就会创建一个pod并且写到etcd中，接下来pod的中启动容器的操作就是controller的事情了。

controller的作用就是监听指定对象的新增、删除、修改等变化，针对这些变化做出相应的响应（例如新增pod的响应为创建docker容器）

关于crd的实战操作：

https://blog.csdn.net/boling_cavalry/article/details/88924194

### 3、PV和PVC

参考资料：https://www.cnblogs.com/along21/p/10342788.html

PV和PVC都是API资源

- PV：是集群中由管理员分配的一段**网络存储**，属于集群的资源，类比集群中的节点。PV是一个容量插件，如volumes，但是它的生命周期独立于任何使用它的pod，也就是使用PV的pod生命周期结束了，PV仍然是存在的。PV这个API对象捕获存储实现的详细信息，包括NFS，iSCSI或特定于云提供程序的存储系统。

- PVC：用户**进行存储的请求**，可以类比于pod，pod是消耗节点资源，PVC是消耗PV资源。

- StorageClass：PV除了大小和模式外，用户可能还会需要不同的属性，这时候就可以使用StorageClass，它为管理员提供了描述所提供的存储属于哪一类的方法。不同的类对应到不同的服务质量级别。

- PV和PVC生命周期
  - Provisioning ——-> Binding ——–>Using——>Releasing——>Recycling

    供应准备Provisioning---通过集群外的存储系统或者云平台来提供存储持久化支持。
    
    - 静态提供Static：集群管理员创建多个PV。 它们携带可供集群用户使用的真实存储的详细信息。 它们存在于Kubernetes API中，可用于消费
    - 动态提供Dynamic：当管理员创建的静态PV都不匹配用户的PersistentVolumeClaim时，集群可能会尝试为PVC动态配置卷。 此配置基于StorageClasses：PVC必须请求一个类，并且管理员必须已创建并配置该类才能进行动态配置。 要求该类的声明有效地为自己禁用动态配置。
    
  - 绑定Binding---用户创建pvc并指定需要的资源和访问模式。在找到可用pv之前，pvc会保持未绑定状态。
  
  - 使用Using---用户可在pod中像volume一样使用pvc。
  
  - 释放Releasing---用户删除pvc来回收存储资源，pv将变成“released”状态。由于还保留着之前的数据，这些数据需要根据不同的策略来处理，否则这些存储资源无法被其他pvc使用。
  
    回收Recycling---pv可以设置三种回收策略：保留（Retain），回收（Recycle）和删除（Delete）。
  
    -  保留策略：允许人工处理保留的数据。
    -  删除策略：将删除pv和外部关联的存储资源，需要插件支持。
    - 回收策略：将执行清除操作，之后可以被新的pvc使用，需要插件支持。

  注：目前只有NFS和HostPath类型卷支持回收策略，AWS EBS,GCE PD,Azure Disk和Cinde



###  4、SVC

参考资料：https://i4t.com/4567.html

Pod是有生命周期(可能挂掉或者重启等)，为了可以给客户端一个固定的访问端点，因此需要在客户端和Pod之间添加一个中间层，这个中间层称之为Service.

**1、Service是如何连接的客户端与Pod的呢？**

  k8s三大IP：

- Node Network 节点网络node ip 节点网络地址是配置在节点网络之上
- Pod Network Pod网络pod ip Pod网络地址是配置在Pod网络之上
- Cluster Network(svc network) virtual IP svc ip没有配置在某个网络接口上，是一个***\*虚拟IP\****，它只是存在service的规则当中，iptables

<img src="https://Lzhang-hub.github.io/images/posts/k8s/Pod RC与Service的关系.png" width = "50%"   height = "70%"  alt="Pod与service关系"/>

  

​	连接service与pod的纽带是**endpoint**

  	在service中会有`selector`配置，`endpoint controller`会根据`selector`生成一个endpoint对象，并存储到etcd中。一般endpoint是和service同名。
  	
  	这样即使pod挂了重启导致ip变化时，此时对应的endpoint也会变化，但是只是变化了地址，其名称是和service对应的，这样service就能够通过配置中的selector来找到endpoint对应的pod地址，从而完成服务。

**2、service的变动(增删)是怎么完成的？**

  每个节点上面都有kube-proxy，kube-proxy通过kubernetes中固有的watch请求方法持续监听apiserver。一旦有service资源发生变动(增删改查)kube-proxy可以及时转化为能够调度到后端Pod节点上的规则，这个规则可以是iptables也可以是ipvs，取决于service实现方式。

  kube-proxy又是通过增删service和endpoint来实现的。

##  二、k8s框架

### k8s基本框架

<img src="https://Lzhang-hub.github.io/images/posts/k8s/k8s-框架-1.png" width = "60%"   height = "80%"  alt="Pod与service关系"/>



各组件启动参数介绍：https://kubernetes.io/zh/docs/reference/command-line-tools-reference/

### 1、 k8s组件之apiserver

参考资料：https://cloud.tencent.com/developer/article/1444692

#### 1.1 概念：

k8s API Server提供了k8s各类资源对象（pod,RC,Service等）的增删改查及watch等HTTP Rest接口，是整个系统的数据总线和数据中心。

#### 1.2 功能：

- 提供了集群管理的REST API接口(包括认证授权、数据校验以及集群状态变更)；

- 提供其他模块之间的数据交互和通信的枢纽（其他模块通过API Server查询或修改数据，只有API Server才直接操作etcd）;

- 是资源配额控制的入口；

- 拥有完备的集群安全机制.

#### 1.3 如何访问apiserver

kube-apiserver这个进程提供服务，该进程运行在单个k8s-master节点上。默认有两个端口。

- 本地端口：

  该端口用于接收HTTP请求；

  该端口默认值为`8080`，可以通过API Server的启动参数`--insecure-port`的值来修改默认值；

  默认的IP地址为`localhost`，可以通过启动参数`--insecure-bind-address`的值来修改该IP地址；

  非认证或授权的HTTP请求通过该端口访问API Server。

- 安全端口：

    该端口默认值为`6443`，可通过启动参数`--secure-port`的值来修改默认值；

    默认IP地址为非本地（Non-Localhost）网络端口，通过启动参数“`--bind-address`”设置该值；

    该端口用于接收HTTPS请求；

    用于基于`Tocken`文件或客户端证书及HTTP Base的认证；

    用于基于策略的授权；

    默认不启动HTTPS安全访问控制。

【常用】: 在k8s集群中一般都是通过安全端口进行访问，node节点上会存在和master想匹配的认证文件，node节点上的kubelet就是通过安全端口向apiserver汇报节点情况。

#### 1.4 kubernetes API访问方式

- curl

  ```
    curl localhost:8080/api
    curl localhost:8080/api/v1/pods
    curl localhost:8080/api/v1/services
    curl localhost:8080/api/v1/replicationcontrollers
  ```

- Kubectl Proxy

  Kubectl Proxy代理程序既能作为API Server的反向代理，也能作为普通客户端访问API Server的代理。通过master节点的8080端口来启动该代理程序。

  ```
  kubectl proxy --port=8080 &
  具体见kubectl proxy --help
  ```

\- kubectl客户端【常用】

命令行工具kubectl客户端，通过命令行参数转换为对API Server的REST API调用，并将调用结果输出。



#### 1.5 通过API Server访问Node、Pod和Service

k8s API Server最主要的REST接口是资源对象的增删改查，另外还有一类特殊的REST接口—k8s Proxy API接口，这类接口的作用是代理REST请求，即kubernetes API Server把收到的REST请求转发到某个Node上的kubelet守护进程的REST端口上，由该kubelet进程负责响应。

- node相关接口

   Node相关的接口的REST路径为：/api/v1/proxy/nodes/{name}，其中{name}为节点的名称或IP地址。

   ```
  /api/v1/proxy/nodes/{name}/pods/   #列出指定节点内所有Pod的信息 注意这里的pod信息是来自node而不是etcd
  /api/v1/proxy/nodes/{name}/stats/  #列出指定节点内物理资源的统计信息
  /api/v1/prxoy/nodes/{name}/spec/   #列出指定节点的概要信息
  ```

- pod相关接口

    ```
  /api/v1/proxy/namespaces/{namespace}/pods/{name}/{path:*}    #访问pod的某个服务接口
  /api/v1/proxy/namespaces/{namespace}/pods/{name}        #访问Pod
  #以下写法不同，功能一样
  /api/v1/namespaces/{namespace}/pods/{name}/proxy/{path:*}    #访问pod的某个服务接口
  /api/v1/namespaces/{namespace}/pods/{name}/proxy        #访问Pod
  ```

- service相关接口

  ```
  /api/v1/proxy/namespaces/{namespace}/services/{name}
  ```

#### 1.6 通过apiserver实现集群功能模块之间的通信

集群内各个功能模块通过API Server将信息存入etcd，当需要获取和操作这些数据时，通过API Server提供的REST接口（GET\LIST\WATCH方法）来实现，从而实现各模块之间的信息交互。

- kubelet与API Server交互

  每个Node节点上的kubelet定期就会调用API Server的REST接口报告自身状态，API Server接收这些信息后，将节点状态信息更新到etcd中。kubelet也通过API Server的Watch接口监听Pod信息，从而对Node机器上的POD进行管理。

- kube-controller-manager与API Server交互

  kube-controller-manager中的Node Controller模块通过API Server提供的Watch接口，实时监控Node的信息，并做相应处理。

- kube-scheduler与API Server交互

  Scheduler通过API Server的Watch接口监听到新建Pod副本的信息后，它会检索所有符合该Pod要求的Node列表，开始执行Pod调度逻辑。调度成功后将Pod绑定到目标节点上。

#### 1.7 API Server启动参数介绍

参考：https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-apiserver/



### 2、 k8s组件之kubelet

参考文档：https://cloud.tencent.com/developer/article/1444694

https://blog.csdn.net/jettery/article/details/78891733

#### 2.1 概念

Kubelet组件运在Node节点上，维持运行中的Pods以及提供kuberntes运行时环境。

功能：

１．监视分配给该Node节点的pods

２．挂载pod所需要的volumes

３．下载pod的secret

４．通过docker/rkt来运行pod中的容器

５．周期的执行pod中为容器定义的liveness探针

６．上报pod的状态给系统的其他组件

７．上报Node的状态



#### 2.2 基本框架

整个kubelet可以按照下图所示的模块进行划分

<img src="https://Lzhang-hub.github.io/images/posts/k8s/k8s-框架-kubelet组件-1.png" width = "80%"   height = "50%"  alt="Pod与service关系"/>

1. kubelet对外暴露的端口

- 10250 kubelet API　–kublet暴露出来的端口，通过该端口可以访问获取node资源以及状态，另外可以配合kubelet的启动参数contention-profiling enable-debugging-handlers来提供了用于调试和profiling的api

- 4194 cAdvisor –kublet通过该端口可以获取到本node节点的环境信息以及node上运行的容器的状态等内容

- 10255 readonly API　–kubelet暴露出来的只读端口，访问该端口不需要认证和鉴权，该http server提供查询资源以及状态的能力．注册的消息处理函数定义src/k8s.io/kubernetes/pkg/kubelet/server/server.go:149

- 10248 /healthz –kubelet健康检查,通过访问该url可以判断Kubelet是否正常work, 通过kubelet的启动参数–healthz-port –healthz-bind-address来指定监听的地址和端口．默认值定义在pkg/kubelet/apis/kubeletconfig/v1alpha1/defaults.go



#### 2.3 核心功能模块

- PLEG（PodLifecycleEvent）PLEG会一直调用container runtime获取本节点的pods,之后比较本模块中之前缓存的pods信息，比较最新的pods中的容器的状态是否发生改变，当状态发生切换的时候，生成一个eventRecord事件，输出到eventChannel中．　syncPod模块会接收到eventChannel中的event事件，来触发pod同步处理过程，调用contaiener runtime来重建pod，保证pod工作正常．

![k8s-框架-kubelet组件-2](https://Lzhang-hub.github.io/images/posts/k8s/k8s-框架-kubelet组件-2.png)

- cAdvisor

  cAdvisor集成在kubelet中，起到收集本Node的节点和启动的容器的监控的信息，启动一个Http Server服务器，对外接收rest api请求．cAvisor模块对外提供了interface接口，可以通过interface接口获取到node节点信息，本地文件系统的状态等信息，该接口被imageManager，OOMWatcher，containerManager等所使用

  cAdvisor相关的内容详细可参考github.com/google/cadvisor

- GPUManager

  对于Node上可使用的GPU的管理，当前版本需要在kubelet启动参数中指定feature-gates中添加Accelerators=true，并且需要才用runtime=Docker的情况下才能支持使用GPU,并且当前只支持NvidiaGPU,GPUManager主要需要实现interface定义的Start()/Capacity()/AllocateGPU()三个函数

- OOMWatcher

  系统OOM的监听器，将会与cadvisor模块之间建立SystemOOM,通过Watch方式从cadvisor那里收到的OOM信号，并发生相关事件

- ProbeManager

  探针管理，依赖于statusManager,livenessManager,containerRefManager，实现Pod的健康检查的功能．当前支持两种类型的探针：LivenessProbe和ReadinessProbe

  - LivenessProbe:用于判断容器是否存活，如果探测到容器不健康，则kubelet将杀掉该容器，并根据容器的重启策略做相应的处理

  - ReadinessProbe: 用于判断容器是否启动完成

  探针有三种实现方式

  - execprobe:在容器内部执行一个命令，如果命令返回码为０，则表明容器健康
  - tcprobe:通过容器的IP地址和端口号执行TCP检查，如果能够建立TCP连接，则表明容器健康
  - httprobe:通过容器的IP地址，端口号以及路径调用http Get方法，如果响应status>=200 && status<=400的时候，则认为容器状态健康

- StatusManager

  该模块负责pod里面的容器的状态，接受从其它模块发送过来的pod状态改变的事件，进行处理，并更新到kube-apiserver中．

- Container/RefManager

  容器引用的管理，相对简单的Manager，通过定义map来实现了containerID与v1.ObjectReferece容器引用的映射．

- EvictionManager

  evictManager当node的节点资源不足的时候，达到了配置的evict的策略，将会从node上驱赶pod，来保证node节点的稳定性．可以通过kubelet启动参数来决定evict的策略．另外当node的内存以及disk资源达到evict的策略的时候会生成对应的node状态记录．

- ImageGC

  imageGC负责Node节点的镜像回收，当本地的存放镜像的本地磁盘空间达到某阈值的时候，会触发镜像的回收，删除掉不被pod所使用的镜像．回收镜像的阈值可以通过kubelet的启动参数来设置．

- ContainerGC

  containerGC负责NOde节点上的dead状态的container,自动清理掉node上残留的容器．具体的GC操作由runtime来实现．

- ImageManager

  调用kubecontainer.ImageService提供的PullImage/GetImageRef/ListImages/RemoveImage/ImageStates的方法来保证pod运行所需要的镜像，主要是为了kubelet支持cni．

- VolumeManager

  负责node节点上pod所使用的ｖolume的管理．主要涉及如下功能

  Volume状态的同步，模块中会启动gorountine去获取当前node上volume的状态信息以及期望的volume的状态信息，会去周期性的sync　volume的状态，另外volume与pod的生命周期关联，pod的创建删除过程中volume的attach/detach流程．更重要的是kubernetes支持多种存储的插件，kubelet如何调用这些存储插件提供的interface.涉及的内容较多，更加详细的信息可以看kubernetes中volume相关的代码和文档．

- containerManager

  负责node节点上运行的容器的cgroup配置信息，kubelet启动参数如果指定–cgroupPerQos的时候，kubelet会启动gorountie来周期性的更新pod的cgroup信息，维持其正确．实现了pod的Guaranteed/BestEffort/Burstable三种级别的Qos,通过配置kubelet可以有效的保证了当有大量pod在node上运行时，保证node节点的稳定性．该模块中涉及的struct主要包括

![k8s-框架-kubelet组件-3](https://Lzhang-hub.github.io/images/posts/k8s/k8s-框架-kubelet组件-3.png)



- runtimeManager

  containerRuntime负责kubelet与不同的runtime实现进行对接，实现对于底层container的操作，初始化之后得到的runtime实例将会被之前描述的组件所使用．

  当前可以通过kubelet的启动参数–container-runtime来定义是使用docker还是rkt.runtime需要实现的接口定义在src/k8s.io/kubernetes/pkg/kubelet/apis/cri/services.go文件里面

- podManager

  podManager提供了接口来存储和访问pod的信息，维持static pod和mirror　pods的关系，提供的接口如下所示

![k8s-框架-kubelet组件-4](https://Lzhang-hub.github.io/images/posts/k8s/k8s-框架-kubelet组件-4.png)

​		跟其他Manager之间的关系，podManager会被statusManager/volumeManager/runtimeManager所调用，并且podManager的接口处理流程里面同样会调用secretManager以及configMapManager.

![k8s-框架-kubelet组件-5](https://Lzhang-hub.github.io/images/posts/k8s/k8s-框架-kubelet组件-5.png)



#### 2.4 kubelet启动参数介绍

参考链接：https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kubelet/



### 3、k8s组件之controler manager
参考资料：https://blog.csdn.net/huwh_/article/details/75675761
#### 3.1 概念
Controller Manager 由 kube-controller-manager 和 cloud-controller-manager 组成，是 Kubernetes 的大脑，它通过 apiserver 监控整个集群的状态，并确保集群处于预期的工作状态。

Controller Manager作为集群内部的管理控制中心，负责集群内的Node、Pod副本、服务端点（Endpoint）、命名空间（Namespace）、服务账号（ServiceAccount）、资源定额（ResourceQuota）的管理，当某个Node意外宕机时，Controller Manager会及时发现并执行自动化修复流程，确保集群始终处于预期的工作状态。

kube-controller-manager 由一系列的控制器组成，cloud-controller-manager 在 Kubernetes 启用 Cloud Provider 的时候才需要，用来配合云服务提供商的控制，也包括一系列的控制器。


#### 3.2 主要控制器

controller manager在启动时会遍历所有的controller，执行初始化函数，所有的controller代码位置：`cmd/kube-controller-manager/app/coontrollermanager.go：387`
常用控制器如下：
![k8s-controller-manager组件-1](https://Lzhang-hub.github.io/images/posts/k8s/k8s-controller-manager组件-1-1644571592159.png)

##### 3.2.1 Replication Controller

为了区分，资源对象Replication Controller简称RC,而本文是指Controller Manager中的Replication Controller，称为副本控制器。副本控制器的作用即保证集群中一个RC所关联的Pod副本数始终保持预设值。
- 只有当Pod的重启策略是Always的时候（RestartPolicy=Always），副本控制器才会管理该Pod的操作（创建、销毁、重启等）。
- RC中的Pod模板就像一个模具，模具制造出来的东西一旦离开模具，它们之间就再没关系了。一旦Pod被创建，无论模板如何变化，也不会影响到已经创建的Pod。
- Pod可以通过修改label来脱离RC的管控，该方法可以用于将Pod从集群中迁移，数据修复等调试。
- 删除一个RC不会影响它所创建的Pod，如果要删除Pod需要将RC的副本数属性设置为0。
- 不要越过RC创建Pod，因为RC可以实现自动化控制Pod，提高容灾能力。

Replication Controller职责和使用场景：
![k8s-controller-manager组件-2](https://Lzhang-hub.github.io/images/posts/k8s/k8s-controller-manager组件-2.png)

##### 3.2.2 Node Controller

kubelet在启动时会通过API Server注册自身的节点信息，并定时向API Server汇报状态信息，API Server接收到信息后将信息更新到etcd中。

Node Controller通过API Server实时获取Node的相关信息，实现管理和监控集群中的各个Node节点的相关控制功能。流程如下
![k8s-controller-manager组件-3](https://Lzhang-hub.github.io/images/posts/k8s/k8s-controller-manager组件-3.png)

Controller Manager在启动时如果设置了--cluster-cidr参数，那么为每个没有设置Spec.PodCIDR的Node节点生成一个CIDR地址，并用该CIDR地址设置节点的Spec.PodCIDR属性，防止不同的节点的CIDR地址发生冲突。
逐个读取节点信息，如果节点状态变成非“就绪”状态，则将节点加入待删除队列，否则将节点从该队列删除。

##### 3.2.3 ResourceQuota Controller

资源配额管理确保指定的资源对象在任何时候都不会超量占用系统物理资源。
支持三个层次的资源配置管理：
1). 容器级别：对CPU和Memory进行限制
2). Pod级别：对一个Pod内所有容器的可用资源进行限制
3). Namespace级别：

- Pod数量
- Replication Controller数量
- Service数量
- ResourceQuota数量
- Secret数量
- 可持有的PV（Persistent Volume）数量

**说明：**

- k8s配额管理是通过Admission Control（准入控制）来控制的；
- Admission Control提供两种配额约束方式：LimitRanger和ResourceQuota；
    - LimitRanger作用于Pod和Container；
    - ResourceQuota作用于Namespace上，限定一个Namespace里的各类资源的使用总额。

ResourceQuota Controller控制流程图：
![k8s-controller-manager组件-4](https://Lzhang-hub.github.io/images/posts/k8s/k8s-controller-manager组件-4.png)

##### 3.2.4 Namespace Controller

用户通过API Server可以创建新的Namespace并保存在etcd中，Namespace Controller定时通过API Server读取这些Namespace信息。  

如果Namespace被API标记为优雅删除（即设置删除期限，DeletionTimestamp）,则将该Namespace状态设置为“Terminating”,并保存到etcd中。同时Namespace Controller删除该Namespace下的ServiceAccount、RC、Pod等资源对象。

##### 3.2.5 Endpoint Controller

Service、Endpoint、Pod的关系：
![k8s-controller-manager组件-5](https://Lzhang-hub.github.io/images/posts/k8s/k8s-controller-manager组件-5.png)

Endpoints表示了一个Service对应的所有Pod副本的访问地址，而Endpoints Controller负责生成和维护所有Endpoints对象的控制器。它负责监听Service和对应的Pod副本的变化。
    1. 如果监测到Service被删除，则删除和该Service同名的Endpoints对象；
        2. 如果监测到新的Service被创建或修改，则根据该Service信息获得相关的Pod列表，然后创建或更新Service对应的Endpoints对象。
        3. 如果监测到Pod的事件，则更新它对应的Service的Endpoints对象。

kube-proxy进程获取每个Service的Endpoints，实现Service的负载均衡功能。

##### 3.2.6 Service Controller

Service Controller是属于kubernetes集群与外部的云平台之间的一个接口控制器。Service Controller监听Service变化，如果是一个LoadBalancer类型的Service，则确保外部的云平台上对该Service对应的LoadBalancer实例被相应地创建、删除及更新路由转发表。

#### controller manager 启动参数
参考资料：https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-controller-manager/

## 待看：
- 一些网络插件的介绍：
https://www.huaweicloud.com/articles/fc0b33b0ae7f5902dd7ae96fca744b32.html
- Admission Control（准入控制）
- Service Controller

