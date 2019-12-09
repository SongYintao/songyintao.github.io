---
title: kubernetes metric and recovery
layout: post
tags: [kubernetes,metric,node-problem-detector]
---

kubernetes监测、自愈方案设计

## 背景

集群中节点数量很多，会经常发生故障、异常。如果没有及时的处理这些，恢复机器，资源效率会比较低下。特别地，一些小的问题，可以通过一些相对应的解决方案，快速恢复。



### 如何快速恢复大规模容器故障

#### 目标

**1-5-10:**

1min发现问题；5分钟定位问题；10min解决问题；















### 有效可靠地管理大规模Kubernetes集群

Kubernetes 的出现使得广大开发同学也能运维复杂的分布式系统，它大幅降低了容器化应用部署的门槛，但运维和管理一个生产级的高可用 Kubernetes 集群仍十分困难。

蚂蚁金服是如何有效可靠地管理大规模 Kubernetes 集群的，并会详细介绍集群管理系统核心组件的设计。

#### 一、系统概述

Kubernetes 集群**管理系统需要具备便捷的集群生命周期管理能力，完成集群的创建、升级和工作节点的管理**。在大规模场景下，集群变更的可控性直接关系到集群的稳定性，因此**管理系统可监控、可灰度、可回滚的能力是系统设计的重点之一**。除此之外，超大规模集群中，节点数量已经达到 10K 量级，**节点硬件故障、组件异常等问题会常态出现**。面向大规模集群的**<u>管理系统在设计之初就需要充分考虑这些异常场景，并能够从这些异常场景中自恢复</u>**。

#####　1. 设计模式

基于这些背景，我们设计了一个**面向终态的集群管理系统**。**<u>系统定时检测集群当前状态，判断是否与目标状态一致，出现不一致时，Operators 会发起一系列操作，驱动集群达到目标状态。</u>**这一设计参考控制理论中常见的负反馈闭环控制系统，系统实现闭环，可以有效抵御系统外部的干扰，在我们的场景下，<u>干扰对应于节点软硬件故障</u>。

![](/Users/nali/songyintao/SongYintao.github.io/img/k8s-1.png)

##### 2. 架构设计

![](/Users/nali/songyintao/SongYintao.github.io/img/k8s-2.png)

元集群是一个高可用的 Kubernetes 集群，用于管理 N 个业务集群的Master 节点。业务集群是一个服务生产业务的 Kubernetes 集群。SigmaBoss 是集群管理入口，为用户提供便捷的交互界面和可控的变更流程。

**元集群**中部署的 **Cluster-Operator** 提供了**业务集群集群创建、删除和升级能力**，**Cluster-Operator 面向终态设计，当业务集群 Master 节点或组件异常时，会自动隔离并进行修复，以保证业务集群 Master 节点达到稳定的终态。**这种采用Kubernetes 管理 Kubernetes 的方案，我们称作 Kube on Kube 方案，简称 KOK方案。

**业务集群**中部署有 <u>**Machine-Operator**</u> 和<u>节点故障自愈组件</u>**用于管理业务集群的工作节点，提供节点新增、删除、升级和故障处理能力**。**在 Machine-Operator提供的单节点终态保持的能力上，SigmaBoss 上构建了集群维度灰度变更和回滚能力**。



#### 二、核心组件

##### 1. 集群终态保持器

基于 K8S CRD，**在元集群中定义了 Cluster CRD 来描述业务集群终态**，**每个业务集群对应一个 Cluster 资源，创建、删除、更新 Cluster 资源对应于实现业务集群创建、删除和升级**。**Cluster-Operator watch Cluster 资源，驱动业务集群Master 组件达到 Cluster 资源描述的终态**。

**业务集群 Master 组件版本集中维护在 ClusterPackageVersion CRD** 中，**Clus-terPackageVersion 资源记录了 Master 组件(如:api-server、controller-manager、scheduler、operators 等)的镜像、默认启动参数等信息**。Cluster 资源唯一关联一个 ClusterPackageVersion，**修改 Cluster CRD 中记录的 ClusterPackageVersion 版本即可完成业务集群 Master 组件发布和回滚**。



##### 2. 节点终态保持器

Kubernetes 集群工作节点的管理任务主要有:

- 节点系统配置、内核补丁管理
- docker / kubelet 等组件安装、升级、卸载
- 节点终态和可调度状态管理(如关键 DaemonSet 部署完成后才允许开启调度)
- 节点故障自愈

为实现上述管理任务，**在业务集群中定义了 Machine CRD 来描述工作节点终态， 每一个工作节点对应一个 <u>Machine 资源</u>，通过修改 Machine 资源来管理工作节点**。 

