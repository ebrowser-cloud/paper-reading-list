# Serving DNNs like Clockwork: Performance Predictability from the Bottom Up

**[Serving DNNs like Clockwork: Performance Predictability from the Bottom Up](https://www.usenix.org/conference/osdi20/presentation/gujarati)** 

## 论文作者：

* Arpan Gujarati：马克斯普朗克软件系统研究所
* Reza Karimi：埃默里大学
* Safya Alzayat、Wei Hao、Antoine Kaufmann：马克斯普朗克软件系统研究所；
* Ymir Vigfusson：埃默里大学
* Jonathan Mace：马克斯普朗克软件系统研究所

## 研究背景：

* 在云上进行推理很困难，主要表现在：1）模型种类特别多，不同的模型类型有着不同的资源需求；2）请求以不同的速率和规律到达；3）每一个请求都有延迟约束；4）GPU可以加速推理，可以很大程度的提高吞吐量，降低推理时延；5）GPU硬件资源更加昂贵，需要进行资源共享。

问题就是：云服务提供者如何在满足推理SLO的情况下，共享GPU，提高GPU利用率。

* 现有的调度框架虽然可以一定程度上提高GPU利用率，但是会导致比较严重的尾部延迟，使得推理具有不可预测性，从而导致推理违反其延迟约束。

本文主要是分析导致DNN推理时间不可预测的因素，通过解决这些问题来最小化尾部延迟，从而保证推理的延迟SLO。

实验观察发现：底层DNN执行时间是具有可预测性的。

![image-20221119163456076](https://s3.uuu.ovh/imgs/2022/11/19/9c8167aff069a8dd.png)

可以看到一个DNN在GPU上单独执行时，其延迟变化可以忽略不计。但是对于多个DNN并发处理，则会出现较大的尾部延迟。作者进行实验分析发现：虽然并发线程确实将推理吞吐量提高了 25%，但上述因素导致尾延迟增加了 100 倍。

⚠️：这里的并发是指多个模型推理进程时间分片的方式共享GPU，所以会导致切换开销。



## 架构设计

* 设计一个可预测的worker
* Central Controller

### 设计一个可预测的worker

* C1：RAM 和 GPU 内存导致的不可预测性。

  * 用作缓存的内存导致命中和不命中具有不可预测性。比如：模型是否加载到GPU上。如果没有加载，则会出现一个模型加载时间；如果已经加载，则不会再次加载，从而导致推理时间具有不可预测性。

  **解决方案**：

  * 预先分配GPU内存。
  * 显示的进行LOAD 和 UNLOAD 模型操作。

* C2：硬件交互具有不可预测性。

  * 由于硬件调度程序的调度方案以非常精细的时间尺度运行，即使两个请求直接到达的时间非常的接近，也会产生不同的调度方案。

  **解决方案**：一次只运行一个推理。

* C3：外部因素可能会引发性能差异。

  * 共享网络的带宽瓶颈。
  * GPU 和 CPU 热节流。

  **解决方案**：为每一个action设置一个 `earliest` `latest` ，超过这个区间的请求直接reject。

## Centralized Controller

* Modeling worker performance

  * memory state：其中调度程序跟踪 worker PageCaches 中存在的模型以及何时需要 LOAD。
  * action profiles：这是过去 10 个动作持续时间的测量值，按模型、工人和批量大小分层，以预测未来动作的持续时间；
  * pending actions：它跟踪提交的操作并估计每个执行者下一次可用的时间。

* Action scheduler

  * Scheduling INFER：当exector未完成的工作只剩下5ms的时间，则会为其调度下一个工作。要安排 INFER 操作，必须选择模型和批量大小。

    调度算法：

    * 为了处理这个问题，每个模型都有一个每个批处理大小的请求队列（我们称之为批处理队列）。新请求被排入每个批处理队列。当请求不再可满足时，请求将从批处理队列中删除。
    * 为了决定安排哪个模型和批量大小，我们使用策略。策略指定模型、最新时间戳和批量大小。每个 INFER  exector都有一个单独的策略队列，按last【最晚执行时间】排序，只包含它加载的模型的策略。调度程序使策略出队，直到它找到一个有效的策略，即：最晚执行时间还没有过去，并且指定批大小的批队列有足够的请求。如果策略有效，只要有额外请求可用，调度程序也会推测性地增加批处理大小。
    * 找到有效策略后，将创建 INFER 操作并将请求出列以填充批处理。此模型的旧策略从策略队列中删除。

  * Scheduling LOAD



### Summary：

系统组成就是：client 客户端，controller 中央控制器，worker 工作执行者。

* 一个worker里面可以有多个GPU。本文将在GPU进行推理的操作划分成两个action，分别是：推理【包括：输入、执行、输出】和 模型参数加载。每一个worker为每一种action 设置了一个执行单元 executor。

* 一个controller也是一台服务器，一个worker也是一台服务器。【应该是一个同构集群，也就是说worker配置都是一样的】

  用户向controller提交请求，controller 里面有一个模型队列，会将该请求放入该模型队列，之后进行调度。

  controller 里面存储了三种很重要的信息：

  * memory state：每一个GPU的page cache 模型参数状态，之后调度 load 调度算法可以根据这个来进行load action。
  * executor state：跟踪已经提交的 action 包括load 和 infer的执行状态，还剩多长时间完成工作。
  * action profiles：按照worker、batch、model 记录了过去10个action的执行时间，来预测未来的执行时间。

流程：

* controller会监测存储的信息，主要是executor state，监测executor的状态，如果该executor剩余工作只有5ms了，controller则会为进行调度，选择合适的工作负载。
* executor 执行完毕也会将该请求的执行信息返回给controller来更新这三种信息。



## 实验评估

### 实验一：对比clockwork 和 INFaaS、Clipper

实验配置：a single cluster machine to use 1 GPU

![image-20221122193230091](https://s3.uuu.ovh/imgs/2022/11/22/92c25c904db113d7.png)

实验结果：

* 左图：Clipper的实际吞吐量要低得多，随着SLO收紧，Clipper和INFaaS的吞吐量和尾部等待时间都会恶化，并且在100ms以下崩溃，只有Clockwork才能继续为100 ms以下的SLO服务。
* 右图：Clockwork的尾部延迟在所有情况下都非常接近SLO。

### 实验二：判断clockwork能否在高负载的情况下工作

实验配置：a single cluster machine to use 1 GPU

实验设置：两种类型的工作负载， minor = 一种模型，请求率固定为200r/s；major = 一共有3600种模型，累计请求率为 1000r/s。

在[-5,0]这段时间，只有minor这种工作负载。从0开始，每一秒增加一种模型，所有活动模型中平均分配 1,000 r/s 的工作负载。所以一开始一个模型的时候，其请求率为1000r/s；两个模型的时候，请求率为500r/s。

![image-20221122193822341](https://s3.uuu.ovh/imgs/2022/11/22/f000f2b7b097462a.png)

随着模型实例增加，所以出现了： 因为模型参数在CPU和GPU进行切换，Cold-Start的比例在不断上升，从而使得PCIe 的利用率迅速上升到了100%。

当t = 20，因为major模型的冷启动次数增加，其bottleneck转移到PCIe，降低了吞吐量，增加了时延。而minor，因为不需要冷启动，所以其吞吐量逐渐回到200r/s。

这个实验说明了 Clockwork 中的瓶颈如何随着工作负载需求的变化而转移。即使在为大量模型提供服务时，Clockwork 也可以处理转移瓶颈。



### 实验三：clockwork 最低可以获得怎么样的吞吐量？

实验配置：six worker: v100 | 32GB GPU Memory + one controller + one client

工作负载满意度是吞吐量与提供的负载之比，为1表示所有请求都在其SLO中收到了成功的响应。

![image-20221122194246646](https://s3.uuu.ovh/imgs/2022/11/22/7a1fea3f24ee0a82.png)

对于 R =600 r/s 和 1200 r/s 的负载，无论模型数量如何，Clockwork 成功地满足了 10 和 22 毫秒的严格 SLO。即使在 R =2400 r/s 时，Clockwork 也能轻松管理 74 毫秒的 SLO。



### 实验四：clockwork 性能隔离

实验配置：six worker: v100 | 32GB GPU Memory + one controller

client:1.six latency-sensitive client；2.several batch clients [no SLO]

![image-20221122194409533](https://s3.uuu.ovh/imgs/2022/11/22/b737a4a1d53549b4.png)

上图指 延迟敏感性 请求的满意率，下图指 组batch 请求的吞吐量。

实验结果：1.与服务没有延迟SLO的批处理请求的其他用户共享系统的情况。2.Clockwork成功地将对延迟敏感的请求优先于批处理请求。同时，Clockwork不会完全限制批处理请求，而是在空闲时间或预期的空闲时间安排它们。在SLO<15ms，因为SLO太紧，所以clockwork直接拒绝了很多延迟敏感性请求，从而允许待处理的批处理请求通过。



### 实验五：验证clockwork预测的准确性

实验配置：12 worker: v100 | 32GB GPU Memory + one controller + one client

![](https://s3.uuu.ovh/imgs/2022/11/22/0ddd356cc3034bea.png)

使用真实trace，运行六小时。过度预测会导致GPU资源空闲，过低预测会导致SLO违反。从图可以看到，在clockwork中，overprediction出现的比underprediction更多。上图：对于INFER动作，高预测和低预测的第99个百分点分别是144us和55us。出现估计错误的概率非常低，只有在极少数情况下会出现，由此可以证明clockwork可预测性的准确性。



### 实验六：clockwork 的可扩展性

验证在不同worker数量下，吞吐量的大小。

![image-20221122194835022](https://s3.uuu.ovh/imgs/2022/11/22/9e1d43ba0b302df9.png)

模拟工作者的行为与真正的worker相同，除了 LOAD 和 INFER 操作不执行任何有意义的工作；相反，他们会根据预先配置的模型测量值等待一段时间，然后再返回响应。

我们运行多个实验，每个实验具有不同的worker数量，从 10 到 150，增量为 10。我们测量达到的峰值输出。该图显示了随着worker数量的增加，峰值输出量呈线性增加。worker数量低于 110 时，吞吐量受worker达到充分利用的限制。在 N =110 时，我们达到了 103,387 r/s 的最大吞吐量。在这一点之后，工人利用率不再是限制因素；相反，瓶颈转移到了 Clockwork 的控制器上。





## 总结：

提出了consolidating choice：尽可能地限制较低系统层可用的选择——这一理念基于我们的观察，即在执行基本上可预测的任务时，只有在系统中的较低层被赋予关于如何执行它的选择时，才会出现性能变化。

设计了clockwork，在consolidating choice的设计思路上通过消除所有导致DNN推理时延有变化的因素，从而使得DNN推理具有可预测性。

本文和之前看到的调度方面的论文的区别在于：clockwork的调度是controller通过监测每一个GPU的执行状态，再按照调度策略选择合适的工作进行调度，是GPU这边提出需求，controller 找到合适的请求调度过去。之前很多都是为请求分配合适的GPU。















