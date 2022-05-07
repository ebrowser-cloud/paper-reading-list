By Jiabin Chen


---

# 使用Medes对无服务器计算进行内存消重

[Memory Deduplication for Serverless Computing with Medes](https://doi.org/10.1145/3492321.3524272)
---

* 文章发表在 EuroSys22 上。
* 现有无服务器计算平台通常在节点中保持 warm 沙盒来减少冷启动的次数，这是一种以资源换性能的权衡。
* 运行在无服务器平台上的 warm 沙盒的内存占用中有很高比例的重复，文章基于这个事实提出了一个无服务器框架 Medes，文章利用这些冗余块来引入一种新的沙盒状态，称为 dedup 状态，它比 warm 状态的内存效率更高，比 cold 状态的恢复速度更快。
* 实验表明，Medes 可以在端到端延迟方面提供高达 1～2.75 倍的改进，Medes 的好处在内存压力情况下得到了增强，在这种情况下，Medes 可以提供高达 3.8 倍的端到端延迟改善。Medes 通过将冷启动次数减少 10%～50% 实现了这一目标。

## 论文作者
* [Divyanshu Saxena](https://divyanshusaxena.github.io)：德克萨斯大学奥斯汀分校计算机科学系的研究生；UT Networked Systems ([UTNS](https://utns.cs.utexas.edu/)) lab.
* Tao Ji，Arjun Singhvi：德克萨斯大学奥斯汀分校。
* [Junaid Khalid](https://pages.cs.wisc.edu/~junaid/)：威斯康星大学麦迪逊分校博士毕业，现在谷歌从事分布式存储系统的工作；研究涉及网络和系统，博士论文主要关注虚拟网络功能。
* [Aditya Akella](https://www.cs.utexas.edu/~akella/)：德克萨斯大学奥斯汀分校系计算机教授；领导 UT Networked Systems ([UTNS](https://utns.cs.utexas.edu/)) lab；致力于构建软件系统，以提高现代计算基础设施和应用程序的性能、效率和正确性。

## 论文动机
### 问题引入：

* 可能有多个 warm 沙盒对应集群中的相同函数，存在相同函数间的内存冗余。
* 不同的函数可能会使用相同的运行时和库，存在不同函数之间的内存冗余。
* 下图图a和图b展示了在不同块粒度大小下相同函数之间的内存冗余度（没使用地址布局随机化和使用了地址布局随机化），图c展示了在 64B 块粒度大小下不同函数之间的内存冗余度，两者都显示出了较高的冗余度。

![image-20220507155759632](https://raw.githubusercontent.com/JBinin/Image-hosting/master/uPic/image-20220507155759632.png)

* 文章进一步根据 Azure 发布的无服务器 trace 中的各种到达模式进行估计，发现相对于目前不利用冗余的最先进平台，如果去除冗余可以节省多达 30% 的内存。

## Medes：

* 文章引入一个新的沙盒状态：dedup 状态，该状态介于 warm 状态和 cold 状态之间，它消除了冗余的内存块。从 dedup 状态启动时，dedup 要先恢复为 warm 状态，它将从副本的位置读取被消除掉的冗余数据块。换句话说，文章希望能重用 warm 沙盒中的那些块，文章将那些块称为可重用沙箱块（RSC）。
* dedup 状态一方面比 warm 状态节省内存，允许同时在内存中保留更多沙盒，另一方面比 cold 状态缩短启动时间，减少单个函数请求的延迟。

#### Medes 整体架构

![image-20220507165854011](https://cdn.jsdelivr.net/gh/Jbinin/Image-hosting@master/uPic/image-20220507165854011.png)

* 控制器包括四个组件：
  * 客户端接口：负责接受请求和返回结果
  * 调度器：存储系统级别的状态（各节点的资源利用率和各节点容器状态），将请求调度到某个沙盒，决定沙盒的状态转换。
  * RSC 指纹库：它是一个哈希表，其中包含 RSCs 的哈希值及其在集群中的对应位置，用于重复数据删除。
  * 策略模块：存储策略参数（例如延迟和内存限制）。
* 每个节点包括两个组件：
  * 守护进程：根据控制器的指令操作本地沙箱并更新节点状态。
  * 重复数据删除代理：为本地沙箱执行重复数据删除的重复数据删除，将请求分配给本地沙箱时，从去重状态恢复本地沙箱。

#### 沙箱的生命周期

![image-20220507195836170](https://cdn.jsdelivr.net/gh/Jbinin/Image-hosting@master/uPic/image-20220507195836170.png)

* 如上图所示，图a和图b分别表示现有无服务器平台和 Medes 的沙盒的生命周期状态机。

#### 去重和恢复

* 去重操作包括两个步骤，冗余识别和冗余去除。本地的重复数据删除代理通过使用 CRIU 工具获取 warm 沙箱的内存状态。重复数据消除代理通过与控制器交互来识别重复的内存块。在识别出冗余后，重复数据消除代理以整个页面的粒度消除（删除）重复的内存块，并基于其 RSC 对应的基页，为被消除的页面计算 patch。
* 恢复操作将一个 dedup 沙箱转换为一个 warm 沙箱。此操作包括读取与此沙箱对应的 patch 的基本页面，然后重建原始页面并重新创建内存检查点，最后，沙箱通过从该检查点恢复转换到 warm 状态。

![image-20220507201504648](https://cdn.jsdelivr.net/gh/Jbinin/Image-hosting@master/uPic/image-20220507201504648.png)

## 实验

实验设置：
* CloudLab 上的 20 节点集群，所有节点都有 64GB 内存和 10Gbps NIC，其中一个节点充当控制器，控制者上没有沙箱，其余的节点都可以通过 RDMA 网络访问

baseline 设置：
* Fixed keep-alive：固定的十分钟持续时间作为 keep-alive 时间
* Adaptive keep-alive：keep-alive 时间根据请求达到间隔的历史信息进行选择

负载说明：

* 请求到达模式：文章使用 Azure Function trace 中的到达模式，文章中将请求速率放大了 5 倍
* 函数：文章使用 Function Bench suite 中的所有十个函数

#### 实验项目1

![image-20220507205957859](https://cdn.jsdelivr.net/gh/Jbinin/Image-hosting@master/uPic/image-20220507205957859.png)

* 上图显示了使用 Medes 对比 baseline 的的应用程序性能优势。
* Medes 改进的主要来源是减少每个函数的冷启动次数，与 Fixed keep-alive 和 Adaptive keep-alive 相比，Medes 可在不同应用中减少多达 1.85 倍和 6.2 倍的冷启动次数。冷启动次数显著减少，这对尾部延迟有好处。冷启动次数的减少是因为平均而言，Medes 重复数据消除约占所有沙箱的 39%。与 Fixed keep-alive 和 Adaptive keep-alive 相比，这种重复数据消除有助于 Medes 在内存中保留 7.74% 和 37.7% 的沙盒。

#### 实验项目2

![image-20220507210837374](https://cdn.jsdelivr.net/gh/Jbinin/Image-hosting@master/uPic/image-20220507210837374.png)

* 上图显示了在不同内存压力下的冷启动次数，Medes 相对于 baseline 的好处随着内存压力的增加而增加，与 Fixed keep-alive 相比，在无内存压力的情况下，冷启动的数量提高了 22% 在内存压力情况下分别为 37% 和 40.67%。

## 文章相关工作：

* Faascache 提出了一个工作负载感知的 keep-alive 策略，该策略考虑了函数的特征如函数大小和初始化开销等
* Seuss 使用 unikernels 来降低开销
* Photon 在同一个沙箱上运行一个函数的多个实例
* Nightcore 在同一个沙箱中运行链式的多个函数
* 函数快照（Catalyzer、REAP）通过从快照中恢复沙箱（存储在磁盘上或共享在内存中）来减少启动开销

