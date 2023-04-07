# Review：MuxFlow Efficient and Safe GPU Sharing in Large-Scale Production Deep Learning Clusters

## 论文作者：

* Yihao Zhao： Peking University
* Xin Liu： ByteDance Inc.
* Shufan Liu： ByteDance Inc.
* Xiang Li： ByteDance Inc.
* Yibo Zhu： ByteDance Inc.
* Gang Huang： Peking University
* Xuanzhe Liu： Peking University
* Xin Jin： Peking University

## motivation：

* 集群GPU利用率低，无论时间上（GPU utilization）的还是空间上（SM activity）的。

  <img src="/Users/echozhou/Library/Application Support/typora-user-images/image-20230406215107562.png" alt="image-20230406215107562" style="zoom:50%;" />

  * 99% 以上的 GPU 的 GPU 利用率和 SM 利用率都低于 60%。
  * 对于大约 90% 的 GPU，GPU 内存使用率低于 60%。
  * 这些数字表明 GPU 在内存和计算能力方面都没有得到充分利用，这表明宝贵的 GPU 存在着巨大的浪费。

* online workloads是波动且可预测的。

  <img src="/Users/echozhou/Library/Application Support/typora-user-images/image-20230406215134276.png" alt="image-20230406215134276" style="zoom:50%;" />

  * GPU 利用率和 SM 活跃度在一天内都会有很大的波动，因为在线请求的数量会不时变化。
  * GPU 内存使用是稳定的，因为 DL 框架缓存中间 GPU 内存以提高效率。
  * 我们观察到 GPU 使用指标的曲线在几分钟内是平滑的，在几天内是周期性的。因此，我们可以通过过去的值来预测 GPU 使用指标。

* 时间复用对提高GPU利用率效率不高。其他空间复用方法也各有缺陷。

  <img src="/Users/echozhou/Library/Application Support/typora-user-images/image-20230406215149895.png" alt="image-20230406215149895" style="zoom:50%;" />

  * 时间复用：单个online workloads通常无法完全填满一个 GPU 的所有 SM[26, 36]，导致 GPU 计算能力的浪费。
  * MIG：分区在工作负载执行过程中无法动态调整，必须为online workloads分配整个instance。MIG仅在新的GPU上可用，A100，H100。
  * cuda stream：只能在一个进程中与其他流共享，在生产集群中将多个workloads合并在一个进程中，难以管理。
  * NVIDIA MPS是 GPU 资源利用率和在线性能之间的最佳权衡。从Kepler以后开始支持使用。

**summary：所以要使用MPS进行空间共享。** 但是MPS使用的过程会出现以下几个问题：

* 生产集群的首要目标是保证online workload的性能。例如，在线请求可能由于特殊活动突然爆发，但无法及时降低offline workload的 SM 百分比，从而不能保证online workload的性能。
* MPS具有严重的错误传播。
* 分配给离线工作负载的不同 SM 百分比会极大地影响两种共享工作负载的效率。图 4(b) 。
* 不同的在线和离线工作负载共享对在图 4(a) 中显示出对共享工作负载的不同影响。

<img src="/Users/echozhou/Library/Application Support/typora-user-images/image-20230406215321947.png" alt="image-20230406215321947" style="zoom:50%;" />

所以本文提出了MuxFlow解决了上述问题：

* 两级性能保护机制来保证online workloads的性能。
* 基于生产错误分析的混合错误处理机制来解决MPS的错误传播。
* 动态SM分配来解决SM对共享工作负载的影响。
* 基于KM的调度算法来找到合适的online 和 offline共享对。

## 算法or机制：

### 两级性能保护机制：

#### workloads level：xCuda

* 在内存方面，xCUDA 可以跟踪 GPU 内存使用情况，确保离线工作负载使用的内存不超过 GPU 内存配额。
* 在计算方面，对于 NVIDIA GPU，SM 时钟代表 SM 执行指令的速度。**我们的目标相当于同时获得高 SM 时钟和高 GPU 利用率。** 当 SM 时钟较低时，我们可以延迟离线工作负载的内核启动，以减少 GPU 负载并提高 SM 时钟。当 GPU 利用率低时，我们可以启动更多的内核来改善它。

