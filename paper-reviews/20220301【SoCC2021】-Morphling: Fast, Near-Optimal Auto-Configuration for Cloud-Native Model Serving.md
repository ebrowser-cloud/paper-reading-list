By AoDong Chen
# Morphling:一个为云原生模型服务自动快速选择接近最优配置的系统
[Morphling: Fast, Near-Optimal Auto-Configuration for Cloud-Native Model Serving](https://cse.hkust.edu.hk/~weiwa/papers/morphling-socc21.pdf)


本文介绍了Morphling，一个为云原生模型服务自动快速搜索出接近最优的配置参数的框架。Morphling离线训练了一个元模型，该模型在面对新型任务时能够快速收敛拟合参数性能空间，然后将该模型作为SMBO的初始回归模型，迭代选择出符合目标函数的最佳配置。  

与当前最先进的参数自动配置方法相比，Morphling能够将搜索成本的中位数减少到1/3甚至1/22，在一个拥有720个解的空间中仅需采样30次就能找到最优的解。

## 论文作者
Luping Wang：香港科技大学。  

Lingyun Yang：香港科技大学科学与工程系的博士研究生。研究方向主要是机器学习系统，云计算和大规模集群的资源管理。  

Yinghao Yu：阿里巴巴研究员。研究方向主要是云计算。   

Wei Wang：香港科技大学计算机科学与工程系副教授。研究方向主要是机器学习系统、云计算和计算机网络。  

## 论文动机
机器学习应用作为云端服务需求量很大，需要投入大量资源。推理服务一般跑在容器上，有很多参数需要配置，例如CPU核数、GPU类型、GPU显存、batchsize、时间共享比例等。一个好的参数配置会对吞吐量提升很大。
之前的工作，一般是关注在怎么调度和选择模型（INFaaS)；而普遍的自动配置技术只能在低维配置空间上表现的好。因此需要一个能够快速处理高维搜索空间的新方法。  

## 论文贡献
本篇论文主要贡献有以下内容：
- 分析出各种参数对各种不同的模型有一个类似的性能影响趋势
- 训练了一个能快速拟合参数配置空间的元学习模型
- 使用SMBO方法通过迭代选择出最佳参数配置

### 类似的性能趋势
![figure4](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20220303/figure4.3w1rm4xh63i0.png)   
配置参数等对不同的推理任务的影响有一个大体上相似的影响，所以促使作者使用元学习来学出一个模型来拟合这个规律。  


### 元学习模型训练
![algorithm1](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20220303/algorithm1.t7m0uyo5jio.webp)    
MAML最大的特点是Model-agnostic (模型无关的)，MAML对模型的形式不做任何假设，这意味着MAML可以用一个普适的方法更新任意模型的参数来使其有快速适应的能力(fast adaptation)。这个想法背后的intuition是内部表示中的一部分表示相比其他表示更加transferrable，即这些表示在任务分布p(T)中都具有广泛的适用性，而非只在一个任务中有效。

### Sequential Model-Based Optimization(SMBO)
![SMBO](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20220303/SMBO.858pnf9smlk.webp)  
SMBO是AutoML中一种搜寻最优超参的方法，用一句话概括就是：在有限的迭代轮数内，按照损失函数的期望值最小同时方差最大的方式选择参数。直观点理解，就是选择loss小，并且最有可能更小的地方进行探索，寻找更优超参。

## 系统工作流程
![figure5](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20220303/figure5.evk89cihcj4.webp)  
要启动配置搜索过程，用户通过RPC调用向Experiment提交一个请求（1），指定容器、参数范围、优化的目标函数和采样次数。在实验过程中，Morphling与一个算法服务器进行迭代通信，该服务器返回下一个要采样的配置(2)，然后启动一个Trial(3)来评估该配置。Trail会启动一个模型服务实例(4)，然后启动一个Client（5），Client进行压力测试(6)。测试结束后，测量的性能（例如，峰值RPS）被存储到数据库中。在所有的结果被送回Experiment后，一个试验就完成了（7）。Morphling反复启动试验，直到采样次数用完。实验因此完成，并获得最佳配置。


