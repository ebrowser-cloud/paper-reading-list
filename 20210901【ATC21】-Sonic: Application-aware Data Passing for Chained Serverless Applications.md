By Jiabin Chen

# Sonic: 链式 serverless 应用的应用感知数据传输

[Sonic: Application-aware Data Passing for Chained Serverless Applications](https://www.usenix.org/conference/atc21/presentation/mahgoub)
---

* 该论文是2021年发表在ATC上的论文，其目标是优化 serverless 平台上数据分析应用所需的成本和性能。
* 数据分析应用通常由链接在一起的多个函数组成，它们组成一个工作流，文章通过优化链式函数之间的数据传输来实现文章目标。
* 文章对比了以往的三种方法：VM-Storage 、Direct-Passing 和 Remote-Storage，没有一种方法可以在所有场景下都取得最优的效果，所以文章使用了一种混合的方法 SONIC，通过为工作流中的任意两个 serverless 函数之间选择最优的数据传输方法（上述三种方法择其一）和优化函数的放置策略，来优化应用的成本和性能。

## 论文作者

* [Ashraf Mahgoub](https://www.linkedin.com/in/ashraf-mahgoub-30972211a)：普渡大学研究助理；博士期间主要研究设计和实现自动调整系统，博士前半段时期专注于为 NoSQL 数据存储提供自动调整功能，博士后半段时期专注于最小化通信延迟和选择最佳容器大小来优化无服务器工作流的性能和成本。
* [Karthick Shankar](https://www.karthickshankar.com)：本科毕业于普渡大学，现为卡内基梅隆大学研究生；主要兴趣是分布式系统和 web 开发。
* [Subrata Mitra](https://research.adobe.com/person/subrata-mitra/)：Adobe 的研究员，博士毕业于普渡大学；研究方向主要是分布式系统、机器学习、高性能计算和移动系统。
* [Ana Klimovic](https://anakli.inf.ethz.ch)：苏黎世联邦理工学院助理教授；Efficient Architectures and Systems Lab；研究方向主要操作系统、计算机体系结构和它们的交叉学科机器学习。
* [Somali Chaterji](https://www.schaterji.io)：普渡大学农业和生物工程系助理教授；生物工程和计算机科学双博士后。
* [Saurabh Bagchi](https://engineering.purdue.edu/~sbagchi/)：普渡大学电气和计算机工程以及计算机科学系教授；主要研究方向是可靠和安全的计算。

## 论文动机

### 问题引入：

* Serverless 工作流由一系列执行的 stage 组成，可以通过 DAG（有向无环图）来表示，DAG 的节点表示 serverless 函数（或者称 lambda），DAG 的边表示有依赖关系的两个函数，下图就是一个视频分析的 DAG 。

<img src="https://cdn.jsdelivr.net/gh/JBinin/Image-hosting@master/20210805/4AD98721-169F-4744-8854-718A960BC740.2nq1340womw.png" alt="4AD98721-169F-4744-8854-718A960BC740" style="zoom:40%;" />

* 由于 serverless 的易用性和按需付费的特性，在 serverless 环境上部署的数据分析应用变得越来越多，而在 serverless 函数之间交换数据则成为了 serverless 工作流中的一个重要挑战。

* 一些云服务商也引进了一些 serverless workflow 服务：

  * AWS StepFunctions
  * Azure Durable Functions
  * Google Cloud Composer

  


### 论文所研究的问题有哪些挑战：

* 在函数间直接交换数据是一个很大的挑战：

  * serverless 平台上的函数不对用户暴露 ip 和相应的端口号。
  * serverless 平台不保证发数据和接收数据两个函数执行时间上的重叠。

* 目前商业实践中的解决方案是 Remote-Storage，即通过远程存储来实现数据交换，但这带来了很大的性能开销。

  <img src="https://cdn.jsdelivr.net/gh/JBinin/Image-hosting@master/20210805/image.20z7bsxjbwl.png" alt="image" style="zoom:25%;" />

* 目前也有一些对 Remote-Storage 的优化方法（但仍需要将数据在网络上传输数次）：

  * 优化对象存储，用基于内存的存储替换基于磁盘的对象存储，但是基于内存的存储更贵。
  * 联合不同的存储介质（DRAM，SSD，NVMe）来适应不同的应用需求。


### 三种解决方案：

<img src="https://cdn.jsdelivr.net/gh/JBinin/Image-hosting@master/20210805/image.5fjwagfx7shs.png" style="zoom:70%;" />

1. VM-Storage：
   * 将发送和接收的函数放在同一个 VM，通过 VM 的本地存储来进行函数之间的数据交换。
   * 对函数的放置有要求，要把发送函数和接收函数放在同一个 VM。
   * 如果 VM 内存不够容纳所有的函数，就会按顺序或者按批次执行这些函数。
   * 在高负载下的资源管理器而言，资源管理器要避免热点来实现负载均衡，而这个策略与之相悖。
   * 不需要额外的数据拷贝
2. Direct-Passing：
   * 发送数据的函数将数据存入 VM 的存储，将 IP 地址和文件路径传给元数据服务器，当有接收函数需要这些数据时，就可以通过 IP 地址和文件路径直接从该 VM 获取数据，过程中不要求两个函数执行时间的重叠。
   * 无函数放置上的要求。
   * 引入网络传输的开销。
   * 需要一次拷贝。
3. Remote-Storage：
   * 通过远程存储来实现数据交换，发送方的函数将数据先存储到远程存储上，接收方函数再统一从远程存储拉取数据。
   * 无函数放置上的要求。
   * 鉴于优秀的存储带宽，随着 fanout 的增加，仍然具有一致的性能。
   * 需要两次拷贝。

### 为什么不采用这些方案：

* 文章在不同 fanout 下跑 LightGBM 应用，结果如下图，发现：
  * fanout 较小时，采用 VM-Storage 的方案可以实现最优的端到端延迟。
  * 随着 fanout 的增大，VM-Storage 需要按序执行接收函数才能保证发送和接收数据的函数都在一个 VM，此时 Direct-Passing 方案的效果最优。
  * fanout 再增加，采用 Direct-Passing 方案时，发送数据的 VM 将面临巨大的带宽压力，此时 Remote-Storage 的效果最优。
* 所以没有一种单一的方法能在任何条件下都达到最好。

<img src="https://cdn.jsdelivr.net/gh/JBinin/Image-hosting@master/20210805/4BB032B7-273E-4C3F-8793-8E8B3C8E1734.46gf7i310o00.png" alt="4BB032B7-273E-4C3F-8793-8E8B3C8E1734" style="zoom:50%;" />

## 论文贡献

1. 分析了 serverless 工作流中三种不同的中间数据传输方法，并表明没有一种方法在延迟和成本方面的所有条件下都占优势。
2. 提出了 SONIC，可以自动为工作流中的任意两个 serverless 函数之间选择数据传输方法，使得端到端时延和成本方面最优，并可以自动根据参数调整。
3. 实验测试了在不同输入数据大小和不同其他参数的情况下 SONIC 的性能，并与其他基准方法作对比，在所测试的三类常见的 serverless 应用下，SONIC 总是能得到比其他方法更好的效果。

## SONIC：

### SONIC 描述：

* 假设：
  * 用户多次运行应用，但每次的输入数据可能不同。
* DAG 参数：
  * 每个 lambda 的内存占用。
  * 每个 lambda 的执行时间（冷启动和热启动）。
  * 每对需要传输数据的 lambda 之间传输数据的大小。
  * 每个 stage 的 fanout degree。
* SONIC 方案简述：对于每一个输入，预测其 DAG 参数，根据预测得到的 DAG 参数，选择合适的函数放置策略以及数据传输方法。

* SONIC 方案流程：
  * SONIC 根据多项式回归的方法得到输入大小和 DAG 参数之间的关系，在收敛前默认使用 Remote-Storage 来进行数据传输。
  * SONIC 利用输入的大小来预测 DAG 的关键参数。
  * SONIC 根据预测得到的 DAG 参数来选择 VM 的规格和每对 stage 之间的数据传输方法：
    * 根据 VM 的计算和存储容量，以及预测的 lambda 的存储占用，用递归的 scoring 算法来产生 lambda 在每个 stage 的所有可能的放置方式。
    * 在 VM 资源限制下（主要是 CPU 和存储资源），遍历 lambda 所有可能的放置方式，并为每两个 stage 之间选择最好的数据传输方式。
    * 构造一个包含所有生成的方案的动态 programming table。
    * 利用 Viterbi 算法在动态 programming table 找出一条最佳路径。
* 资源管理器根据 SONIC 的建议，决定怎么在实际的物理机器上分配函数，并把结果返回给 SONIC（SONIC 仅仅是建议）。

![BFAA5870-70B6-4B31-9993-246ED794D444](https://cdn.jsdelivr.net/gh/JBinin/Image-hosting@master/20210805/BFAA5870-70B6-4B31-9993-246ED794D444.5lzs2h26m64g.png)

### 对比以往方案，SONIC 有什么优势：

* 自动为用户选择 VM 和内存配置。
* 根据应用的 DAG 参数，自动选择最合适的函数放置策略和中间数据传输方法，得到更好的 Perf/$ 性能。

## 实验

性能指标：

* Perf/$：1/Latency(sec) * 1/Price($)
* Latency(sec)

baseline 设置：

1. OpenLambda + S3：Remote-Storage 的实现，采用 OpenLambda 实现。
2. OpenLambda + Pocket（DRAM） ：Remote-Storage 的实现，但用了更贵的 DRAM 作为存储，采用 OpenLambda 实现。
3. Sand：VM-Storage 的实现。
4. AWS-lambda +S3 / ElasticCache-Redis：Remote-Storage 的实现，采用了商用的 AWS-lambda 平台实现。
5. Oracle SONIC：完全精确估计 DAG 的参数，没有传输数据的延迟，尽管不切实际，但是这是三种方法的性能上限。

运行的测试应用介绍：

1. Video analytics：
   1. 一个 lambda 负责切分视频成等长时间的小块（文中例子使用10s），第二个 lambda 对每个块进行关键帧提取，第三个 lambda 使用一个预先需脸的深度学习模型（MXNET）对提取的帧进行分类，它输出在1000个类的概率分布 ，最后结果写入长期存储中。
   2. 3min长的视频，15MB，fanout degree 为19。
2. LightBGM：
   1. 首先一个 lambda 读取训练数据并执行 PCM，然后多个 lambda 对决策树进行并行训练（lambda 数量由用户定义），每个 lambda 选择90%用作训练，10%用作验证，最后用一个 lambda 收集合并所有的训练模型并在提供的测试集上进行测试。
   2. NIST和MNIST用作输入。 
   3. 输入数据200MB，fanout degree 为6，产生6棵树的随机森林。
3. MapReduce Sort：
   1. 首先，K 个并行的 lambda 从远程存储获取数据并产生临时文件（lambda 就是 mapper，K 是用户定义的参数），然后 K 个并行的 lambda 对临时文件进行排序并将排序输出写入到存储。
   2. 输入数据是1.5GB，fanout degree 为30。

#### 实验项目1

E2E Evaluation：

* 延迟计算时包含 11ms 的 DAG 参数推断和 120ms 的 Viterbi 优化时间。
* 文章对35个作业执行在线评测和训练，从而使所有应用程序的平均绝对百分比误差（MAPE）都很低（≤ 15%）。
* baseline 的内存配置：
  * memory-sized：仅仅配置 lambda 运行所需的内存，通过在配置最大内存（3GB for AWS lambda）的情况下运行一次 DAG 得到 lambda    运行所需的内存。
  * latency-optimized：只要延迟减少就给增加内存分配，使用运行延迟最小时的内存配置。

虽然 AWS-lambda 允许用户自己配置内存大小，这通常并不能保证得到最好的 Perf/$ 或者时延。

* 视频分析应用在 memory-sized 配置下，SONIC 的 Perf/$ 是 AWS-lambda + S3 的442%，是 AWS-lambda + ElasticCache-Redis 的12.9倍。
* 视频分析应用在 latency-optimized 配置下，AWS-lambda 的 Perf/$ 明显提升，SONIC 的对 AWS-lambda + S3 和 AWS-lambda +  ElasticCache-Redis 的增益分别降到了76%和36%，同时时延上也有相似的效果。
* 上述现象原因是 AWS-lambda 分配的资源（CPU和带宽等）与选择的内存大小成比例，随着分配的内存上升，时延降低并且成本升高，而时延降低的速度比成本升高的速度更快。但是，超过一定的点后，时延无法再降低了，因此 Perf/$ 开始下降。

![image](https://cdn.jsdelivr.net/gh/JBinin/Image-hosting@master/20210805/image.4eo7hzwb92tc.png)

#### 实验项目2

Scalability：

实验设置：52台 m2.large VM，每台 VM 都有两个计算核心

* 比较了 SONIC 与 SAND 和 OpenLambda + S3 在上述三种应用负载的混合负载下的表现，其中混合负载中三种应用的请求数量是均分的。

* SNAD 由于要求把所有函数都放在同一个 VM 上，所以并不能利用所有计算核心，导致了极大的时延。

* OpenLambda 使用了 Remote-Storage 方案来进行函数间数据传输，并不要求函数放在同一个 VM，所以可以利用所有的计算资源。

* SONIC 在三种不同并行请求数量的情况下，比起另外两种方法，具有一致的增益，SONIC 可以随着并行请求数的增加而相应地扩展。

  <img src="https://cdn.jsdelivr.net/gh/JBinin/Image-hosting@master/20210805/C2226758-8956-415D-80BC-9CB5ABF8AD33.2q25b70rzh1c.png" alt="C2226758-8956-415D-80BC-9CB5ABF8AD33" style="zoom:80%;" />

#### 实验项目3

输入内容的敏感性：

文章仅考虑输入大小和 DAG 参数之间的关系，而忽略了输入内容的影响。

对于视频分析应用，文章选取了油管上不同类别的视频，实验对比了以下几种训练数据下，测试一个 396秒的运动类别视频的效果：

* SONIC Same Category：用60个运动类别视频作训练。此时性能最好。

* SONIC All：用60个不同类别视频作训练，一共5个类别，每个类别12个视频。相对于 SONIC Same Category，性能降低了8%。

* SONIC Unseen Category：用60个非运动类视频作训练，其中视频平均比特率比运动类视频低大约25%。与 SONIC All 相比，性能降低了19%，这是由于 SONIC 错误预测了 fanout degree（40%）和 Split_Video 和 Extract_Frame 传输的中间数据的大小（21%）。

  <img src="https://cdn.jsdelivr.net/gh/JBinin/Image-hosting@master/20210805/image.15oibonjnj7k.png" alt="image" style="zoom:80%;" />

* 训练数据与测试数据特征差异越大，性能下降就越大，但是仍然比 OpenLambda + S3 和 SAND 要好。

* 文章提到的一种解决方案：将不同特征的 job 分到不同的集群中，为每个集群分别训练不同的预测模型。

## 其他

文章相关工作：

* Pocket and Locus：多层远程存储，综合各类存储获得良好的综合性能/时延效果。

* 使用 NAT 技术来直接通信，ExCamera 用一台 rendezvous 服务器和一群长时间存活的临时 workers 来实现直接通信，需要两个函数在传输数据过程中保持运行。
* gg 是一个突发并行应用程序框架，支持多个中间数据存储引擎，包括 lambda 之间的直接通信，用户可以从中选择。

* 集群配置的自动调整系统，但是这些系统依赖于黑盒式的机器学习优化，并依赖于上百次线下的评测，以此获取精确的性能模型。

* Cloudburst 提出在每个拥有函数的 VM 上使用 cache 来以快速检索远程键值存储中经常访问的数据。

组会上思考和问题记录：

* 为什么选择维特比算法，相对与启发式算法有什么优势？
  * 选择 Viterbi 算法是因为它可以保证找到真正的最大后验 (MAP) 解决方案 ，这与基于启发式的搜索算法（例如遗传算法或模拟退火）不同。

* 细粒度传输方式选择，两个 stage 之间也采用混合的数据传输方法是否可行？