#### GPU level：SysMonitor

MuxFlow 使用 SysMonitor 通过状态机来监控 GPU 设备状态。状态机有五个状态，每个状态都有一组用于 GPU 利用率、SM 活动、SM 时钟和 GPU 内存使用的度量阈值。阈值是根据经验选择的。请注意，离线工作负载只能安排到健康的 GPU。

<img src="/Users/echozhou/Library/Application Support/typora-user-images/image-20230406220100430.png" alt="image-20230406220100430" style="zoom:50%;" />

### 混合错误处理机制

分析出现传播错误的原因如下：

<img src="/Users/echozhou/Library/Application Support/typora-user-images/image-20230406220117200.png" alt="image-20230406220117200" style="zoom:50%;" />

针对错误进行处理：

* SIGINT/SIGTERM 99%：xCUDA 收到 SIGINT 和 SIGTERM 信号时，它将冻结所有内核启动并主动释放 CUDA 上下文。
* 其他错误只占1%：对于这些错误，我们总结了它们的错误模式。自动检测器会监控 GPU，并在满足错误模式时发出警报。一旦 xCUDA 收到警报，它将重置 CUDA 上下文和 MPS 服务器。

### 动态SM分配

根据online workload单独运行时SM activity设定百分比，从而不影响online workload的性能。

### 基于KM的调度算法

当与不同的online workloads共享时，一个offline workload具有不同的吞吐量。

本文主要探讨offline workload调度算法，假设online workload已经放好了。将offline workload与哪个online workload配对。

**DL模型预测共享对的速度**：训练一个DL模型来预测共享对的速度。

采用KM算法来选择最佳共享对方案。二分图匹配问题。权重为offline workload归一化后的吞吐量。



## 实验：

**硬件：**

使用 125 台机器和 1, 000 个 GPU 进行测试台实验。每台机器配备8个NVIDIA Tesla T4 GPU，2个Intel Xeon Platinum CPU，128G内存，100+25G网卡。我们将 PyTorch v1.8.0 与 CUDA 11.1 用于离线工作负载。

**workloads**：

* 在线：选择了很多种DL模型，包括：CNN、GNN、LLM，速率从20 到 190 查询每秒。【模拟集群：三种部署在companyX的负载】
* 离线：随机的选择了四种模型，包括：ResNet50，VGG16，DenseNet201，Inception-V3。trace 使用 Microsoft 公开的Trace。生成的trace可以在12h完成【模拟集群：trace可以在24h完成】

**baseline**：

* Online-only：只有在线工作负载并显示在线工作负载的最佳延迟。
* Time-sharing：分时共享工作负载，并通过 Gandiva 采用的 GPU 驱动策略将 GPU 的时间片分配给共享的工作负载。
* PBtime-sharing：将在线工作负载设置为高优先级，并将更多的 GPU 时间片分配给高优先级工作负载，以保护高优先级工作负载的性能，这被 AntMan 和 PAI 采用。

**Metrics**：

* online workloads：平均延迟、99% - th 延迟
* offline workloads：JCT、makespan、归一化吞吐量：定义为共享时的吞吐量除以单独运行时的吞吐量。【共享时的速度/单独运行的速度】、oversold GPU：表示给offline workload提供了多少GPU资源。
* GPU指标：GPU utilization、SM activity、GPU memory utilization.

### 实验一：整体评测

<img src="/Users/echozhou/Library/Application Support/typora-user-images/image-20230406221109526.png" alt="image-20230406221109526" style="zoom:50%;" />

