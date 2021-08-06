By Jiabin Chen

# FaaSNet：阿里云函数计算平台，serverless容器运行时的可扩展快速部署方案
[FaaSNet: Scalable and Fast Provisioning of Custom Serverless Container Runtimes at Alibaba Cloud Function Compute](https://www.usenix.org/conference/atc21/presentation/wang-ao)
---

* 该文是2021年发表在 ATC 上的论文，Alibaba Group 团队第一个提出在 FaaS 场景下的去中心化快速镜像分发技术，其目标是缩短突发流量下 FaaS 中的大规模容器镜像启动（函数冷启动）时延。

* 该文继承了 Alibaba Group 在2020年ATC的文章 [DADI: Block-Level Image Service for Agile and Elastic Application Deployment](https://www.usenix.org/conference/atc20/presentation/li-huiba) 中的容器镜像加速技术，设计了一种具有高伸缩性的轻量级系统中间件 FaasNet，它的的核心组件是一个 Function Tree（FT），一个去中心化的、自平衡的二叉树状拓扑结构，树状拓扑中所有节点等价。

* 该文将提出的 FaaSNet 集成到阿里云函数计算产品上，做到了：
	* 在1000个 VM 上建立2500个函数容器花费8.3s，容器启动速度是阿里云现有 FaaS 平台的13.4倍，是 Kraken（一个基于P2P的容器仓库）的16.3倍。
	* 并且对于由于突发请求量带来的端到端延迟不稳定时间，FaaSNet 相比阿里云现有 FaaS 平台少用 75.2% 的时间可以将端到端延迟恢复到正常水平。

## 论文作者
* [Ao Wang](https://wangaoone.github.io)：乔治梅森大学在读博士；LeapLab实验室；主要研究方向有 serverless、云计算和云存储。
* Shuai Chang：Alibaba Group
* [Huangshi Tian](http://home.cse.ust.hk/~htianaa/)：香港科技大学在读博士。
* Hongqi Wang, Haoran Yang, Huiba Li, and Rui Du：Alibaba Group
* [Yue Cheng](https://cs.gmu.edu/~yuecheng/)：乔治梅森大学助理教授；LeapLab实验室；主要研究方向有分布式系统、存储系统、基于容器的虚拟化技术、serverless、云计算和物联网。

## 论文动机
近来Serveless上引入自定义容器镜像成为一种趋势，但快速容器镜像分发仍存在着巨大的挑战：
* FaaS 上的 wordload 具有高动态性和突发性，突发 wordload 下，从同一镜像仓库拉取镜像会造成网络瓶颈
* 镜像体积庞大

现行容器镜像分发解决方案，是用P2P来实现大规模容器分发加速：
1. [Dragonfly](https://d7y.io/en-us/)
* 是一个基于P2P的镜像、文件分发网络
* 多个 dfget（Peer 节点）分别贡献同一个文件的不同 pieces 来达到给目标节点点对点传输的目的）
* 依赖若干性能规格较大的 Supernode （Master 节点），通过 Supernode 节点管理和维护一个全连接拓扑结构
* 尽管 Supernode 节点能做出最佳决定，但整个集群的吞吐量受到一个或者几个主机处理能力的限制，性能会随着 blob 的大小和集群规模的增大而线性降低

![9F4379BD-39D3-41C0-804F-A7F6837A5C9D](https://cdn.jsdelivr.net/gh/JBinin/Image-hosting@master/20210805/9F4379BD-39D3-41C0-804F-A7F6837A5C9D.4bbl8igobslc.png)

2. [Kraken](https://github.com/uber/kraken)
* 以 layer 为粒度进行组网
* origin、tracker 节点作为中央节点管理整个网络，agent 存在于每个 peer 节点上
* tracker 节点仅负责管理组织集群中 peer 的连接，Kraken 会让 peer 节点间自行沟通数据传输，所以 Kraken 比 Dragonfly 能更好地处理大 blobs
			
![76FFDA16-83FA-4998-8633-8B500295D5B1](https://cdn.jsdelivr.net/gh/JBinin/Image-hosting@master/20210805/76FFDA16-83FA-4998-8633-8B500295D5B1.u0k5rzkn9gg.png)

3.[DADI](https://www.usenix.org/conference/atc20/presentation/li-huiba)
* 提供了一种高效的镜像加速格式，可以实现按需读取（FaaSNet 继承了这种技术）
* 以 layer 为粒度进行组网
* 使用了树状拓扑结构进行镜像分发
* DADI 的 P2P 分发需要依赖若干性能规格较大的 DADI-Root 节点，这些节点担任数据回源和拓扑管理的角色
* 树状拓扑结构偏静态，因为容器部署的速度一般不会持续很久，所以默认情况下，DADI-Root 节点回在20分钟后将逻辑拓扑解散
		
![6ABEC0E0-1F8B-403B-9545-AD6F088A8448](https://cdn.jsdelivr.net/gh/JBinin/Image-hosting@master/20210805/6ABEC0E0-1F8B-403B-9545-AD6F088A8448.1ahlww0c5uzk.png)

		
为什么不直接引入上述三种方案：
* 引入额外、专用、中心化的组件，增加成本，同时引入了性能瓶颈
* 阿里云 FaaS 的架构使用了一个动态的资源受限 VM 池，VM 可以随时加入和离开，所以需要高度自适应的解决方案
	
## 论文贡献
1. 提出FaasNet，实现 FaaSNet 并集成进阿里云平台
2. 实验评估 FaaSNet 性能
3. 公开了FT（function tree）原型代码和一个包含生产环境 FaaS 冷启动数据的匿名数据集

## FaaSNet

![396B87F8-34EB-448B-AB1F-22B173986720](https://cdn.jsdelivr.net/gh/JBinin/Image-hosting@master/20210805/396B87F8-34EB-448B-AB1F-22B173986720.6kqgwepazitc.png)

* 图中灰色的部分是文章 FaaSNet 进行了系统改造的部分，其他白色模块均延续了 FC 现有的系统架构。
* FaaSNet 组件：
	* FT：一个去中心化的、自平衡的二叉树状拓扑结构，且树状拓扑中所有节点是等价的。（FT 是 FaaSNet 的核心组件，其设计来源于 AVL 树算法的启发。在 FT 中，目前不存在节点权重这个概念，所有节点等价，当有节点加入或删除后，FT 会自己调整树的形状从而达到平衡结构）
	* 网关：身份管理认证、将函数请求发到调度器、容器镜像格式转换
	* 调度器：接受函数请求并服务、向 VM manager 申请 VM、访问 FT 并通过 RPC 请求 FaaSNet worker 进行容器部署
	* VM agent：VM 本地函数管理
	*FaaSNet worker：容器部署
* 设计特点：
	* 以函数为粒度进行组网
	* 数据平面和控制平面分离
	* 自平衡的二叉树
* 其他优化技术
	* 高效IO数据格式：原始数据被分为大小相同的块进行压缩
	* 按需IO：以块粒度按需拉取层数据
	* RPC和数据流：实现了一个用户级、零拷贝的RPC库，FaaSNet worker 接收到一个完整数据块时立刻传输给下游节点

## 实验
实验设置：
* 平台：阿里云
* VM规格：2 CPUs, 4 GB memory, 1 Gbps network 
* 维持一个空闲的 VM 池，需要时直接从其中调用，实验中计算 function 冷启动时就可以忽略 VM 冷启动带来的影响
* 每个 VM 仅跑一个 containerized function
* 块大小为512KB
* 实验运行的 function 是一个Python 3.8 PyStan application，运行时间为2秒
* 函数容器镜像的大小758MB
* 给函数配置的内存大小为3008MB

实验方案描述：
* Baseline：从一个中心化仓库直接拉取镜像
* On-demand：从一个中心化仓库按需拉取镜像
* Kraken：方案如上面描述，实验中设置一个 origin 节点
* DADI-P2P：方案如上面描述，实验中设置一个资源受限的 VM 做 DADI-Root 节点
* FaaSNet：本文提出的方案，方案如上面描述

#### FaaS 应用 workloads 下的对比表现
![408606B2-7AF1-4E59-A341-C5F5885D38D6](https://cdn.jsdelivr.net/gh/JBinin/Image-hosting@master/20210805/408606B2-7AF1-4E59-A341-C5F5885D38D6.61z089rii70g.png)
* 从上图可以看出，FaaSNet 时延的峰值更小，回归正常响应时间速度更快。

#### 压力测试
![795D031E-CD34-43A5-BA46-02B100655900](https://cdn.jsdelivr.net/gh/JBinin/Image-hosting@master/20210805/795D031E-CD34-43A5-BA46-02B100655900.5c50hyrl8qo0.png)
* 压测部分记录的延迟为用户感知的端到端冷启动平均延迟。
* 镜像加速功能相比于传统的 FC 可以显著提升端到端延迟，但是随着并发量的提高，更多的机器同时对中央的 container registry 拉取数据，造成了网络带宽的竞争导致端到端延迟上升。
* 但是在 FaaSNet 中，由于去中心化的设计，对源站的压力无论并发压力多大，只会有一个 root 节点会从源站拉取数据，并向下分发，所以具有极高系统伸缩性，平均延迟不会由于并发压力的提高而上升。

#### 多函数
![640](https://cdn.jsdelivr.net/gh/JBinin/Image-hosting@master/20210805/640.4keoqrs5gygw.webp)
* 上图探究了一个 VM 上如果放置不同 image（多函数）的函数的性能表现，上图纵轴表示标准化后的端到端延迟水平。
* 随着不同镜像的函数的数量增多，DADI P2P 由于 layer 变多，并且平台内每台 ECS 的规格较小，对每台 VM 的带宽压力过大，造成了性能下降。
* FaaSNet 由于在镜像级别建立连接，连接数目远远低于 DADI P2P 的 layer tree，所以仍然可以保持较好的性能。

## 其他
文章中提到的缺点及未来工作：
* 以块为粒度拉取过程中，不可避免地会拉取一些不需要的数据，即存在 read amplification，未来需要一些优化技术来减轻。
* 随着 function 数量增长，会出现带宽的争用，文章通过编程来避免出现多函数（在一个 VM 上放置多个不同的 funtcions ）。未来预计随着需求增加，通过扩展容器放置逻辑来解决这个问题。一般目标是在配置多个 funtions 时平衡每个VM 的入站和出站通信。直观地说，通过调整容器位置，我们可以控制 VM 所涉及的 FTs 数量和 VM 所服务的角色，从而控制带宽消耗。进一步的优化是将共享公共层的函数放在一起，这样可以减少数据传输量。
* 现在的平台，租户间不能共享 VM，所以 FaaSNet 的 FTs 是租户隔离的
* FaaSNet 不适合亚秒级的短生命周期函数和请求稀疏的场景，可用 Function environment caching 和 pre-provisioning 来处理这些场景

组会上思考记录：
* 在按需拉取块过程中，p2p传输块，实现更细粒度传输，是否可行


