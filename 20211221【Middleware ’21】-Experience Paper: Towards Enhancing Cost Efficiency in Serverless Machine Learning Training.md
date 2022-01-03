By Jiabin Chen

# 经验文章：提高无服务器机器学习训练的成本效益
[Experience Paper: Towards Enhancing Cost Efficiency in Serverless Machine Learning Training](https://dl.acm.org/doi/10.1145/3464298.3494884)
---

* 论文发表在会议 Middleware ’21 (CCF B) 上，目的是为在 serverless 平台上进行 ML 训练能获得比传统服务器 ML 系统更好的成本效益。
* 文章提出了一个基于 serverless 的 ML 训练原型 MLLess，实现了一个显著性过滤器和一个收缩调度器，并利用这两个关键的优化来提高成本效益。
* 文章结果证明，对于具有快速收敛性的 ML 模型（如稀疏逻辑回归和矩阵分解），MLLess 可以在较低的成本下比 serverful ML 系统快 15 倍。

## 论文作者
* [Marc Sánchez-Artigas](https://artigas81.github.io)：罗维拉-维吉利大学计算机工程和数学系副教授；主要研究方向是分布式计算。
* [Pablo Gimeno Sarroca](https://github.com/pablogs98)：罗维拉-维吉利大学博士；主要研究方向是 serverless 和 ML。

## 论文动机
### 问题引入：

* 在 FaaS 上构建 ML 已经成为一个新的研究领域，由于推理是 serverless 计算的一个微不足道的例子，人们的注意力已经转移到了更为困难的模型训练上。
* 在什么情况下，在 FaaS 之上进行 ML训练可能是有益，这是不知道的。

### serverless ML 训练存在的问题：

*  现今的 FaaS 平台仅仅支持无状态的函数调用，函数的资源和执行时间都有限制。例如，文章使用的 IBM Cloud Functions 最多使用 2GB 的 RAM，并且需要在 10 分钟内执行完毕。这限制了一些自然的做法，例如将所有训练数据加载到本地内存中，所以任何未考虑这些约束而设计的 ML 框架都不能在 serverless 上使用。
* FaaS 不能直接通信，需要通过共享外部存储器在函数之间传递状态。这不仅造成了显著的额外延迟（通常为数百毫秒），还使得其无法利用 ML 中采用的 HPC 通信拓扑，如树结构和环结构的 All-Reduce。

## MLLess：

MLLess 的系统结构示意图如下所示：

![image](https://cdn.jsdelivr.net/gh/JBinin/Image-hosting@master/Experience-Paper/image.7a3mhny64jcw.png)

* driver 运行在用户的本地机器，当用户启动一个 ML 训练作业时，driver 会调用需要数量的 worker 以数据并行的方式执行训练作业。每一个 worker 维护一份模型的局部拷贝，并使用 MLLess 的库训练它们。

#### Supervisor

* Supervisor 的职责是收集和汇总统计数据，同步 worker 进度，以限制模型副本之间的差异，并在满足停止标准时终止训练作业。
* Supervisor 的一个核心属性是，当 worker 对收敛的边际贡献很小，甚至由于通信成本增加而为负时，自动移除 worker。

#### Communication channels

* 为了在 worker 和 Supervisor 之间交换控制消息，它使用了一个构建在 RabbitMQ2 之上的消息传递服务。
* 为了共享模型训练期间生成的中间数据（例如，局部梯度），MLLess 使用了 Redis。
* 使用 IBMCOS（一种无服务器对象存储服务）来存储数据集 mini-batches。

### 两个关键优化

#### 显著性过滤器

* 云服务商不允许函数之间的直接通信，梯度的快速聚合无法使用最佳原语（如ring All-Reduce）实现，只能通过外部存储来实现，这是一个代价很高的操作，会大大降低并行带来的收益。为了减小上述影响， MLLess 使用了一种 Approximate Synchronous Parallel (ASP) 模型的变体，文章将这种变体命名为 Insignificance-bounded Synchronous Parallel (ISP) 。ISP 得益于许多 ML 算法（例如，逻辑回归、 矩阵分解、折叠 Gibbs 等）的鲁棒性，这些算法容忍一定数量的不一致性。
* 为了在限制任意两个模型副本之间的偏差的同时减少通信，一种“聪明”的压缩技术是让每个 worker 在本地更新不重要时聚合其本地更新。这样，如果累积的更新最终变得重要，则 worker 将非重要更新的完整历史编码为一个更新广播到其他 worker。

#### 收缩调度器

* 与集群计算相比，FaaS 模型的一个主要优点是，它能够随着时间的推移快速调整 worker 的数量。这种能力为新型调度器的发明打开了大门，这可能会带来更具成本效益的训练，传统的 “serverful” 云计算根据保留虚拟机保持活动的时间向客户收费。
* 为了证明使用 FaaS 计算可以获得更好的成本效率比，MLLess 包含了一个动态的细粒度调度器，用于删除“不需要” 的 worker。ML 训练通常是一个迭代过程，其中质量改进的水平随着训练步骤的增加而降低。
* SGD 在凸问题上近似地作为一个几何级数减少损失。这意味着，虽然在第一个训练步骤中需要更多的 worker 来大幅减少损失，但当损失减少速度放缓时，大量 worker 只会带来边际回报，这最终会恶化成本效率比。

#### 算法

* 最初 worker 数量为 P， 调度器根据 ML 算法的反馈动态地减少 worker 池。
* 根据历史损失信息，调度器首先检测收敛速度中的“拐点”，超过这个时间点之后损失减少速度显著减慢，并使用此时损失值的历史来拟合参考训练损失曲线。 调度程序将使用该曲线来量化未来 worker 被移除所导致的与原始收敛速度的偏差。
* 调度器在每个调度间隔上重复以下操作序列：

  * 估计阶段：使用自上次移除 worker 以来收集的损失值，拟合一条新的的训练损失曲线 。
  * 决策阶段：调度器根据固定时间内预计损失减少的相对误差（新的训练损失曲线和参考训练曲线之间的误差），决定移除一个新的 worker。

## 实验
baseline：

* Distributed PyTorch on CPUs：
  * PyTorch v1.8.1
  * 使用 all-reduce 操作
* PyWren-IBM：
  * 利用 map 阶段并行处理 mini-batch，用 reduce 任务以聚合本地更新。
  * 所有通信都是通过 IBM COS完成的，包括更新的共享，以保持其纯无服务器的通用体系结构。

数据集（所有数据集都是高度稀疏的，以验证 MLLess对稀疏数据的支持）：

* Criteo
* MovieLens-10M
* MovieLens- 20M

ML 模型：

* Criteo for logistic regression（LR）
* MovieLens-10M/20M for probabilistic matrix factorization (PMF)

实验设置：

* 用于实验的 VM 实例部署在IBM云上。
* MLLess：一个 C1.4x4 实例（4vCPUs，4GB内存）来承载消息服务，以及一个 M1.2x16 实例（2VCPU，16GB的RAM）来部署Redis，以及一定量的 FaaS worker 。
* PyTorch 为了使用和 MLLess 相同数量 worker，PyTorch 集群将由 3 或 6 个 B1.4x8 实例（4vCPU，8GB内存）。
* 所有实例都有 1Gbps NIC。
* 作为 MLLess worker，使用 2GB 最大内存的函数。

#### 实验项目1

![image](https://raw.githubusercontent.com/JBinin/Image-hosting/master/Experience-Paper/image.40cgccu0jt34.png)

* PyWren IBM在所有工作中都非常低效。这主要是由于两个事实。首先，本地更新仅通过慢速存储（即IBM COS）在 worker 之间进行通信。第二个是 PyWren IBM 并不是专门针对迭代 ML 训练的框架。
* 自动调整器在任何作业中都不会减慢收敛速度。相反，它有助于在降低成本的同时提高收敛速度。
* 对于大型模型（如ML-20M），ISP 一致性的使用对于确保最初几秒钟内的快速收敛至关重要。

#### 实验项目2

![image](https://raw.githubusercontent.com/JBinin/Image-hosting/master/Experience-Paper/image.3xp52i82la2o.png)

上图显示每个系统在固定预算（以美元计）下的收敛程度。条形图上方的数字报告了每个可能预算可承受的最大执行时间。从图中可以看出，MLLess+All 在所有应用程序中都提供了最佳的性价比折衷方案。

## 文章相关工作：

#### serverless ML

* Cirrus 是一个 serverless ML 系统，它在 VM 上实现了一个参数服务器（PS），所有 FaaS 的 worker 都与这个集中的 PS 层通信。它的优势在于 PS 的计算能力比通过外部存储进行间接通信节省了 200% 的通信成本。Cirrus 比 VMs 快 3-5 倍，但成本高出 7倍。
* SIREN 提出了一个异步 ML 框架，其中每个 worker 独立运行，worker 从远程存储（如AWS S3）读取（过时）模型，用 mini-batch  本地数据更新模型，将新模型写回存储。它的主要优势在于其基于强化学习的调度器，该调度器根据一定的预算动态调整 worker 的数量。与 MLLess 相比，它的调度器更粗粒度，成本效率比更低。
* LambdaML 与本文几乎并行，是一个基于 FaaS 的训练系统，用于确定 FaaS 对 IaaS 具有影响力的案例。两篇文章结果都反映了 FaaS 对于快速收敛的模型更具成本效益的结论。

## 其他
文章中提到的缺点及未来工作：
* MLLess 仅仅适合快速收敛的模型，不适合深度学习和突发 ML 负载。



