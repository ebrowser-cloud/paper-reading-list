By Xinyi Zhang

# Singularity: 针对AI 工作负载的行星级、抢占式和弹性调度

[Singularity: Planet-Scale, Preemptive and Elastic Scheduling of AI Workloads](https://arxiv.org/pdf/2202.07848.pdf)



* Singularity，作为Microsoft的分布式调度服务，用于高效可靠地执行深度学习训练和推理工作负载。
* Singularity 可以透明地抢占和弹性扩展深度学习工作负载，以提高利用率，而不会影响其在 AI 加速器（例如 GPU、FPGA）中的正确性或性能。
* 默认情况下，Singularity 中的所有作业都是可抢占、可迁移和动态调整大小（弹性）的：作业可以动态且透明地 (a) 抢占并迁移到一组不同的节点、集群、数据中心或区域，并准确地从被抢占的点执行，以及 (b) 在给定类型的一组不同的加速器上调整大小（即，弹性地放大/缩小）。


## 论文作者

* Dharma Shukla：微软员工，研究方向：分布式系统、人工智能系统。  
* Muthian Sivathanu：微软研究院，研究方向：大规模分布式系统。  

## 论文动机

对于深度学习工作负载的要求主要在降低成本和高利用率上；针对调度服务的要求主要集中在高效、可靠、可同时用于深度学习训练和推理工作负载。

## 设计目标

* 基于给定的加速器（例如GPU）容量，通过最大化作业吞吐量来降低运行成本
* 基于不同定价下的资源，为作业提供严格的 SLA

### 设计原则

* 无空闲资源：将所有资源视为一个共享集群
* 提供作业级的SLAs：使具有较低 SLA 的作业能够机会性地使用备用容量，并迅速被抢占
* 针对失败作业：作业从被抢占的地方恢复，从而最大限度地减少浪费。

## 关键机制概述

* 对用户透明
  * 目前针对checkpoint和弹性的方法主要为用户直接编写代码或使用特定的库
  * Singularity代码不需要用户的参与，隐藏了复杂的细节

* 工作节约
  * Checkpoint由workers的连续快照所组成，这些快照涵盖了完整的程序状态，例如指令指针、堆栈等
  * 作业可随时恢复，不会丢失数据

* 解耦执行
  * 解耦作业和底层资源之间的映射，方便后续作业的抢占和恢复
  * 提高作业的可靠性、提高资源的利用率


# Device Proxy

### 概述

* Device Proxy是GPU的硬件抽象服务
* 有一个服务器组件（每个设备一个）
* 与设备交互的每个进程中都会嵌入一个客户端组件。  
* 主机调用的所有GPU API都会被拦截并发送到proxy
* Proxy运行在独立的地址空间中：
  * 使主机地址空间免受设备映射和其他例如CRIU的影响
  * 允许proxy在弹性时间切片期间在多个工作进程之间有效共享  

![](https://github.com/XinyiZhang-ecnu/photo/blob/main/device%20proxy.png?raw=true) 



### 组成

#### Dispatch Interceptors (D<sub>INT</sub> ) 

* 拦截作业调用的所有设备调用并将其发送到到 device-proxy 服务器
* 将 API 跨地址空间传送到 device-proxy 服务器
* 实现：底层，即用于启动内核的驱动程序API



#### Semantics-Aware Interceptors (SA<sub>INT</sub>)

- 在客户端或服务器端实现自定义逻辑，大部分逻辑使用一个硬件抽象层映射到设备特定的API，硬件抽象层中封装了通用功能

- 内存分配：GPU 内存的哪些区域实际在使用中，有助于减少checkpoint的大小；使用自定义内存分配机制来在同一 GPU 上进行时间切片以实现弹性。

- 通信：实现distributed barrier算法，以在多个worker之间同步，获得一致的检查点；有助于在弹性时间切片期间管理集体调用。

- 设备同步、主机特定功能（选择CPU库）



# **Transparent Migration**

* Singularity 的 checkpointing 逻辑有两种执行方式：

（1）当调度器决定作业需要抢占时，基于外部命令按需执行，

（2）基于用户指定的时间间隔定期执行（epoch-level或基于时间）。

* 抢占、恢复和调整作业大小涉及四种状态下的checkpointing：

（a）CPU 中的程序状态（例如，堆栈、堆、指令指针等）

（b）模型训练状态GPU（例如，模型参数、优化器状态等）

（c）处理 CPU 和 GPU 之间交互的控制状态（活动流、同步事件）

（d）不同类型的 GPU 间和节点间通信状态（数据/管道/张量并行等）。



### **Distributed barrier**

* 原因：为了checkpoint 通信状态，需要暂停作业，这样能保证没有正在进行的调用。由于每个worker不能独立暂停，需要完成集体调用（例如 allreduce），所有参与的worker都必须完成该调用。

* 算法目的: 所有woker发出同一组集体调用

* 算法分为两个阶段：

​	1. 稳定状态

​	2. 收到barrier请求时，串联meta-allreduce 是有效载荷上的 SUM allreduce，由两个整数组成：

​			 need_barrier：worker收到命令发送1，否则发送0；当SUM(need)>0，则切换状态2 

​			 ack_barrier：worker切换状态2发送1，否则发送0；如果SUM(ack)=work_size，则表示每个worker都收到命令，能够安全获得barrier

​	一旦 worker 进入状态2，表明它就会进入同步模式：该 worker 执行的每个collective calls都是同步的；这确保了barrier的及时终止。



## **Transparent Elasticity**

现状： 自上一个checkpoint以后的初始化和迭代被重做

Singularity:

* 作业以相同的workers运行
* 调度程序可以将每个worker一对一映射到物理 GPU（放大），或使用多对一映射，其中物理 GPU 被虚拟化并跨多个workers进行时间切片（缩小）


![](https://github.com/XinyiZhang-ecnu/photo/blob/main/elasticity.png?raw=true)


* Transparent Elasticity建立迁移的支持之上
  * 将作业从 4 个 GPU 缩减到 1 个 GPU，只需采用 3 个 CRIU 检查点，然后通过时间片将这些进程迁移到单个 GPU

* 难点：
  * 在同一个 GPU 上对同一个训练作业的多个worker进行时间切片时， worker之间的细粒度通信（例如 allreduce）必须像worker在不同的 GPU 中运行一样进行；这要求时间片既要语义感知又要细粒度（每个小批量多个上下文切换）。
  * 对于大型模型，每个worker可能会使用 GPU 上几乎整个 RAM；在同一个 GPU 上切换worker开销巨大
  * 对于数据并行、管道并行和张量并行组合的作业，需要在 GPU 上仔细放置worker ，使得只有同一模型并行分片的数据并行副本可在同一GPU上进行时间分片，防止通信调度死锁



### **Replica splicing**

* 原因：上下文切换速度较慢且开销较大
* 训练作业消耗的 GPU 内存分为四类： 
  1. 参数 (P)。模型每一层的权重/参数；在这些张量上运行正向和反向传递。 
  2. 优化器状态 (O)。优化器跟踪状态以计算每次迭代用于参数的增量。跟踪历史状态（例如，梯度的第一和第二时刻） 
  3. 梯度 (G)。每个replica都有自己的梯度拷贝，对应于它的小批量。在反向传递之后，所有replica的梯度被平均，然后用于一致地更新权重。
  4. 激活 (A)。每层前向传递的中间输出；在反向传播期间用于计算相对于反向传播输入的梯度。

实现技术的重难点：

* 一致分配
  * 解决：同一个P和O的缓冲区在不同rank得到不同地址
  * Device-proxy使用双向内存分配器，P和O分配在地址空间的高端，其他缓冲区分配在低端
  * 瞬态分配（例如激活）中的不稳定性不会影响高区域中的内存分配器数据，从而确保稳定的缓冲区（例如 P 和 O）在副本之间获得相同的地址。



* “挤压”选择性操作

  * 解决：避免处理P和O的多个副本（导致切换成本和显存增加）

  * 所有数据并行replicas在完成mini-batch后到达相同版本的 P 和 O 缓冲区

  * 只有在replicas间梯度的 allreduce 完成后，才会更新 P 和 O 缓冲区。

  * 如果可以识别更新参数和优化器状态的操作，可以只在共享设备的一个等级（“根”等级）中执行这些操作，并简单地“挤压”其他等级中的这些操作

  * 为了压缩操作，设备代理只是省略了向 GPU 发出 cudaLaunchKernel 以执行这些操作

    

## 实现难点

* 序列化不透明参数：D<sub>INT</sub>签名不透明
  * SA<sub>INT</sub>使用cuObjDump，是CUDA 工具包中的一个二进制实用程序，用于解析生成的内核库并提取参数信息
  * 通过解析生成的 PTX 来提取参数签名



* 隐藏调用延迟
  * 对最频繁的调用使用特定领域的优化
  * 例如，对于 cudaGetLastError，在每次内核启动期间在服务器上发出它，并将它与它的响应一起搭载，以便device-proxy客户端可以在 PyTorch 发出它时从缓存中返回它



* 隐藏上下文切换开销
  * 切换开销例如：在时间切片期间涉及计算所有活动设备张量的校验和，并进行比较，必要时执行缓冲区移动（几毫秒的 CPU 活动）。
  * 解决方案：eager dispatch。Device-proxy与切换逻辑并行地提供服务，从而将下一个队列的有用工作（CPU 逻辑、设备操作的调度）与切换延迟重叠。




## 实验

### 实验设置


![](https://github.com/XinyiZhang-ecnu/photo/blob/main/model.png?raw=true)

- 实验放置在  NVIDIA DGX-2 servers (8 V100 GPUs per node with NVLink connectivity);

- 每个服务器： Xeon Platinum 8168 with 2 sockets of 20 cores each and 692 GB of RAM.

  

### 实验一 device-proxy的开销

![](https://github.com/XinyiZhang-ecnu/photo/blob/main/device%20proxy%20overhead.png?raw=true)

#### 结论： 

Device-proxy 对端到端性能的影响可以忽略不计，在大多数模型中开销低于 3%，包括使用数据并行、张量并行和管道组合的 GPT-2 和 InternalT -并行性。



### 实验二 checkpoint 大小


![](https://github.com/XinyiZhang-ecnu/photo/blob/main/checkpoint%20size.png?raw=true)


![](https://github.com/XinyiZhang-ecnu/photo/blob/main/checkpoint%20calculate.png?raw=true)

S<sub>G</sub>是单个数据并行副本的 GPU 状态（参数和优化器状态）大小，S<sub>pwCr</sub>是每个worker的 CRIU 转储大小



#### 结论： 

* S<sub>G</sub>与用户级checkpoints相当
* 于 CPU 状态的 CRIU 检查点，有两个数字：第一个checkpoint的大小和后续checkpoints的大小（代表连续检查点场景）。
* 由于时间重复数据会被删除，对于许多模型，后续检查点比第一个检查点小一个数量级。
* 对于第一个检查点，即使对于大型作业，这非常易于管理的。
* 正如预期的那样，8-worker 的 CRIU 转储大小是 4-worker 配置的两倍。



### 实验三 **Replica splicing**的性能

![](https://github.com/XinyiZhang-ecnu/photo/blob/main/replica%20splicing.png?raw=true)

#### 结论：

图中显示了在两种配置下各种模型的replica splicing导致的开销：2 路时间切片和 4 路时间切片。在每一个中，我们将我们的时间切片运行与完全放大模式下未修改的 PyTorch（没有 Singularity）的基线运行进行比较。使用 N 路时间切片，小批量时间预计会增加 N 倍（即相同的工作但更少的 GPU）；超出此范围的任何增加都是开销。可以看出，对于大多数模型，按比例缩小模式（右 Y 轴）中时间切片引入的开销小于 3%，证明了replica splicing的有效性。



### 实验四 迁移以及资源动态缩放的延迟

![](https://github.com/XinyiZhang-ecnu/photo/blob/main/migration%20and%20resizing%20latency.png?raw=true)



 延迟原因：

(a)	获取屏障，

(b)	生成 GPU 和 CRIU 转储

(c)	将转储上传到远程存储

(d)	在不同的节点上下载转储

(e)	恢复 CRIU 转储和 GPU 转储，并释放屏障



#### 结论：

对于大多数模型，迁移或调整大小的这种端到端延迟为数十秒，其中一半以上的时间用于从远程 Azure blob 存储上传和下载（显示为传输时间）。 



## 文章相关工作

### DNN Schedulers

* 现今大多数调度器提出迁移和弹性资源共享，但他们没有解决如何处理绝大多数不可迁移或弹性的工作的问题。
* Singularity 中，所有工作都具有可抢占性、可恢复性和弹性。



### Job Elasticity

* Singularity 是第一个能够对现有的、未修改的 DNN 训练作业实现高效、真正透明的弹性的系统
* Singularity对用户来说是完全透明的：作业总是以相同数量的worker和相同的参数运行； Singularity 只是将这些worker重新映射到不同数量的物理设备。
* 现今：要求用户使用接管训练循环的自定义库。资源缩放不透明、为每个配置指定不同参数、统计效率低等等



### Job Migration

* 背景：必须处理每个工作负载带来的众多极端情况，在生产环境中的广泛采用具有挑战性
* Singularity 中的迁移是特定于域的，因为checkpoint的时间安排使得进程处于相对干净的状态
* Gandiva中的checkpoint通过对深度学习框架进行大量更改来实现的，很难维护和保持更新
* proxy-based checkpointing与框架解耦，同时支持透明性和可维护性。



## 总结

* Singularity 将弹性等小众特性转化为主流特性
* 调度程序可以依赖这些特性来实施严格的 SLA
* Singularity 使作业可抢占并可调整大小且性能开销忽略不计，使作业能够利用备用容量并保证 SLA
* Singularity 实现了非常简单的用户体验：用户只关注 ML 任务，不需要考虑检查点或弹性；这些机制是对用户完全透明的基础设施优化。