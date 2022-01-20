By AoDong Chen
# 用Pesto实现DNN中operator的最佳放置和调度
[Towards Optimal Placement and Scheduling of DNN Operations with Pesto]([extension://oikmahiipjniocckomdccmplodldodja/pdf-viewer/web/viewer.html?file=https%3A%2F%2Fwww3.cs.stonybrook.edu%2F~anshul%2Fmiddleware21_pesto.pdf#=&zoom=180](https://dl.acm.org/doi/pdf/10.1145/3464298.3476132))

在本文中，作者设计并实现了Pesto，这是一种快速且接近最优的模型放置技术，用于在多个GPU上自动分割任意的DNN。Pesto的关键思想是在细粒度的operator层面上同时优化模型放置和调度，最大限度地减少GPU间的通信，同时最大限度地增加模型在GPU间的并行度。通过将该问题表述为一个整数线性方程，Pesto可以提供最佳的放置和调度。作者在TensorFlow中实现了Pesto，并表明Pesto与最先进的方法相比，在几个大型的DNN模型中，可以将模型的训练时间降低31%。  

## 论文作者
Ubaid Ullah Hafeez：石溪大学PACE实验室的博士生。研究方向主要是分布式机器学习系统。  

Qian Li：暂无信息。  

Anshul Gandhi：石溪大学计算机科学系的副教授。研究方向主要是系统和性能建模。  

Christos Kozyrakis：石溪大学应用数学与统计学系的助理教授。研究方向主要是复杂分布式系统。

## 论文动机
- 模型越来越大，一个GPU的显存放不下，需要模型并行
- 模型并行：模型（或DNN图）被划分到多个GPU中，这样每个GPU只评估模型的一个子集。
- 模型并行的关键问题是怎么切分模型？切的好可以降低通信开销
- 之前的工作，要么是人工，要么是基于自动学习的技术但是耗时长，或者基于算法的切分技术，时间短但是效果不好。而且它们都没有考虑通信拥塞和内存限制等因素。
 

## 论文贡献
本篇论文介绍了Pesto，一种快速且接近最优的模型放置技术，用于在多个GPU上自动切分任意DNN。主要贡献有以下内容：
- 作者提出了一种接近最优的、同时优化DNN的operator放置和调度的算法，该方法可以应用到任何一种DNN上。
- 作者设计了一个在线模型并行化框架Pesto，它可以最大限度地减少模型在多个GPU上的训练时间。
- 作者在TensorFlow上实现了Pesto，并表明与Expert策略和Baechi相比，Pesto可以将DNN模型训练时间平均减少15.5%和23.4%。 
- 与现有的DNN放置技术不同，作者在多个巨型模型上对Pesto进行了广泛的评估，每个模型都有不同的变体


### 主要工作概述
- 首先预估每个operation的计算时间和不同设备间的通信时间
- 为了考虑通信开销，作者使用一个**图增强技术**——给原DAG图加节点
- 构建一个**整数线性方程**，将以上预测的值作为方程输入，解出这个方程，然后就能得到切分方案和调度顺序
- 因为DAG图节点太多，解方程太耗时。作者设计出一个**图粗化技术**——去边合并顶点。


### 数学建模
![model](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20220119/model.26ix1dswu4ps.webp)
Cmax整个DAG图的运行时间；Si为算子i的开始时间；Ci为算子i的完成时间；pi为算子i通信时间或者计算时间；Zk为是否需要通信；Xi=0或1代表算子i放在GPU0还是GPU1上；内存限制没说清楚，应该是通过每个算子的输入输出张量大小之和估计算子占用的内存，保证他不超过显存大小。CPU的约束没有Ml那一项，因为这里的实验只有一个CPU。CPU-GPU间的通信限制同理可得。

![non-overlapping](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20220119/non-overlapping.43aw631k2us0.webp)
保证GPU每次只调度一个算子。假设Xi = Xj = 1,代表i和
j都放置gpu-1上。

![congestion-constraint](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20220119/congestion-constraint.37gkq1dnf7g0.webp)
保证每次总线上只传递一对算子的数据。假设a=1,b=0,
c=1,b=0。

### 图增强技术
![graph-enhancement](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20220119/graph-enhancement.7gbflr0vppg0.webp)
然后通过判断i、j是否在同一个设备上来设置k的计算时间，即等价于考虑了通信时间，并且放在同一设备上通过条件约束使其只能按顺序执行
   
### 图粗化技术
- 定理一：如果边(u,v)是图中u到v的唯一路径，则可以将u，v合并。（合并两个成对的顶点而不产生循环的充分必要条件）
- 推论：如果v前序节点个数等于1或者u的后序节点个数等于1，那么(u,v)是u到v的唯一路径
以上规则每次只能去一条边。

- 上面处理图的速度太慢了，又提出要给批量合并边的规则，也给出了定理。然后根据规则设计粗化算法，可在有限时间内完成任务。

合并后的图中一个顶点代表了原图中的多个顶点，当方程解出来要把合并后的点放在gpu-0上时，则代表原来的多个顶点都放在gpu-0上，然后这些顶点按照DAG顺序执行。


## 实验
### 基础实验设置
作者在一台装有Intel Xeon Silver 4116处理器和两颗英伟达Tesla V100 SXM2 16GB GPU的服务器上进行实验。每个GPU通过专用的PCIe连接到CPU上。GPU之间直接通过NVlink进行通信。作者使用TensorFlow r1.15作为DNN模型训练框架。

#### 选用的实验模型
RNNLM、NMT、Transformer、NASNet。

#### Baselines：
- Expert:由领域专家进行人工切分，被称为 "专家 "策略。
- RNN-based:Mirhoseini等人使用循环神经网（RNN）来优化设备放置。这种方法通过使用分层模型进一步改进，以更好地分组和放置计算操作。
- Placeto：也是一种基于学习的算法，它使用图嵌入和强化学习来迭代改进所学到的放置策略。
- Baechi：采用传统的工作调度算法，为DNN模型图寻找在多个GPU上的内存感知的放置方案。
### 实验一  每一步的训练时间对比
![figure7](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20220119/figure7.16d81s7shvcw.webp)
#### 结论：
pesto表现最好，然后由于NMT和RNN中具备LSTM单元的网格结构，Pesto能够比别的找到更好的放置方案；而对于NASNet，Expert出现显存爆炸的现象，由于Pesto考虑了显存在各个GPU上的负载均衡约束，没有这种问题；

### 实验二 得到放置方案的时间对比
![table2](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20220119/table2.1914lag17h8g.webp)
#### 结论：
baechi虽然找的快，但是前面的实验表明它的训练时间长，即方案不好。

### 实验三 整体训练时间对比（以人工为基础）
![table3](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20220119/table3.4cq04omiijg0.webp)
#### 结论：
由于进行了图粗化，能够很快找到放置方案，然后又考虑了通信拥塞，又提前知道计算时间和通信时间可以获得一个很好的调度方案且考虑显存约束，所以更好。

### 实验四 通过模拟器验证Pesto处理通信开销的能力
![figure8](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20220119/figure8.2s69ddu1cws0.webp)
#### 结论：
随着硬件算能力的上升，Pesto对比Expert会越来越强，因为计算力越大，则训练时间越受到通信开销的限制；然后Pesto随通信速度的上升也变化不大，因为它本来就能很好的处理通信开销。

## 总结
本篇论文的数学建模个人认为挺精巧的，然后加节点考虑通信开销，又融合节点减少方程求解时间的方式，十分新颖。