Machine CRD 定义如下图所示，**spec 中描述了节点需要安装的组件名和版本**， **status 中记录有当前这个工作节点各组件安装运行状态**。除此之外，**Machine CRD 还提供了插件式终态管理能力，用于与其它节点管理 Operators 协作**，这部分会在后文详细介绍。 

![](/Users/nali/songyintao/SongYintao.github.io/img/k8s-3.png)

**工作节点上的组件版本管理**由 **MachinePackageVersion CRD** 完成。**MachinePackageVersion 维护了每个组件的 rpm 版本、配置和安装方法等信息。一个 Machine 资源会关联 N 个不同的 MachinePackageVersion，用来实现安装多个组件。** 

<u>在 Machine、MachinePackageVersion CRD 基 础 上， 设 计 实 现 了 节 点 终 态 控 制 器 Machine-Operator</u>。**Machine-Operator watch Machine 资 源， 解 析 MachinePackageVersion，在节点上执行运维操作来驱动节点达到终态，并持续守护终态**。

 

##### 3. 节点终态管理

随着业务诉求的变化，节点管理已不再局限于安装 docker / kubelet 等组件，我们需要实现如等待日志采集 DaemonSet 部署完成才可以开启调度的需求，而且这类需求变得越来越多。**如果将终态统一交由 Machine-Operator 管理，势必会增加Machine-Operator 与其它组件的耦合性，而且系统的扩展性会受到影响**。因此，
我们设计了一套节点终态管理的机制，来协调 Machine-Operator 和其它节点运维Operators。设计如下图所示:

![](/Users/nali/songyintao/SongYintao.github.io/img/k8s-4.png)

**全量 ReadinessGates**: 记录节点可调度**需要检查的 Condition 列表**
**Condition ConfigMap**: 各节点运维 Operators **终态状态上报 ConfigMap**



协作关系:

1. 外部节点运维 Operators 检测并上报与自己相关的子终态数据至对应的
   Condition ConfigMap;
2. Machine-Operator 根据标签获取节点相关的所有子终态 Condition ConfigMap，并同步至 Machine status 的 conditions 中 ;
3. Machine-Operator 根 据 全 量 ReadinessGates 中 记 录 的 Condition 列 表，检查节点是否达到终态，未达到终态的节点不开启调度 



##### 4.节点故障自愈

​		我们都知道物理机硬件存在一定的故障概率，随着集群节点规模的增加，集群中 会常态出现故障节点，如果不及时修复上线，这部分物理机的资源将会被闲置。 为解决这一问题，**我们设计了一套故障发现、隔离、修复的闭环自愈系统。** 

​		如下图所示，**故障发现方面，采取 Agent 上报和监控系统主动探测相结合的方 式**，**保证了故障发现的实时性和可靠性(Agent 上报实时性比较好，监控系统主动探 测可以覆盖 Agent 异常未上报场景)**。故障信息统一存储于**<u>事件中心</u>**，**关注集群故障的组件或系统都可以订阅事件中心事件拿到这些故障信息**。 



![](/Users/nali/songyintao/SongYintao.github.io/img/k8s-5.png)

​		节点故障自愈系统会根据故障类型创建不同的维修流程，例如:硬件维系流程、系统重装流程等。维修流程中优先会隔离故障节点(暂停节点调度)，然后将节点上Pod 打上待迁移标签来通知 PAAS 或 MigrateController 进行 Pod 迁移，完成这些前置操作后，会尝试恢复节点(硬件维修、重装操作系统等)，修复成功的节点会重新开启调度，长期未自动修复的节点由人工介入排查处理。

![](/Users/nali/songyintao/SongYintao.github.io/img/k8s-6.png)

##### 5. 风险防范

在 Machine-Operator 提供的原子能力基础上，系统中设计实现了集群维度的灰度变更和回滚能力。此外，为了进一步降低变更风险，Operators 在发起真实变更时都会进行风险评估，架构示意图如下。

![](/Users/nali/songyintao/SongYintao.github.io/img/k8s-7.png)



高风险变更操作(如:删除节点、重装系统)接入统一限流中心，限流中心维护 了不同类型操作的限流策略，若触发限流，则熔断变更。 

为了评估变更过程是否正常，我们会在变更前后，对各组件进行健康检查，组件 的健康检查虽然能够发现大部分异常，但不能覆盖所有异常场景。所以，风险评估过 程中，系统会从事件中心、监控系统中获取集群业务指标(如:Pod 创建成功率)，如 果出现异常指标，则自动熔断变更。 

