# Pollux: Co-adaptive Cluster Scheduling for Goodput-Optimized Deep Learning
## 论文作者

**Aurick Qiao**：CEO[@PetuumInc](https://twitter.com/PetuumInc) ,毕业于卡内基·梅隆大学计算机科学系博士，研究方向：可扩展的机器学习。

**Qirong Ho**: Co-founder,CTO[@PetuumInc](https://twitter.com/PetuumInc) ,2014年毕业于卡内基·梅隆大学计算机科学系博士,现在研究方向：在大数据和大尺度模型上的分布式机器学习软件系统

**Willie Neiswanger** ：毕业于卡内基·梅隆大学计算机科学系博士，现在工作内容：开发分布式推理和概率编程的方法，以帮助扩展和自动化机器学习

**Sang Choe**：卡内基·梅隆大学语言技术研究所二年级博士生，研究方向：高效的大规模机器学习算法和系统。

## 问题定义

* 大多数现有调度程序希望用户为每个作业指定资源数量，这通常会导致资源使用效率低下。近来，一些调度程序帮助用户选择分配给作业的资源，但却忽略了重新优化深度学习训练，无法更好地利用所提供的资源。
* 一般调度器的工作是为了给模型训练提供合适的资源，如分配多少个GPU，需要考虑集群内部争用和被提交工作集的伸缩性；用户提交的模型又期望配置更合适的参数（batch size 和 learning rate）来使当前已分配资源（如GPU）。调度器的工作和用户调参使存在相关性的，现存的调度器并没有考虑它们之间的相关性管理。本论文针对这个问题开展研究，提出了pollux架构。

## 介绍

深度学习（DL）训练在许多诸如云和数据中心的共享资源环境下已经快速成为主流的工作负载。为了快速完成资源密集型和长运行时间DL作业，一般会在分布式集群使用昂贵的硬件设备，如GPU。于此需求，有专用集群提供此服务，这类集群会有特定调度器调和不同争用的DL作业。然而，现存的调度器是需要用户手动配置他们的DL作业，这会降低训练性能和资源利用率。最近的*elastic*调度器可以给每个工作动态分配合适的资源，但是它没有考虑内部独立且训练相关的配置，例如一个DL作业的batch size 和 learning rate会影响训练模型所需的资源。这两个参数很难合适地配置。

那么如何才能动态配置用户提交的DL作业呢？作者认为需要在两个对立的指标找到平衡：每小时处理的训练实例的数量——**系统吞吐量**（system throughput），和每处理一个单位的训练数据所取得的训练进展量——**统计量效率**（statistical efficiency）.

[<img src="https://i.postimg.cc/KYQCDPtp/fig1.jpg" alt="fig1.jpg" style="zoom:150%;" />](https://postimg.cc/mczdbFX7)

图1a证实了吞吐量可以增加batch size来提高，但是在[An empirical model of large-batchtraining](https://arxiv.org/abs/1812.06162)和[Measuring the Effects of Data Parallelism on Neural Network Training](https://arxiv.org/abs/1811.03600)指出即使有优化调整学习率（learning rate），增大batch size经常导致statistical efficiency下降。然而图1b显示对于每一个不同的GPU分配，可以找到最佳batch size来平衡throughput提高时伴随的statistical efficiency下降。相对于batch size大小statistical efficiency的下降取决于训练进度（这在论文[An empirical model of large-batch
training](https://arxiv.org/abs/1812.06162)有解释），与训练初期相比，训练后期的工作有可能容忍10倍或更大的批量，而不会降低统计效率。

本论文主要有以下贡献：

* 提出了DL作业的goodput，它是对性能的整体衡量，其中考虑了系统吞吐量（throughput）和统计效率（statistical efficiency）。
* 实验表明，在训练期间，DL作业的goodput的模型可以通过观察吞吐量和统计效率习得，并用于预测不同资源分配和batch size下得性能。
* 设计和实现了一个调度架构——Pollux，它使用goodput模型为每个待定和运行的DL作业配置资源和训练参数的正确组合。
* 评估实验中，使用来自微软集群跟踪的工作负载在集群测试平台上评估Pollux，与最近的DL调度器[Tiresias](https://www.usenix.org/conference/nsdi19/presentation/gu)和[Optimus](https://www.usenix.org/conference/nsdi19/presentation/gu)相比,Pollux减少了平均作业完成时间达73%, 即使所有的作业都事先进行了手动调整，Pollux也将平均作业完成时间减少了37%-50%。同时，Pollux将完成时间的[公平性](https://www.usenix.org/conference/nsdi20/presentation/mahajan)提高了1.5倍-5.4倍。
* 在云环境中，使用基于Pollux的goodput驱动的自动扩展，有可能将大型模型的训练成本降低25%.

## 背景:分布式机器学习

[![loss-function.jpg](https://i.postimg.cc/0yYTbRjs/loss-function.jpg)](https://postimg.cc/MnGPFL1P)

损失函数可以使用随机梯度下降法（SGD）或其变体如AdaGrad和Adam进行最小化。为了解释系统的吞吐量和统计效率，论文使用SGD作为运行实例。SGD反复应用以下更新，直到损失收敛到一个稳定的值：
$$
\omega(t+1)= \omega(t)-h\hat g(t)
$$
h被称为学习率，它是一个控制每次更新幅度的标量，而 g(t)是损失函数L的随机梯度估计，使用训练数据的随机小批M(t)⊂X来评估。

### 系统吞吐量

系统吞吐量在本论文定义为每小时处理的训练实例的数量，它与资源得配置和分配，分布式执行的方法和同步技术，以及batch size有关。

#### 数据并行执行

同步数据并行是主流分布式DL训练方法。

[![local-gfradient.jpg](https://i.postimg.cc/mkKvFT7b/local-gfradient.jpg)](https://postimg.cc/1VcCkhWT)
$$
模型参数 \omega^t被复制分配到GPUs\,1,...,k\, 并且每个mini-batch\, \mathcal{M}^t\,分配到每个节点等大小的partition，\, 每个GPU计算一个本地的梯度值\hat{\mathcal{g}^t}\\
$$
其中计算梯度值得时间为**Tgrad**，计算平均梯度（使用collective all-reduce）与同步**w**到所有GPU得时间之和为**Tsync**。

#### 统计量效率（Statistical Efficiency）

* 统计量效率在本论文定义为每处理一个单位的训练数据所取得的训练进展量，它受参数batch size和学习率所影响。该论文还引用[梯度噪声（GNS）](https://arxiv.org/abs/1812.06162)与统计量效率联系。较大的GNS意味着训练参数，如批量大小和学习率可以增加到更高的值，而统计效率的降低相对较少。GNS在不同的DL模型之间可以有很大的不同。它也是非恒定的，在训练过程中往往会逐渐增加，最多可达10倍或更多。
* 因此，在训练过程中，对于大批量的后期，有可能达到明显更好的统计效率。梯度噪声尺度在数学上捕捉到了对批次大小如何影响统计效率的直观解释。当随机梯度的噪声较低时，在每个小批中增加更多的训练实例并不能显著提高每个梯度估计值，这就降低了统计效率。当随机梯度具有高噪声时，在每个小批中增加更多的训练例子可以减少每个梯度估计的噪声，从而保持较高的统计效率。在接近收敛时，随机梯度的信号相对低于噪声，因此在训练后期，较大的批次规模会更有用。

## 机器学习训练的Goodput 和 Pollux

* DL作业训练中的每个迭代（iteration）t的goodput可以定义为：[<img src="https://i.postimg.cc/5NCdWFMj/goodput.png" alt="goodput.png" style="zoom:200%;" />](https://postimg.cc/svs0pxxR)

其中**⋆=(a,m,s)**，**a**表示每个节点（node）分配的GPU数量，**m**表示每个GPU的batch size，**s**表示梯度累加（gradient accumulation）的步骤数。**M(a,m,s)=SUM(a)×m×(s+1)** 表示所有分配的GPU上的总批处理量。

### 建模EFFICIENCYt (M)

* **EFFICIENCYt (M)** 定义为：相对于使用初始**M0**,使用**M**的每个训练例子所取得的进展量。对于基于SGD的训练，这个数量可以用梯度噪声尺度（GNS）来表示。为了支持不同SGD的变体，本论文使用pre-conditioned gradient noise scale (PGNS)，用下式表示

[![PGNS.jpg](https://i.postimg.cc/d0JzCHZT/PGNS.jpg)](https://postimg.cc/zyPtZkjJ)

g是真实梯度，P是自适应SGD算法的预处理矩阵，S是每个样本随机梯度的协方差矩阵。

* **EFFICIENCYt (M)**定义为[![efficiency.jpg](https://i.postimg.cc/zfjYPZYj/efficiency.jpg)](https://postimg.cc/VSdhdh6r)

### 建模THROUGHPUT(⋆）

* 对数据并行DL的系统吞吐率建模和预测，首先预测每次训练迭代所花费时间，然后计算吞吐量为[![throughput.jpg](https://i.postimg.cc/6675TqXx/throughput.jpg)](https://postimg.cc/yWHzbVxL)

* 总时间**Titer**由计算平均梯度时间与参数同步时间和**Tsync**和计算梯度时间**Tgrad**组成，由于这两部分计算可能会有重叠，而且由于batch size过大可能会出现内存溢出（OOM）而使用了梯度累加——时间换空间的方法将其分为**s**次计算梯度值，所以最后建模成：

[![image.jpg](https://i.postimg.cc/nzNxqKJb/image.jpg)](https://postimg.cc/xJGB2z8t)

[![Tsync.jpg](https://i.postimg.cc/fTmzjPyH/Tsync.jpg)](https://postimg.cc/CzMpwmjD)



[![Tgrad.jpg](https://i.postimg.cc/hhFYtpqv/Tgrad.jpg)](https://postimg.cc/q6LwmcqH)

K表示总GPU数量，N表示节点（node）数量，其他为适应参数。

## Pollux 的设计和架构

[![pollux.jpg](https://i.postimg.cc/fySFs1p1/pollux.jpg)](https://postimg.cc/RJ9GQpMT)

* 首先，一个PolluxAgent与每个作业一起运行。它适合该作业的EFFICIENCYt和THROUGHPUT函数，并调整其批次大小和学习率，以有效利用其当前分配的资源。PolluxAgent定期向PolluxSched报告其作业的goodput函数。
* 其次，PolluxSched定期优化集群中所有作业的资源分配，同时考虑到每个作业的当前goodput函数和集群范围内的资源争夺。PolluxSched做出的调度决定还考虑了与资源重新分配相关的开销、多个作业之间的网络干扰导致的速度减慢以及资源公平性。
* PolluxAgent和PolluxSched共同适应对方。当PolluxAgent适应每个训练作业以有效利用其分配的资源时，PolluxSched动态地重新分配每个作业的资源，同时考虑到PolluxAgent对其作业的调整能力。

### PolluxAgent：作业级优化

* 每个训练作业都有一个PolluxAgent的实例被启动。在训练期间，它不断地测量作业的梯度噪声规模和系统吞吐量，并以固定的时间间隔向PolluxSched报告。它还使用这些信息来确定其作业在当前资源分配下最有效的批次大小，并使用适当的插件LR缩放规则（例如SGD的AdaScale或Adam的平方根缩放）使其作业的学习率适应这个批次大小。

[![image.jpg](https://i.postimg.cc/6phBFGdn/image.jpg)](https://postimg.cc/9Dr370gQ)

* 上式是构建THROUGHTPUT函数的参数，与PGNS和初始batch size M0一起作为描述GOODPUT函数的配置。在每次迭代（iteration），PolluxAgent根据这些参数指标决定每个GPU最效率的batch size和梯度累加步骤数。一旦找到一个新的配置，作业将在其后续的训练迭代中使用它，使用插件LR缩放规则来适当调整其学习率。由于作业的EFFICIENCYt函数随时间变化，PolluxAgent将定期重新评估最有效的配置。

### PolluxSched：集群级优化

* 为了确定如何高效分配资源，论文定义了一个**fitnees**函数，以及**Speedup**函数——给定的资源分配与使用公平资源分配之比，其中**J**表示任务数量，**Aj**表示对**j**任务的GPU数量分配向量，**af**表示对该任务的公平资源分配。



[![fitnees.jpg](https://i.postimg.cc/KcVSWDwd/fitnees.jpg)](https://postimg.cc/LhzbYLHT)

[![speedup.jpg](https://i.postimg.cc/3Rc6P3Sg/speedup.jpg)](https://postimg.cc/TLnCbvXw)

* PolluxSched利用这种预测GOODPUT的能力，通过搜索程序使FITNESS最大化，然后将输出的分配应用于集群。

#### 重复分配的惩罚机制

* 每次作业被重新分配到不同的GPU集，都会产生一些延迟来重新配置训练过程。使用流行的检查点重启方法，作者测量了15到120秒的延迟，这些延迟取决于被训练的模型的大小和训练代码中的其他初始化任务。为了防止过多的重新分配，当PolluxSched评估给定分配矩阵的适配函数时，它对每一个需要重新分配的工作进行惩罚。**Tj**是训练作业的时期（age），**Rj**是该作业迄今为止发生的重新分配次数，**d**是对重新分配延迟的估计。直观地说，REALLOC_FACTORj(d)根据工作**j**的历史平均再分配率将无限期地持续到未来的假设，对SPEEDUPj(Aj)进行缩放。因此，历史上经历过较高重新分配率的工作将在未来的重新分配中受到更大的惩罚。

[![image.jpg](https://i.postimg.cc/s2D3zDc8/image.jpg)](https://postimg.cc/8732BGxB)

[![2.jpg](https://i.postimg.cc/wv1HtfnQ/2.jpg)](https://postimg.cc/njfNgkFX)

#### 干扰避免

当多个分布式DL作业共享一个节点时，它们在同步梯度和模型参数时的网络使用可能会相互干扰，导致两个作业的速度减慢。PolluxSched通过不允许不同的分布式作业（每个作业在多个节点上使用GPU）共享同一节点来缓解这一问题。避免干扰是作为Pollux搜索算法中的一个约束条件来实现的，确保每个节点上最多分配一个分布式作业。

## 实验

### 1. 实验环境设置

* 16个节点和64个GPU组成的集群

* 每个节点是一个AWS EC2 g4dn.12xlarge实例，有4个NVIDIA T4 GPU，48个vCPUs，192GB内存和一个900GB SSD

* 集群上部署了Kubernetes 1.18.2，以及CephFS 14.2.8

* 合成工作负载构建：

[![workload.jpg](https://i.postimg.cc/hGyctWZG/workload.jpg)](https://postimg.cc/qt391YTf)

* 合成工作负载由上表中描述的模型和数据集组成。

在跟踪中和上表中根据其总的GPU时间对每个工作进行了分类。小型（0-1个GPU-小时），中型（1-10个GPU-小时），大型（10-100个GPU-小时），以及特大（100-1000个GPU-小时）。对于追踪中的每个作业，从上表中挑选了一个属于同一类别的训练作业。

* 对比的DL 调度器： [Tiresias](https://www.usenix.org/conference/nsdi19/presentation/gu)和 [Optimus](https://www.usenix.org/conference/nsdi19/presentation/gu)（手动调整配置）

### 2. 模型验证

[![efficiency.jpg](https://i.postimg.cc/MZjqqL7W/efficiency.jpg)](https://postimg.cc/CdYW4mrt)

* 上图：验证指标与训练进度的关系，三种不同的 批量大小。M0，一个中间批次大小，以及我们为每个DL任务设置的最大批次大小限制。衡量标准如表1所定义 除YoloV3外，其他指标如表1所定义，其中验证损失显示在表1中。尽管几个DL任务的验证曲线存在差异（特别是在早期的历时中），但在这评估的不同批次规模中，它们取得了类似的最佳值（除了DeepSpeech2的±4%，所有任务的相对差异为±1%）

* 中图：测量的统计效率与训练进度 两种不同的批量大小。最上面两行的训练进度（X轴）以 "统计代（epoch） "为单位，该定义类似于[scale-invariant iterations](https://openreview.net/forum?id=rygxdA4YPS)，定义为

[![epochs.jpg](https://i.postimg.cc/1t3hkC9Z/epochs.jpg)](https://postimg.cc/67gmRh2H)

其中|X|是训练集大小。

* 底图：测量的EFFICIENCYt与预测的EFFICIENCYt相比，在一个早期训练历时（epoch）（大约是训练的1/8），使用每个范围内的中位数批次大小测量的![apha](D:\BaiduNetdiskWorkspace\ecnuicloud\Pollux_fig\apha.jpg) 统计历时 （epoch)是由EFFICIENCYt归一化的训练迭代次数，这样每个统计历时在理论上，如模型所预测的那样，在不同的批次规模中取得了相同的训练进度。因此，不同批次规模的验证曲线之间的相似程度是EFFICIENCYt作为实际训练进度预测器的准确性的一个指标。中图和底图显示了在训练期间和不同批次大小的情况下测量和预测的EFFICIENCYt。一般来说，较大的批次规模在训练初期的EFFICIENCYt较低，但在训练后期缩小差距。

[![throughput.jpg](https://i.postimg.cc/fyXNC3z3/throughput.jpg)](https://postimg.cc/kRnzXgt9)

上图显示了THROUGHPUT函数与一系列资源分配和批次大小的测量吞吐量值的拟合实例。每个DL任务都是用PyTorch实现的，它重叠了后向传递的计算和通信。梯度与NCCL 2.7.8同步，它根据检测到的GPU和它们的位置以及它自己的内部性能估计，使用环状全还原或树状全还原。总的来说该模型可以紧密地代表观察到的数据，同时改变资源量以及批次大小。

### 3. 测试平台宏观基准（Macrobenchmark）实验

* Optimus+Oracle和Tiresias相比，Pollux（p=-1）分别实现了50%和37%的平均JCT，27%和27%的尾部（第99百分位）JCT。

[![testbed.jpg](https://i.postimg.cc/2SYzHxBC/testbed.jpg)](https://postimg.cc/LnyK5LTW)

* Pollux改进的一个关键来源是它在训练期间能够在高吞吐量/低效率和低吞吐量/高效率模式之间进行折衷。下图显示了在执行合成工作负载期间分配的GPU总数和平均EFFICIENCYt。 在集群竞争较少的时期，Pollux可以分配更多的GPU（用（A）表示），并使用更大的批处理量来提高训练吞吐量，甚至以较低的统计效率为代价，因为这样做会带来更高的整体吞吐量。另一方面，在集群竞争激烈的时期，Pollux可以使用较小的批处理量来提高统计效率（如（B）所示）。

[![comparison.jpg](https://i.postimg.cc/VvSy6JrG/comparison.jpg)](https://postimg.cc/PvkVVrMY)





### 4. 模拟器实验

该模拟器是通过测量**表1**中每个模型的性能和梯度统计，在许多不同的资源和批量大小的配置下，并为每个模拟作业重新播放来构建的。

#### 模拟器构建

对于表1中的每项工作，我们测量了在我们由16个节点和64个GPU组成的测试平台集群中146个不同的GPU分配的每个训练迭代的时间。

对于每个分配，我们测量了一系列的批次大小，直到GPU的内存极限。为了模拟一个作业的吞吐量，作者在测量的配置上查询了一个多维线性插值。对于每个模型，还测量了在训练过程中使用一系列批处理量的（预设条件）梯度噪声规模，并跨越每个历时。为了模拟使用某一批次大小的作业的统计效率，在测量的两个最近的批次大小之间线性插值其PGNS的值。

#### 调度器的公平

作者使用[完成时间公平性](https://www.usenix.org/conference/nsdi20/presentation/mahajan)（用p表示）来评估Pollux的调度公平性，它被定义为在共享资源上运行的作业的JCT (作业完成时间) 与在隔离和平等分区的集群中运行的作业的比率。在这个指标下，p<1的工作被集群调度器处理得比公平好，而p>1的工作被处理得比公平差。
在下图中，作者比较了Pollux与Optimus+Oracle+TunedJobs和Tiresias+TunedJobs的完成时间的公平性。

* Pollux的p=1导致公平性较差，与Tiresias+TunedJobs类似，这明显表现为p>4的长尾工作。Optimus+Oracle+TunedJobs获得了更好的公平性，因为其分配算法试图使每个工作的JCT改进相等。
* Pollux的p=-1提供了最好的公平性，99%的作业实现了p<2，并且在这样做的同时仍然提供了显著的性能提升（表2）。对于p=-10，我们观察到整体的公平性稍差，这是由于PolluxSched在任何时候都忽略了成本而偏向于均衡加速，从而引起了更多的重新分配。
* Tiresias和Optimus的曲线与[Mahajan等人的报告Themis](https://www.usenix.org/conference/nsdi20/presentation/mahajan)(针对不同的工作负载）一致。虽然他们的Themis系统不能直接比较，但Pollux的p=-1的r范围与Themis的报告范围相似。与Tiresias和Optimus相比，最大p的改进（1.5倍和5.4倍）也是类似的。

[![fairness.jpg](https://i.postimg.cc/qRDV4sR6/fairness.jpg)](https://postimg.cc/CB864fbw)

#### 其他影响实验



[![other-Effects.jpg](https://i.postimg.cc/rmm3nDJV/other-Effects.jpg)](https://postimg.cc/kV0TDXzz)

* 对工作负载的敏感度：上图a显示随着负载的增加，所有三个调度策略都遭受了更长的平均JCT和makespan。在所有的作业负载中，Pollux保持了与基线调度器类似的相对改进。

* 调度间隔的影响：作者使用一系列的调度间隔值来运行Pollux，如上图b所示Pollux在2分钟以内的平均JCT方面表现相似，而更长的间隔会导致性能下降。由于新提交的作业只能在下一个调度间隔期间开始，作者预计由于较长的调度间隔，平均排队时间会增加。然而排队促成了大约一半的性能下降，表明Pollux仍然受益于相对频繁的资源分配调整。

* 避免干扰的影响：作者人为地给共享同一节点的分布式作业注入不同程度的放缓。上图c显示在启用干扰避免的情况下，平均JCT甚至不受严重减速的影响，因为网络争用被完全缓解了。然而，在没有干扰规避的情况下，当干扰放缓到50%时，平均JCT要长1.4倍。另一方面，在理想情况下，当干扰导致的减速为零时，无论是否启用干扰避免，PolluxSched的表现都很相似。这表明，PolluxSched仍然能够找到有效的集群分配，同时服从干扰避免约束。

### 5 其他平台上的Pollux实验

#### 云环境上自动扩展

* 在云环境中，计算资源可以根据需要获得和释放，而用户则为他们持有这些资源的时间付费。Goodput驱动的调度提供了一个独特的机会：当DL模型的统计效率在训练期间增加时，在大型训练工作的后期，提供更多的云资源和使用更大的批处理量可能会更有成本效益。
* **作者使用上述集群模拟器提出了一些初步证据，并指出基于goodput的自动扩展系统的完整设计可能是未来工作的主题。**

* 自动缩放ImageNet训练：作者使用Pollux的goodput函数实现了一个简单的自动缩放策略。在训练过程中，只要

[<img src="https://i.postimg.cc/QMsrbNwC/cloudauto.jpg" alt="cloudauto.jpg" style="zoom:200%;" />](https://postimg.cc/Mns4ZwWJ)

即goodput超过假设完美扩展性的理想goodput的某个分数U，就扩大节点的数量。这里作者设定U=2/3，并增加到一个节点数，使预测的良好吞吐量约为预测的理想良好吞吐量的L=1/2。

* 下图比较了本论文的基于Pollux的自动缩放器和[Andrew Or等人提出的自动缩放器](https://www.cs.princeton.edu/~andrewor/deep-learning-elasticity.pdf)，该自动缩放器允许在训练期间增加批次大小，但使用系统吞吐量而不是良好吞吐量来模拟工作性能。由于系统吞吐量不随训练进度而变化，Or等人的自动缩放器迅速扩展到更多的节点和更大的批次规模（下图a），此后保持不变。另一方面，Pollux从少量的节点开始，随着时间的推移，较大的批处理规模的有效性提高，逐渐增加节点的数量。图b显示，Pollux在整个训练过程中保持了较高的统计效率。总的来说，与Or等人的基于吞吐量的自动扩展相比，Pollux训练ImageNet的成本要低25%，完成时间只长6%。

[![goodput-Cloud.png](https://i.postimg.cc/K8hXGvZf/goodput-Cloud.png)](https://postimg.cc/3d1f9h00)

#### 超参数优化实验

* 超参数优化（HPO）是一个重要的DL工作负载。在HPO中，用户定义了一个相关模型超参数的搜索空间。一个HPO算法（又称试验调度器）提交许多训练作业（试验），以评估特定超参数的有效性，如模型精度或能源效率等目标。不同的HPO算法类型以不同的方式管理试验。例如，贝叶斯优化算法可能一次提交几个训练作业，并根据以前试验的充分训练结果确定未来的试验。
* **关于Pollux如何影响不同HPO算法类型的全面评估是未来的工作**。

* 实验环境和配置：对树状结构的Parzen估计器（TPE）进行了配置，使4个试验同时运行，总共运行100个试验。测试平台由两个NVIDIA DGX A100节点组成，每个节点有8个A100 GPU。基准调度程序为每个试验分配了4个GPU的静态分配（都在同一个节点上），并为每个试验使用固定的每GPU批处理量。

[![theLast.jpg](https://i.postimg.cc/59Z9zJ4Y/theLast.jpg)](https://postimg.cc/BtBGWd14)

* 上表显示了在CIFAR10数据集上训练的ResNet18模型的调整结果，该模型使用了流行的基于贝叶斯优化的HPO算法。搜索空间涵盖了学习率和退火、动量、权重衰减和网络宽度等超参数。

本实验实现了类似的精度值，但由于随着试验的进行，Pollux完成HPO的速度快了30%，因为它是自适应（重新）分配资源和自适应批次大小。

## 相关工作

除了上述提到的1）基于goodput的自动扩展系统的完整设计可能是未来工作的主题和2）关于Pollux如何影响不同HPO算法类型的全面评估，还有以下额外的未来方向。

* 自适应批次大小的训练：最近关于DL训练算法的工作探索了动态适应批量大小以获得更好的效率和并行化。AdaBatch在训练过程中以预先确定的迭代次数增加批处理量，同时线性放大学习率。Smith等人建议在训练过程中不要使学习率衰减，而应该增加批次大小。CABS使用与Pollux类似的梯度统计，在训练过程中自适应地调整批次大小和学习率。这些工作有一个共同的假设，即在需要的时候，额外的计算资源可以用来并行化更大的批处理量，这在共享资源环境中很少是真的。Pollux是对现有自适应批处理策略的补充，它将批处理规模和学习率与当前可用的资源量结合起来进行调整。另外，Anytime minibatch[17]适应批次大小以减少分布式训练中的散兵游勇。KungFu支持自适应训练算法，包括自适应批次大小，它允许应用程序定义自定义的适应策略，并在训练过程中实现有效的适应和监控。尽管KungFu是针对单任务训练的，而Pollux是针对集群调度的，但作者相信KungFu提供了有用的工具，可以用来实现PolluxAgent使用的自适应策略。

* 超参数调整：大量的工作集中在调整ML和DL模型的超参数，这通常涉及许多训练作业。虽然批量大小和学习率属于这些系统经常优化的超参数空间，但Pollux的目标是根本不同的。HPO算法寻找最高的模型质量，而Pollux则是在不降低模型质量的前提下，为每个作业调整批量大小和学习率，以达到最有效的执行。

## 总结

* Pollux是一个DL集群调度器，它可以共同适应性地分配资源，同时调整每个训练作业以最好地利用这些资源。

* 本论文提出了一个结合系统吞吐量和分布式DL训练的统计效率的goodput表述。基于goodput最大化的原则，Pollux自动联合调整DL作业的资源分配、批量大小和学习率。这些对用户来说是特别难以手动配置的，即使用户能够很好地配置他们的作业，Pollux也比最近的DL调度器表现得更好、更公平，并且在用户知识更现实的情况下提供更大的好处。

* 相对于最先进的DL调度器，Pollux将平均作业完成时间减少了37-50%，即使他们为每项作业提供了理想的资源和训练配置。

## 思考

对于深度学习模型训练的资源分配和调度有一个初步总览，该片论文调度上有在作业级粒度上优化，对于深度学习的参数以及相关模型训练的需求也需要整体考虑进去，如batch size大小，学习率以及吞吐量等等指标或参数。

本论文有诸多亮点，其中之一就在建模GOODPUT上，它不仅仅考虑吞吐量。作者在构建efficiency里参考了随机梯度噪声这一概念并加以改进用在自己的idea里，并用实验证明模型的有效性，可以加以借鉴。另外在实验部分对比实验里，baseline的选择搭配可以用【模型+】的方式组合成作为参照实验。

