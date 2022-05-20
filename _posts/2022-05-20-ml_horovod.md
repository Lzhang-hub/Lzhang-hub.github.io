---
layout: post
title: 分布式训练框架horovod
date: 2022-05-20
tags: 机器学习   
---



## 一、基本概念

### 1.1 并行机制：

- 模型并行

  - 模型很大的情况下，一个节点放不下，放到不同的卡上训练，即将模型的图拆分。

  - 模型后面的计算必须依赖前面的计算

- 数据并行

  - 模型不是很大的情况，通常情况下，数据量很大
  - 多节点上分割数据集进行训练

- 使用：

  - 卷积层，占据了90% ~ 95% 的计算量，5% 的参数，但是对结果具有很大的表达能力。
  - 全连接层，占据了 5% ~ 10% 的计算量， 95% 的参数，但是对于结果具有相对较小的表达的能力。

  **综上**：卷积层计算量大，所需参数系数 W 少，全连接层计算量小，所需参数系数 W 多。因此对于卷积层适合使用数据并行，对于全连接层适合使用模型并行。

### 1.2 通信

分布式训练，特别是多机多卡的情况，就会涉及到通信问题，即如何对分布训练的结果做聚合。

#### 1.2.1 方法和架构

- 通信方法：Share memory 和 Message passing。