* 在线工作负载的效率。增加了avg latency 和 p99，大约是10ms左右。文章提到：**对于大多数在线工作负载来说，10ms减速几乎察觉不到，例如推荐服务和机器翻译。** 
* 离线工作负载的效率。我们发现 MuxFlow 为离线工作负载提供了高达 86.42% 的 GPU 资源，考虑到生产中的大量 GPU，这是一个相当大的数字。【86.42应该是oversold GPU】
* GPU资源利用率：MuxFlow 将 GPU 利用率提高了 4.0 倍，SM 活动提高了 4.7 倍，GPU 内存使用率提高了 1.5 倍。
* 因为12h内没有错误发生，从而验证了安全性

### 实验二：与相关工作进行比较

![image-20230406221136671](/Users/echozhou/Library/Application Support/typora-user-images/image-20230406221136671.png)

比较了latency，acg JCT，oversold GPU。

* MuxFlow 将平均 JCT 提高了 1.10 − 2.24 倍，将oversold GPU 提高了 1.08 − 1.97 倍，同时将在线工作负载减慢了不到 20%。
* Time sharing使在线工作负载减慢高达 50%，这表明对在线工作负载的影响很大。
* PB-time-sharing 利用优先级来保护在线工作负载的性能，但由于两个原因，它对离线工作负载的指标比 MuxFlow 差。首先，MuxFlow 可以利用空间在线工作负载浪费的 GPU 资源。其次，MuxFlow 采用调度算法来提高离线工作负载的效率。

### 实验三：对MuxFlow具体分析

#### 速度预测模型的分析

研究了不同layer数量以及hidden size对模型准确性的影响。

<img src="/Users/echozhou/Library/Application Support/typora-user-images/image-20230406203028673.png" alt="image-20230406203028673" style="zoom:50%;" />

最后选择了4层，以及64*64的hidden size，因为其性能最好。

#### 研究动态SM分配机制和基于匹配的调度算法对offline workloads的性能影响

<img src="/Users/echozhou/Library/Application Support/typora-user-images/image-20230406203258194.png" alt="image-20230406203258194" style="zoom:50%;" />

与 MuxFlow-S-M 相比，MuxFlow-S 和 MuxFlow-M 均提高了平均 JCT 和oversold GPU。这些改进证实只有在线保护机制是不高效的，动态SM分配机制和基于匹配的调度对于离线效率很重要。

#### 系统开销【即KM和DL模型开销】

profiling开销和scheduling开销：

* profiling开销：每个离线工作负载的分析时间不到 10 分钟。分析开销很小，因为离线工作负载通常需要数小时甚至数天。
* scheduling开销：调度算法的开销包括两个时期。
  * 第一阶段是预测共享性能并构建二分图。每个预测只需要不到一毫秒，公司内部集群都由数千个 GPU 和数千个工作负载组成。这个周期只需要几秒钟。
  * 第二期是执行KM算法。几千个workloads需要几分钟。注意，调度算法可以与工作负载执行并行执行。因此，我们可以隐藏每个调度间隔内的调度开销。



## 总结反思：

本文使用的指标是：

* SM activity，这个可以从空间上反映GPU利用率。
* GPU Utilization：从时间上可以反映GPU利用率。

测量GPU性能指标工具：GCDM，NVML(nvidia-smi)

本文的调度间隔是：15min。调度开销也需要几分钟。但是可以提前计算下一轮的调度策略。从而掩盖了调度开销。

本文是将推理和训练共置的角度来提高GPU利用率，有点类似AntMan的想法，但是AntMan是时间上的共享。

### 局限性：

局限主要是本文只能一个online workload和一个offline workload共置。如果要将一个online和多个offline共置则会出现很多问题：

* 首先，我们需要保证所有在线工作负载的性能。
* 其次，我们需要限制多个离线工作负载使用的 SM 总百分比，这不能简单地通过 MPS 参数来限制。
* 第三，xCUDA 需要监控所有离线工作负载的内核启动，并根据优先级决定延迟或启动哪个内核。
* 第四，选择具有三个以上工作负载的共享对的调度算法成为一个 NP-hard 问题 [63]。如何解决这些挑战以更好地利用 GPU 是一个有趣的未来方向。