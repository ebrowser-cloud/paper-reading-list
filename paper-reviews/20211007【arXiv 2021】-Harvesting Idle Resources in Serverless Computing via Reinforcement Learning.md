By Jiabin Chen

# 通过强化学习回收 serverless 计算中的空闲资源

[Harvesting Idle Resources in Serverless Computing via Reinforcement Learning](http://arxiv.org/abs/2108.12717)
---

* 本文提出了 FaaSRM，一种用于无服务器平台的新的资源管理器，它应用深度强化学习，动态地将闲置资源从资源供应过剩的函数中分配到供应不足的函数中，从而实现资源效率最大化。
* 文章在一个 13 个节点的 Apache OpenWhisk 集群中实现并部署了 FaaSRM 原型。结果表明，与基准资源管理器相比，FaaSRM 通过从 38.8% 的调用中获取闲置资源并加速 39.2% 的调用，将 98% 的函数调用的执行时间降低了 35.81%。

## 论文作者

* [Hanfei Yu](https://hanfeiyu.github.io)：路易斯安那州立大学计算机科学与工程系在读博士；IntelliSys Lab；主要研究方向为云计算和机器学习，致力于应用机器学习（例如强化学习）来改进无服务器计算并构建更好的系统来为机器学习服务。
* [Hao Wang](https://haow.ca)：路易斯安那州立大学 CSE 部门助理教授；Hanfei Yu 导师；主要研究方向为数据分析、机器学习、分布式计算和数据中心网络。
* [Jian Li](https://www.binghamton.edu/electrical-computer-engineering/people/profile.html?id=lij)：纽约州立大学宾汉姆顿分校电气与计算机工程助理教授；主要研究方向为强化学习、在线算法、网络、边缘和云计算系统、基于机器学习的数据分析、物联网和博弈论。
* [Seung-Jong Park](http://www.csc.lsu.edu/~sjpark/index.html)：路易斯安那州立大学计算机科学与工程系的教授；主要研究方向基于大数据和深度学习的数据科学、数据密集型计算的网络基础设施、无线传感器/自组织网络和用于高性能计算的高速网络。

## 论文动机

### 问题引入：

* 现有的 serverless 平台都是采用静态策略来分配资源的。
* 用户无法准确估计函数运行所需要的资源量，这会导致资源的过度分配和欠分配。
* serverless 计算场景下的高并发和细粒度的资源隔离，放大了这种资源分配造成的影响。

文章对比两个不同的 RM 在相同负载下的工作表现：
* Fixed RM：用户预定义内存配置，平台分配等比例 CPU 资源。
* Greedy RM：以固定步长，主动优化资源配置。

![image](https://raw.githubusercontent.com/JBinin/Image-hosting/master/FaaSRM/image.dauh3om88rc.png)

与 Fixed RM 相比，Greedy RM 从超额配置的函数（函数 2、4、5 和 6 ）中获取闲置的 CPU 核，而没有显著降低它们的性能，并以额外的资源来加速低配置的函数（函数 1 和 3 ）。在相同的环境和工作负载下，Greedy RM 通过显著改善整个工作负载而优于 Fixed RM。

### 论文所研究的问题有哪些挑战：

* 人类设计的资源配置策略很难理解平台和函数的特性，这可能导致某些问题。例如，Greedy RM 可能由于过度收集资源而无法处理负载高峰，或者在提供函数资源时提供超过需要的资源。

* 对运行在微软 Azure Functions 上的 serverless 应用程序的特征分析表明，大多数 serverless 工作负载在持续时间、调用和资源消耗的痕迹上都有明显的特征。
* FaaSRM 通过学习 serverless 平台和函数背后的模式来管理和提供计算资源。而据文章描述，现有的 FaaS 平台都不能根据 serverless 函数的工作负载模式来智能地管理和配置资源。

云服务商在解决资源分配不合适时存在的挑战：

* 用户的函数被视为一个黑盒，serverless 系统很难准确估计函数的资源需求。
* 云服务上的庞大应用拆分后部署在 serverless 平台上会产生很多的函数，这些函数有不同的资源需求和动态的输入负载。
* serverless 函数的资源分配是空间上是细粒度的，时间上是短暂的。

## 论文贡献

1. 在 Apache OpenWhisk 平台上实现了支持多进程的基于 PyTorch 的 FaaSRM。
2. 在模拟和 OpenWhisk 实验中使用 Azure Functions traces 对 FaaSRM 和其他三个 baseline 的 RM 进行评估。

## FaaSRM：

FaaSRM 在函数级别重新平衡资源，收集资源过度分配函数的空闲资源来加速资源欠分配函数的运行速度。

### 解决方案描述：

* FaaSRM 的目标：
  * 收集资源过度分配函数的空闲资源。
  * 加速资源欠分配函数的执行。
  * 保证没有函数会产生明显的性能降低。
  
* FaaSRM 架构：

  ![image](https://raw.githubusercontent.com/JBinin/Image-hosting/master/FaaSRM/image.268hant3n43k.png)

  * 对于每一次函数调用，FaaSRM 都会根据 DRL agent 的预测提供一个最佳的资源分配策略。为了给函数调用提供最佳的资源分配策略，FaaSRM 收集平台和函数的信息，并将这些信息作为状态输入到 DRL agent。DRL agent 根据输入信息，计算得出一个最佳的结果。
  * 解除 CPU 和内存的绑定，用相同但是独立的系统分别管理 CPU 和内存。
  * 将用户定义的资源量作为安全机制的基线，用来保证单个函数调用的 SLOs。
  
* 安全机制：
  
  * 安全机制调用即使用用户资源默认配置来进行调用。
  * 假如当前的调用是第一次调用时，用 safeguard 调用，收集资源使用信息并校准基线。
  * 非第一次调用，查询上一次调用的资源使用峰值、资源分配量和上一次基线校准之后的资源使用的最高峰值。
  * FaaSRM 首先检测函数的状态，既过度分配和欠分配，文章假定资源使用量小于用户定义的资源量的 90% 即为过度分配。
    * 对于资源过度分配函数，FaaSRM 检测函数最近一次调用的资源使用量峰值。如过达到了分配资源的 90%，则推断可能出现了负载的峰值，这可能会使用比现在分配的更多的资源。这会导致一次 safeguard 调用并重新校准基线。如果没有达到 90%，则从 [recent_peak + 1, user_defined] 之间选择一个策略。
    * 对于资源欠分配的函数，则从 [recent_peak + 1, max_per_function] 之间选择一个策略。
  
* 重新平衡 serverless 函数资源的 3 个挑战：

  * CPU 和内存的耦合。现有平台 CPU 和内存是绑定的。
  * 巨大的状态空间。例如，AWS Lambda 允许用户配置从 128 MB 到 10240 MB 的内存，并可以访问 6 个 CPU 内核，则总共有 60,672 个选择。巨大的动作空间会导致神经网络的输出规模巨大，效率低下且难以训练。
  * 性能降低。保证单个函数的 SLO 至关重要，即要保证收集的函数没有显著的性能下降。

  为了解决前两个挑战，文章提出了一个使用评分函数在 FaaSRM 中做出决策的 DRL agent。 通过重用相同的 score function，设法解耦 CPU 和内存资源，并使用固定大小的动作空间做出二维决策。 对于最后一个挑战，文章设计了一种保护机制来保护 FaaSRM 收集的函数性能。 

  ![image](https://cdn.jsdelivr.net/gh/JBinin/Image-hosting@master/FaaSRM/image.6cr9006xhz40.png)
  
* 上图描述了 FaaSRM 的流程，DRL 采用了 PPO 的方式。

## 实验

实验设置：

* 两个神经网络，每个神经网络都有两个全链接的隐含层，第一个隐含层有 32 个神经元，第二个隐含层有 16 个神经元。
* 神经元都是用 tanh 作为激活函数。
* 在 OpenAI Gym 模拟
  * 训练 5000 episodes。 
  * 从 Azure Functions traces 中抽取了1,000 个不同的函数。
  * 20 个 worker 服务器, 每个服务器配置 8 个 CPU 核心和 2 GB 内存 。
  * 每个函数最多可以得到 8 个 CPU 核心和 512 MB 内存。
* 在 OpenWhisk 上实验
  * 训练 500 episodes，实验训练时间为 150 小时，每次训练 1 个episode 都重启 OpenWhisk 平台。
  * 选取了 10 个实际 serverless 应用进行实验，来自三个 benchmark：SeBS、ServerlessBench 和 ENSURE-workloads。
  * 在一个有 13 台服务器的 OpenWhisk 集群上部署和评估。一台服务器托管 REST 前端、API 网关和 Redis 服务；一台后端服务器托管控制器、分布式消息和数据库服务；一台服务器托管 FaaSRM agent；其余 10 台服务器是执行函数的 Invokers。
  * 托管 FaaSRM agent 的服务器有 16 个 Intel Xeon Skylake CPU 内核和 64 GB 内存，而其他 12 台服务器每台有 8 个 Intel Xeon Skylake CPU 内核和 32 GB 内存。
  * 每个函数最多可以配置 8 个CPU 核心和 512 MB 的内存。
  * 使用 100 毫秒的窗口来监测容器内的函数调用的 CPU 和内存使用情况。

baseline 设置：

* Fixed RM：现有大多 serverless 的默认 RM，用户预先定义函数的内存使用量，然后平台会分配用户定义的内存量和等比例的 CPU 资源量。文章假设用户在运行过程中不改变函数的配置。
* Greedy RM：以固定步长主动优化资源分配，对于每一次调用，它都会根据函数的最近资源利用率来进行资源分配。在文章中的步长设置为：CPU 为 1 个核心，内存是 64 MB。
* ENSURE：在函数级别管理 CPU 资源，仅在检测到函数性能降低时动态调整 CPU 资源。但是没有对内存进行调整的策略。

#### 实验项目1

##### 模拟：

![image](https://cdn.jsdelivr.net/gh/JBinin/Image-hosting@master/FaaSRM/image.503o9a3el9q8.png)

* 上图显示了由四种 RM 处理的 85,470 个函数调用的 RFETs，FaaSRM 最小平均 RFET 为0.76，而 Fixed RM、Greedy RM、ENSURE 分别为1.00、1.49 和 0.91。

* Fixed RM 没有性能下降，但也没有资源调整。Greedy RM 在收获太多资源的同时，出现了一些严重的违反 SLO 的情况。ENSURE 也违反了调用的 SLO，甚至只调整了 CPU 资源。相比之下，FaaSRM 合理地从超额配置的函数中获取闲置资源，并为配置不足的函数提供显著的加速。



![image](https://cdn.jsdelivr.net/gh/JBinin/Image-hosting@master/FaaSRM/image.1rqmd48hyn8g.png)

* 上图显示了所有调用的函数执行时间的累积分布函数（CDF），FaaSRM处理大多数函数调用的速度比其他 baseline 的 RM 快。

* FaaSRM完成 98% 的调用，执行时间小于 15 秒，而 Fixed RM、Greedy RM 和 ENSURE 完成执行时间分别为121秒、76 秒和 48 秒。

#### 实验项目2

##### 实验：

![image](https://cdn.jsdelivr.net/gh/JBinin/Image-hosting@master/FaaSRM/image.758y6l9phngg.png)

* 上图显示了由四个 RM 处理的 268 个函数调用的 RFETs。FaaSRM 工作负载的平均 RFET 为 0.89，而 Fixed RM, Greedy RM、ENSURE 分别为 1.09, 0.93 和 1.12。

* 在OpenWhisk实验中，由于性能变化，Fixed RM 的平均 RFET 不是严格意义上的1.0。

* Fixed RM 没有资源调整。Greedy RM 和 ENSURE 在收集资源时都严重违反了一些调用的 SLO。与模拟中调用的完美 SLO 相比，在 OpenWhisk 中评估 FaaSRM 会出现一些性能变化。FaaSRM 仍然实现了最小的平均 RFET，并将所有调用的 RFET 的 98% 保持在 1.28 以下，而 Fixed RM、Greedy RM 和 ENSURE 分别为 1.27、2.52 和 5.2。

  ![image](https://cdn.jsdelivr.net/gh/JBinin/Image-hosting@master/FaaSRM/image.41i3r0cw73ls.png)

* 上图显示了所有调用的函数执行时间的累积分布函数（CDF）。FaaSRM 处理大多数函数调用的速度比其他基线 RM 快。

* FaaSRM 以低于16.91 秒的执行时间完成了 98% 的调用，而 Fixed RM、Greedy RM 和 ENSURE分别是 32.16 秒、25.80 秒和 27.39秒完成执行。

## 其他

结论：

* 文章提出了一个新的资源管理器 FaaSRM，收集资源过度分配的闲置资源，并用这些资源加速资源欠分配的函数。
* 在实际的 serverless 工作负载下，利用强化学习和安全机制提升大多数函数的执行效率并安全地收集闲置资源。
* 在模拟和 OpenWhisk 集群的实验中表明，FaaSRM 有比其它 baseline 更好的表现。与默认的 RM 相比，FaaSRM 减少了 98% 的函数调用时间，在相同的负载下，模拟结果显示减少了 87.60%，实验结果显示减少了 35.81%。在 OpenWhisk 的实验中，FaaSRM 收集了38.78% 函数的空闲资源，并将这些资源用于加速 39.18% 的函数。

文章相关工作：

* 资源调度
  * SmartHarvest：提出了采用在线学习的 VM 资源收集算法，与 FaaSRM 将收集的资源用来加速函数执行时间不同，SmartHarvest 利用收集的资源提供新的低优先级 VM 服务。
  * Spock：提出了一个基于 serverless 的 VM 伸缩系统来提升 SLOs 并减少开销。
  * 在检测到性能降低时自动调整 CPU 资源。
    * Automated Fine-Grained CPU Cap Control in Serverless Computing Platform
    * Efficient Scheduling and Autonomous Resource Management in Serverless Environments
  * 提出了一个 serverless 平台的中心化调度器，将 worker 服务器的 CPU 核分配给调度器的 CPU 核，以实现精细的核对核管理。
    * Centralized Core-Granular Scheduling for Serverless Functions

## 补充

强化学习的缺点：

* 所需样本数量过大。对于简单问题，强化学习尚需百万、千万级别的样本，对于现实世界的复杂问题，强化学习需要多少样本呢。在电子游戏中获取上亿样本或许不太难，但是在现实世界中获取每一个样本都是困难的。
* 探索阶段代价太大。强化学习要求智能体与环境交互，用收集到的经验去更新策略。在交互的过程中，智能体会改变环境。在仿真、游戏的环境中，智能体对环境造成任何影响都无所谓。但是在现实世界中，智能体对环境的影响可能会造成巨大的代价。
* 超参数的制约。深度强化学习对超参数的设置极其敏感，需要很小心调参才能找到好的超参数。超参数分两种：神经网络结构超参数和算法超参数。这两类超参数的设置都严重影响实验效果。换句话说，完全相同的方法，由不同的人实现，效果会有天壤之别。
* 稳定性极差。强化学习的过程中充满了随机性。在监督学习中，由于随机初始化和随机梯度中的随机性，即使使用同样的超参数，训练出来的模型表现也会不一致，测试的准确率可能会差几个百分点。但是几乎不会出现完全不收敛的情况，但是强化学习中确可能出现，即使超参数设置完全正确。