- Share memory：共享内存，只能在同一个节点内使用，不能跨节点。

  ![img](https://Lzhang-hub.github.io/images/posts/ml/1850883-20210606212418325-1488844464.png)

- Message passing 通信架构：
  
  ![img](https://Lzhang-hub.github.io/images/posts/ml/1850883-20210606212438201-2042403909.png)
  
  - Client-Server 架构: 一个 server 节点协调其他节点工作，其他节点是用来执行计算任务的 worker。
  - Peer-to-Peer 架构：每个节点都有邻居，邻居之间可以互相通信。

#### 1.2.2 异步vs同步

- 同步：同步指的是所有的设备都是采用相同的模型参数来训练，等待所有设备的mini-batch训练完成后，收集它们的梯度然后取均值，然后执行模型的一次参数更新。<u>存在拖油瓶问题，其中一个没完成整体就没完成。</u>

  ![img](https://Lzhang-hub.github.io/images/posts/ml/1850883-20210606212511542-933900105.png)

- 异步：异步训练中，各个设备完成一个mini-batch训练之后，不需要等待其它节点，直接去更新模型的参数，这样总体会训练速度会快很多。<u>存在是梯度失效问题。</u>

  ![img](https://Lzhang-hub.github.io/images/posts/ml/1850883-20210606212533329-1159359838.png)

- 对比：
  - 异步更新可能会更快速地完成整个梯度计算。
  - 同步更新 可以更快地进行一个收敛。

### 1.3 具体架构

- MapReduce：消息传递，client-server，批同步
- Parameter Server：消息传递，client-server，异步
- Decentralized：消息传递，P2P，同步或异步

#### 1.3.1  Map

执行步骤：

- Spark Driver 就是 Server，Spark Executor 就是 Worker 节点，每一个梯度下降过程包含一个广播、map和一个 reduce 操作。
- Server 定义了 map操作（就是具体的训练），也可以把信息广播到worker节点。
- Worker 会执行 map 操作进行训练，在此过程中，数据被分给 worker 进行计算。
- 计算结束后，worker把计算结果传回 driver 处理，这个叫做reduce。
- 在 reduce 过程中，Server 节点对 worker 传来的计算结果进行聚合之后，把聚合结果广播到各个worker节点，进行下一次迭代。

#### 1.3.2 参数服务器 (PS)

<u>MapReduce只有等所有map都完成了才能做reduce操作。Parameter server 也是一种client-server架构。和MapReduce不同在于 Parameter server 可以是异步的</u>

执行步骤：

- 所有的参数都存储在参数服务器中，而 工作节点（worker） 是万年打工仔。
- 工作节点 们只负责计算梯度，待所有计算设备完成梯度计算之后，把计算好的梯度发送给参数服务器，这样参数服务器收到梯度之后，执行一定的计算（梯度平均等）之后，就更新其维护的参数，做到了在节点之间对梯度进行平均，利用平均梯度对模型进行更新。
- 然后参数服务器再把更新好的新参数返回给所有的工作节点，以对每个节点中的模型副本应用一致化更新。
- 打工仔们会再进行下一轮的前后向计算。

![img](https://Lzhang-hub.github.io/images/posts/ml/1850883-20210606212602276-1607577597.png)

#### 1.3.3 Decentralized Network

Decentralized Network 就是去中心化网络，其特点如下：

- 去中心化网络没有一个中心节点，属于 Peer-to-Peer 架构。
- 采用 message passing 进行通信，且节点只和邻居通信。
- 并行方式可以采用异步或者同步。
- 去中心化网络的收敛情况取决于网络连接情况：
  - 连接越紧密，收敛性越快，当强连接时候，模型可以很快收敛；
  - 如果不是强连接，它可能不收敛；

### 1.4 All Reduce

在每个子集上面算出一些局部统计量，然后整合出全局统计量，并且再分配给各个节点去进行下一轮的迭代，这样一个过程就是AllReduce。

all reduce类似于map reduce，map reduce主要是通用任务的多阶段处理，而all reduce则在必要的时候可以占领一台机器一直跑到底。

#### 1.4.1 并行任务通信分类

并行任务的通信一般可以分为 Point-to-point communication 和 Collective communication。

- P2P 这种模式只有一个sender和一个receiver，实现起来比较简单，比如NV GPU Direct P2P技术服务于单机多卡的单机卡间数据通信 。
- Collective communication包含多个sender和多个receiver，一般的通信原语包括 broadcast，gather,all-gather，scatter，reduce，all-reduce，reduce-scatter，all-to-all等。如nccl

#### 1.4.2 ring allreduce

特点：

- Ring-Allreduce 的命名中的 Allreduce 则代表着没有中心节点，架构中的每个节点都是梯度的汇总计算节点。
- 此种算法各个节点之间只与相邻的两个节点通信，并不需要参数服务器。因此，所有节点都参与计算也参与存储，也避免产生中心化的通信瓶颈。
- 相比PS架构，Ring-Allreduce 架构是带宽优化的，因为集群中每个节点的带宽都被充分利用。
  - 在 ring-allreduce 算法中，每个 N 节点与其他两个节点进行 2 * (N-1) 次通信。在这个通信过程中，一个节点发送并接收数据缓冲区传来的块。在第一个 N - 1 迭代中，接收的值被添加到节点缓冲区中的值。在第二个 N - 1 迭代中，接收的值代替节点缓冲区中保存的值。百度的文章证明了这种算法是带宽上最优的，这意味着如果缓冲区足够大，它将最大化地利用可用的网络。

策略：

Ring-based AllReduce 策略包括 Scatter-Reduce 和 AllGather 两个阶段。[详细细节参考](https://www.cnblogs.com/rossiXYZ/p/14856464.html)

- 首先是scatter-reduce，scatter-reduce 会逐步交换彼此的梯度并融合，最后每个 GPU 都会包含完整融合梯度的一部分，是最终结果的一个块。

  假设环中有 N 个 worker，每个 worker 有长度相同的数组，需要将 worker 的数组进行求和。在 Scatter-Reduce 阶段，每个 worker 会将数组分成 N 份数据块，然后 worker 之间进行 N 次数据交换。在第 k 次数据交换时，第 i 个 worker 会将自己的 (i - k) % N 份数据块发送给下一个 worker。接收到上一个 worker 的数据块后，worker 会将其与自己对应的数据块**求和**。

- 然后是allgather。GPU 会逐步交换彼此不完整的融合梯度，最后所有 GPU 都会得到完整的最终融合梯度。

  在执行完 Scatter-Reduce 后，每个 worker 的数组里都有某个数据块是最终求和的结果，现在需要将各数据块的最后求和结果发送到每个 worker 上。和 Scatter-Reduce 一样，也需要 N 次循环。在第 k 次循环时，第 i 个 worker 会将其第 (i+1-k)%N 个数据块发送给下一个 worker 。接收到前一个 worker 的数据块后，worker 会用接收的数据快**覆盖**自己对应的数据块。进行 N 次循环后，每个 worker 就拥有了数组各数据块的最终求和结果了。



## 二、hovorod框架

### 2.1 hovorod机制

hovorod采用的是**数据并行**的机制在GPU上训练。

每一个迭代的操作方法如下：

1. 每个 worker 将维护自己的模型权重副本和自己的数据集副本。

2. 收到执行信号后，每个工作进程都会从数据集中提取一个不相交的批次，并计算该批次的梯度。

3. Workers 使用ring all-reduce算法来同步彼此的梯度，从而在本地所有节点上计算同样的平均梯度。

   1. 将每个设备上的梯度 tensor 切分成长度大致相等的 num_devices 个分片，后续每一次通信都将给下一个邻居发送一个自己的分片（同时从上一个邻居接受一个新分片）。

   2. ScatterReduce 阶段：通过 num_devices - 1 轮通信和相加，在每个 device 上都计算出**一个** tensor 分片的和，即每个 device 将有一个块，其中包含所有device 中该块中所有值的总和；具体如下：

      ![img](https://Lzhang-hub.github.io/images/posts/ml/1850883-20210606214604856-157216298.png)

   3. AllGather 阶段：通过 num_devices - 1 轮通信和覆盖，将上个阶段计算出的每个 tensor 分片的和 广播到其他 device；最终所有节点都拥有**所有**tensor分片和。具体如下：

      ![img](https://Lzhang-hub.github.io/images/posts/ml/1850883-20210606214657923-462449065.png)

   4. 在每个设备上合并分片，得到梯度和，然后除以 num_devices，得到平均梯度；

4. 每个 worker 将 梯度更新 应用于其模型的本地副本。

5. 执行下一个batch。

### 2.2 代码阅读

官方代码样例：


```
import tensorflow as tf
import horovod.tensorflow.keras as hvd

// Horovod: initialize Horovod.
hvd.init() # 初始化 Horovod，启动相关线程和MPI线程，启动MPI会去调用C++部分

// Horovod: pin GPU to be used to process local rank (one GPU per process)
// 依据 local rank 为不同的进程分配不同的GPU
gpus = tf.config.experimental.list_physical_devices('GPU')
for gpu in gpus:
    tf.config.experimental.set_memory_growth(gpu, True)
if gpus:
    tf.config.experimental.set_visible_devices(gpus[hvd.local_rank()], 'GPU')

(mnist_images, mnist_labels), _ = \
    tf.keras.datasets.mnist.load_data(path='mnist-%d.npz' % hvd.rank())

// 切分数据  
dataset = tf.data.Dataset.from_tensor_slices(
    (tf.cast(mnist_images[..., tf.newaxis] / 255.0, tf.float32),
             tf.cast(mnist_labels, tf.int64))
)
dataset = dataset.repeat().shuffle(10000).batch(128)

mnist_model = tf.keras.Sequential([
    tf.keras.layers.Conv2D(32, [3, 3], activation='relu'),
    ......
    tf.keras.layers.Dense(10, activation='softmax')
])

// Horovod: adjust learning rate based on number of GPUs.
scaled_lr = 0.001 * hvd.size() // 根据Worker的数量增加学习率的大小
opt = tf.optimizers.Adam(scaled_lr)

//Horovod: add Horovod DistributedOptimizer.
// 把常规TensorFlow Optimizer通过Horovod包装起来，进而使用 ring-allreduce 来得到平均梯度
opt = hvd.DistributedOptimizer(
    opt, backward_passes_per_step=1, average_aggregated_gradients=True)

// Horovod: Specify `experimental_run_tf_function=False` to ensure TensorFlow
// uses hvd.DistributedOptimizer() to compute gradients.
mnist_model.compile(loss=tf.losses.SparseCategoricalCrossentropy(),
                    optimizer=opt, metrics=['accuracy'],
                    experimental_run_tf_function=False)

callbacks = [
    hvd.callbacks.BroadcastGlobalVariablesCallback(0), // 广播初始化，将模型的参数从第一个设备传向其他设备，以保证初始化模型参数的一致性
    hvd.callbacks.MetricAverageCallback(),
    hvd.callbacks.LearningRateWarmupCallback(initial_lr=scaled_lr, warmup_epochs=3, verbose=1),
]

// Horovod: save checkpoints only on worker 0 to prevent other workers from corrupting them. // 只有设备0需要保存模型参数作为checkpoint
if hvd.rank() == 0:
    callbacks.append(tf.keras.callbacks.ModelCheckpoint('./checkpoint-{epoch}.h5'))

// Horovod: write logs on worker 0.
verbose = 1 if hvd.rank() == 0 else 0

// Train the model.
// Horovod: adjust number of steps based on number of GPUs.
mnist_model.fit(dataset, steps_per_epoch=500 // hvd.size(), callbacks=callbacks, epochs=24, verbose=verbose)
```


#### 2.2.1 初始化部分

- 首选导入Python文件，在Python的初始化中会引入So库，SO库就是C++编译的结果，它可以调用C++的函数，python 的 _allreduce 函数就会把功能转发给 C++，由 MPI_LIB.horovod_allreduce 完成。
- hvd,init()部分，`horovod` 管理的所有状态都会传到 hvd 对象中。会调用HorovodBasics 的函数，这部分会一直深入到 C++世界，调用了大量的 MPI_LIB_CTYPES 函数
- 然后就是在C++中进行初始化，调用 MPI_Comm_dup 获取一个 Communicator，这样就有了和 MPI 协调的基础，然后调用 InitializeHorovodOnce
- InitializeHorovodOnce 是初始化的主要工作，主要是：
  - 依据是否编译了 mpi 或者 gloo，对各自的 context 进行处理，为 globalstate 创建对应的 controller；
  - 启动了后台线程 BackgroundThreadLoop 用来在各个worker之间协调；
- HorovodGlobalState起到了集中管理各种全局变量的作用，在 horovod 中是一个全局变量，其中的元素可以供不同的线程访问。是在backgroundThreadLoop 中完成初始化，比较重要的有：
  - **controller** 管理总体通信控制流；
  - **tensor_queue** 会处理从前端过来的通信需求（allreduce，broadcast 等）；

![img](https://Lzhang-hub.github.io/images/posts/ml/1850883-20210606214739435-703722722.png)

#### 2.2.2 用户代码

##### hvd概念

- local rank：hovorod会为每个设备上的GPU启动一个该训练的副本，`local rank`就是某一台机器上每个训练的唯一编号(进程号或者GPU ID号)，范围是0-n-1，n就是GPU的总个数；
- rank：可以认为是代表分布式任务里的一个执行训练的唯一全局编号（用于进程间通讯），Rank 0 在Horovod中有特殊作用：
  - Rank 0 会充当 coordinator 的角色。它会协调来自其他 Rank 的 MPI 请求；
  - Rank 0 也用来把参数广播到其他进程 & 存储 checkpoint。
- world_size：进程总数量，会等到所有world_size个进程就绪之后才会开始训练。

##### 数据处理

- 首先，训练的数据需要放置在任何节点都能访问的地方。
- 其次，Horovod 需要对数据进行分片处理，需要在不同机器上按Rank进行切分，以保证每个GPU进程训练的数据集是不一样的。
- 数据集本体需要出于数据并行性的需求而被拆分为多个分片，Horovod的不同工作节点都将分别读取自己的数据集分片。

##### 广播初始化变量

```
hvd.callbacks.BroadcastGlobalVariablesCallback(0)
```

这句代码保证的是 rank 0 上的所有参数只在 rank 0 初始化，然后广播给其他节点，即变量从第一个流程向其他流程传播，以实现参数一致性初始化。	

broadcast_variables 调用了 _make_broadcast_group_fn 完成功能，可以看到对于 执行图 的每个变量，调用了 broadcast。broadcast 就是调用了 MPI 函数真正完成了功能。

在训练过程中也会定期同步参数，同步参数也是调用broadcast功能完成。

##### distributedoptimizer

```
// Horovod: add Horovod DistributedOptimizer.
opt = hvd.DistributedOptimizer(
    opt, backward_passes_per_step=1, average_aggregated_gradients=True)

```

TF Optimizer 是模型训练的关键API，可以获取到每个OP的梯度并用来更新权重。HVD 在原始 TF Optimizer的基础上包装了**hvd.DistributedOptimizer**。

- hvd.DistributedOptimizer继承 keras Optimizer，在计算时候，依然由传入的原始优化器做计算。
- 在得到计算的梯度之后，调用 hvd.allreduce 或者 hvd.allgather 来计算。
- 最后实施这些平均之后的梯度。从而实现整个集群的梯度归并操作。

### 2.3 总结

- Hovorod 怎么进行数据分割？
  - 答案：有的框架可以自动做数据分割。如果框架不提供，则需要用户自己进行数据分割，以保证每个GPU进程训练的数据集是不一样的。
- Hovorod 怎么进行模型分发？
  - 用户需要手动拷贝训练代码到各个节点上。
- Hovorod 启动时候，python 和 C++ 都做了什么？
  - 答案：python 会引入 C++库，初始化各种变量和配置。C++部分会对 MPI，GLOO上下文进行初始化，启动后台进程处理内部通信。
- 如何确保 Hovorod 启动时候步骤一致；
  - 答案： rank 0 上的所有参数只在 rank 0 初始化，然后广播给其他节点，即变量从第一个流程向其他流程传播，以实现参数一致性初始化。


[参考](https://www.cnblogs.com/rossiXYZ/p/14856543.html)

参考[2](https://zhuanlan.zhihu.com/p/40578792)