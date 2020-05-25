ApiServer创建流程



## 配置

首先，我们通过配置来了解一下ApiServer包含了什么东西，这些配置的意义是什么。

再根据这些配置，来反向解析具体代码的实现。







### CreateNodeDialer

主要进行了以下几件事：

（1）调用CreateNodeDialer，创建与节点交互的工具。

（2）配置API Server的Config。这里同时还配置了Extension API Server的Config，用于配置用户自己编写的API Server。

（3）根据Config，创建API Server和Extension API Server。

（4）运行API Server。通过调用PrepareRun方法实现。

（5）创建并运行aggregator（将API Server和Extension API Server整合在一起，暂时不提）。



apiExtensionsServer：

kubeAPIServer：

aggregatorServer：