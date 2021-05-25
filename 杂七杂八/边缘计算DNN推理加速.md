# 移动边缘计算DNN推理加速论文总结

移动边缘计算(Multi-Access Edge Computing, MEC)是一种新兴的计算范式，将计算和存储能力“下沉”到网络边缘。越来越多的研究开始尝试在边缘侧部署应用，其中不乏一些计算密集型的深度神经网络(Deep Neural Networks, DNN)应用。但DNN模型的高算力需求和边缘侧设备的弱算力供应之间存在不可调和的矛盾，缓解矛盾的方式大体上有两种：1.优化DNN模型，在不降低模型性能的前提下减少计算量；2.充分调动边缘侧资源，加速DNN模型在边缘设备上的推理速度。本文就为大家总结了近三年移动边缘计算DNN推理加速的相关研究。

## 0.研究分类

按照数据在DNN模型中的处理流程，可以将移动边缘计算DNN推理加速研究分为3大类：早期退出、模型划分、局部并行、全局并行。

## 1.早期退出

### Adaptive Parallel Execution of Deep Neural Networks on Heterogeneous Edge Devices (ICDCS, 2017)

![](https://gitee.com/molinchn/BlogImage/raw/master/duanyang/20201224143948.gif)

该研究的思想很简单，但也十分有效。研究者认为，DNN模型推理不一定在最后一层才达到最高的精度，有可能在推理到中间某一层的时候，精度已经可以满足设定的阈值，此时推理就可以退出了，该层之后的计算都可以不用进行。极大地减少的计算量，自然整体推理速度就提高了。

## 2.模型划分+早期退出

### 2.1 Edge AI: On-Demand Accelerating Deep Neural Network Inference via Edge Computing (TWC, 2020)

![](https://gitee.com/molinchn/BlogImage/raw/master/duanyang/20201224144229.gif)

该研究的关键点有两个：1.DNN划分，根据可用带宽在移动设备和边缘服务器之间自适应地对DNN计算进行划分，以便利用边缘服务器的计算能力；2.移动设备上模型早期退出，因为DNN划分后的执行性能仍然受到在移动设备上运行的模型的制约，可以通过早期退出中间DNN层的推理来加速整体推理。通过对DNN分区和 退出点调整进行联合优化，可以最大限度地提高准确性，同时保证延迟要求。

### 2.2 ADDA: Adaptive Distributed DNN Inference Acceleration in Edge Computing Environment (ICPADS, 2019)

![](https://gitee.com/molinchn/BlogImage/raw/master/duanyang/20201224144310.gif)

该研究考虑了边缘计算执行框架优化的两个方面：1.DNN计算路径优化，构造多个DNN计算路径，在精度保证下实现不同输入数据推理的早期退出；2.DNN计算划分优化，考虑到边缘服务器的负载、网络带宽和延迟，对具有不同层划分的多路径DNN网络的总推理执行时间进行了分析和建模，可以实现终端设备和边缘服务器之间的动态最佳DNN分区，以进一步降低推理延迟。

## 3.模型划分

### 3.1 Dynamic Adaptive DNN Surgery for Inference Acceleration on the Edge (INFOCOM, 2019)

![](https://gitee.com/molinchn/BlogImage/raw/master/duanyang/20201224144653.gif)

该研究关注边缘侧和云的结合，认为DNN不同层的划分会导致不同的计算时间和传输时间，DNN的划分实际上是一种计算和传输之间的权衡，因此是存在最佳划分的。设计了动态自适应DNN划分方案，通过持续监控网络状况来优化DNN网络划分，以寻找具有动态网络条件的集成边缘和云计算环境中的最佳DNN分区。

### 3.2 Towards Real-time Cooperative Deep Inference over the Cloud and Edge End Devices (Proceedings of the ACM on Interactive, Mobile, Wearable and Ubiquitous Technologies, 2020)

![](https://gitee.com/molinchn/BlogImage/raw/master/duanyang/20201224144735.png)

该研究将DNN模型自适应划分为两部分，并在不同的设备上执行不同的部分，以最小化总推理延迟。研究者把最优DNN划分转化为有向无环图(DAG)中的最小割问题，并提出了一种新的两阶段快速深度模型划分(QDMP)方法来求解它。

### 3.3 Joint Device-Edge Inference over Wireless Links with Pruning (SPAWC, 2020)

![](https://gitee.com/molinchn/BlogImage/raw/master/duanyang/20201224145224.gif)

该研究考虑了结合网络剪枝，在给定信道条件和物联网设备计算约束条件下，找到最佳的DNN划分点和剪枝参数，以确保在给定场景下尽可能达到最佳的精度和效率。

### 3.4 JointDNN: An Efficient Training and Inference Engine for Intelligent Mobile Cloud Computing Services (TMC, 2018)

该研究关注DNN在移动设备和云的联合平台中推理和训练，将DNN结构建模为一个有向无环图(DAG)，证明了在不同情况下，即最佳性能或能量消耗的最优计算调度问题，可以归结为多项式时间多种资源约束下的最短路径问题。

![](https://gitee.com/molinchn/BlogImage/raw/master/duanyang/20201224145309.png)

### 3.5 Auto-tuning Neural Network Quantization Framework for Collaborative Inference Between the Cloud and Edge (ICANN, 2018)

该研究对DNN的算子进行剖面，并生成候选图层作为划分点。当使用神经网络时，框架将自动调整网络划分。推理时，网络的第一部分在边缘设备上量化和执行以减少存储和计算，网络的第二部分全精度网络在云服务器中执行。

![](https://gitee.com/molinchn/BlogImage/raw/master/duanyang/20201224145342.gif)

## 4.局部并行

### 4.1 Fully Distributed Deep Learning Inference on Resource-Constrained Edge Devices (SAMOS, 2019)

![484168_1_En_6_Fig6_HTML](https://gitee.com/molinchn/BlogImage/raw/master/duanyang/20201224145413.png)

该研究考虑了DNN模型划分后的数据通信开销，优化了覆盖DNN所有层分布式执行的存储、计算和通信需求。对于给定数量的边缘设备，该方案是联合应用的，以最小化设备之间的交换数据量，优化运行时间。

### 4.2 MeDNN: A distributed mobile system with enhanced partition and deployment for large-scale DNNs (ICCAD, 2017)

![8203852-fig-3-source-large](https://gitee.com/molinchn/BlogImage/raw/master/duanyang/20201224145424.gif)

该研究将MapReduce的概念用到分布式体系结构中，由Group Owner对特征图进行划分，并由每个Worker Node进行卷积计算，计算结果再由Group Owner进行归并以进行下一次计算。

### 4.3 Adaptive parallel execution of deep neural networks on heterogeneous edge device (SEC, 2019)

![image-20201217164706537](https://gitee.com/molinchn/BlogImage/raw/master/duanyang/20201224145548.png)

该研究使用预测模型来预测DNN每一层在设备上的执行时间，并为每个划分选择最佳的划分点和并行化策略。

### 4.4 Distributed Inference Acceleration with Adaptive DNN Partitioning and Offloading (INFOCOM, 2020)

该研究提出了一个细粒度的自适应划分方案，将DNN分成比单个图层更小的部分从而减少网络中的通信开销。使用一种基于交换匹配的分布式算法将DNN推理从端设备卸载到雾网络中的节点。

![](https://gitee.com/molinchn/BlogImage/raw/master/duanyang/20201224145640.gif)

## 5.全局并行

### 5.1 DeepThings: Distributed Adaptive Deep Learning Inference on Resource-Constrained IoT Edge Clusters (IEEE Transactions on Computer-Aided Design of Integrated Circuits and Systems, 2018)

![](https://gitee.com/molinchn/BlogImage/raw/master/duanyang/20201224145658.png)

该研究重点讨论能够执行早期卷积层的划分和分布方法，关键思想是将原始的CNN层分割成独立的可分配执行单元，DeepThings将动态平衡边缘集群之间的工作负载。

### 5.2 Adaptive Distributed Convolutional Neural Network Inference at the Network Edge with ADCNN (ICPP, 2020)

![](https://gitee.com/molinchn/BlogImage/raw/master/duanyang/20201224145719.jpg)

该研究提出了一种在动态边缘计算环境中运行CNN推理任务的新方法，将卷积层划分为许多小型的独立计算任务，根据边缘节点的当前条件动态地将分区任务分配给边缘节点，以进行并行处理，还提出一个压缩方案，进一步最大限度地降低跨层的通信成本。

