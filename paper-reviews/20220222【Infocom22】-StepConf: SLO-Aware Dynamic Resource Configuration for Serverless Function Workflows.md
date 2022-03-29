By Jiabin Chen

# StepConf:无服务器函数工作流的SLO感知动态资源配置
[StepConf: SLO-Aware Dynamic Resource Configuration for Serverless Function Workflows](https://fangmingliu.github.io/files/INFOCOM22-serverless.pdf)
---

* 论文发表在 Infocom'22 上，为无服务器函数工作流提出了一个动态资源配置框架 StepConf，以满足 SLOs 的同时减少成本开销。
* StepConf 主要思想是在工作流中执行每个函数之前，实时动态地做出配置决策，即选择合适的函数内存大小，以及函数间和函数内的并行度。
* 在 AWS Lambda 上进行实验对比，StepConf 与现有的静态策略相比，在保证 SLOs 的情况下，减少了 40.9% 的成本开销。

## 论文作者
* Zhaojie Wen, Yishuo Wang：华中科技大学
* [Fangming Liu](https://fangmingliu.github.io)：华中科技大学计算机科学与技术学院教授；主要研究方向为分布式和软件系统。

## 论文动机
### 问题引入：

* 开发人员需要给函数配置合适的资源，以使得其能满足 SLO 并且节省金钱开销。然而由于云函数的执行经常会遇到冷启动和性能波动的影响，所以制定资源配置策略具有挑战性，文章认为需要一个动态的资源配置策略。但是这需要解决一些问题：

  * Vague impact of resource configuration：函数的成本和性能高度依赖于用户配置的资源参数。然而，目前尚不清楚如何选择合适的资源参数可以实现高性能和低成本。

  * Exponential growth of configuration space：用户需要为函数配置一系列资源参数，随着工作流中函数数量的增加，资源参数的决策空间呈指数增长，这使得获得最优配置变得具有挑战性。同时，根据不同的 SLO 动态调整配置使资源配置问题进一步复杂化。

* 哪些资源需要配置：

  * 函数的内存
  * 函数内并行度（在函数内采取多线程或者多进程的方式）
  * 函数间并行度（将任务映射到多个并行的函数）

<img src="https://cdn.jsdelivr.net/gh/Jbinin/Image-hosting@master/uPic/202203/image-20220305220629506.png" alt="image-20220305220629506" style="zoom:67%;" />

## StepConf：

### 建模

* 工作流执行模型
  * 通过同时运行相同无服务器函数的实例来执行并行任务的过程称为 function step
  * 使用有向无环图 G = (V, E) 来表函数工作流
  * 路径 $L \in \mathcal{L}$ 表示从起点到终点的顶点集合
  * 定义 function step $v_i$ 的运行时间为 $t_i$，则函数工作流的完成时间为 $T=\max _{L \in \mathcal{L}} \sum_{v_{i} \in L} t_{i}$
  * 将 function step $v_i$ 的成本定义为 $c_i$，处理工作流的总成本可以计算为 $C=\sum_{v_{i} \in V} c_{i}$


* function step 模型

  * 对于每个 function step $v_i$，内存大小、函数间并行度和函数内并行度分别定义为 $m_i$、$p_i$ 和 $q_i$

  * 如图 7 所示，由于编排开销和冷启动，函数不会同时启动，文章定义 mapping delay 为从 function step 开始到云服务商准备好函数实例沙箱的持续时间，表示为 $t_{i}^{m a p}(p)$

  * 定义 function step 的执行时间为 $t_{i}=\max _{1 \leq j \leq p_{i}}\left[t_{i}^{m a p}(j)+t_{i, j}\right]$，为了简化，这里用平均值来代替函数的执行时间，则 $t_{i}=t_{i}^{\operatorname{map}}\left(p_{i}\right)+\hat{t_{i}}$

  * 定义 function step 中的作业数为 $\gamma_{i}$，则函数实例的作业数为 $\bar{\gamma}_{i}=\frac{\gamma_{i}}{p_{i}}$

  * 云厂商为一个函数准确分配完整单核资源的内存大小设置为 $m_s$，其中 s 代表 single，其相对性能记为 $\delta_s$

    <img src="https://cdn.jsdelivr.net/gh/Jbinin/Image-hosting@master/uPic/202203/image-20220306111430698.png" alt="image-20220306111430698" style="zoom: 67%;" />

  * 以 AWS 的定价模型为代表,将每 GB-s 函数的价格表示为 $μ_0$，并且函数请求和编排的价格为 $μ_1$，其中 $μ_0$ $μ_1$ 是常数，则 function step $v_i$ 的定价为 $c_i=p_i·(t_i·m_i·μ_0+μ_1)$

  * 文章为函数执行时间定义了一个分段拟合函数，对于多核友好的程序（case 1），使用指数函数进行拟合；对于只能利用单线程性能或者内存小于$m_s$ 的程序（case 2），使用反比例函数进行拟合，其中 $\alpha_{i, 1}, \alpha_{i, 2}, \beta_{i, 1}, \beta_{i, 2}, \phi_{i, 1}, \phi_{i, 2}$ 都是模型参数
    $$
    \hat{t}_{i}= \begin{cases}\left(\alpha_{i, 1} \cdot \bar{\gamma}_{i}+\beta_{i, 1}\right) e^{-\alpha_{i, 2} \cdot \min \left(\frac{m_{i}}{m_{s}}, q_{i}\right)}+\phi_{i, 1}, & \text { for case } 1 \\ \frac{\alpha_{i, 2} \cdot \bar{\gamma}_{i}}{\delta_{s} \cdot \min \left(\frac{m_{i}}{m_{s}}, q_{i}\right)+\beta_{i, 2}}+\phi_{i, 2}, & \text { for case } 2\end{cases}
    $$

* SLO 感知模型配置问题
  * 定义每个 function step $v_i$ 的配置为 $\theta_i=(m_i,p_i,q_i)$，定义所有配置的集合为 $\Theta_i$，使用二进制变量 $x_i(\theta)$ 来表示函数的配置是否为 $\theta$，则问题的限制条件之一为 $\sum_{\theta \in \Theta_{i}} x_{i}(\theta)=1, \forall v_{i} \in V$
  
  * 让 $\mathcal{S}$ 表示 SLO，SLO 的限制课表述为 $T=\max _{L \in \mathcal{L}} \sum_{v_{i} \in L} \sum_{\theta \in \Theta_{i}} t_{i}(\theta) \cdot x_{i}(\theta)<\mathcal{S}$
  
  * 在上述 2 个限制条件下，文章的 Function Workflow Configuration Problems (FWCP) 可表示为 
    $$
    \min C=\min \sum_{v_{i} \in V} \sum_{\theta \in \Theta_{i}} c_{i}(\theta) \cdot x_{i}(\theta)
    $$
    
  
* SLO 感知模型配置问题可在多项式时间内由 multiple-choices knapsack problem 规约得到，故是一个 NP 难问题

### 问题解决方案

文章提出了一种启发式的方法来解决上述的问题，文章称之为 Problem Relaxation and Global-cached Most Cost-effective Critical Path Algorithm：

* 为将问题化简，将应用的 SLO 进行划分，给每个 function step 设置一个 sub-SLO 限制，sub-SLO 满足 $\forall L \in \mathcal{L}, \sum_{v_{i} \in L} s_{i} \leq \mathcal{S}$，其中 $s_i$ 表示 sub-SLO 限制，则原先的 SLO 限制变成了 $t_{i}=\sum_{\theta \in \Theta_{i}} t_{i}(\theta) \cdot x_{i}(\theta)<s_{i}$

* 原先的优化问题也就变成了
  $$
  \min c_{i}=\min \sum_{\theta \in \Theta_{i}} c_{i}(\theta) \cdot x_{i}(\theta)
  $$
  

* 最后一个问题是怎么确定每个 function step 的 $s_i$ ，文章将关键路径上 fucntion step 的执行时间作为权重，并按权重比例分配 SLO 为 sub-SLO。但是在线确定函数的执行时间存在困难，文章发现靠近最佳性价比的配置，其性价比也较高，所以文章将在最佳性价比配置下的函数执行时间作为权重，最佳配置的计算方式为 $\theta^{*}=\arg \max _{\theta} \frac{c_{i}(\theta)}{t_{i}^{-1}(\theta)}$，而 sub-SLO 的计算方式则可以使用 $s_{i}=\frac{t_{i}\left(\theta^{*}\right)}{\sum_{v_{i} \in \mathcal{L}_{\text {critical }}} t_{i}\left(\theta^{*}\right)} \cdot \mathcal{S}$

## 实验

实验设置：
* 基于 AWS Lambda 和 AWS Step Function 实现 StepConf 框架
* 负载：
  * 视频处理程序，包括 5 个 function step
  * 基于 serverless benchmark suite 合成的工作流


baseline 设置：
* No-No：使用静态配置策略，不考虑函数间或函数内并行度。静态策略是指在工作流开始运行之前为所有功能步骤做出配置决策，在工作流运行期间配置不变。
* Static：使用与 No-No 相同的静态启发式算法，但具有与 StepConf 相同的配置因素（函数间和函数内并行性）。
* Optimal: 不考虑性能波动的离线优化配置。

#### 实验项目1: Workflow performance under different SLOs

![image-20220307105930642](https://cdn.jsdelivr.net/gh/Jbinin/Image-hosting@master/uPic/202203/image-20220307105930642.png)

* 对比图 14 和 15 的箱线图和图 16 和 17 的 CDF 图，文章的动态配置策略不仅更好地满足了 SLO，而且带来了更小的性能波动。由于库的高度依赖，视频处理工作流中静态和动态配置在波动级别上的差距比合成工作流大。静态方法不能保证 SLO，而文章的动态策略可以在工作流运行过程中发生性能变化时实时调整配置以纠正错误。

#### 实验项目2：Cost saving and performance improvement

![image-20220307110457854](https://cdn.jsdelivr.net/gh/Jbinin/Image-hosting@master/uPic/202203/image-20220307110457854.png)

* StepConf 考虑了函数间和函数内的并行性。分别比较了函数内和函数间并行性对工作流性能和成本的影响。No-No 不使用任何函数并行性，将其用作归一化基准，与 No-No 相比，图 18 表明 StepConf 在相同的成本预算下可以将性能提高 5.32 倍，并在相同的性能下节省 40.9% 的成本。
* 当 SLO 相对较小时，StepConf 比 inter-only 或 intra-only 方法具有更大的成本优势。当 SLO 比较大时，任何 inter 或 intra 并行都可以达到很好的效果。
* 与离线优化方法相比，StepConf 的成本在文章的测试平台上最多高出 16.48%。 
* StepConf 的启发式算法虽然不能保证全局最优配置，但也能保证大多数时候优化的效果。

## 文章相关工作：

#### Performance and cost optimization for serverless

* Akhtar et al. 通过使用较少测试的统计学习方法优化函数链的配置。 (Cose: Configuring serverless functions using statistical learning) -- Infocom20
  * COSE，一个使用贝叶斯优化来寻找无服务器函数的最佳配置的框架。 COSE 使用统计学习技术智能地收集样本并预测无服务器功能在看不见的配置值中的成本和执行时间。


* Elgamal et al. 共同优化函数布局和函数大小以节省成本。(Costless: Optimizing cost of serverless computing through function fusion and placement) -- SEC18
  * (1) 融合一系列函数，(2) 跨边缘和云资源拆分函数，以及 (3) 为每个函数分配内存。


* Eismann et al. 使用混合密度模型来预测函数的执行时间分布以优化无服务器工作流的成本 (Predicting the costs of serverless workflows) -- ICPE20

* Carver et al. 为数据局部性优化的 DAG 工作流设计 FaaS 计算框架。 (Wukong: A Scalable and Locality-Enhanced Framework for Serverless Parallel Computing) -- SOCC20

* Bhasi et al. 提出一个工作流感知的无服务器函数资源自适应自动缩放器。 (Kraken: Adaptive container provisioning for deploying dynamic dags in serverless platforms) -- SOCC21


* Zhang et al. 引入一个新的调度问题来确定无服务器数据分析的理想任务启动时间。 (Caerus: Nimble task scheduling for serverless analytics.) -- NSDI21

* Kijak et al. 提出了科学工作流的时限感知功能资源调度问题，并提出了一种节约成本的配置策略。 (Challenges for scheduling scientific workflows on cloud functions) -- CLOUD18

* Lin et al. 将函数工作流描述为无 DAG 模式，其中包含循环和自循环。他们提出了一种概率精炼的关键路径算法来解决性能约束下的最佳成本问题和预算约束下的最佳性能问题。(Modeling and optimization of performance and cost of serverless applications) -- TPDS20

## 其他

