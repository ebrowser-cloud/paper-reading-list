By AoDong Chen

# Gillis:通过自动化模型切分在无服务器函数中运行大型神经网络

Gillis: Serving Large Neural Networks in Serverless Functions with Automatic Model Partitioning([gillis-icdcs21.pdf (ust.hk)](https://www.cse.ust.hk/~weiwa/papers/gillis-icdcs21.pdf))

本文介绍了Gillis，一个基于无服务器架构的模型服务系统，它可以在多个函数之间自动并行化大型DNN模型，以更快地进行推理并减少每个函数的内存占用。论文提出了两种切分模型的算法，一种可以满足最低延时服务，另一种可以在满足SLO的情况下最小化成本。实验评估表明，Gillis加速了模型推理，支持非常大的深度学习模型，并且能在满足指定延迟SLO的情况下显著节省成本。

## 论文作者

Minchen Yu：香港科技大学计算机科学与工程系的博士研究生。研究方向主要是是分布式系统和云计算，包括机器学习系统、分布式存储和无服务器计算。

Zhifeng Jiang：香港科技大学计算机科学与工程系博士研究生。研究方向主要是在大规模数据分析背景下的网络系统和机器学习的交叉，包括构建分布式数据分析系统和使用学习技术优化系统。

Hok Chun Ng：香港科技大学学生。

## 论文动机

推理服务通常是由具有高性能CPU和GPU加速器的虚拟机提供。基于虚拟机的推理服务为运行稳定的工作负载提供性价比高的性能。然而，它在处理动态查询时效率很低。这是因为虚拟机的启动延迟很长(例如，几分钟)，通常需要很高的过度供应值来适应意外的负载峰值，例如，SageMaker建议将过度供应系数设置为2。与VM相比，无服务器函数的启动延迟要短得多(例如，热启动时10毫秒)，并且可以快速扩展到大量实例，以便在短时间内适应激增的推理请求。另一方面，无服务器函数并不适合为稳定的工作负载提供服务，因为它们每个请求的价格很高。因此，一个很有吸引力的方法是用无服务器函数来增强vm，也就是说，使用vm来处理稳定的推理请求，同时使用无服务器函数来处理瞬时负载突发情况。但深度学习社区正在构建越来越大的深度神经网络，以实现对各种实际应用的更高预测精度。这一趋势，由硬件加速器的进步和训练数据的快速增长所驱动，预计将在未来会继续下去。而当前的无服务器产品只支持资源（CPU核、内存、带宽等）受限的功能。使用资源受限的函数来服务于非常大的模型是有问题的——它要么由于有限的CPU核而导致非常长的推断延迟，要么在模型太大而无法放入函数的内存时变得不可用。因此需要设法解决大模型的问题，一般有两种方法：（1）模型压缩和（2）模型切分。模型压缩会损失精度而且需要耗费开发人员大量精力研究网络，对于本篇论文遇到的问题不可取。而模型切分不减少精度，同时还能降低内存占用，所以文章选择使用模型切分方法来处理大模型问题。

## 论文贡献

在本文中，我们提出Gillis，一个基于无服务器架构的模型服务系统，自动探索在多个函数间并行执行DNN推理的方案，能够加速推理并且减少每个函数的内存占用。主要贡献有以下内容：

- 设计了一个遵循 fork-join 的计算模型——在接收到推理请求时，调用一个master函数来启动多个worker函数，每个worker函数承载模型的一个部分。master通过无状态连接(函数调用)与worker交互。
- 提出了两种应用于不同场景的粗粒度(此处即不是对每一层的网络进行切分)模型切分算法：
  - 最小化推理延时场景：
  - 在满足用户指定的SLO的条件下最小化推理服务的成本：

### Gillis工作流程  

![rl-overview](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20211017/rl-overview.u4086nsw140.jpg)  

图3给出了Gillis的工作流程概述。Gillis从运行时分析开始。对于每种类型的DNN层，Gillis在单个函数的环境中运行并记录其执行时间。Gillis还记录了函数之间的通信延迟。基于分析结果，Gillis构建了一个性能模型，并使用它来预测模型在各种并行方案下的执行时间。该模型分为两部分（1）回归模型预测运行时间。（2）n阶统计量预测函数通信延迟。然后进入模型切分阶段，把接收一个服务模型作为输入，根据用户选择的场景选用两个算法的其中之一，算法过程中利用性能模型来找到最优的模型切分与并行方案。最后进入部署阶段，模型的各部分被打包到函数中，并部署在无服务器平台上。Gillis支持通过向无服务器平台发送并发ping来周期性地预热函数。由于函数实例在很长时间内都是活动的，所以预热成本可以通过提供大量推理查询来分摊，因此可以忽略不计。

### Fork-Join计算模型  
![fork-join-model](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20211017/fork-join-model.57rgz34pwq80.jpg)  

图4展示了Gillis用于协调多个无服务器函数的fork-join模型。主函数在接收到推理查询时被触发运行。按照被计算出来的切分方案，master异步调用多个worker函数。每个worker计算所服务模型的一个分区，将结果返回给master，并结束它的执行。如果有足够的内存，master还可以帮助计算一个分区，这可以导致更少的worker和更低的成本。master将所有worker返回的结果组装成一个完整的张量，还可以启动更多的worker来继续并行化模型执行。fork-join过程可能需要多轮才能完成，最终的推断结果由master给出。

### 粗粒度的并行化

Gillis执行粗粒度并行化:它将多个连续的层合并为一个组，并跨无服务器函数并行化每个组。因此，一组中的所有层都在函数内进行局部计算——只需要传输第一层的输入张量和该组中最后一层的输出，这大大降低了通信开销。分组有两步：

- （1）合并连续的元素级的层(如ReLU)和分支模块(如果有的话)，缩小寻找最优分组的搜索空间。
- （2）对每个组进行切分从而可以并行执行，一个组的切分必须满足每个部分的计算具有独立性。

### 延时最优的模型切分算法

目标函数与约束条件如下：  
![objective-function](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20211017/objective-function.6ttrhyscjd80.png)  

我们的目标是最小化 （1），约束条件为（2）。

使用动态规划算法解决以上数学问题：  

![dp](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20211017/dp.7cft7l7plrs0.jpg)  

动态规划算法需要知道 t(l(k,j)，b) ，在master上用内存预算b最优化并行一个组 l(k,j) 的的时间。该时间可以通过一下算法得到：  
![algorithm1](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20211017/algorithm1.6vro8lrbyxg0.jpg)

### 满足SLO情况下成本最优的模型切分算法

将问题建模可得：  
![CsG](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20211017/CsG.2wgujyivp3i0.jpg)
![cost-formulation](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20211017/cost-formulation.3jdkzvbgl140.jpg)

即在最小化成本的同时，平均推理延时需要小于用户指定的SLO——$T_{max}$ 。

强化学习是非常适合解决我们的问题,文章将切分策略进行编码并输入到一个神经网络——使用大量的模拟实验训练它:RL的Agent做出切分决策,使用性能模型预测成本和延迟，然后迭代改进策略。  

![rl-overview](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20211017/rl-overview.u4086nsw140.jpg)

作者设计了一个分层的RL模型来学习最优划分策略。图8给出了解决方案的概述。论文的RL模型有两个agent, 分别是partitioner和placer。每个agent都是一个两层神经网络。partitioner将DNN层作为输入，并决定如何将这些层融合到组中，以及每组如何并行化。给定层组，placer决定如何在master和workers上放置分区，从而制定出详细的功能执行计划。在模拟实验中，可以利用性能模型获得执行计划的延迟和成本，然后用策略梯度一同训练partitioner和placer。  
![reward-function](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20211017/reward-function.3bkyl82cs0i0.jpg)

作者定义了同时考虑推理成本和延迟的奖励函数。直觉上，只有当延迟SLO得到满足时，才能获得积极的奖励，同时较短的计费持续时间也会带来较高的奖励。

## 实验

基准模型：作者使用四类常用的dnn作为基准模型:VGG、ResNet、Wide ResNet和Multi-layer recurrent neural networks (RNNs)。

无服务器平台：作者使用AWS Lambda函数和Google Cloud Functions各自的最大可能实例内存(即分别为3GB和4GB)来评估Gillis。还在KNIX上评估Gillis，这是一个开源的无服务器平台，能够比Lambda更快地进行函数通信。作者将KNIX部署在一个EC2 c5.12xlarge实例上，并配置每个KNIX函数的资源，以匹配Lambda实例的性能。

### 实验一	Gillis在延迟最优模式

#### Baselines:

- 默认服务(Default)使用单个函数来服务一个模型。由于内存限制，它只适用于中等大小的模型。
- 管道执行(Pipeline)将大型模型的层划分为多个小分区，并将它们存储在S3中。然后，它启动一个函数，在管道中依次执行这些分区，每次执行一个分区。

#### 实验结果：  

![fig9](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20211017/fig9.5vklmh25ihc0.jpg)

如图，在多个函数之间并行化CNN模型明显比在两个平台上使用单个函数(默认)提供服务具有更快的推理速度。与谷歌云功能相比，Lambda可以为Gillis提供更多延迟改进，因为它的每个实例的资源更少。  

![fig10](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20211017/fig10.4nv592if8uu0.jpg)

由于KNIX的低延迟通信，与Lambda相比，Gillis可以从并行化中获得更多好处。    

![fig11](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20211017/fig11.553ym0nchjk0.jpg)

由于网络带宽有限，pipeline的通信开销成为严重的瓶颈。相比之下，Gillis无需从远程仓库传输权张量。而且，并行执行使其比流水线的顺序执行快两倍（计算时间）。  

![fig12](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20211017/fig12.5uk630iwokc0.jpg)

由于RNN层不能并行化，Gillis在单个函数能够处理的小模型中没有显示出比Default更大的优势。然而，单个函数只能支持多达9层的RNN模型。Gillis对模型大小没有这样的限制，可以线性地扩展到大的rnn。

### 实验二	Gillis在满足SLO条件下成本最优模式

#### Baselines：

- 蛮力搜索(BF)枚举所有可能的符合SLO的并行化策略，并找到成本最小的策略。考虑到其计算难度，我们只将其应用于VGG-11，这是我们基准测试中最小的模型，仍然需要超过24小时才能完成。
- 贝叶斯优化(BO)将推理代价建模为高斯过程，并通过抽样迭代优化模型以寻找最优策略。
 
#### 实验结果：  

![fig13](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20211017/fig13.1tifbztdbiv4.jpg)

在最优化并行VGG-11同时最小化成本的情况下，与BF相比，Gillis (SA)学习了相同的分区策略(图13a)，但更小的计算开销获得了相似的延迟和成本。与BO算法相比，Gillis的RL算法在模拟实验中使用性能模型来指导最优划分的搜索，结果更好。事实上，BO在一些情况下无法满足延迟SLOs，特别是对于像WRN-50-4这样的复杂模型，或者像$T_{amx}$ = 500ms的严格延迟要求。相比之下，Gillis (SA)总能在满足SLOs的同时比SO更节省成本。

## 总结

本文的性能模型的源代码没有给出，因此关于作者如何测量模型每一层的计算时间存在疑问， 此外，作者使用的模型总计算时间为模型的各层计算时间之和，是否确实如此？而对于文中模型分支合并、连续层合并的底层操作并不十分清晰，是以后需要仔细学习了解的部分。

