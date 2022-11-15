# DART:Pipelined Data-Parallel CPU/GPU Scheduling for Multi-DNN Real-Time Inference

**[Pipelined Data-Parallel CPU/GPU Scheduling for Multi-DNN Real-Time Inference](https://ieeexplore.ieee.org/document/9052147)**  

http://2019.rtss.org/wp-content/uploads/2020/05/240_Presentation.pdf

解决的主要问题：在多核的异构集群上对推理请求的调度，主要实现了保证real-time 任务的时间以及最大化best-effort任务的吞吐量。【该异构集群上的CPU共享主存】

## 论文作者：

* Yecheng Xiang：在攻读加利福尼亚大学河滨分校博士学位，他专注于提高安全关键应用（例如自动驾驶汽车）的多核和 GPU 架构的中间件的可靠性、可预测性和性能效率。 他一直致力于研究流行的 AI/DNN 和 ROS 框架，并致力于通过提出新的系统抽象、调度程序和资源分配/预留技术来从系统级的角度提高它们的性能。
* Hyoseung Kim：在韩国大学信息安全研究生院攻读信息安全博士学位。他的研究兴趣包括功能加密、功能签名和匿名证明。

## 研究背景和动机

现有的DNN调度框架【Caffe, TensorFlow and Torch】顺序使用每个 DNN 模型的单独进程以顺序方式处理推理作业，不为任务提供优先级和实时任务支持。当有着不同时间约束的多个任务到达时，real-time 任务会产生无法预测的延迟，这导致了任务的调度性难以被分析。这是在安全关键应用程序中使用现有框架的主要限制因素。

因此，本文设计了DART，具有可分析实时保证的 DNN 调度框架。该调度框架主要目标是：

* real-time 任务，确保该类型任务的时间约束，最小化该任务的响应时间。
* best-effort 任务，最大化该类型任务的吞吐量。



## 系统建模

系统：多核的异构集群【⚠️：一个很大的server，多个CPU和GPU核】

*  $R$ 表示系统配备的不同的CPU和GPU设备，$R = {c_1,c_2,...,c_r}$ ，$c_i$ 要么是 CPU 要么是 GPU。

* 引入一个节点的概念，每一个节点 $p_k$ 都是 $R$ 的一个子集， $R = \bigcup p_k$ ，并且保证 $i!=j,p_i \bigcap p_j = \varnothing$ 。

  * $p_k$ 是一个CPU节点，则只有一种CPU类型的核。【同质CPU集群】

  * $p_k$ 是一个GPU节点，它有一个 GPU 以及一个 CPU 来支持 GPU 相关的操作。

* $P = \bigcup {p_k} = {p_1,p_2,...,p_k}$ 【给定系统中的CPU核GPU，有很多种方式来组合成节点，所以会产生多种节点的配置 $P$ 】





一个任务是多个连续到达【请求间隔时间为minimum inter-arrival time】的相同模型的请求。一个任务可以用一个五元组来描述：
$$
\tau_i := (C_i,T_i,D_i,L_i)
$$

* $C_i$ 表示一个请求的最坏执行时间。
* $T_i$ 表示minimum inter-arrival time。
* $D_i$  每个请求的相对截止时间。【对于没有 $D_i$ 的best-effort，该值则会被设置成 ∞。
* $L_i$ 表示模型的层数。



每一个请求的执行会被分割成多个层的执行，每一层的执行时间取决于该层在哪个节点上执行。

$\tau_{i,j}$ 表示任务 $\tau_i$  的第 $j$ 层。$C_{i,j} (p_k)$ 表示 $\tau_{i,j}$ 在 $p_k$ 这个节点上执行所花费的时间。

* 如果在CPU节点上执行，则 $C_{i,j}(p_k)$ 表示在该节点速度最慢的CPU核上执行所需要的时间。

* 如果在GPU节点上执行，则 $C_{i,j}(p_k) = (G_i^{hd}(pk),G_{i,j}^e(p_k),G_{i,j}^m(p_k),G_{i,j}^{dh})$ 

  * $G_i^{hd}(pk)$ ：表示从CPU拷贝到GPU花费的时间。
  * $G_{i,j}^e(p_k)$ ：表示在GPU上执行花费的时间。
  * $G_{i,j}^m(p_k)$：表示$\tau_{i,j}$ 在CPU处理花费的时间。
  * $G_{i,j}^{dh}$：表示从GPU拷贝到CPU花费的时间。

  所以：$C_{i,j}(p_k) = G_i^{hd}(pk)+G_{i,j}^e(p_k)+G_{i,j}^m(p_k)+G_{i,j}^{dh}$

所以：$C_i = \sum_{j = 1}^{L_i} C_{i,j}(p_{k_{i,j}})$  ，其中 $p_{k_{i,j}}$  表示执行请求 $i$ 的第 $j$ 层被分配的节点。



## DART 框架

### 调度架构

三个关键抽象：

* stage：任务 $\tau_i$ 的一组连续的层，用 $s$ 表示。$\bigcup s = \bigcup\tau_{i,j}$ ，即使是相同的模型，可以使用不同的分割方式来划分stage，所以会导致最后stage集合的多样性。
* execution pipeline：请求的stage的执行顺序。是串行执行的，后面的stage的输入依赖于前面的stage的结果。
* worker：一个调度器单元，它执行到达相应节点的每个任务阶段。

三个抽象设置的原因是：DNN 任务的每一层可能对异构计算资源提供的并行化级别具有不同的敏感性。



每一个节点上有两个worker，一个是执行real-time任务的worker，一个是执行best-effort任务的worker。【有两个worker的原因，可能是考虑real-time和best-effort任务的优先级以及目标不同，所以直接创建两个worker】

* real-time task worker：
  * 优先队列：DM策略
  * 更高的优先级
* best-effort worker
  * 优先队列：EDF 策略
  * 较低的优先级

虽然 RT worker 可以随时抢占同一节点上的 BE worker，但每个 worker 内的任务执行是非抢占的【这个主要是因为GPU的限制】。==因为GPU只有两个优先级，高和低，所以同为高或者同为低是无法抢占的==。 

#### CPU节点调度

每一个线程对应一个CPU核，假设一个节点有c个CPU核。

所以对于real-time task worker和best-effort worker，OpenBLAS-rt会为这两个worker都创建c个线程。

* 对于real-time task 线程，会被赋予Linux 实时优先级。

* 对于best-effort 线程，会被赋予普通优先级。

实验：在三个场景下，比较CPU调度策略的性能：

* 不用多线程库，在现有的调度框架下启动多个实例。
* 使用现有的多线程库的单进程来处理。
* 使用OpenBLAS-rt的DART

<img src="https://s3.bmp.ovh/imgs/2022/11/10/6c018a4cdf654487.png" alt="image-20221110100519636" style="zoom:50%;" />

实验配置：

* 三个BE任务，一个RT任务，都是同一种模型请求，该模型第一层有更多的CPU不会加快速度，第二层则会。
* RT任务在1时刻达到，11时刻是deadline。
* 系统有三个CPU。

实验分析：

* 对于a和b两个场景，RT任务都会超时，因为现有的调度框架忽略了DNN任务的优先级。
* b场景比a场景花费更长的时间，这个是因为现有的多线程库总是使用预先设定的线程数量，不管增加CPU是否会加速处理。
* 对于DART不仅没有让RT任务超时，而且获得了更短的完成时间。



#### GPU节点调度

Pascal 以及之后架构的GPU有着CUDA Stream优先级和线程级别的抢占。

RT 任务采用高优先级；BE 任务采用低优先级。

RT woker一次只执行一个kernel，也就是每次只使用一个高优先级的CUDA Stream；BE worker 一次可以使用多个低优先级的CUDA Stream，这使得BE任务的多个kernel 并发执行。【BE woker的CUDA Stream的数量是可配置的，根据内核参数以及GPU的计算能力共同决定】

实验：比较三个场景：

* 使用一个默认的CUDA流，没有内核的优先级。
* 启动多个实例，不同的cuda context。
* DART：具有流优先级的共享 CUDA 上下文。

<img src="https://s3.bmp.ovh/imgs/2022/11/10/61572223673ab87b.png" alt="image-20221110104450166" style="zoom:50%;" />

实验分析：

* 对于场景a，每次只启动一个实例，序列化了CPU和GPU的操作，所以速度最慢。
* 对于场景b，同时启动多个实例，但是因为是每一个task是从不同的cuda context启动的，所以不能并发执行。
* 对于场景c，BE task可以并发执行，提高了GPU利用率，获得了很高的吞吐量；同时存在Stream 优先级，所以RT任务会抢占GPU，缩短了RT任务的响应时间。



#### 批量执行

==只允许对 DART 中的 BE 任务进行批处理。==

批量大小可由用户离线配置。



### 资源管理

DART框架的资源管理模式包含两个算法：

* 为DNN任务==划分stage==，并将==stage分配到节点==上。
* 另一种是找到满足所有RT任务的时序约束并最小化它们的响应时间的==节点配置== 。

#### 可调度性分析【 ==只会分析RT任务的可调度性== 】

* $\mathbb{C}_{i,k}$ ：任务 $\tau_i$ 的一些stage被分配到 $p_k$ ，其累计执行时间

$$
\mathbb{C}_{i,k} = \left\{\begin{matrix}
 & (\sum_{\tau_{i,j}\in p_k}C_{i,j}(p_k)) + \epsilon_{i,k} & :\exist \tau_{i,j} \in p_k\\ 
 & 0 & :otherwise 
\end{matrix}\right.
$$

$\epsilon_{i,k}$ 表示在节点 $p_k$ 上总共的stage执行开销。这个既可以用于CPU也可以用于GPU，但是如果连续几层都在同一个GPU上执行，则会消除很多不必要的内存拷贝，从而缩短stage的执行时间。
$$
\mathbb{C}_{i,k} = \sum_{\tau_{i,j}\in p_k}(G_{i,j}^e(p_k) + G_{i,j}^m(p_k)) + G_{i,j1}^{hd}(p_k) + G_{i,j2}^{dh}(p_k) + \epsilon_{i,k}
$$
其中 $j1$ 和 $j2$ 分别表示在 $p_k$ 节点上执行 $\tau_i$ 的第一层和最后一层。

* 分析 $\epsilon_{i,k}$ 

  $\epsilon_{i,k}^s$  表示 $\tau_i$ 从节点 $p_k$  通知下一个节点的时间。

  $\epsilon_k^{cp}$ 表示CPU抢占开销

  $\epsilon_k^{gp}$ 表示GPU抢占开销

  $\epsilon_k^{gm}$ 由BE task 引起的CUDA 内存拷贝阻塞时间。

  <img src="https://s3.bmp.ovh/imgs/2022/11/10/5130c2146311519f.png" alt="image-20221110164722321" style="zoom:50%;" />

  等式(3) 表示在 GPU 节点上运行的所有 BE 任务的层中占用的内存复制时间最长的时间。对于有些使用batch的，则内存复制时间要乘以b。

  <img src="https://s3.bmp.ovh/imgs/2022/11/10/2e1dfc844dff4373.png" alt="image-20221110165229935" style="zoom:50%;" />

  因为$\epsilon_k^{gp}$ 和$\epsilon_k^{gm}$ 不会同时出现，所以取较大值。

* 最坏响应时间 $R_i$ 

  <img src="https://s3.bmp.ovh/imgs/2022/11/10/6a51df227e05405a.png" alt="image-20221110165442693" style="zoom: 50%;" />

  该分析是基于论文 **[Transforming Distributed Acyclic Systems into Equivalent Uniprocessors under Preemptive and Non-Preemptive Scheduling](https://ieeexplore.ieee.org/document/4573119)** 

  <img src="https://s3.bmp.ovh/imgs/2022/11/10/bb733d9437aebb6a.png" alt="image-20221110165634606" style="zoom:50%;" />

  不断迭代(5)直到满足最坏的执行时间 $R_i < D_i$ 则表示该任务是可调度的。



#### 设计任务管道的stage

给定了节点的配置，划分合适的stages。

<img src="https://s3.bmp.ovh/imgs/2022/11/10/98a2982f82aeb170.png" alt="image-20221110170527541" style="zoom:50%;" />

$M[i,j]$ 表示task的前面 $i$ 层分配给了前面 $j$ 个节点的最小的最大利用率。

通过这个可以将模型的若干层划分成多个stage。



#### 查找任务的节点配置

使用了两个算法：

* Algo2 暴力搜到全部可能的节点分配。【限制条件只有一个：CPU节点同构，GPU节点只有一个GPU】

* Algo1 为一个静态的任务集找到一个`加权最坏情况响应时间` 最短的节点配置。

在==初始化阶段==，在一个静态的任务集中使用两个算法来==确定一个节点配置==；在==运行阶段==，则使用预先设定好的节点配置，而不会更新节点配置。



### 其他杂项组件

#### Layer-wise Execution Time Profiling

为了获得每一层的最坏执行时间，DART会在所有可能的节点配置运行所有支持的模型，重复运行 $n$ 次，然后最坏执行时间为运行过程中观察到的最长执行时间。

#### Admission Control

运行过程中，每一个新任务到来，系统会调用DP算法来对这个任务的stage进行分割，并且识别该任务是BE还是RT。如果任务是RT，那么这个任务只有在包括新任务的所有RT任务都是可调度的，才会接受该任务。如果任务是BE任务，那么只有当该任务最长的GPU内存拷贝时间小于等于 $\epsilon_k^{gm}$ 才会接受该任务。

#### Run-time Task Enforcement

每一层的执行时间都会被检测，如果存在任务某一层的最坏执行时间超过了记录在数据库中的时间。

* 如果该任务是RT任务，则会将该任务降级到BE任务，该任务的剩余stage则会在BE任务调度中进行。
* 如果该任务是BE任务，则会将该任务的完成时间置为∞。

DART接下来则会测试在新的最坏执行时间下，其RT任务的可调度性。如果仍然是可调度的，则会将之前的RT任务恢复到之前的RT级别调度。否则，DART会通知用户无法满足所有RT任务的deadline，需要重新进行配置。





## 实验评估

### 环境设置：

#### 两个baseline：

baseCPU、baseGPU：分发到不同处理器的优先队列【RT任务优先于BE任务】，会为每一个DNN推理创建一个线程。BaseCPU 通过多线程 OpenBLAS 库在 CPU 上执行所有操作；BaseGPU 使用 GPU。

#### 两个平台：

* Xeon 平台具有 8 核 2.1GHz Intel Xeon E2620 v4 CPU 和 Nvidia GTX 1080 独立 GPU。 
* TX2 平台配备了具有四核 ARM Cortex-A57 CPU 集群、双核 Denver CPU 集群和集成 GPU 的 SoC。

#### 四种DNN模型：

目标分类：Alexnet, LeNet, VGGnet

自动驾驶：Pilotnet





### DNN 模型层执行时间分析

![image-20221111151627862](https://s3.bmp.ovh/imgs/2022/11/11/2763818de0e189dd.png)

实验：

测试了LeNet这个模型不同层在不同处理器上的执行时间。

模型一共有9层，处理器有6种：1.一个ARM CPU，2.两个ARM，3.三个ARM，4.一个Denver CPU，5.两个Denver，6.CPU【配备了一个ARM CPU，所以前面最多只有3个】

实验分析：

* CPU 内核数量增加带来的加速因层而异。比如：2、4层增加CPU数量有较明显的加速，3、5则没有。
* CPU 内核数量增加带来的收益是有限的，随着数量CPU不断增加，收益逐渐变少。
* 不同处理器的性能差异。GPU处理时间最短，Denver CPU核性能比ARM CPU核性能更好。



### 可调度性实验

实验一：10个RT任务，但是RT任务中重模型占比不同，分析可调度任务的比例。如图Fig. 11

实验二：不同数量的RT任务，分析可调度任务的比例。如图Fig. 12

<img src="https://s3.bmp.ovh/imgs/2022/11/11/b13c14eaaaca73b6.png" alt="image-20221111153214130" style="zoom:50%;" />

<img src="https://s3.bmp.ovh/imgs/2022/11/11/104475f4cfda52b8.png" alt="image-20221111153304580" style="zoom: 50%;" />

实验分析：

* 随着重型模型任务比率的提高，所有三种方法的可调度任务集的百分比都会降低，因为执行时间较长的任务更难满足最后期限。
* Xeon比 TX2 具有更高的可调度性，Xeon比 TX2 的计算能力更强。
* DART 在所有实验条件下均优于其他两种。

实验结果：结果表明，DART 有效地利用了给定的 CPU 和 GPU 资源，并且在调度实时 DNN 任务方面具有显着优势。



### 响应时间和吞吐量

任务集：

<img src="https://s3.bmp.ovh/imgs/2022/11/11/01763642066535ea.png" alt="image-20221111154809187" style="zoom:50%;" />

考虑了在不同batch size下的响应时间。

<img src="https://s3.bmp.ovh/imgs/2022/11/11/e7bee6ece81f1ca2.png" alt="image-20221111155155026" style="zoom:50%;" />

![image-20221111155300529](https://s3.bmp.ovh/imgs/2022/11/11/0c402224f0688aa4.png)

Fig 13和Fig 14 分析了pilot_rt_2 、alexnet_rt_2，这两个任务在两个平台下的响应时间。选择这两个是因为其分别是第二高和最低的实时优先级，这很好地证明了来自更高优先级 RT 任务和其他 BE 任务的干扰的影响。

Fig 15 测量了在四个RT任务并发执行的情况下，

实验分析：

* Fig 13 Fig 14
  * 在各种不同的批量大小下，DART 的尾部延迟比其他延迟小得多。
  * 在所有方法下，尾部延迟都会随着批量大小的增加而增加，但在 DART 中性能下降速度比 BaseGPU 小得多。
* Fig 15
  * 批处理执行显着提高了 DART 和 BaseGPU 下的吞吐量。
  * 批量大小达到 16 后，性能提升会减弱。
  * DART 提供了最高的吞吐量。





## 其他

### 疑问：

* 为什么没有评估离线profile时间？

### 实验设置猜想：

* 评估离线profile时间。
* 评估离线得到的配置在运行时的效率。
* 和其他调度进行对比。
* DART 对于 BE任务提高的吞吐量，对RT任务的违规率。



### 未来工作：

* 因为这些都是离线profile，确定好节点配置，然后运行时直接使用固定的策略。可以考虑在线设计一个启发式算法来进行调度。也可以有一个profile过程，在所有节点上对支持的模型进行测试，从而预估每一层的时间，这个时间之后在线不断调整。



### 相关工作：

* DNN 推理框架：

  * Glimpse 和 MCDNN：是与云协作的实时移动 DNN 框架。【Glimpse：目前许多终端设备上都带有摄像头，这使得物体检测应用可以在终端设备上运行。但物体检测算法需要进行大量运算，终端设备的运算力通常不能满足实时物体检测所要求的低时延。若将算法交给服务器运行，则传输时延变得不可接受。Glimpse周期性地将一些关键帧发送给服务器，由服务器标记关键帧中的物体，终端设备缓存一部分帧，并且根据关键帧的标记在缓存帧中追踪要检测的物体。】
  * Neurosurgeon：具有轻量级调度程序的分布式推理框架，它在层粒度中对本地和云机器之间的 DNN 计算进行分区以改善延迟。
  * DeepEye：通过使用模型压缩和层交错在内存有限的嵌入式设备上实现 DNN 推理。
  * DeepMon：实现层缓存、分解和乘法技术，以实现低延迟的移动 GPU DNN 框架。
  * $S^3DNN$ ：为实时工作负载而开发，并通过在 GPU 上应用监督流和调度来提高平均案例响应时间。
  
* DNN 优化技术：

  * 压缩DNN模型或者层，以牺牲准确性来换取性能的提升。这些工作也可以放到DART中，两者是正交的。

    Dynamic network surgery for efficient dnns.

    Deep compression: Compressing deep neural network with pruning, trained quantization and huffman coding.

    Compression of deep convolutional neural networks for fast and low power mobile applications.

    CNNPack: Packing convolutional neural networks in the frequency domain.

    Compressing deep neural network structures for sensing systems with a compressor-critic framework.

* 流水线和 GPU 调度：

  * 提出延迟组合定理来分析抢占式和非抢占式调度下的实时管道，然后将其扩展到 DAG 任务：Transforming distributed acyclic systems into equivalent uniprocessors under preemptive and non-preemptive scheduling.
  * 本地截止日期分配：Meeting end-to-end deadlines through distributed local deadline assignments.该论文介绍了一种分布式方法来为每个处理器上的作业分配本地截止日期。 由于竞争作业具有不同的路径，该方法通过考虑处理器之间的不同工作负载来提高可调度性结果。

* Memory-induced Interference：

  * 由共享内存资源（如缓存、DRAM 和内存总线）引起的时间干扰已被认为是多核实时系统中的一个严重问题	
  * Protecting real-time GPU applications on integrated CPU-GPU SoC platforms.当在 CPU 上共同安排内存带宽密集型合成任务时，这种内存引起的干扰可能导致 CPU-GPU 集成 SoC（如 Nvidia TX2）中的 GPU 内核执行速度减慢多达 3 倍。





















