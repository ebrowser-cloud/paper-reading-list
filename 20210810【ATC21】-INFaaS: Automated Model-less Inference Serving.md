By AoDong Chen
# INFaas:自动化无模型推理服务
[INFaaS: Automated Model-less Inference Serving](https://www.usenix.org/conference/atc21/presentation/romero)

## 论文主要工作
  本文介绍了INFaaS（Inference as a service），一种提供分布式推理服务的自动化(自动选择模型变体)无模型（无需人为指定模型）系统，在该系统中过，开发人员只需为其应用程序指定性能和准确性要求，而无需为每个推理查询指定特定的模型变体。INFaaS能够从已经训练过的模型中生成模型变体，并代替开发人员有效地选择模型变体，以满足不同应用程序的需求。
  与AWS EC2上最先进的推理服务系统相比，INFaaS将吞吐量提高为的1.3倍，延时违反量降为1/1.6，成本降为1/21.6。