## 实验
### 实验设置
![table1](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20220303/table1.6rbrn66uq9c0.webp)  
遵循MLPerf Inference基准的指导方针，作者在15个模型家族中选择了42个不同规模的模型（表1），包括图像分类模型，如ResNet，EfficientNet和MobileNet，以及语言模型如BERT，ALBERT和Universal Sentence Encoder。这些预先训练的模型由TensorFlow模型zoo提供。作者将模型和服务框架打包在一个Docker容器中，同时还有一个接口用于在容器启动时配置资源和批量大小。Morphling中使用的元模型是一个有两个隐藏层的神经网络，每个隐藏层有128个隐藏单元。

Baselines：
- 贝叶斯搜索 
- Ernest为每个工作负载建立了一个专门的回归模型，并通过几次抽样对其进行训练。作者使用与Morphling相同的神经网络架构，但为每个推理服务从头开始训练。
- Google Vizier是一个基于相似性的搜索框架，它使用高斯回归器作为核函数，并采用迁移学习来加速新的搜索。
- Fine-Tuning，用十个模型训练了一个回归模型。（这种微调模型只是针对预测的准确率而训练的）
- 随机搜索通过对搜索空间进行随机抽样，产生不同的配置，并取其中性能最好的一个。作者规定同样的配置只被抽样一次。

Metrics：
- RPS/Cost
- 搜索成本——找到满足某种性能要求的配置所需的采样次数。

### 实验一 不同采样次数得到的最终性能对比
![figure6a](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20220303/figure6a.77nyv1tx4bs.webp)  
#### 结论： 
在相同的采样预算下，Morphling总是比五个baseline返回更好的配置。Morphling通过对不超过30个配置（不到搜索空间的5%）进行采样，确定了所有模型的最佳配置；第二好的算法Fine-Tuning需要采样200个配置，但仍然不能保证所有模型的最佳性能。  

### 实验二 不同性能所需要的搜索成本（采样次数）
![figure6b](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20220303/figure6b.1nbekywdy56o.webp)  
#### 结论：
 在所有情况下，Morphling都优于其他baseline：性能要求越高，它的效率就越高。当搜索最佳配置（100%优化）时，Morphling比Fine-Tuning（采样中位数为54）的效率高3倍，比BO和Google Vizier高9.4倍，比Ernest和随机搜索高22倍以上。  

### 实验三 证明Morphling的高效对不同目标函数具有普适性
![figure7](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20220303/figure7.60o1eucg7no0.webp)   
#### 结论：
图7比较了Morphling和五个baseline在两个目标函数下的搜索成本。Morphling保持了它的优势，总是在30次采样内返回最佳配置。与图6b相比，BO在两个新目标函数下看到了效率的急剧下降，需要超过400次采样（中位数，搜索空间的55%）才能找到最佳配置。像搜索最高的RPS这样的目标往往会导致一个具有多个局部最优的高度不平衡的配置-性能平面，这种情况下BO很容易被卡住。  

### 实验四 证明元模型对不同推理任务的快速适应能力
![figure8](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20220303/figure8.51jresnq5x40.webp)  
#### 结论：
图d是人工测试出的不同配置对应的性能空间，模型的目标就是要快速的拟合这个平面，a图为最开始的随机初始化模型对应的解空间，b和c则是采样8次和28次之后的性能空间，显然与真实情况很接近了。  
![figure9](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20220303/figure9.745yhghwq940.webp)  
#### 结论：
相比之下，图9显示了BO对同一模型NNLM-128的拟合过程，在28次采样后，拟合的平面仍然远离实际情况。  

### 实验五 实际生产环境下的结果对比
![figure11](https://cdn.jsdelivr.net/gh/CAD2115/image-hosting@main/20220303/figure11.1qt8ppukjs74.webp)  
#### 结论：  
图11a比较了不同方案在不同的采样预算下为25个推理服务推荐的配置的归一化性能。图11b进一步比较了这些算法为寻找最佳配置所需的搜索成本。Morphling在配置性能和搜索成本方面都领先于五个baseline。Morphling以9次抽样（中位数）和19次抽样的最大值确定了最佳配置，比baseline的效率高得多。对于Morphling来说，生产服务下的搜索成本（100个选项中的9个抽样）略高于开源模型下的成本（720个选项中的18个抽样）。因为后者有更多的配置选项，因此有更大的搜索空间。

## 总结
本篇论文在参数配置领域使用了一种机器学习中流行的元学习思路来训练模型，使得最终训练出来的模型能够快速拟合新模型的参数配置空间。其中关键的特性是参数配置对不同的模型有一个类似的性能影响趋势，因此在未来自己研究的领域中若是也有这种特点，不妨也可以借用元学习的方法。
