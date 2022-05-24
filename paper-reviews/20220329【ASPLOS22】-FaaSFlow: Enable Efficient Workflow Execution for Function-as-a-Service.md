By Jiabin Chen


---

# FaaSFlow：函数即服务的高效工作流

[FaaSFlow: Enable Efficient Workflow Execution for Function-as-a-Service](https://doi.org/10.1145/3503222.3507717)
---

* 文章发表在 ASPLOS22 上，目标是实现一个高效的无服务器计算工作流。

* 论文提出并实现了一个无服务器计算工作流 FaaSFlow，使用了 worker-side 工作流调度模式，将调度下放到每一个 worker 中，以减轻采用 master-side 工作流调度模式的调度开销。

* 文章在 FaaSFlow 基础上实现了一个自适应存储库 FaaStore，利用分配给容器的多余内存，来使得同一结点内的数据交换不需要通过远程数据存储，从而减轻在同一节点中的函数间的数据传输开销。

* 实验结果表明，FaaSFlow 有效地降低了平均 74.6% 的工作流调度开销和最多 95% 的数据传输开销。当网络带宽波动时，FaaSFlow-FaaStore 将吞吐量降低 23.0%，并能将网络带宽利用率提高 1.5 倍到 4 倍。

## 论文作者
* Zijun Li, Yushi Liu, Linsong Guo, Jiagan Cheng：上交
* [Quan Chen](https://www.cs.sjtu.edu.cn/~chen-quan/)：上交计算机教授；新兴并行计算研究中心；研究兴趣是计算机系统，并行分布式计算，数据中心，任务调度。
* [Wenli Zheng](https://www.cs.sjtu.edu.cn/PeopleDetail.aspx?id=371)：上交计算机副教授；新兴并行计算研究中心；研究兴趣是大规模计算系统与高能效计算、大数据计算与分布式机器学习和多主体计算。
* [Minyi Guo](https://cs.sjtu.edu.cn/~guo-my/)：上交计算机教授；新兴并行计算研究中心；研究兴趣是并行和分布式处理、并行化编译器、普适计算、软件工程、嵌入式系统和绿色计算。

## 论文动机
### 问题引入：

* 主要的云服务提供商都提供了无服务器工作流，比如 AWS step functions，Microsoft Durable Functions，Google Workflows 和 Alibaba Serverless Workflows。开源的无服务器系统如 OpenWhisk 和 Fission 同样支持处理函数序列的工作流。
* 上述的无服务器工作流系统通常会有一个位于 master 节点的中心化的工作流引擎，负责管理工作流执行状态和给 worker 节点分配函数任务。文章将这种调度模式称为 master-side 工作流调度模式（MasterSP），因为在 Master 节点的中心化的工作流引擎会负责决定一个函数任务是否被触发执行。

![image-20220326171826894](https://cdn.jsdelivr.net/gh/Jbinin/Image-hosting@master/uPic/202203/image-20220326171826894.png)

* MasterSP 并不能很好的适应无服务器平台的事件驱动的特性，因为其引入了巨大的调度开销和数据移动开销。
  * 函数执行状态会很频繁地从 master 节点传到 worker 节点，而每个函数的执行时间又很短，所以会造成很大的调度开销。
  * 工作流引擎随机将函数任务分配给 worker 节点以实现负载均衡，云服务商对函数的输入和输出数据大小施加配额，以避免严重消耗网络带宽。在生产型无服务器平台中，用户通常依赖额外的数据库存储服务来进行临时数据存储和传递，因此会承受巨大的数据移动开销。

![image-20220326214558806](https://cdn.jsdelivr.net/gh/Jbinin/Image-hosting@master/uPic/202203/image-20220326214558806.png)

* 上图是在 MasterSP 下测得的几个不同工作流的调度开销，由于函数的执行时间很短，所以这是一个很大的开销，特别是对毫秒级别延迟敏感的应用来说更为严重。

![image-20220326215856747](https://cdn.jsdelivr.net/gh/Jbinin/Image-hosting@master/uPic/202203/image-20220326215856747.png)

* 如图所示，与构建单独的应用程序（同一容器中运行的所有函数都使用直接的内部调用）相比，函数隔离无疑会带来更多的任务间数据通信开销。对于无服务器工作流来说，有必要增强数据的局部性，减少通过网络的数据传输。

## FaaSFlow：

* FaaSFlow 实现了 worker-side 的工作流调度模式，并且实现了一个 FaaStore 来管理混合存储。
* 在 workerSP 中，master 中的图调度器仅负责将工作流的图划分为子图分配给 worker，并确保相应的 worker 有足够的资源执行这些子图，而每个 worker 的引擎会负责执行本地函数的触发和调用。

![image-20220327085312054](https://cdn.jsdelivr.net/gh/Jbinin/Image-hosting@master/uPic/202203/image-20220327085312054.png)

* 如上图所示，每个 worker 都按照 Workflow State 和 FunctionInfo 处理其本地引擎中函数的同步。当一个函数的所有前置函数都被标记为已执行时，本地引擎将触发本地调用，并在其本地子图中同步更新状态。如果函数具有跨 worker 依赖关系，则本地引擎将通过 TCP 连接将执行状态传递给远程 worker 的引擎。

![image-20220327112102987](https://cdn.jsdelivr.net/gh/Jbinin/Image-hosting@master/uPic/202203/image-20220327112102987.png)

* 上图展示了 FaaStore 的协同设计机制，用户的代码和数据会被容器的执行器导入到内存中，FaaStore 会检查函数的后续函数是否位于同一节点上，并相应地选择适当的数据存储。

### FaaSFlow 实现

![image-20220327112751914](https://cdn.jsdelivr.net/gh/Jbinin/Image-hosting@master/uPic/202203/image-20220327112751914.png)

* FaaSFlow 的图分割算法：
  * 输入为：工作流图 G，函数 f1，f2...fn
  * 算法如下：
    * 每个函数独立作为一个组，随机为每个组选择一个 worker 节点
    * 迭代以下过程，直到不再有合并发生：
      * 选择 G 的关键路径
      * 将关键路径上的节点按权重降序排列
      * 对关键路径上的每条边进行迭代：
        * 边对应的两个函数 f1 和 f2 所在的组将在满足以下条件下进行合并
          * 存在一个节点有足够容量容纳合并后的组
          * f1 和 f2 不存在严重的干扰
        * 根据装箱算法选择一个最合适的节点放置新的组

## 实验

负载设置：

* Pegasus 的 4 个科学工作流程：Cycles (Cyc), Epigenomics (Epi), Genome (Gen), SoyKB (Soy),
* 4 个现实世界的应用：Video-FFmpeg (Vid), Illegal Recognizer (IR), File Processing (FP), Word Count (WC)

实验设置：
* 8 节点集群，1 个节点作为 master 节点部署图调度器和产生工作流调用，7 个节点作为 worker 节点部署工作流引擎
* 使用 CouchDB 作为远程键值存储，使用 Redis 作为内存存储
* 硬件配置如下：

![image-20220328010008829](https://cdn.jsdelivr.net/gh/Jbinin/Image-hosting@master/uPic/202203/image-20220328010008829.png)

#### 实验项目1：调度开销

![image-20220328101514875](https://cdn.jsdelivr.net/gh/Jbinin/Image-hosting@master/uPic/202203/image-20220328101514875.png)

* FaaSFlow 将科学工作流的平均调度开销从 712ms 减少到 141.9ms，将实际应用的平均调度开销从 181.3ms 减少到 51.4 ms，所有的应用都获得了平均 74.6% 的调度开销的减少。

#### 实验项目2：数据移动开销

![image-20220328120538950](https://cdn.jsdelivr.net/gh/Jbinin/Image-hosting@master/uPic/202203/image-20220328120538950.png) 

#### 实验项目3：尾部延迟和吞吐量

![image-20220328145140669](https://cdn.jsdelivr.net/gh/Jbinin/Image-hosting@master/uPic/202203/image-20220328145140669.png)

* 文章将超过 60s 视为执行超时，并将此调用的执行时间设置为 60s
* Gen 和 Cyc 的执行超时，是由于 parallel 和 foreach 操作的网络瓶颈，大量函数实例竞争网络资源造成的。

#### 实验项目4：共置干扰

![image-20220328151323046](https://cdn.jsdelivr.net/gh/Jbinin/Image-hosting@master/uPic/202203/image-20220328151323046.png)

* 网络带宽的竞争

#### 实验项目5：调度器的可扩展性

![image-20220328153304312](https://cdn.jsdelivr.net/gh/Jbinin/Image-hosting@master/uPic/202203/image-20220328153304312.png)

* 根据分析，随着函数节点数量的增加，开销大致遵循 𝑂（n^2）。
* 当工作节点数量增加、平均 CPU 和内存使用的资源保持稳定。
* 函数节点少于 50 个时，对于大多数当前 internet 应用程序，文章的 Graph Scheduler 在调度方面表现出了更高的性能。

## 文章相关工作：

* 无服务器工作流优化
  * WuKong 实现了一个基于发布者/订阅者的无服务器并行框架。它通过 lambda 执行器执行任务调度，并将 DAG 划分为多个子图。中间任务输入和输出可以存储在同一执行器中，以增强数据的局部性。
  * Viil等人也使用类似的想法，即使用任务分片和迁移来自动调配和配置资源。
  * NightCore 假设应用程序的所有函数都可以安排在同一台服务器上，通过减少同一节点上函数的 RPC 开销。然而，Nightcore 在不同的工作服务器之间使用 MasterSP，因为它们的网关充当分配任务和分配执行状态的集中管理器。

