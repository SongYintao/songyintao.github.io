---
title: 分布式系统一致性原理、实现对比分析
subtitle: 服务发现比较
layout: post
tags: [zk,etcd,consul,eureka]
---

服务发现组件很多，各自的优缺点及其实现原理深入了解研究一下。

# 背景



## 一、组件对比

| 特性                   | Consul                 | zookeeper               | etcd             | eureka                       |
| ---------------------- | ---------------------- | ----------------------- | ---------------- | ---------------------------- |
| 健康检查               | 服务状态、内存、磁盘等 | 长连接（弱），keepalive | 连接心跳         | 可配支持                     |
| 多数据中心             | 支持                   | —                       | —                | —                            |
| kv存储服务             | 支持                   | 支持                    | 支持             | —                            |
| 一致性                 | raft                   | paxos                   | raft             | —                            |
| CAP                    | cp                     | cp                      | cp               | ap                           |
| 使用接口（多语言支持） | 支持http、dns          | 客户端                  | http、grpc       | http（sidecar）              |
| watch支持              | 全量／支持long polling | 支持                    | 支持long polling | 支持 long polling/大部分增量 |
| 自身监控               | metrics                | —                       | metrics          | metrics                      |
| 安全                   | acl /https             | acl                     | https支持（弱）  | —                            |
| spring cloud集成       | 已支持                 | 已支持                  | 已支持           | 已支持                       |



#### 1. 健康检查

Euraka 使用时需要显式配置健康检查；

Zookeeper,Etcd 则在失去了和服务进程的连接情况下任务不健康；

 Consul 相对更为详细点，比如内存是否已使用了90%，文件系统的空间是不是快不足了。

#### 2. 多数据中心

Consul 通过 WAN 的 Gossip 协议，完成跨数据中心的同步；而且其他的产品则需要额外的开发工作来实现；

#### 3. KV 存储服务

除了 Eureka ,其他几款都能够对外支持 k-v 的存储服务，所以后面会讲到这几款产品追求高一致性的重要原因。而提供存储服务，也能够较好的转化为动态配置服务。

#### 4.产品设计中 CAP 理论的取舍

Eureka 典型的 AP,作为分布式场景下的服务发现的产品较为合适，服务发现场景的可用性优先级较高，一致性并不是特别致命。

其次 CA 类型的场景 Consul,也能提供较高的可用性，并能 k-v store 服务保证一致性。 

而Zookeeper,Etcd则是CP类型 牺牲可用性，在服务发现场景并没太大优势；

#### 5. 多语言能力与对外提供服务的接入协议

Zookeeper的跨语言支持较弱，其他几款支持 http11 提供接入的可能。

Euraka 一般通过 sidecar的方式提供多语言客户端的接入支持。

Etcd 还提供了Grpc的支持。

 Consul除了标准的Rest服务api,还提供了DNS的支持。

#### 6. Watch的支持（客户端观察到服务提供者变化）

Zookeeper 支持服务器端推送变化。 Eureka 1,Consul,Etcd则都通过长轮询的方式来实现变化的感知；

#### 7.自身集群的监控

除了 Zookeeper ,其他几款都默认支持 metrics，运维者可以搜集并报警这些度量信息达到监控目的；

#### 8.安全

Consul,Zookeeper 支持ACL，另外 Consul,Etcd 支持安全通道https.

#### 9. Spring Cloud的集成

目前都有相对应的 boot starter，提供了集成能力。

总的来看，目前Consul 自身功能，和 spring cloud 对其集成的支持都相对较为完善，而且运维的复杂度较为简单（没有详细列出讨论），Eureka 设计上比较符合场景，但还需持续的完善。



# 基础知识

[Paxos、Raft分布式一致性最佳实践](https://zhuanlan.zhihu.com/p/31727291)

主要说明分布式一致性问题与分布式一致性算法的典型应用场景，帮助后面大家更好的理解Paxos、Raft等分布式一致性算法。

## 一、分布式一致性 (Consensus)

分布式一致性问题，简单的说，就是在一个或多个进程提议了一个值应当是什么后，使系统中所有进程对这个值达成一致意见。

这样的协定问题在分布式系统中很常用，比如：

- **领导者选举（leader election）**：进程对leader达成一致；
- **互斥（mutual exclusion）**：进程对进入临界区的进程达成一致；
- **原子广播（atomic broadcast）**：进程对消息传递（delivery）顺序达成一致。

对于这些问题有一些特定的算法，但是，分布式一致性问题试图探讨这些问题的一个更一般的形式，如果能够解决分布式一致性问题，则以上的问题都可以解决。

**分布式一致性问题**

> - 为了达成一致，每个进程都提出自己的提议（propose）
> - 最终通过分布式一致性算法，所有正确运行的进程决定（decide）相同的值。

定义如下图所示：



![img](https://pic3.zhimg.com/80/v2-fec5a5ee8ee501ffcec3c0a48ce60e12_hd.jpg)



实例：

![img](https://pic1.zhimg.com/80/v2-3042b89d939ba9acb765ebcda23f5bd4_hd.jpg)



**分布式系统存在的问题：**

> 实际分布式系统一般是基于消息传递的异步分布式系统。
>
> - 进程可能会慢、被杀死或者重启；
> - 消息可能会延迟、丢失、重复、乱序等。

**一致性算法的挑战**：

> - 在一个可能发生上述异常的分布式系统中，**如何就某个值达成一致，形成一致的决议**；
> - 保证不论发生以上任何异常，都**不会破坏决议的一致性**。



## 二、分布式一致性算法典型应用场景

我们在分布式存储系统中经常使用多副本的方式实现容错，每一份数据都保存多个副本，这样部分副本的失效不会导致数据的丢失。每次更新操作都需要更新数据的所有副本，使多个副本的数据保持一致。

####  0.问题

如何在一个可能出现各种故障的异步分布式系统中保证同一数据的多个副本的一致性 (Consistency) 呢？

#### 1. 传统的主从同步

以最简单的两副本为例，首先来看看传统的**主从同步方式**。

![img](https://pic1.zhimg.com/80/v2-6a26bdb3f035ebcabb18691703a89600_hd.jpg)



**过程**：写请求首先发送给主副本，主副本同步更新到其它副本后返回。

**优势**：这种方式可以保证副本之间数据的强一致性，写成功返回之后从任意副本读到的数据都是一致的。

**劣势**：但是可用性很差，只要任意一个副本写失败，写请求将执行失败。



![img](https://pic2.zhimg.com/80/v2-6cbe60837d44e7eceb2f000dc66d0c65_hd.jpg)





#### 2. 主从同步的弱可用性

如果采用异步复制的方式，主副本写成功后立即返回，然后在后台异步的更新其它副本。

##### 2.1 主从异步复制

![img](https://pic1.zhimg.com/80/v2-67145cc44047cdbb0a7d55014f4d13bc_hd.jpg)

**过程**：写请求首先发送给主副本，主副本写成功后立即返回，然后异步的更新其它副本。

**优势**：这种方式可用性较好，只要主副本写成功，写请求就执行成功。

**劣势**：但是不能保证副本之间数据的强一致性，写成功返回之后从各个副本读取到的数据不保证一致，只有主副本上是最新的数据，其它副本上的数据落后，只提供最终一致性。

##### 2.2 异步复制失败

![img](https://pic1.zhimg.com/80/v2-73d8d3ea3ae1ad5af6d9669982a88d70_hd.jpg)

如果出现断网导致后台异步复制失败，则主副本和其它副本将长时间不一致，其它副本上的数据一直无法更新，直到网络重新连通。

##### 2.3 主副本写成功后立即宕机

![img](https://pic1.zhimg.com/80/v2-b47a1eaada575a8fda1af357718e6410_hd.jpg)

如果主副本在写请求成功返回之后和更新其它副本之前宕机失效，则会**造成成功写入的数据丢失，一致性被破坏**。

#### 3. 应用示例

在Oracle里面，上述同步和异步复制方式分别对应`Oracle Data Guard`的一种**数据保护模式**。

- **同步复制**为**最高保护模式 (Maximum Protection)**
- **异步复制**为**最高性能模式 (Maximum Performance)**
- **最高可用性模式 (Maximum Availability) 介于两者之间**，在正常情况下，它和最高保护模式一样，但一旦同步出现故障，立即切换成最高性能模式。

#### 4. 存在的问题

**传统的主从同步无法同时保证数据的一致性和可用性**，此问题是典型的分布式系统中**一致性**和**可用性**不可兼得的例子，分布式系统中著名的CAP理论从理论上证明了这个问题。

**而Paxos、Raft等分布式一致性算法则可在一致性和可用性之间取得很好的平衡，在保证一定的可用性的同时，能够对外提供强一致性，因此Paxos、Raft等分布式一致性算法被广泛的用于管理副本的一致性，提供高可用性**。



## 三、CAP理论

CAP理论是分布式系统、特别是分布式存储领域中被讨论的最多的理论。其中C代表**一致性 (Consistency)**，A代表**可用性 (Availability)**，P代表**分区容错性 (Partition tolerance)**。CAP理论告诉我们C、A、P三者不能同时满足，最多只能满足其中两个。

![img](https://pic4.zhimg.com/80/v2-44e5fb78a506c61d3f03169b8ab43c27_hd.jpg)

**CAP理论**

- **一致性** (Consistency)：一个写操作返回成功，那么之后的读请求都必须读到这个新数据；如果返回失败，那么所有读操作都不能读到这个数据。所有节点访问同一份最新的数据。
- **可用性** (Availability)：对数据更新具备高可用性，请求能够及时处理，不会一直等待，即使出现节点失效。
- **分区容错性** (Partition tolerance)：能容忍网络分区，在网络断开的情况下，被分隔的节点仍能正常对外提供服务。



理解CAP理论最简单的方式是**想象两个副本处于分区两侧，即两个副本之间的网络断开，不能通信**。

> - 如果允许其中一个副本更新，则会导致数据不一致，即丧失了C性质。
> - 如果为了保证一致性，将分区某一侧的副本设置为不可用，那么又丧失了A性质。
> - 除非两个副本可以互相通信，才能既保证C又保证A，这又会导致丧失P性质。



一般来说**<u>使用网络通信的分布式系统，无法舍弃P性质</u>**，**那么就只能<u>在一致性和可用性</u>上做一个艰难的选择**。

CAP理论的表述很好地服务了它的目的，开阔了分布式系统设计者的思路，在多样化的取舍方案下设计出多样化的系统。在过去的十几年里确实涌现了不计其数的新系统，也随之**在一致性和可用性的相对关系上产生了相当多的争论。**

既然在分布式系统中一致性和可用性只能选一个。**那Paxos、Raft等分布式一致性算法是如何做到在保证一定的可用性的同时，对外提供强一致性呢**。

在CAP理论提出十二年之后，其作者又出来辟谣。“三选二”的公式一直存在着误导性，它会过分简单化各性质之间的相互关系：

- 首先，**由于分区很少发生，那么在系统不存在分区的情况下没什么理由牺牲C或A**。
- 其次，C与A之间的取舍可以在同一系统内以非常细小的粒度反复发生，而每一次的决策可能因为具体的操作，乃至因为牵涉到特定的数据或用户而有所不同。
- 最后，这三种性质都可以在程度上衡量，并不是非黑即白的有或无。**可用性显然是在0%到100%之间连续变化的，一致性分很多级别，连分区也可以细分为不同含义，如系统内的不同部分对于是否存在分区可以有不一样的认知。**

所以一致性和可用性并不是水火不容，非此即彼的。Paxos、Raft等分布式一致性算法就是在一致性和可用性之间做到了很好的平衡的见证。



## 四、多副本状态机

#### 1. 背景知识

将多副本管理的模型抽象出来，可得到一个通用的模型：**多副本状态机 (Replicated State Machine)** 。

> **多副本状态机**是指多台机器具有完全相同的状态，并且运行完全相同的**确定性状态机**。

通过使用这样的状态机，**可以解决很多分布式系统中的容错问题**。

**优点：**

因为多副本状态机通常**可以容忍半数节点故障**，且**所有正常运行的副本节点状态都完全一致**，所以可以使用多副本状态机来实现需要避免单点故障的组件。

**使用场景：**

> **多副本状态机**在分布式系统中被用于解决各种容错问题。**比如，集中式的选主或是互斥算法中的协调者 (Coordinator)**。
>
> - 集中式的领导者或互斥算法逻辑简单
> - 但**最大的问题是协调者的单点故障问题**
> - 通过采用**多副本状态机**来实现协调者实现了高可用的“单点”，回避了单点故障。

 **高可用"单点"的集中式架构**

![img](https://pic3.zhimg.com/80/v2-94203ffad7b259917a89ed212cfb92be_hd.jpg)



`GFS，HDFS`等典型地使用一个独立的多副本状态机来管理领导者选举与保存集群配置信息，以备节点宕机后信息能够保持。`Chubby`与`ZooKeeper`等都是使用多副本状态机的例子。

**核心逻辑：**

多副本状态机的**每个副本上都保存有完全相同的操作日志，保证所有状态机副本按照相同的顺序执行相同的操作，这样由于状态机是确定性的，则会得到相同的状态**。



#### 2. 多副本状态机结构

![img](https://pic4.zhimg.com/80/v2-af8b01bd38458b43eff9fa0d8191fc43_hd.jpg)

每个服务器在日志中保存一系列命令，所有的状态机副本按照同样的顺序执行，**分布式一致性算法**管理着来自客户端的包含状态机命令的日志复制，每条日志以同样的顺序保存同样的命令，因此**每个状态机执行同样的命令序列**。因为状态机是确定的，每个都计算出同样的状态与同样的输出。

**保证复制到各个服务器上的日志的一致性正是分布式一致性算法的工作**。一致性算法**保证所有状态机副本上的操作日志具有完全相同的顺序，如果状态机的任何一个副本在本地状态机上执行了一个操作，则绝对不会有别的副本在操作序列相同位置执行一个不同的操作**。

具体流程：

- 服务器上的**一致性模块**，接收来自客户端的命令，并且添加到他们的日志中。

- 它**与其它服务器的一致性模块通讯**，以确保每个日志最终以同样的顺序保存同样的请求，即使有些服务器失败。
- 一旦命令**被适当地复制**，每台服务器的状态机以日志的顺序执行，将输出返回给客户端。
- 结果多台服务器就像来自单个高度一致的高可用状态机。





# 历史协议

2PC (两阶段提交)协议和3PC (三阶段提交)协议本身其实很简单. 我尽量通过少写字, 多上图的方法, 用一个coordinator和三个voter达成共识的例子来说明这两个协议的想法.

## 2PC (两阶段提交)协议

#### 1. 2PC的原理

顾名思义, 2PC协议有两个阶段:Propose和Commit. **在无failure情况下**的2PC协议流程的画风是这样的:

- Propose阶段:

- - coordinator: "昨夜验人有惊喜, 今天都投票出六娃"
  - voter1/voter2/voter3: "好的女王大人!"

- Commit阶段

- - coordinator: "投六娃"
  - voter1/voter2/voter3: "投了女王大人!" (画外音: 六娃扑街)

![img](https://pic1.zhimg.com/80/v2-e2f7149a81d9ad3aa46589e25503d688_hd.jpg)

​									图1: 2PC, coordinator提议通过, voter{1,2,3}达成新的共识

如果有至少一个voter (比如voter3)在Propose阶段投了反对票, 那么propose通过失败. coordinator就会在Commit(or abort)阶段跟所有voter说, 放弃这个propose.

![img](https://pic4.zhimg.com/80/v2-d40abfa365ed84e84e264ba13900f64b_hd.jpg)

​									图2: 2PC, coordinator提议没有通过, voter{1,2,3}保持旧有的共识

#### 2. 2PC的缺陷

2PC的缺点在于**不能处理fail-stop形式的节点failure.** 比如下图这种情况. 

假设**coordinator和voter3都在Commit这个阶段crash了**, 而voter1和voter2没有收到commit消息. 这时候voter1和voter2就陷入了一个困境. 因为他们并不能判断现在是两个场景中的哪一种:

 (1)上轮全票通过然后voter3第一个收到了commit的消息并在commit操作之后crash了

(2)上轮voter3反对所以干脆没有通过.

![img](https://pic3.zhimg.com/80/v2-a9e4ef8b9082ffdf76bc426e61ba3ed2_hd.jpg)

​								图3: 2PC, coordinator和voter3 crash, **voter{1,2}无法判断当前状态而卡死**

2PC在这种fail-stop情况下会失败是因为voter在得知Propose Phase结果后就直接commit了, 而并没有在commit之前告知其他voter自己已收到Propose Phase的结果. **从而导致在coordinator和一个voter双双掉线的情况下, 其余voter不但无法复原Propose Phase的结果, 也无法知道掉线的voter是否打算甚至已经commit**. 为了解决这一问题, 3PC了解一下.



------

## 3PC (三阶段提交)协议

#### 1. 3PC的原理

简单的说来, 3PC就是**把2PC的Commit阶段拆成了PreCommit和Commit两个阶段**. 

- 通过进入增加的这一个PreCommit阶段, voter可以得到Propose阶段的投票结果, 但不会commit; 
- 而通过进入Commit阶段, voter可以盘出其他每个voter也都打算commit了, 从而可以放心的commit.

**换言之, 3PC在2PC的Commit阶段里增加了一个barrier**(即相当于告诉其他所有voter, 我**收到了Propose的结果**啦). 

- 在这个barrier之前coordinator掉线的话, 其他voter可以得出结论不是每个voter都收到Propose Phase的结果, 从而放弃或选出新的coordinator;
-  **在这个barrier之后coordinator掉线的话, 每个voter会放心的commit, 因为他们知道其他voter也都做同样的计划**.

![img](https://pic2.zhimg.com/80/v2-28c17c86e689007015a4853f0d0c4a89_hd.jpg)

​									图4: 3PC, coordinator提议通过, voter{1,2,3}达成新的共识

#### 2.3PC的缺陷

##### 1. 网络分区

3PC可以有效的处理fail-stop的模式, 但**不能处理网络划分(network partition)的情况---节点互相不能通信**. 假设在PreCommit阶段所有节点被一分为二, 收到preCommit消息的voter在一边, 而没有收到这个消息的在另外一边. 在这种情况下, **两边就可能会选出新的coordinator而做出不同的决定**.

![img](https://pic4.zhimg.com/80/v2-7a18adc27a2bd7b5f5926dd999bc7bb3_hd.jpg)

​										图5: 3PC, network partition, voter{1,2,3}失去共识

##### 2. fail-recover

除了网络划分以外, **3PC也不能处理fail-recover的错误情况**. 

简单说来**当coordinator收到preCommit的确认前crash**, 于是其他某一个voter接替了原coordinator的任务而开始组织所有voter commit. 而与此同时原coordinator重启后又回到了网络中, 开始继续之前的回合---发送abort给各位voter因为它并没有收到preCommit. 

此时有可能会出现**原coordinator和继任的coordinator给不同节点发送相矛盾的commit和abort指令, 从而出现节点的状态分歧.**

这种情况等价于一个更真实或者更负责的网络环境假设: 异步网络. 在这种假设下, 网络传输时间可能任意长. 为了解决这种情况, 那就得请出下一篇的主角: Paxos

## 总结

1. 2PC使用两个roundtrip来达成新的共识或维持旧有的共识. 其局限性在于不能保证有节点永久性崩溃(fail-stop)的情况下算法能向前推进;
2. 3PC扩展了2PC, 使用三个roundtrip达成共识. 其局限性在于不能保证在节点暂时性崩溃(fail-recover), 或是有网络划分的情况下, 共识依旧成立.

## 推荐阅读

- [Consensus Protocols: Two-Phase Commit - Paper Trail](https://link.zhihu.com/?target=http%3A//the-paper-trail.org/blog/consensus-protocols-two-phase-commit/)
- [Consensus Protocols: Three-phase Commit - Paper Trail](https://link.zhihu.com/?target=http%3A//the-paper-trail.org/blog/consensus-protocols-three-phase-commit/)
- [In distributed systems, what is a simple explanation of the Paxos algorithm?](https://link.zhihu.com/?target=https%3A//www.quora.com/In-distributed-systems-what-is-a-simple-explanation-of-the-Paxos-algorithm)



# Paxos简介

上面讲述了分布式一致性问题与分布式一致性算法的典型应用场景。作为分布式一致性代名词的Paxos算法号称是最难理解的算法。本文试图用通俗易懂的语言讲述Paxos算法。

## 一、Paxos算法背景

Paxos算法是Lamport宗师提出的一种基于消息传递的分布式一致性算法，使其获得2013年图灵奖。

Paxos由Lamport于1998年在《The Part-Time Parliament》论文中首次公开，最初的描述使用希腊的一个小岛Paxos作为比喻，描述了Paxos小岛中通过决议的流程，并以此命名这个算法，但是这个描述理解起来比较有挑战性。后来在2001年，Lamport觉得同行不能理解他的幽默感，于是重新发表了朴实的算法描述版本《Paxos Made Simple》。

自Paxos问世以来就持续垄断了**分布式一致性算法**，**Paxos这个名词几乎等同于分布式一致性**。Google的很多大型分布式系统都采用了Paxos算法来解决分布式一致性问题，如Chubby、Megastore以及Spanner等。开源的ZooKeeper，以及MySQL 5.7推出的用来取代传统的主从复制的MySQL Group Replication等纷纷采用Paxos算法解决分布式一致性问题。

然而，Paxos的最大特点就是难，不仅难以理解，更难以实现。

## 二、Paxos算法流程

**Paxos算法解决的问题**：

分布式一致性问题，即**一个分布式系统中的各个进程如何就某个值（决议）达成一致**。

**适用场景：**

Paxos算法运行在允许宕机故障的异步系统中，不要求可靠的消息传递，可容忍消息丢失、延迟、乱序以及重复。它利用大多数 (Majority) 机制保证了2F+1的容错能力，即2F+1个节点的系统最多允许F个节点同时出现故障。

**简单流程**：

一个或多个`提议进程 (Proposer)` 可以发起`提案 (Proposal)`，`Paxos算法`使所有提案中的某一个提案，在所有进程中达成一致。系统中的多数派同时认可该提案，即达成了一致。最多只针对一个确定的提案达成一致。

Paxos将系统中的角色分为`提议者 (Proposer)`，`决策者 (Acceptor)`和`最终决策学习者 (Learner)`:

- **Proposer**: 提出提案 (Proposal)。Proposal信息包括**提案编号** (Proposal ID) 和**提议的值** (Value)。
- **Acceptor**：参与决策，回应Proposers的提案。收到Proposal后可以接受提案，若Proposal获得多数Acceptors的接受，则称该Proposal被批准。
- **Learner**：不参与决策，从Proposers/Acceptors学习最新达成一致的提案（Value）。

在`多副本状态机`中，每个副本**同时具有Proposer、Acceptor、Learner三种角色**。

![img](https://pic1.zhimg.com/80/v2-2c0d971fcca713a8e045a93d7881aedc_hd.jpg)



Paxos算法通过一个决议分为两个阶段（Learn阶段之前决议已经形成）：

1. **第一阶段**：Prepare阶段。`Proposer`向`Acceptors`发出Prepare请求，`Acceptors`针对收到的Prepare请求进行**Promise承诺**。
2. **第二阶段**：Accept阶段。`Proposer`收到多数`Acceptors`承诺的Promise后，**向`Acceptors`发出Propose请求，`Acceptors`针对收到的Propose请求进行Accept处理**。
3. **第三阶段**：Learn阶段。**`Proposer`在收到多数`Acceptors`的Accept之后，标志着本次Accept成功，决议形成，将形成的决议发送给所有`Learners`**。

![img](https://pic2.zhimg.com/80/v2-a6cd35d4045134b703f9d125b1ce9671_hd.jpg)



Paxos算法流程中的每条消息描述如下：

- **Prepare**:` Proposer`**生成全局唯一且递增的Proposal ID** (可使用时间戳加Server ID)，**向所有`Acceptors`发送Prepare请求**，这里无需携带提案内容，只携带Proposal ID即可。
- **Promise**: `Acceptors`收到Prepare请求后，做出“**两个承诺，一个应答**”。

> **两个承诺**：
>
> 1. **不再接受Proposal ID小于等于（注意：这里是<= ）当前请求的Prepare请求**。
>
> 2. **不再接受Proposal ID小于（注意：这里是< ）当前请求的Propose请求**。
>
> **一个应答：**
>
> 1. 不违背以前作出的承诺下，**回复已经Accept过的提案中Proposal ID最大的那个提案的Value和Proposal ID，没有则返回空值**。

- **Propose**: `Proposer` **收到多数`Acceptors`的Promise应答后，从应答中选择Proposal ID最大的提案的Value，作为本次要发起的提案。**如**果所有应答的提案Value均为空值，则可以自己随意决定提案Value**。**然后携带当前Proposal ID，向所有`Acceptors`发送Propose请求。**
- **Accept**: `Acceptor`收到Propose请求后，**在不违背自己之前作出的承诺下，接受并持久化当前Proposal ID和提案Value。**
- **Learn**: **`Proposer`收到多数`Acceptors`的Accept后，决议形成，将形成的决议发送给所有Learners**。



**Paxos算法伪代码描述如下：**

![img](https://pic2.zhimg.com/80/v2-8d4eaf5fdeb145e8bdf5e3bb1af408c9_hd.jpg)

1. 获取一个Proposal ID n，为了保证Proposal ID唯一，可采用时间戳+Server ID生成；

2. `Proposer`**向所有Acceptors广播Prepare(n)请求**；

3. `Acceptor`比较n和minProposal，如果n>minProposal，minProposal=n，**并且将 acceptedProposal 和 acceptedValue 返回；**

4. **Proposer接收到过半数回复后，如果发现有acceptedValue返回，将所有回复中acceptedProposal最大的acceptedValue作为本次提案的value，否则可以任意决定本次提案的value**；

5. 到这里可以进入第二阶段，广播Accept (n,value) 到所有节点；

6. Acceptor比较n和minProposal，如果n>=minProposal，则acceptedProposal=minProposal=n，acceptedValue=value，本地持久化后，返回；否则，返回minProposal。

7. 提议者接收到过半数请求后，如果发现有返回值result >n，表示有更新的提议，跳转到1；否则value达成一致。

   

下面举几个例子

#### 1. 实例1

![img](https://pic1.zhimg.com/80/v2-ac7e4a827f77dc57d316c77ae95e1940_hd.jpg)

图中P代表Prepare阶段，A代表Accept阶段。3.1代表Proposal ID为3.1，其中3为时间戳，1为Server ID。X和Y代表提议Value。

实例1中**P 3.1达成多数派，其Value(X)被Accept**，然后**P 4.5学习到Value(X)，并Accept**。



#### 2. 实例2

![img](https://pic2.zhimg.com/80/v2-3ae48cb81d39079022666ccb35821c71_hd.jpg)



实例2中**P 3.1没有被多数派Accept（只有S3 Accept）**，**但是被P 4.5学习到，P 4.5将自己的Value由Y替换为X，Accept（X）。**



#### 3. 实例3

![img](https://pic2.zhimg.com/80/v2-931f9487900f0f002867c9e116dec255_hd.jpg)



实例3中**P 3.1没有被多数派Accept（只有S1 Accept），同时也没有被P 4.5学习到**。

由于**P 4.5 Propose的所有应答，均未返回Value，则P 4.5可以Accept自己的Value (Y)。后续P 3.1的Accept (X) 会失败，已经Accept的S1，会被覆盖。**

#### 4. Paxos算法形成活锁

Paxos算法可能形成活锁而永远不会结束，如下图实例所示：

![img](https://pic1.zhimg.com/80/v2-0e18b29659367076ff1c0156ae46eca0_hd.jpg)

回顾两个承诺之一，`Acceptor`不再应答`Proposal ID`小于等于当前请求的Prepare请求。意味着**需要应答Proposal ID大于当前请求的Prepare请求**。

两个Proposers交替Prepare成功，而Accept失败，形成活锁（Livelock）。

## 三、Multi-Paxos算法

**Basic Paxos缺点：**

- 只能对一个值形成决议；
- 决议的形成至少需要两次网络来回，在高并发情况下可能需要更多的网络来回；
- 极端情况下甚至可能形成活锁；
- 如果想连续确定多个值，Basic Paxos搞不定了。

因此**Basic Paxos几乎只是用来做理论研究，并不直接应用在实际工程中**。



实际应用中几乎都需要连续确定多个值，而且希望能有更高的效率。Multi-Paxos正是为解决此问题而提出。



**Multi-Paxos基于Basic Paxos做了两点改进**：

1. 针对每一个要确定的值，运行一次Paxos算法实例（Instance），形成决议。每一个Paxos实例使用唯一的Instance ID标识。
2. **在所有Proposers中选举一个Leader，由Leader唯一地提交Proposal给Acceptors进行表决**。这样**<u>没有Proposer竞争，解决了活锁问题</u>**。在系统中仅有一个Leader进行Value提交的情况下，Prepare阶段就可以跳过，从而将两阶段变为一阶段，提高效率。



#### 1.Multi-Paxos流程

![img](https://pic1.zhimg.com/80/v2-e5cd197abc9c922ca4ca91c3df74fa70_hd.jpg)

 **前置条件：**

- Multi-Paxos**首先需要选举Leader**，Leader的确定也是一次决议的形成，所以**可执行一次Basic Paxos实例来选举出一个Leader**。
- 选出Leader之后只能由Leader提交Proposal。
- 在Leader宕机之后服务临时不可用，需要重新选举Leader继续服务。

**在系统中仅有一个Leader进行Proposal提交的情况下，Prepare阶段可以跳过**。

Multi-Paxos通过改变Prepare阶段的作用范围至后面Leader提交的所有实例，从而使得**Leader的连续提交只需要执行一次Prepare阶段，后续只需要执行Accept阶段，将两阶段变为一阶段，提高了效率**。**为了区分连续提交的多个实例，每个实例使用一个Instance ID标识，Instance ID由Leader本地递增生成即可**。

**Multi-Paxos允许有多个自认为是Leader的节点并发提交Proposal而不影响其安全性，这样的场景即退化为Basic Paxos。**

Chubby和Boxwood均使用Multi-Paxos。ZooKeeper使用的Zab也是Multi-Paxos的变形。



## 四、附Paxos算法推导过程

Paxos算法的设计过程就是从正确性开始的，对于分布式一致性问题，很多进程提出（Propose）不同的值，共识算法保证最终只有其中一个值被选定，Safety表述如下：

- 只有被提出（Propose）的值才可能被最终选定（Chosen）。
- 只有**一**个值会被选定（Chosen）。
- 进程只会获知到已经确认被选定（Chosen）的值。

Paxos**以这几条约束作为出发点进行设计**，只要算法最终满足这几点，正确性就不需要证明了。



**Paxos算法中共分为三种参与者：Proposer、Acceptor以及Learner，通常实现中每个进程都同时扮演这三个角色。**

Proposers向Acceptors提出Proposal，为了保证最多只有**一**个值被选定（Chosen），Proposal必须被超过一半的Acceptors所接受（Accept），且每个Acceptor只能接受一个值。



为了保证正常运行（必须有值被接受），所以Paxos算法中：



**P1：Acceptor必须接受（Accept）它所收到的第一个Proposal。**



先来先服务，合情合理。但这样产生一个问题，如果多个Proposers同时提出Proposal，很可能会导致无法达成一致，因为没有Propopal被超过一半Acceptors的接受，因此，Acceptor必须能够接受多个Proposal，不同的Proposal由不同的编号进行区分，当某个Proposal被超过一半的Acceptors接受后，这个Proposal就被选定了。

既然允许Acceptors接受多个Proposal就有可能出现多个不同值都被最终选定的情况，这违背了Safety要求，为了保证Safety要求，Paxos进一步提出：



**P2：如果值为v的Proposal被选定（Chosen），则任何被选定（Chosen）的具有更高编号的Proposal值也一定为v。**



只要算法同时满足**P1**和**P2**，就保证了Safety。**P2**是一个比较宽泛的约定，完全没有算法细节，我们对其进一步延伸：



**P2a：如果值为v的Proposal被选定（Chosen），则对所有的Acceptors，它们接受（Accept）的任何具有更高编号的Proposal值也一定为v。**

如果满足**P2a**则一定满足**P2**，显然，因为只有首先被接受才有可能被最终选定。但是**P2a**依然难以实现，因为acceptor很有可能并不知道之前被选定的Proposal（恰好不在接受它的多数派中），因此进一步延伸：

**P2b：如果值为v的Proposal被选定（Chosen），则对所有的Proposer，它们提出的的任何具有更高编号的Proposal值也一定为v。**

更进一步的：

**P2c：为了提出值为v且编号为n的Proposal，必须存在一个包含超过一半Acceptors的集合S，满足(1) 没有任何S中的Acceptors曾经接受（Accept）过编号比n小的Proposal，或者(2) v和S中的Acceptors所接受过(Accept)的编号最大且小于n的Proposal值一致。**



满足**P2c**即满足**P2b**即满足**P2a**即满足**P2**。至此Paxos提出了Proposer的执行流程，以满足**P2c**：

1. Proposer选择一个新的编号n，向超过一半的Acceptors发送请求消息，Acceptor回复: (a)承诺不会接受编号比n小的proposal，以及(b)它所接受过的编号比n小的最大Proposal（如果有）。该请求称为Prepare请求。
2. 如果Proposer收到超过一半Acceptors的回复，它就可以提出Proposal，Proposal的值为收到回复中编号最大的Proposal的值，如果没有这样的值，则可以自由提出任何值。
3. 向收到回复的Acceptors发送Accept请求，请求对方接受提出的Proposal。

仔细品味Proposer的执行流程，其完全吻合**P2c**中的要求，但你可能也发现了，当多个Proposer同时运行时，有可能出现没有任何Proposal可以成功被接受的情况（编号递增的交替完成第一步），这就是Paxos算法的Liveness问题，或者叫“活锁”，论文中建议通过对Proposers引入选主算法选出Distinguished Proposer来全权负责提出Proposal来解决这个问题，但是即使在出现多个Proposers同时提出Proposal的情况时，Paxos算法也可以保证Safety。



接下来看看Acceptors的执行过程，和我们对**P2**做的事情一样，我们对**P1**进行延伸：



**P1a：Acceptor可以接受（Accept）编号为n的Proposal当且仅当它没有回复过一个具有更大编号的Prepare消息。**



易见，**P1a**包含了**P1**，对于Acceptors：

1. 当收到Prepare请求时，如果其编号n大于之前所收到的Prepare消息，则回复。
2. 当收到Accept请求时，仅当它没有回复过一个具有更大编号的Prepare消息，接受该Proposal并回复。

以上涵盖了满足**P1a**和**P2b**的一套完整一致性算法。



## 五、Zab简介

Zab也是一个**强一致性算法**，也是**multi-Paxos**的一种，全称是**Zookeeper atomic broadcast protocol**，是Zookeeper内部用到的一致性协议。相比Paxos，也易于理解。其保证了消息的全局有序和因果有序，拥有强一致性。Zab和Raft也是非常相似的，只是其中有些概念名词不一样。

### 一、基本概念

**节点状态：**

- **Leading**：说明当前节点为Leader

- **Following**：说明当前节点为Follower

- **Election**：说明节点处于选举状态。整个集群都处于选举状态中。

**Epoch逻辑时钟：**

Epoch相当于paxos中的proposerID，Raft中的term，相当于一个国家，朝代纪元。

**epoch**可以理解为当前集群所处的年代或者周期，每个 leader 就像皇帝，都有自己的年号，所以每次改朝换代，leader 变更之后，都会在前一个年代的基础上加 1。这样就算旧的 leader 崩溃恢复之后，也没有人听他的了，因为 follower 只听从当前年代的 leader 的命令。

**Quorums：**

多数派，集群中超过半数的节点集合。

**节点中的持久化信息：**

- **history:**a log of transaction proposals accepted; **历史提议日志文件**

- **acceptedEpoch:**the epoch number of the last NEWEPOCH message accepted; **集群中的最近最新Epoch**

- **currentEpoch:**the epoch number of the last NEWLEADER message accepted; **集群中的最近最新Leader的Epoch**

- **lastZxid:**zxid of the last proposal in the history log; **历史提议日志文件的最后一个提议的zxid**

> 在 ZAB 协议的事务编号 Zxid 设计中，**Zxid**是一个 64 位的数字，
>
> - 低 32 位是一个简单的单调递增的计数器，针对客户端每一个事务请求，计数器加 1；
> - 高 32 位则代表 Leader 周期 epoch 的编号，每个当选产生一个新的 Leader 服务器，就会从这个 Leader 服务器上取出其本地日志中最大事务的ZXID，并从中读取 epoch 值，然后加 1，以此作为新的 epoch，并将低 32 位从 0 开始计数。

### 二、ZAB协议

Zab协议分为四个阶段：



![img](https://ask.qcloudimg.com/http-save/developer-news/ts10n0qfkg.jpeg?imageView2/2/w/1620)

#### Phase 0: Leader election（选举阶段，Leader不存在）

节点在一开始都处于选举阶段，只要有一个节点得到超半数Quorums节点的票数的支持，它就可以当选prospective  leader（提名leader）。只有到达 Phase 3 prospective leader 才会成为established  leader(EL)。

**这一阶段的目的是就是为了选出一个prospective leader（PL）**，然后进入下一个阶段。

协议并没有规定详细的选举算法，后面我们会提到实现中使用的Fast Leader Election。

#### Phase 1: Discovery（发现阶段，Leader不存在）

在这个阶段，**PL收集Follower发来的acceptedEpoch**，并确定了PL的Epoch和Zxid最大，则会**生成一个NEWEPOCH分发给Follower，Follower确认无误后返回ACK给PL**。

**这个一阶段的主要目的是PL生成NEWEPOCH，同时更新Followers的acceptedEpoch，并寻找最新的historylog，赋值给PL的history。**

**这个阶段的根本：发现最新的history log，发现最新的history log，发现最新的history log。**

一个 follower 只会连接一个 leader，如果有一个节点 f 认为另一个 follower 是 leader，f 在尝试连接时会被拒绝，f 被拒绝之后，就会进入 Phase 0。

#### Phase 2: Synchronization（同步阶段，Leader不存在）

同步阶段主要是**利用 leader 前一阶段获得的最新提议历史，同步集群中所有的副本**。只有当 quorum 都同步完成，PL才会成为EL。**follower 只会接收 zxid 比自己的 lastZxid 大的提议**。

**这个一阶段的主要目的是同步PL的historylog副本。**

#### Phase 3: Broadcast（广播阶段，Leader存在）

到了这个阶段，Zookeeper 集群才能正式对外提供事务服务，并且 leader 可以进行消息广播。同时如果有新的节点加入，还需要对新节点进行同步。

**这个一阶段的主要目的是接受请求，进行消息广播**

值得注意的是，ZAB 提交事务并不像 2PC 一样需要全部 follower 都 ACK，只需要得到 quorum （超过半数的节点）的 ACK 就可以了。

### 三、zookeeper中zab协议的实现

协议的 Java 版本实现跟上面的定义有些不同，选举阶段使用的是 Fast Leader Election（FLE），它包含了 Phase 1 的发现职责。因为**FLE 会选举拥有最新提议历史的节点作为 leader，这样就省去了发现最新提议的步骤**。实际的实现将 Phase 1 和 Phase 2 合并为 Recovery Phase（恢复阶段）。

#### 1. ZAB 的实现

所以，ZAB 的实现只有三个阶段：

##### Phase 1:Fast Leader Election(快速选举阶段，Leader不存在)

前面提到 FLE 会选举拥有最新提议历史（lastZixd最大）的节点作为 leader，这样就省去了发现最新提议的步骤。这是基于拥有最新提议的节点也有最新提交记录的前提。

每个节点会同时向自己和其他节点发出投票请求，互相投票。

选举流程：

- 选epoch最大的

- epoch相等，选 zxid 最大的

- epoch和zxid都相等，选择serverID最大的（就是我们配置zoo.cfg中的myid）

节点在选举开始都默认投票给自己，当接收其他节点的选票时，会根据上面的条件更改自己的选票并重新发送选票给其他节点，当有一个节点的得票超过半数，该节点会设置自己的状态为 leading，其他节点会设置自己的状态为 following。



##### Phase 2:Recovery Phase（恢复阶段，Leader不存在）

这一阶段 follower 发送它们的 lastZixd 给 leader，leader 根据 lastZixd 决定如何同步数据。这里的实现跟前面 Phase 2 有所不同：Follower 收到 TRUNC 指令会中止 L.lastCommittedZxid 之后的提议，收到 DIFF 指令会接收新的提议。

##### Phase 3:Broadcast Election(广播阶段，Leader存在)

同上

#### 2. Leader故障

如果是Leader故障，首先进行Phase 1:Fast Leader Election，然后Phase 2:Recovery Phase，恢复阶段保证了如下两个问题，这两个问题同时也和Raft中的Leader故障解决的问题是一样的，**总之就是要保证Leader操作日志是最新的**：

- 已经被处理的消息不能丢



![img](https://ask.qcloudimg.com/http-save/developer-news/t1f078ibjj.jpeg?imageView2/2/w/1620)

- 被丢弃的消息不能再次出现



![img](https://ask.qcloudimg.com/http-save/developer-news/gunsb2n6q4.jpeg?imageView2/2/w/1620)

### 四、总结

Zab和Raft都是强一致性协议，但是Zab和Raft的实质是一样的，都是mutli-paxos衍生出来的强一致性算法。简单而言，他们的算法都都是先通过Leader选举，选出一个Leader，然后Leader接受到客户端的提议时，都是先写入操作日志，然后再与其他Followers同步日志，Leader再commit提议，再与其他Followers同步提议。如果Leader故障，重新走一遍选举流程，选取最新的操作日志，然后同步日志，接着继续接受客户端请求等等。过程都是一样，只不过两个的实现方式不同，描述方式不同。**实现Raft的核心是Term，Zab的核心是Zxid，反正Term和Zxid都是逻辑时钟。**



# Raft简介

[Paxos算法详解](https://zhuanlan.zhihu.com/p/31780743)一文讲述了晦涩难懂的Paxos算法，以可理解性和易于实现为目标的Raft算法极大的帮助了我们的理解，推动了分布式一致性算法的工程应用，本文试图以通俗易懂的语言讲述Raft算法。

## 一、Raft算法概述

不同于Paxos算法直接从分布式一致性问题出发推导出来，Raft算法则是从多副本状态机的角度提出，用于管理多副本状态机的日志复制。Raft实现了和Paxos相同的功能，它将一致性分解为多个子问题：

- Leader选举（Leader election）
- 日志同步（Log replication）
- 安全性（Safety）
- 日志压缩（Log compaction）
- 成员变更（Membership change）等。

同时，Raft算法使用了更强的假设来减少了需要考虑的状态，使之变的易于理解和实现。

Raft将系统中的角色分为**领导者（Leader**）、**跟从者（Follower）**和**候选人（Candidate）**：

- **Leader**：<u>**接受客户端请求，并向Follower同步请求日志，当日志同步到大多数节点上后告诉Follower提交日志。**</u>
- **Follower**：<u>**接受并持久化Leader同步的日志，在Leader告之日志可以提交之后，提交日志**</u>。
- **Candidate**：Leader选举过程中的临时角色。

![img](https://pic2.zhimg.com/80/v2-40d42747bec5c00503e4bd47566beb65_hd.jpg)

Raft要求系统**在任意时刻最多只有一个Leader**，正常工作期间只有Leader和Followers。

Raft算法角色状态转换如下：

![img](https://pic2.zhimg.com/80/v2-7f64a2df8f8817932ed047d35878bca9_hd.jpg)

Follower只响应其他服务器的请求。**如果Follower超时没有收到Leader的消息**，它会成为一个Candidate并且开始一次Leader选举。收到大多数服务器投票的Candidate会成为新的Leader。Leader在宕机之前会一直保持Leader的状态。

![img](https://pic1.zhimg.com/80/v2-d3cc1cb525ac72dc59ed34148cb3199c_hd.jpg)

Raft算法将时间分为一个个的任期（term），每一个term的开始都是Leader选举。在成功选举Leader之后，Leader会在整个term内管理整个集群。如果Leader选举失败，该term就会因为没有Leader而结束。



## 二、Leader选举

Raft 使用心跳（heartbeat）触发Leader选举。**当服务器启动时，初始化为Follower。Leader向所有Followers周期性发送heartbeat。如果Follower在选举超时时间内没有收到Leader的heartbeat，就会等待一段随机的时间后发起一次Leader选举。**

Follower将其当前term加一然后转换为Candidate。它首先给自己投票并且给集群中的其他服务器发送 RequestVote RPC （RPC细节参见八、Raft算法总结）。

结果有以下三种情况：

- 赢得了多数的选票，成功选举为Leader；
- 收到了Leader的消息，表示有其它服务器已经抢先当选了Leader；
- 没有服务器赢得多数的选票，Leader选举失败，等待选举时间超时后发起下一次选举。

![img](https://pic2.zhimg.com/80/v2-0471619d1b78ba6d57326d97825d9495_hd.jpg)

选举出Leader后，Leader通过定期向所有Followers发送心跳信息维持其统治。若Follower一段时间未收到Leader的心跳则认为Leader可能已经挂了，再次发起Leader选举过程。

**Raft保证选举出的Leader上一定具有最新的已提交的日志**，这一点将在 四、安全性中说明。



## 三、日志同步

Leader选出后，就开始接收客户端的请求。Leader把请求作为`日志条目（Log entries）`加入到它的日志中，然后并行的向其他服务器发起 `AppendEntries RPC `（RPC细节参见八、Raft算法总结）复制日志条目。**当这条日志被复制到大多数服务器上，Leader将这条日志<u>应用到它的状态机</u>并向客户端返回执行结果**。

Raft日志同步过程

![img](https://pic3.zhimg.com/80/v2-7cdaa12c6f34b1e92ef86b99c3bdcf32_hd.jpg)

**某些Followers可能没有成功的复制日志，Leader会无限的重试 AppendEntries RPC直到所有的Followers最终存储了所有的日志条目**。

日志由**有序编号`（log index）`的日志条目组成。每个日志条目包含它被<u>创建时的任期号</u>（term），和<u>用于状态机执行的命令</u>。如果一个日志条目被复制到大多数服务器上，就被认为可以提交（commit）了**。

![img](https://pic3.zhimg.com/80/v2-ee29a89e4eb63468e142bb6103dbe4de_hd.jpg)

Raft日志同步保证如下两点：

- 如果不同日志中的**两个条目有着相同的索引和任期号，则它们所存储的命令是相同的**。
- 如果不同日志中的**两个条目有着相同的索引和任期号，则它们之前的所有条目都是完全一样的**。

**第一条特性**源于<u>Leader在一个term内在给定的一个log index最多创建一条日志条目，同时该条目在日志中的位置也从来不会改变</u>。

**第二条特性**源于 <u>AppendEntries 的一个简单的一致性检查。当发送一个 AppendEntries RPC 时，**Leader会把新日志条目紧接着之前的条目的log index和term都包含在里面**。如果Follower没有在它的日志中找到log index和term都相同的日志，它就会拒绝新的日志条目。</u>

一般情况下，Leader和Followers的日志保持一致，因此 AppendEntries 一致性检查通常不会失败。然而，Leader崩溃可能会导致日志不一致：旧的Leader可能没有完全复制完日志中的所有条目。

**Leader和Followers上日志不一致**

![img](https://pic4.zhimg.com/80/v2-d36c587901391cae50788061f568d24f_hd.jpg)

上图阐述了一些Followers可能和新的Leader日志不同的情况。

- 一个Follower可能会丢失掉Leader上的一些条目;
- 也有可能包含一些Leader没有的条目，也有可能两者都会发生。
- 丢失的或者多出来的条目可能会持续多个任期。

Leader通过强制Followers复制它的日志来处理日志的不一致，Followers上的不一致的日志会被Leader的日志覆盖。

Leader为了使Followers的日志同自己的一致，Leader需要找到Followers同它的日志一致的地方，然后覆盖Followers在该位置之后的条目。

Leader会从后往前试，每次AppendEntries失败后尝试前一个日志条目，直到成功找到每个Follower的日志一致位点，然后向后逐条覆盖Followers在该位置之后的条目。



**Follower crashes**

如果一个follower故障了，则不会再接受AppendEntriesandvoterequests，并且Leader会不断尝试与这个节点保持心跳。

如果这个节点恢复了，则会接受Leader的最新的log，并且将log应用到state machine中去，执行log中的操作

方格指的是client发出的一条请求。

方格虚线，说明一条log entry写入了log。

方格实线，说明一条log entry应用到state machine中



![img](https://ask.qcloudimg.com/http-save/developer-news/10yd5vts1x.gif)

**Leader crashes**

则会进行Leader election。

如果碰到Leader故障的情况，集群中所有节点的日志可能不一致。

old leader的一些操作日志没有通过集群完全复制。new leader将通过强制Followers复制自己的log来处理不一致的情况，步骤如下：

对于每个Follower，new leader将其日志与Followers的日志进行比较，找到他们的达成一致的最后一个log entry。

然后删除掉Followers中这个关键entry后面的所有entry，并将其替换为自己的log entry。该机制将恢复日志的一致性。

下面这种情况集群中所有节点的日志可能不一致：

![](/Users/nali/songyintao/SongYintao.github.io/img/raft-1.gif)







## 四、安全性

Raft增加了如下两条限制以保证安全性：

- 拥有**最新的已提交的**log entry的Follower才有资格成为Leader。

> 这个保证是在RequestVote RPC中做的，Candidate在发送RequestVote RPC时，要带上自己的最后一条日志的term和log index，其他节点收到消息时，如果发现自己的日志比请求中携带的更新，则拒绝投票。日志比较的原则是，如果本地的最后一条log entry的term更大，则term大的更新，如果term一样大，则log index更大的更新。

- Leader**只能推进commit index来提交当前term的已经复制到大多数服务器上的日志**，旧term日志的提交要等到提交当前term的日志来间接提交（log index 小于 commit index的日志**被间接提交**）。

![img](https://pic4.zhimg.com/80/v2-12a5ebab63781f9ec49e14e331775537_hd.jpg)

> **之所以要这样，是因为可能会出现已提交的日志又被覆盖的情况：**
>
> - 在阶段a，term为2，S1是Leader，且S1写入日志（term, index）为(2, 2)，并且日志被同步写入了S2；
>
> - 在阶段b，S1离线，触发一次新的选主，此时S5被选为新的Leader，此时系统term为3，且写入了日志（term, index）为（3， 2）;
>
> - 在阶段c，S5尚未将日志推送到Followers就离线了，进而触发了一次新的选主，而之前离线的S1经过重新上线后被选中变成Leader，此时系统term为4，此时S1会将自己的日志同步到Followers，按照上图就是将日志（2， 2）同步到了S3，而此时由于该日志已经被同步到了多数节点（S1, S2, S3），因此，此时日志（2，2）可以被提交了。；
>
> - 在阶段d，S1又下线了，触发一次选主，而S5有可能被选为新的Leader（这是因为S5可以满足作为主的一切条件：1. term = 5 > 4，2. 最新的日志为（3，2），比大多数节点（如S2/S3/S4的日志都新），然后S5会将自己的日志更新到Followers，于是S2、S3中已经被提交的日志（2，2）被截断了。



增加上述限制后，即使日志`（2，2）`已经被**大多数节点（S1、S2、S3）**确认了，但是它不能被提交，因为它是来自之前`term（2）`的日志，直到S1在当前`term（4）`产生的日志`（4， 4）`被**大多数Followers确认**，S1方可提交日志（4，4）这条日志，当然，根据Raft定义，（4，4）之前的所有日志也会被提交。**此时即使S1再下线，重新选主时S5不可能成为Leader，因为它没有包含大多数节点已经拥有的日志（4，4）。**



## 五、日志压缩

在实际的系统中，不能让日志无限增长，否则系统重启时需要花很长的时间进行回放，从而影响可用性。Raft采用对整个系统进行snapshot来解决，snapshot之前的日志都可以丢弃。

**每个副本独立的对自己的系统状态进行snapshot，并且只能对已经提交的日志记录进行snapshot。**

Snapshot中包含以下内容：

- 日志元数据。最后一条已提交的 log entry的 log index和term。这两个值在snapshot之后的第一条log entry的AppendEntries RPC的完整性检查的时候会被用上。
- 系统当前状态。

**当Leader要发给某个日志落后太多的Follower的log entry被丢弃，Leader会将snapshot发给Follower。或者当新加进一台机器时，也会发送snapshot给它。发送snapshot使用InstalledSnapshot RPC（RPC细节参见八、Raft算法总结）。**

做snapshot既不要做的太频繁，否则消耗磁盘带宽， 也不要做的太不频繁，否则一旦节点重启需要回放大量日志，影响可用性。推荐当日志达到某个固定的大小做一次snapshot。

做一次snapshot可能耗时过长，会影响正常日志同步。**可以通过使用copy-on-write技术避免snapshot过程影响正常日志同步**。



## 六、成员变更

**成员变更是在集群运行过程中副本发生变化，如增加/减少副本数、节点替换等。**

成员变更也是一个分布式一致性问题，既所有服务器对新成员达成一致。但是成员变更又有其特殊性，因为在成员变更的一致性达成的过程中，参与投票的进程会发生变化。

如果将成员变更当成一般的一致性问题，直接向Leader发送成员变更请求，Leader复制成员变更日志，达成多数派之后提交，各服务器提交成员变更日志后从**旧成员配置**（Cold）切换到**新成员配置**（Cnew）。

因为各个服务器提交成员变更日志的时刻可能不同，造成各个服务器从旧成员配置（Cold）切换到新成员配置（Cnew）的时刻不同。

成员变更不能影响服务的可用性，但是成员变更过程的某一时刻，可能出现在Cold和Cnew中同时存在两个不相交的多数派，进而可能选出两个Leader，形成不同的决议，破坏安全性。

![img](https://pic3.zhimg.com/80/v2-c8e4ead21f6f2e9d40361717739519c6_hd.jpg)

> 成员变更的某一时刻Cold和Cnew中同时存在两个不相交的多数派

由于成员变更的这一特殊性，成员变更不能当成一般的一致性问题去解决。

**为了解决这一问题，Raft提出了两阶段的成员变更方法。**

集群先从旧成员配置Cold切换到一个过渡成员配置，称为共同一致（joint consensus），共同一致是旧成员配置Cold和新成员配置Cnew的组合Cold U Cnew，**一旦共同一致Cold U Cnew被提交，系统再切换到新成员配置Cnew**。

![img](https://pic3.zhimg.com/80/v2-6b85a141cd131aa129a4e70d060f37be_hd.jpg)

Raft两阶段成员变更过程如下：

1. Leader收到成员变更请求从Cold切成Cnew；
2. Leader**在本地生成一个新的log entry，其内容是Cold∪Cnew，代表当前时刻新旧成员配置共存，写入本地日志，同时将该log entry复制至Cold∪Cnew中的所有副本。在此之后新的日志同步需要保证得到Cold和Cnew两个多数派的确认；**
3. Follower收到Cold∪Cnew的log entry后更新本地日志，并且此时就以该配置作为自己的成员配置；
4. **如果Cold和Cnew中的两个多数派确认了Cold U Cnew这条日志，Leader就提交这条log entry；**
5. 接下来Leader生成一条新的log entry，其内容是新成员配置Cnew，同样将该log entry写入本地日志，同时复制到Follower上；
6. Follower收到新成员配置Cnew后，将其写入日志，并且从此刻起，就以该配置作为自己的成员配置，并且如果发现自己不在Cnew这个成员配置中会自动退出；
7. Leader收到Cnew的多数派确认后，表示成员变更成功，后续的日志只要得到Cnew多数派确认即可。Leader给客户端回复成员变更执行成功。

异常分析：

- 如果Leader的Cold U Cnew尚未推送到Follower，Leader就挂了，此后选出的新Leader并不包含这条日志，此时新Leader依然使用Cold作为自己的成员配置。
- 如果Leader的Cold U Cnew推送到大部分的Follower后就挂了，此后选出的新Leader可能是Cold也可能是Cnew中的某个Follower。
- 如果Leader在推送Cnew配置的过程中挂了，那么同样，新选出来的Leader可能是Cold也可能是Cnew中的某一个，此后客户端继续执行一次改变配置的命令即可。
- 如果大多数的Follower确认了Cnew这个消息后，那么接下来即使Leader挂了，新选出来的Leader肯定位于Cnew中。

两阶段成员变更比较通用且容易理解，但是实现比较复杂，同时两阶段的变更协议也会在一定程度上影响变更过程中的服务可用性，因此我们期望增强成员变更的限制，以简化操作流程。

两阶段成员变更，之所以分为两个阶段，是因为对Cold与Cnew的关系没有做任何假设，为了避免Cold和Cnew各自形成不相交的多数派选出两个Leader，才引入了两阶段方案。

如果增强成员变更的限制，假设Cold与Cnew任意的多数派交集不为空，这两个成员配置就无法各自形成多数派，那么成员变更方案就可能简化为一阶段。

那么如何限制Cold与Cnew，使之任意的多数派交集不为空呢？方法就是每次成员变更只允许增加或删除一个成员。

可从数学上严格证明，只要每次只允许增加或删除一个成员，Cold与Cnew不可能形成两个不相交的多数派。

一阶段成员变更：

- 成员变更限制每次只能增加或删除一个成员（如果要变更多个成员，连续变更多次）。
- 成员变更由Leader发起，Cnew得到多数派确认后，返回客户端成员变更成功。
- 一次成员变更成功前不允许开始下一次成员变更，因此新任Leader在开始提供服务前要将自己本地保存的最新成员配置重新投票形成多数派确认。
- Leader只要开始同步新成员配置，即可开始使用新的成员配置进行日志同步。



## 七、Raft与Multi-Paxos的异同

Raft与Multi-Paxos都是基于领导者的一致性算法，乍一看有很多地方相同，下面总结一下Raft与Multi-Paxos的异同。

Raft与Multi-Paxos中相似的概念：

![img](https://pic1.zhimg.com/80/v2-a932cb62a02604d5ec57dc0a046a1414_hd.jpg)

Raft与Multi-Paxos的不同：

![img](https://pic3.zhimg.com/80/v2-7679d235c0ac8056552ba88b677e73a2_hd.jpg)

## 八、Raft算法总结

Raft算法各节点维护的状态：

![img](https://pic1.zhimg.com/80/v2-9b53bd65fa9e11eeefd5331833d41c78_hd.jpg)

Leader选举：

![img](https://pic3.zhimg.com/80/v2-05b80ce9095004381b5846c6179f932e_hd.jpg)

日志同步：

![img](https://pic2.zhimg.com/80/v2-8713b773762e9644c38defa5086afacd_hd.jpg)

Raft状态机：

![img](https://pic1.zhimg.com/80/v2-4abb923772ec1be269843c977b5af3c8_hd.jpg)

安装snapshot：

![img](https://pic3.zhimg.com/80/v2-793f4024bfcb648d9aab2a3dfe6b80de_hd.jpg)



Raft要求具备唯一Leader，并把一致性问题具体化为保持日志副本的一致性，以此实现相较Paxos而言更容易理解、更容易实现的目标。Raft是state machine system，Zab是primary-backup system。

[Raft动画](http://thesecretlivesofdata.com/raft/)



# CAP详解

CAP 理论是分布式系统设计中的一个重要理论，虽然它为系统设计提供了非常有用的依据，但是也带来了很多误解。本文将从 CAP 诞生的背景说起，然后对理论进行解释，最后对 CAP 在当前背景下的一些新理解进行分析，澄清一些对 CAP 的误解。

## CAP 理论诞生的背景

CAP 理论的是在“数据一致性 VS 可用性”的争论中产生。CAP 的作者 Brewer 在 90 年代的时候就开始研究基于集群的跨区域系统（实质上是早期的云计算），对于这类系统而言，系统可用性是首要目标，因此他们采用了缓存或者事后更新的方式来优化系统的可用性。尽管这些方法提升了系统的可用性，但是牺牲了系统数据一致性。

Brewer 在 90 年代提出了 BASE 理论（基本可用、软状态、最终一致性），这在当时还不怎么被接受。因为大家还是比较看重 ACID 的优点，不愿意放弃强一致性。**因此，Brewer 提出了 CAP 理论，目的就是为了开阔分布式系统的设计空间，通过“三选二”的公式，解放思想，不要只抓着一致性不放。**

理解了 CAP 诞生的背景，我们才能更加深入的理解 CAP 理论，以及它带来的启示。“三选二”的观点虽然帮助大家开拓了设计思路，但是也带来了很多误解。下面我们会逐一分析，首先来看一下 CAP 理论的解释。

## CAP 理论的经典解释

CAP 定理是分布式系统设计中最基础，也是最为关键的理论。它指出，分布式数据存储不可能同时满足以下三个条件。

- **一致性（Consistency）**：每次读取要么获得最近写入的数据，要么获得一个错误。
- **可用性（Availability）**：每次请求都能获得一个（非错误）响应，但不保证返回的是最新写入的数据。
- **分区容忍（Partition tolerance）**：尽管任意数量的消息被节点间的网络丢失（或延迟），系统仍继续运行。

CAP 定理表明，在存在网络分区的情况下，一致性和可用性必须二选一。**当网络发生分区（不同节点之间的网络发生故障或者延迟较大）时，要么失去一致性（允许不同分区的数据写入），要么失去可用性（识别到网络分区时停止服务）。**而在没有发生网络故障时，即分布式系统正常运行时，一致性和可用性是可以同时被满足的。这里需要注意的是，CAP 定理中的一致性与 ACID 数据库事务中的一致性截然不同。ACID 的 C 指的是事务不能破坏任何数据库规则，如键的唯一性。与之相比，CAP 的 C 仅指单一副本这个意义上的一致性，因此只是 ACID 一致性约束的一个严格的子集。

CAP 理论看起来难理解，其实只要抓住一个核心点就能推导出来，不用死记硬背。在出现网络分区的时候，

- 如果系统不允许写入，那么意味着降低了系统的可用性，但不同分区的数据能够保持一致，即选择了一致性。
- 如果系统允许写入，那么意味着不同分区之间的数据产生不一致，系统可用性得到保障，即选择可用性。

## CAP 的新理解

CAP 经常被误解，很大程度上是因为在讨论 CAP 的时候可用性和一致性的作用范围往往都是含糊不清的。如果不先定义好可用性、一致性、分区容忍在具体场景下的概念，CAP 实际上反而会束缚系统设计的思路。首先，由于分区很少发生，那么在系统不存在分区的情况下没什么理由牺牲 C 或 A。其次，C 与 A 之间的取舍可以在同一系统内以非常细小的粒度反复发生，而每一次的决策可能因为具体的操作，乃至因为牵涉到特定的数据或用户而有所不同。最后，这三种性质都可以在程度上都可以进行度量，并不是非黑即白的有或无。可用性显然是在 0% 到 100% 之间连续变化的，一致性分很多级别，连分区也可以细分为不同含义，如系统内的不同部分对于是否存在分区可以有不一样的认知。

### 什么是分区容忍

在现实世界中，正常情况下分布式系统各个节点之间的通信是可靠的，不会出现消息丢失或者延迟很高的情况，但是网络是不可靠的，总会偶尔出现消息丢失或者消息延迟很高的情况，这个时候不同区域的节点之间在一段时间内就会出现无法通信的情况，也就是发生了分区。

**分区容忍就是指分布式系统在出现网络分区的时候，仍然能继续运行，对外提供服务。**注意，这里所说的仍然能够对外提供服务跟可用性的要求不一样，可用性要求的是对于任意请求都能得到响应，意味着即使出现网络分区所有节点都能够提供服务。而分区容忍的重点在于出现网络分区之后，系统仍然是可用的（包括部分可用）。

举个例子：使用 Paxos 进行数据复制的系统就是典型的 CP 系统，即使出现网络分区，主分区也能够提供服务，所以它是分区容忍的。再举个反例：使用 2PC 进行数据复制的系统没有分区容忍的特性，当出现网络分区时，整个系统都会阻塞。

### 可用性的范围

可用性其实很直观：每次请求都能获得一个（非错误）响应，但不保证返回的是最新写入的数据。换一个说法就是**对于分布式系统中的每个节点，都能够对外部请求做出响应，但不要求一致性。**

经常让我们疑惑的问题是衡量系统可用性的标准是什么？其实关键点在于可用性的范围，脱离了具体场景下的可用性范围是没有意义的。讨论可用性是要有具体场景来划分边界的，简单的认为某个算法是满足可用性要求其实并不严谨，因为在工程实现中会有很多的技巧去弥补修正。

举个例子：谷歌文档就是非常典型的 AP 系统，它在网络断了的情况下也能够使用。诀窍在于它在发现网络断了之后会进入离线模式，允许用户继续进行编辑，然后在网络恢复之后再对修改的内容进行合并处理。可以发现对于谷歌文档来说，用户的浏览器也是它系统的一个节点，当出现网络分区时，它仍然能够为用户提供服务，但是代价是放弃了一致性，因为用户做的修改只有本地知道，服务端是不清楚的。所以在这个例子里面，可用性的范围是包括了用户浏览器在内的，不是我们常规理解的分布式系统的节点一定就是服务端的机器。

值得注意的是在现实世界中，我们一般不会去追求完美的可用性，所以一般的说法是高可用，即尽可能保证更多的节点服务可用。这也是为什么 Paxos 这类的一致性算法越来越流行的原因之一。

### 一致性的范围

**讨论一致性的时候必须要明确一致性的范围，即在一定的边界内状态是一致的，超出边界之外的一致性是无从谈起的。**比如 Paxos 在发生网络分区的时候，在一个主分区内可以保证完备的一致性和可用性，而在分区外服务是不可用的。值得注意的是，当系统在分区的时候选择了一致性，也就是 CP，并不意味着完全失去了可用性，这取决于一致性算法的实现。比如标准的两阶段提交发生分区的时候是完全不可用的，而 Paxos 则保证了主分区的一致性和可用性。

经过上面的讨论可以发现，可用性的范围要求比一致性的范围要求要更严格，CAP 理论中的可用性要求的是整个系统的可用性，即使出现部分节点不可用也算是违反了可用性约束。而一致性的要求则没有那么高，发生网络分区的时候只要保证主分区数据一致性，也认为系统是符合一致性约束的。为什么这么说呢？因为当出现网络分区的时候，客户端只要通过访问主分区就能得到最新的值（访问超过半数以上节点，如果值都相同说明访问的数据是最新的），此时系统是满足 CAP 理论中一致性的要求的。

## 管理分区

网络分区是分布式系统中必然发生的事情，经典的 CAP 理论是忽略网络延迟的，但是在现实世界中，网络延迟跟分区密切相关。也就是说当系统在有限的时间内无法通信达成一致（网络延迟很高），就意味着发生了分区。此时就需要在一致性和可用性之间做出选择：选择继续重试就意味着选择一致性，放弃可用性；放弃数据一致性让操作完成就意味着选择了可用性。值得注意的是在分区的时候放弃数据一致性并不是意味着完全不管，一般工程实现会采用重试的方式达到最终一致性。

通过上面的分析可以发现，平衡分区期间可用性和一致性的影响是分布式系统设计中的关键问题。因此，管理分区不仅是需要主动发现分区，还需要针对分区期间产生的影响准备恢复过程。也就是说**我们可以从另一个角度来应用 CAP 理论：系统进入分区模式的时候，如何在一致性和可用性之间做出选择。**

管理分区有三个步骤：

![img](https://pic1.zhimg.com/80/v2-1e001027e836ad8a88aed24ffb4d73c8_hd.jpg)



- 检测到分区开始
- 明确进入分区模式，限制某些操作
- 当通信恢复后启动分区恢复过程

当系统进入分区模式之后，有两种选择：

- 选择一致性：例如 Paxos 算法，只有大多数的主分区能够进行操作，其他分区不可用，当网络恢复之后少数节点跟多数节点同步数据。
- 选择可用性：例如谷歌文档，出现分区时进入离线模式，等网络恢复了客户端跟服务端数据进行合并恢复。

## 总结

**理论抽象于现实，服务于现实，但绝不等于现实。对 CAP 理论“三选二”的误解就源于我们经常把理论等同于现实。**CAP 的诞生主要是为了拓宽设计思路，不要局限在强一致性的约束中。简单的把“三选二”进行套用反而限制了设计思路。在现实世界中，不同业务场景对可用性和一致性的要求不一样，并且一致性和可用性的范围和区间是动态变化的，并不是非此即彼。因此，准确理解 CAP 理论，从管理分区的角度出发，结合具体的业务场景，才能做出更好的系统设计。



# 实现原理

### 1. Zookeeper

## What is ZooKeeper?

ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services. All of these kinds of services are used in some form or another by distributed applications. Each time they are implemented there is a lot of work that goes into fixing the bugs and race conditions that are inevitable. Because of the difficulty of implementing these kinds of services, applications initially usually skimp on them, which make them brittle in the presence of change and difficult to manage. Even when done correctly, different implementations of these services lead to management complexity when the applications are deployed.

Learn more about ZooKeeper on the [ZooKeeper Wiki](https://cwiki.apache.org/confluence/display/ZOOKEEPER/Index).

**zookeeper提供的原语服务**

1. 创建节点。

2. 删除节点

3. 更新节点

4. 获取节点信息

5. 权限控制

6. 事件监听

### 2. eureka

[Eureka](https://link.zhihu.com/?target=https%3A//github.com/Netflix/eureka)是Netflix开源的一款提供服务注册和发现的产品。

> Eureka is a REST (Representational State Transfer) based service that is primarily used in the AWS cloud for locating services for the purpose of load balancing and failover of middle-tier servers. **We call this service, the Eureka Server**. Eureka also comes with a Java-based client component,the **Eureka Client**, which **makes interactions with the service much easier**. The client also has a built-in load balancer that does basic round-robin load balancing.

Eureka的实现短小精悍，今天就来详细介绍一下。

#### 1. OverView

![](/Users/nali/songyintao/SongYintao.github.io/img/eureka-overview.png)

上图简要描述了Eureka的基本架构，由3个角色组成：

**Eureka Server**

提供服务注册和发现

**Service Provider**

服务提供方。将自身服务注册到`Eureka`，从而使服务消费方能够找到。

**Service Consumer**

服务消费方。从`Eureka`获取注册服务列表，从而能够消费服务。

需要注意的是，上图中的3个角色都是逻辑角色。在实际运行中，这几个角色甚至可以是同一个实例，比如在我们项目中，`Eureka Server`和`Service Provider`就是同一个JVM进程。

![](/Users/nali/songyintao/SongYintao.github.io/img/euraka0.png)

#### 2. 更详细一点

![](/Users/nali/songyintao/SongYintao.github.io/img/eureka2.png)

上图更进一步的展示了3个角色之间的交互。

1. `Service Provider`会向`Eureka Server`做`Register`（服务注册）、`Renew`（服务续约）、`Cancel`（服务下线）等操作。
2. `Eureka Server`之间会做注册服务的同步，从而保证状态一致。
3. `Service Consumer`会向`Eureka Server`获取注册服务列表，并消费服务。

#### 3. Eureka Server 实现细节

首先来看下`Eureka Server`的几个对外接口实现。

##### 1. Register

首先来看`Register`（服务注册），这个**接口会在`Service Provider`启动时被调用来实现服务注册。同时，当`Service Provider`的服务状态发生变化时（如自身检测认为`Down`的时候），也会调用来更新服务状态**。

接口实现比较简单，如下图所示。

1. `ApplicationResource`类接收Http服务请求，调用`PeerAwareInstanceRegistryImpl`的`register`方法
2. `PeerAwareInstanceRegistryImpl`完成服务注册后，调用`replicateToPeers`向其它`Eureka Server`节点（Peer）做状态同步（异步操作）

![](/Users/nali/songyintao/SongYintao.github.io/img/eureka3.png)

注册的服务列表保存在一个嵌套的`hash map`中：

- 第一层`hash map`的`key`是`app name`，也就是应用名字
- 第二层`hash map`的key是`instance name`，也就是实例名字

`RESERVATION-SERVICE`就是`app name`，`jason-mbp.lan:reservation-service:8000`就是`instance name`。

`Hash map`定义如下：

```java
private final ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry =new ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>>();
```

##### 2.  Renew

`Renew`（服务续约）操作由`Service Provider`定期调用，类似于`heartbeat`。主要是用来告诉`Eureka Server`  `Service Provider`还活着，避免服务被剔除掉。接口实现如下图所示。

可以看到，接口实现方式和`register`基本一致：**首先更新自身状态，再同步到其它Peer**。

![](/Users/nali/songyintao/SongYintao.github.io/img/eureka4.png)

##### 3.  Cancel

`Cancel`（服务下线）一般在`Service Provider shut down`的时候调用，用来把自身的服务从`Eureka Server`中删除，以防客户端调用不存在的服务。接口实现如下图所示。

![](/Users/nali/songyintao/SongYintao.github.io/img/eureka5.png)

##### 4. Fetch Registries

`Fetch Registries`由`Service Consumer`调用，用来获取`Eureka Server`上注册的服务。

为了提高性能，服务列表在`Eureka Server`会缓存一份，同时每30秒更新一次。

![](/Users/nali/songyintao/SongYintao.github.io/img/eureka7.png)

##### 5. Eviction

`Eviction`（**失效服务剔除**）用来定期（默认为每60秒）在`Eureka Server`检测失效的服务，检测标准就是超过一定时间没有`Renew`的服务。

默认失效时间为90秒，也就是如果有服务超过90秒没有向`Eureka Server`发起`Renew`请求的话，就会被当做失效服务剔除掉。

**失效时间**可以通过`eureka.instance.leaseExpirationDurationInSeconds`进行配置，**定期扫描时间**可以通过`eureka.server.evictionIntervalTimerInMs`进行配置。

接口实现逻辑见下图：

![](/Users/nali/songyintao/SongYintao.github.io/img/eureka8.png)

##### 6. How Peer Replicates

在前面的`Register`、`Renew`、`Cancel`接口实现中，我们看到了都会有`replicateToPeers`操作，这个就是用来做**Peer之间的状态同步**。

通过这种方式，`Service Provider`只需要通知到任意一个`Eureka Server`后就能保证状态会在所有的Eureka Server中得到更新。

具体实现方式其实很简单，就是**接收到Service Provider请求的Eureka Server，把请求再次转发到其它的Eureka Server，调用同样的接口，传入同样的参数，除了会在header中标记isReplication=true，从而避免重复的replicate。**

**Peer之间的状态是采用异步的方式同步的，所以不保证节点间的状态一定是一致的，不过基本能保证最终状态是一致的**。

**结合服务发现的场景，实际上也并不需要节点间的状态强一致。在一段时间内（比如30秒），节点A比节点B多一个服务实例或少一个服务实例，在业务上也是完全可以接受的（Service Consumer侧一般也会实现错误重试和负载均衡机制）。**

所以按照CAP理论，Eureka的选择就是放弃C，选择AP。

##### 7. How Peer Nodes are Discovered

那大家可能会有疑问，**Eureka Server是怎么知道有多少Peer的呢？**

`Eureka Server`在启动后会调用`EurekaClientConfig.getEurekaServerServiceUrls`来获取所有的Peer节点，并且会定期更新。定期更新频率可以通过`eureka.server.peerEurekaNodesUpdateIntervalMs`配置。

这个方法的**默认实现是从配置文件读取**，所以如果`Eureka Server`节点相对固定的话，可以通过在配置文件中配置来实现。

**如果希望能更灵活的控制Eureka Server节点，比如动态扩容/缩容，那么可以override getEurekaServerServiceUrls方法，提供自己的实现，比如我们的项目中会通过数据库读取Eureka Server列表**。

具体实现如下图所示：

![](/Users/nali/songyintao/SongYintao.github.io/img/eureka9.jpg)

##### 8.How New Peer Initializes

最后再来看一下**一个新的Eureka Server节点加进来，或者Eureka Server重启后，如何来做初始化，从而能够正常提供服务**。

具体实现如下图所示，**简而言之就是启动时把自己当做是Service Consumer从其它Peer Eureka获取所有服务的注册信息。然后对每个服务，在自己这里执行Register，isReplication=true，从而完成初始化**。

![](/Users/nali/songyintao/SongYintao.github.io/img/eureka10.png)

#### 4. Service Provider实现细节

现在来看下**Service Provider**的实现细节，主要就是`Register`、`Renew`、`Cancel`这3个操作。

##### 1. Register

**Service Provider**要对外提供服务，**一个很重要的步骤就是把自己注册到Eureka Server上**。

这部分的实现比较简单，**只需要在启动时和实例状态变化时调用Eureka Server的接口注册即可**。需要注意的是，需要确保配置`eureka.client.registerWithEureka=true`。

![](/Users/nali/songyintao/SongYintao.github.io/img/eureka11.png)

##### 2. Renew

Renew操作会**在Service Provider端定期发起**，**用来通知Eureka Server自己还活着**。 这里有两个比较重要的配置需要注意一下：

1. `instance.leaseRenewalIntervalInSeconds`

Renew频率。默认是30秒，也就是每30秒会向Eureka Server发起Renew操作。

2. `instance.leaseExpirationDurationInSeconds`

服务失效时间。默认是90秒，也就是如果Eureka Server在90秒内没有接收到来自Service Provider的Renew操作，就会把Service Provider剔除。

具体实现如下：

![](/Users/nali/songyintao/SongYintao.github.io/img/eureka12.png)

##### 3. Cancel

在`Service Provider`服务`shut down`的时候，需要及时通知`Eureka Server`把自己剔除，从而避免客户端调用已经下线的服务。

逻辑本身比较简单，通过对方法标记`@PreDestroy`，从而在服务`shut down`的时候会被触发。

![](/Users/nali/songyintao/SongYintao.github.io/img/eureka13.png)

##### 4. How Eureka Servers are Discovered

这里大家疑问又来了，**Service Provider是怎么知道Eureka Server的地址呢**？

其实这部分的主体逻辑和**How Peer Nodes are Discovered**几乎是一样的。

也是**默认从配置文件读取，如果需要更灵活的控制**，可以通过`override getEurekaServerServiceUrls`方法来提供自己的实现。定期更新频率可以通过`eureka.client.eurekaServiceUrlPollIntervalSeconds`配置。

![](/Users/nali/songyintao/SongYintao.github.io/img/eureka14.png)

#### 5. Service Consumer实现细节

**Service Consumer**这块的实现相对就简单一些，因为它只涉及到**从Eureka Server获取服务列表和更新服务列表**。

##### 1.Fetch Service Registries

`Service Consumer`在启动时会从`Eureka Server`获取所有服务列表，并在本地缓存。需要注意的是，需要确保配置`eureka.client.shouldFetchRegistry=true`。

![](/Users/nali/songyintao/SongYintao.github.io/img/eureka15.png)

##### 2.Update Service Registries

由于在本地有一份缓存，所以需要定期更新，定期更新频率可以通过`eureka.client.registryFetchIntervalSeconds`配置。

![](/Users/nali/songyintao/SongYintao.github.io/img/eureka16.png)

##### 3. How Eureka Servers are Discovered

`Service Consumer`和`Service Provider`一样，也有一个**如何知道Eureka Server地址的问题**。

其实由于`Service Consumer`和`Service Provider`本质上是同一个Eureka客户端，所以这部分逻辑是一样的，这里就不再赘述了。



### 3. Etcd





### 4. Consul





# Eureka VS Zookeeper

Eureka本身是Netflix开源的一款提供服务注册和发现的产品，并且提供了相应的Java封装。在它的实现中，**节点之间相互平等，部分注册中心的节点挂掉也不会对集群造成影响，即使集群只剩一个节点存活，也可以正常提供发现服务。哪怕是所有的服务注册节点都挂了，Eureka Clients（客户端）上也会缓存服务调用的信息。这就保证了我们微服务之间的互相调用足够健壮。**

Zookeeper主要**为大型分布式计算提供开源的分布式配置服务、同步服务和命名注册**。曾经是Hadoop项目中的一个子项目，用来控制集群中的数据，目前已升级为独立的顶级项目。很多场景下也用它作为Service发现服务解决方案。

## 1. 对比

在分布式系统中有个著名的CAP定理（**C-数据一致性**；**A-服务可用性**；**P-服务对网络分区故障的容错性**，这三个特性在任何分布式系统中不能同时满足，最多同时满足两个）；

## 2. Zookeeper

Zookeeper是**基于CP**来设计的，即**任何时刻对Zookeeper的访问请求能得到一致的数据结果，同时系统对网络分割具备容错性，但是它不能保证每次服务请求的可用性。**从实际情况来分析，在使用Zookeeper获取服务列表时，如果zookeeper正在选主，或者Zookeeper集群中半数以上机器不可用，那么将无法获得数据。所以说，Zookeeper不能保证服务可用性。

#### 优点：

- 任何时刻对Zookeeper的访问请求能得到一致的数据结果，同时系统对网络分割具备容错性。

#### 缺点：

- 在使用Zookeeper获取服务列表时，如果zookeeper正在选主，或者Zookeeper集群中半数以上机器不可用，那么将无法获得数据。

#### 使用场景：

- **数据存储**：在大多数分布式环境中，尤其是涉及到**数据存储的场景**，数据一致性应该是首先被保证的，这也是zookeeper设计成CP的原因。
- **服务发现**：针对同一个服务，即使注册中心的不同节点保存的服务提供者信息不尽相同，也并不会造成灾难性的后果。**因为对于服务消费者来说，能消费才是最重要的——拿到可能不正确的服务实例信息后尝试消费一下，也好过因为无法获取实例信息而不去消费。**（尝试一下可以快速失败，之后可以更新配置并重试）所以，对于服务发现而言，**可用性比数据一致性更加重要**——AP胜过CP。



## 3. Eureka

 **Netflix在设计Eureka时遵守的就是AP原则**。Eureka Server也可以运行多个实例来构建集群，解决单点问题，但**不同于ZooKeeper的选举leader的过程，Eureka Server采用的是Peer to Peer对等通信**。这是一种**去中心化的架构，无master/slave区分，每一个Peer都是对等的**。在这种架构中，节点通过彼此互相注册来提高可用性，每个节点需要添加一个或多个有效的serviceUrl指向其他节点。**每个节点都可被视为其他节点的副本**。

如果某台Eureka Server宕机，Eureka Client的请求会自动切换到新的Eureka Server节点，当宕机的服务器重新恢复后，Eureka会再次将其纳入到服务器集群管理之中。**当节点开始接受客户端请求时，所有的操作都会进行replicateToPeer（节点间复制）操作，将请求复制到其他Eureka Server当前所知的所有节点中**。

一个新的Eureka Server节点启动后，会首先尝试从邻近节点获取所有实例注册表信息，完成初始化。Eureka Server通过getEurekaServiceUrls()方法获取所有的节点，并且会通过心跳续约的方式定期更新。默认配置下，如果Eureka Server在一定时间内没有接收到某个服务实例的心跳，Eureka Server将会注销该实例（默认为90秒，通过eureka.instance.lease-expiration-duration-in-seconds配置）。当Eureka Server节点在短时间内丢失过多的心跳时（比如发生了网络分区故障），那么这个节点就会进入自我保护模式。

> 什么是自我保护模式？默认配置下，如果Eureka Server每分钟收到心跳续约的数量低于一个阈值（instance的数量*(60/每个instance的心跳间隔秒数)*自我保护系数），并且持续15分钟，就会触发自我保护。在自我保护模式中，Eureka Server会保护服务注册表中的信息，不再注销任何服务实例。当它收到的心跳数重新恢复到阈值以上时，该Eureka Server节点就会自动退出自我保护模式。它的设计哲学前面提到过，那就是宁可保留错误的服务注册信息，也不盲目注销任何可能健康的服务实例。该模式可以通过eureka.server.enable-self-preservation = false来禁用，同时eureka.instance.lease-renewal-interval-in-seconds可以用来更改心跳间隔，eureka.server.renewal-percent-threshold可以用来修改自我保护系数（默认0.85）。

## 4. 总结

- ZooKeeper基于CP，不保证高可用，如果zookeeper正在选主，或者Zookeeper集群中半数以上机器不可用，那么将无法获得数据。

- Eureka基于AP，能保证高可用，即使所有机器都挂了，也能拿到本地缓存的数据。作为注册中心，其实配置是不经常变动的，只有发版和机器出故障时会变。**对于不经常变动的配置来说，CP是不合适的，而AP在遇到问题时可以用牺牲一致性来保证可用性，既返回旧数据，缓存数据**。

所以理论上Eureka是更适合作注册中心。而现实环境中大部分项目可能会使用ZooKeeper，那是因为集群不够大，并且基本不会遇到用做注册中心的机器一半以上都挂了的情况。所以实际上也没什么大问题。





**1. 前言**

**服务注册中心**：给客户端提供可供调用的服务列表，客户端在进行远程服务调用时，根据服务列表然后选择服务提供方的服务地址进行服务调用。

服务注册中心在分布式系统中大量应用，是分布式系统中不可或缺的组件，例如rocketmq的name server，hdfs中的namenode，dubbo中的zk注册中心，spring cloud中的服务注册中心eureka。

在spring cloud中，除了可以使用eureka作为注册中心外，还可以通过配置的方式使用zookeeper作为注册中心。既然这样，我们该如何选择注册中心的实现呢？

著名的CAP理论指出，一个分布式系统不可能同时满足C(一致性)、A(可用性)和P(分区容错性)。由于分区容错性在是分布式系统中必须要保证的，因此我们只能在A和C之间进行权衡。在此Zookeeper保证的是CP, 而Eureka则是AP。

**2. Zookeeper保证CP**

当向注册中心查询服务列表时，我们可以容忍注册中心返回的是几分钟以前的注册信息，但不能接受服务直接down掉不可用。也就是说，服务注册功能对可用性的要求要高于一致性。但是zk会出现这样一种情况，当master节点因为网络故障与其他节点失去联系时，剩余节点会重新进行leader选举。问题在于，选举leader的时间太长，30 ~ 120s, 且选举期间整个zk集群都是不可用的，这就导致在选举期间注册服务瘫痪。在云部署的环境下，因网络问题使得zk集群失去master节点是较大概率会发生的事，虽然服务能够最终恢复，但是漫长的选举时间导致的注册长期不可用是不能容忍的。

**3. Eureka保证AP**

Eureka看明白了这一点，因此在设计时就优先保证可用性。Eureka各个节点都是平等的，几个节点挂掉不会影响正常节点的工作，剩余的节点依然可以提供注册和查询服务。而Eureka的客户端在向某个Eureka注册或如果发现连接失败，则会自动切换至其它节点，只要有一台Eureka还在，就能保证注册服务可用(保证可用性)，只不过查到的信息可能不是最新的(不保证强一致性)。除此之外，Eureka还有一种自我保护机制，如果在15分钟内超过85%的节点都没有正常的心跳，那么Eureka就认为客户端与注册中心出现了网络故障，此时会出现以下几种情况：

1. Eureka不再从注册列表中移除因为长时间没收到心跳而应该过期的服务；
2. Eureka仍然能够接受新服务的注册和查询请求，但是不会被同步到其它节点上(即保证当前节点依然可用)；
3. 当网络稳定时，当前实例新的注册信息会被同步到其它节点中；

因此， Eureka可以很好的应对因网络故障导致部分节点失去联系的情况，而不会像zookeeper那样使整个注册服务瘫痪。

**4. 更深入的探讨**



**为什么不应该使用ZooKeeper做服务发现**

本文作者通过ZooKeeper与Eureka作为Service发现服务（注：WebServices体系中的UDDI就是个发现服务）的优劣对比，分享了Knewton在云计算平台部署服务的经验。本文虽然略显偏激，但是看得出Knewton在云平台方面是非常有经验的，这篇文章从实践角度出发分别从云平台特点、CAP原理以及运维三个方面对比了ZooKeeper与Eureka两个系统作为发布服务的优劣，并提出了在云平台构建发现服务的方法论。

**4.2 背景**

很多公司选择使用ZooKeeper作为Service发现服务（Service Discovery），但是在构建Knewton（Knewton是一个提供个性化教育平台的公司、学校和出版商可以通过Knewton平台为学生提供自适应的学习材料）平台时，我们发现这是个根本性的错误。在这边文章中，我们将用我们在实践中遇到的问题来说明，为什么使用ZooKeeper做Service发现服务是个错误。

**4.3 请留意服务部署环境**

让我们从头开始梳理。我们在部署服务的时候，应该首先考虑服务部署的平台（平台环境），然后才能考虑平台上跑的软件系统或者如何在选定的平台上自己构建一套系统。例如，对于云部署平台来说，平台在硬件层面的伸缩（注：作者应该指的是系统的冗余性设计，即系统遇到单点失效问题，能够快速切换到其他节点完成任务）与如何应对网络故障是首先要考虑的。当你的服务运行在大量服务器构建的集群之上时，则肯定会出现单点故障的问题。对于knewton来说，我们虽然是部署在AWS上的，但是在过往的运维中，我们也遇到过形形色色的故障；**所以，你应该把系统设计成“故障开放型”（expecting failure）的。**其实有很多同样使用AWS的公司跟我们遇到了相似的问题。你必须能够提前预料到平台可能会出现的问题如：意外故障（box failure），高延迟与网络分割问题（network partitions）——同时我们要能构建足够弹性的系统来应对它们的发生。

永远不要期望你部署服务的平台跟其他人是一样的！当然，如果你在独自运维一个数据中心，你可能会花很多时间与钱来避免硬件故障与网络分割问题，这是另一种情况了；但是在云计算平台中，如AWS，会产生不同的问题以及不同的解决方式。当你实际使用时你就会明白，但是，你最好提前应对它们（意外故障、高延迟与网络分割问题）的发生。

**4.4 ZooKeeper作为发现服务的问题**

ZooKeeper（注：ZooKeeper是著名Hadoop的一个子项目，旨在解决大规模分布式应用场景下，**服务协调同步（Coordinate Service）**的问题；它可以为同在一个分布式系统中的其他服务提供：**统一命名服务**、**配置管理**、**分布式锁服务**、**集群管理**等功能）是个伟大的开源项目，它很成熟，有相当大的社区来支持它的发展，而且在生产环境得到了广泛的使用；**但是用它来做Service发现服务解决方案则是个错误。**

在分布式系统领域有个著名的CAP定理（C-数据一致性；A-服务可用性；P-服务对网络分区故障的容错性，这三个特性在任何分布式系统中不能同时满足，最多同时满足两个）；ZooKeeper是个CP的，即任何时刻对ZooKeeper的访问请求能得到一致的数据结果，同时系统对网络分割具备容错性；但是它不能保证每次服务请求的可用性（也就是在极端环境下，ZooKeeper可能会丢弃一些请求，消费者程序需要重新请求才能获得结果）。但是别忘了，ZooKeeper是分布式协调服务，它的职责是保证数据（配置数据，状态数据）在其管辖下的所有服务之间保持同步、一致；所以就不难理解为什么ZooKeeper被设计成CP而不是AP特性的了，如果是AP的，那么将会带来恐怖的后果（ZooKeeper就像交叉路口的信号灯一样，你能想象在交通要道突然信号灯失灵的情况吗？）。而且，作为ZooKeeper的核心实现算法Zab，就是解决了分布式系统下数据如何在多个服务之间保持同步问题的。

**作为一个分布式协同服务，ZooKeeper非常好，但是对于Service发现服务来说就不合适了；因为对于Service发现服务来说就算是返回了包含不实的信息的结果也比什么都不返回要好；再者，对于Service发现服务而言，宁可返回某服务5分钟之前在哪几个服务器上可用的信息，也不能因为暂时的网络故障而找不到可用的服务器，而不返回任何结果。所以说，用ZooKeeper来做Service发现服务是肯定错误的，如果你这么用就惨了！**

而且更何况，如果被用作Service发现服务，ZooKeeper本身并没有正确的处理网络分割的问题；而在云端，网络分割问题跟其他类型的故障一样的确会发生；所以最好提前对这个问题做好100%的准备。就像Jepsen在ZooKeeper网站上发布的博客中所说：**在ZooKeeper中，如果在同一个网络分区（partition）的节点数（nodes）数达不到ZooKeeper选取Leader节点的“法定人数”时，它们就会从ZooKeeper中断开，当然同时也就不能提供Service发现服务了。**

**如果给ZooKeeper加上客户端缓存（注：给ZooKeeper节点配上本地缓存）或者其他类似技术的话可以缓解ZooKeeper因为网络故障造成节点同步信息错误的问题。**Pinterest与Airbnb公司就使用了这个方法来防止ZooKeeper故障发生。这种方式可以从表面上解决这个问题，具体地说，当部分或者所有节点跟ZooKeeper断开的情况下，每个节点还可以从本地缓存中获取到数据；但是，即便如此，ZooKeeper下所有节点不可能保证任何时候都能缓存所有的服务注册信息。如果ZooKeeper下所有节点都断开了，或者集群中出现了网络分割的故障（由于交换机故障导致交换机底下的子网间不能互访）；那么ZooKeeper会将它们都从自己管理范围中剔除出去，外界就不能访问到这些节点了，即便这些节点本身是“健康”的，可以正常提供服务的；所以导致到达这些节点的服务请求被丢失了。（注：这也是为什么ZooKeeper不满足CAP中A的原因）

**更深层次的原因是，ZooKeeper是按照CP原则构建的，也就是说它能保证每个节点的数据保持一致，而为ZooKeeper加上缓存的做法的目的是为了让ZooKeeper变得更加可靠（available）；但是，ZooKeeper设计的本意是保持节点的数据一致，也就是CP。所以，这样一来，你可能既得不到一个数据一致的（CP）也得不到一个高可用的（AP）的Service发现服务了；因为，这相当于你在一个已有的CP系统上强制栓了一个AP的系统，这在本质上就行不通的！一个Service发现服务应该从一开始就被设计成高可用的才行！**

如果抛开CAP原理不管，正确的设置与维护ZooKeeper服务就非常的困难；错误会经常发生，导致很多工程被建立只是为了减轻维护ZooKeeper的难度。这些错误不仅存在与客户端而且还存在于ZooKeeper服务器本身。Knewton平台很多故障就是由于ZooKeeper使用不当而导致的。那些看似简单的操作，如：正确的重建观察者（reestablishing watcher）、客户端Session与异常的处理与在ZK窗口中管理内存都是非常容易导致ZooKeeper出错的。同时，我们确实也遇到过ZooKeeper的一些经典bug：ZooKeeper-1159 与ZooKeeper-1576；我们甚至在生产环境中遇到过ZooKeeper选举Leader节点失败的情况。**这些问题之所以会出现，在于ZooKeeper需要管理与保障所管辖服务群的Session与网络连接资源**；但是它不负责管理服务的发现，所以**使用ZooKeeper当Service发现服务得不偿失。**

**4.5 做出正确的选择：Eureka的成功**

我们把Service发现服务从ZooKeeper切换到了Eureka平台，它是一个开源的服务发现解决方案，由Netflix公司开发。（Eureka由两个组件组成：Eureka服务器和Eureka客户端。Eureka服务器用作服务注册服务器。Eureka客户端是一个java客户端，用来简化与服务器的交互、作为轮询负载均衡器，并提供服务的故障切换支持。）**Eureka一开始就被设计成高可用与可伸缩的Service发现服务**，这两个特点也是Netflix公司开发所有平台的两个特色。

自从切换工作开始到现在，我们实现了在生产环境中所有依赖于Eureka的产品没有下线维护的记录。我们也被告知过，在云平台做服务迁移注定要遇到失败；但是我们从这个例子中得到的经验是，**一个优秀的Service发现服务在其中发挥了至关重要的作用**！

首先，**在Eureka平台中，如果某台服务器宕机，Eureka不会有类似于ZooKeeper的选举leader的过程；客户端请求会自动切换到新的Eureka节点；当宕机的服务器重新恢复后，Eureka会再次将其纳入到服务器集群管理之中；而对于它来说，所有要做的无非是同步一些新的服务注册信息而已。**所以，再也不用担心有“掉队”的服务器恢复以后，会从Eureka服务器集群中剔除出去的风险了。Eureka甚至被设计用来应付范围更广的网络分割故障，并实现“0”宕机维护需求。当网络分割故障发生时，每个Eureka节点，会持续的对外提供服务（注：ZooKeeper不会）：接收新的服务注册同时将它们提供给下游的服务发现请求。这样一来，就可以实现在同一个子网中（same side of partition），新发布的服务仍然可以被发现与访问。

但是，Eureka做到的不止这些。**正常配置下，Eureka内置了心跳服务，用于淘汰一些“濒死”的服务器；如果在Eureka中注册的服务，它的“心跳”变得迟缓时，Eureka会将其整个剔除出管理范围（这点有点像ZooKeeper的做法）。这是个很好的功能，但是当网络分割故障发生时，这也是非常危险的；因为，那些因为网络问题（注：心跳慢被剔除了）而被剔除出去的服务器本身是很”健康“的，只是因为网络分割故障把Eureka集群分割成了独立的子网而不能互访而已。**

**幸运的是，Netflix考虑到了这个缺陷。如果Eureka服务节点在短时间里丢失了大量的心跳连接（注：可能发生了网络故障），那么这个Eureka节点会进入”自我保护模式“，同时保留那些“心跳死亡“的服务注册信息不过期。此时，这个Eureka节点对于新的服务还能提供注册服务，对于”死亡“的仍然保留，以防还有客户端向其发起请求。当网络故障恢复后，这个Eureka节点会退出”自我保护模式“。所以Eureka的哲学是，同时保留”好数据“与”坏数据“总比丢掉任何”好数据“要更好，所以这种模式在实践中非常有效。**

最后，**Eureka还有客户端缓存功能**（注：Eureka分为客户端程序与服务器端程序两个部分，客户端程序负责向外提供注册与发现服务接口）。所以即便Eureka集群中所有节点都失效，或者发生网络分割故障导致客户端不能访问任何一台Eureka服务器；Eureka服务的消费者仍然可以通过Eureka客户端缓存来获取现有的服务注册信息。甚至最极端的环境下，所有正常的Eureka节点都不对请求产生相应，也没有更好的服务器解决方案来解决这种问题时；得益于Eureka的客户端缓存技术，消费者服务仍然可以通过Eureka客户端查询与获取注册服务信息，这点很重要。

**Eureka的构架保证了它能够成为Service发现服务。它相对与ZooKeeper来说剔除了Leader节点的选取或者事务日志机制，这样做有利于减少使用者维护的难度也保证了Eureka的在运行时的健壮性。而且Eureka就是为发现服务所设计的，它有独立的客户端程序库，同时提供心跳服务、服务健康监测、自动发布服务与自动刷新缓存的功能。但是，如果使用ZooKeeper你必须自己来实现这些功能。Eureka的所有库都是开源的，所有人都能看到与使用这些源代码，这比那些只有一两个人能看或者维护的客户端库要好。**

维护Eureka服务器也非常的简单，比如，切换一个节点只需要在现有EIP下移除一个现有的节点然后添加一个新的就行。**Eureka提供了一个web-based的图形化的运维界面，在这个界面中可以查看Eureka所管理的注册服务的运行状态信息：是否健康，运行日志等。Eureka甚至提供了Restful-API接口，方便第三方程序集成Eureka的功能。**

**4.6 结论**

关于Service发现服务通过本文我们想说明两点：

1、留意服务运行的硬件平台；

2、时刻关注你要解决的问题，然后决定使用什么平台。

Knewton就是从这两个方面考虑使用Eureka替换ZooKeeper来作为service发现服务的。云部署平台是充满不可靠性的，Eureka可以应对这些缺陷；同时Service发现服务必须同时具备高可靠性与高弹性，Eureke就是我们想要的！



# Consul vs. Eureka

Eureka is a service discovery tool. The architecture is primarily client/server, with a set of Eureka servers per datacenter, usually one per availability zone. Typically clients of Eureka use an embedded SDK to register and discover services. For clients that are not natively integrated, a sidecar such as Ribbon is used to transparently discover services via Eureka.

Eureka provides a weakly consistent view of services, using best effort replication. When a client registers with a server, that server will make an attempt to replicate to the other servers but provides no guarantee. Service registrations have a short Time-To-Live (TTL), requiring clients to heartbeat with the servers. Unhealthy services or nodes will stop heartbeating, causing them to timeout and be removed from the registry. Discovery requests can route to any service, which can serve stale or missing data due to the best effort replication. This simplified model allows for easy cluster administration and high scalability.

Consul provides a super set of features, including richer health checking, key/value store, and multi-datacenter awareness. Consul requires a set of servers in each datacenter, along with an agent on each client, similar to using a sidecar like Ribbon. The Consul agent allows most applications to be Consul unaware, performing the service registration via configuration files and discovery via DNS or load balancer sidecars.

Consul provides a strong consistency guarantee, since servers replicate state using the [Raft protocol](https://www.consul.io/docs/internals/consensus.html). Consul supports a rich set of health checks including TCP, HTTP, Nagios/Sensu compatible scripts, or TTL based like Eureka. Client nodes participate in a [gossip based health check](https://www.consul.io/docs/internals/gossip.html), which distributes the work of health checking, unlike centralized heartbeating which becomes a scalability challenge. Discovery requests are routed to the elected Consul leader which allows them to be strongly consistent by default. Clients that allow for stale reads enable any server to process their request allowing for linear scalability like Eureka.

The strongly consistent nature of Consul means it can be used as a locking service for leader elections and cluster coordination. Eureka does not provide similar guarantees, and typically requires running ZooKeeper for services that need to perform coordination or have stronger consistency needs.

Consul provides a toolkit of features needed to support a service oriented architecture. This includes service discovery, but also rich health checking, locking, Key/Value, multi-datacenter federation, an event system, and ACLs. Both Consul and the ecosystem of tools like consul-template and envconsul try to minimize application changes required to integration, to avoid needing native integration via SDKs. Eureka is part of a larger Netflix OSS suite, which expects applications to be relatively homogeneous and tightly integrated. As a result, Eureka only solves a limited subset of problems, expecting other tools such as ZooKeeper to be used alongside.



# Consul vs. ZooKeeper, doozerd, etcd

ZooKeeper, doozerd, and etcd are all similar in their architecture. All three have server nodes that require a quorum of nodes to operate (usually a simple majority). They are strongly-consistent and expose various primitives that can be used through client libraries within applications to build complex distributed systems.

Consul also uses server nodes within a single datacenter. In each datacenter, Consul servers require a quorum to operate and provide strong consistency. However, Consul has native support for multiple datacenters as well as a more feature-rich gossip system that links server nodes and clients.

All of these systems have roughly the same semantics when providing key/value storage: reads are strongly consistent and availability is sacrificed for consistency in the face of a network partition. However, the differences become more apparent when these systems are used for advanced cases.

The semantics provided by these systems are attractive for building service discovery systems, but it's important to stress that these features must be built. ZooKeeper et al. provide only a primitive K/V store and require that application developers build their own system to provide service discovery. Consul, by contrast, provides an opinionated framework for service discovery and eliminates the guess-work and development effort. Clients simply register services and then perform discovery using a DNS or HTTP interface. Other systems require a home-rolled solution.

A compelling service discovery framework must incorporate health checking and the possibility of failures as well. It is not useful to know that Node A provides the Foo service if that node has failed or the service crashed. Naive systems make use of heartbeating, using periodic updates and TTLs. These schemes require work linear to the number of nodes and place the demand on a fixed number of servers. Additionally, the failure detection window is at least as long as the TTL.

ZooKeeper provides ephemeral nodes which are K/V entries that are removed when a client disconnects. These are more sophisticated than a heartbeat system but still have inherent scalability issues and add client-side complexity. All clients must maintain active connections to the ZooKeeper servers and perform keep-alives. Additionally, this requires "thick clients" which are difficult to write and often result in debugging challenges.

Consul uses a very different architecture for health checking. Instead of only having server nodes, Consul clients run on every node in the cluster. These clients are part of a [gossip pool](https://www.consul.io/docs/internals/gossip.html) which serves several functions, including distributed health checking. The gossip protocol implements an efficient failure detector that can scale to clusters of any size without concentrating the work on any select group of servers. The clients also enable a much richer set of health checks to be run locally, whereas ZooKeeper ephemeral nodes are a very primitive check of liveness. With Consul, clients can check that a web server is returning 200 status codes, that memory utilization is not critical, that there is sufficient disk space, etc. The Consul clients expose a simple HTTP interface and avoid exposing the complexity of the system to clients in the same way as ZooKeeper.

Consul provides first-class support for service discovery, health checking, K/V storage, and multiple datacenters. To support anything more than simple K/V storage, all these other systems require additional tools and libraries to be built on top. By using client nodes, Consul provides a simple API that only requires thin clients. Additionally, the API can be avoided entirely by using configuration files and the DNS interface to have a complete service discovery solution with no development at all.







# 分布式系统中的一致性模型

> 最近看到的一篇超棒的关于分布式系统中强一致性模型的blog，实在没有不分享的道理。最近比较闲，所以干脆把它翻译了，一是为了精读，二是为了更友好地分享。其中会插入一些乱七八糟的个人补充，评论区的精彩讨论也会有选择性的翻译。原文在这：[Strong consistency models](https://link.zhihu.com/?target=https%3A//aphyr.com/posts/313-strong-consistency-models)



网络分区是大概率会[发生](https://link.zhihu.com/?target=https%3A//aphyr.com/posts/288-the-network-is-reliable)的。交换机，网络接口控制器（*NIC，Network Interface Controller*），主机硬件，操作系统，硬盘，虚拟机，语言运行时，更不用说程序语义本身，所有的这些将导致我们的消息可能会延迟，被丢弃，重复或被重新排序。在一个充满不确定的世界里，我们希望程序保持一种**直观的正确性**。

是的，我们想要直观的正确性。那就做正确的事吧！但什么是正确的事？我们如何描述它？在这篇文章里，我们会见到一些“强”一致性模型，并且看到他们是如何互相适应的。

## **正确性（Correctness）**

其实有很多种描述一个算法抽象行为的方式——但为了统一，我们说一个系统是由**状态**和一些**导致状态转移的操作**组成的。在系统运行期间，它将随着操作的演进从一个状态转移到另一个状态。

![img](https://pic3.zhimg.com/80/v2-898f9c90b95d6ecefcccb7c5c4376d8a_hd.jpg)uniprocessor history

举个例子，我们的状态可以是个变量，**操作**可以是对这个变量的读和写。在这个简单的Ruby程序里，我们会对一个变量进行几次读写，以输出的方式表示写。

```rb
x = "a"; puts x; puts x
x = "b"; puts x
x = "c"
x = "d"; puts x
```

我们对程序的正确性已经有了一个直观的概念：以上这段程序应该输出`“aabd”`。为什么呢？因为它们是有序发生的。首先我们`写入值a`，然后`读到值a`，接着`读到值a`，`写入值b`，如此进行。

一旦我们把变量写为某个值，比如`a`，那么读操作就应该返回`a`，直到我们再次改变变量。读到的值应该总是返回最近写入的值。我们把这种系统称为——单值变量——单一**寄存器***（并不单指硬件层次的寄存器，而是 act like a register）*。

从编程的第一天开始，我们就把这种模型奉为圭臬，像习惯一样自然——但这并不是变量运作的唯一方式。一个变量可能被读到任何值：`a`，`d`，甚至是`the moon`。这样的话，我们说系统是**不正确**的，因为这些操作没有与我们模型期望的运作方式对应上。

这引出了对系统**正确性**的定义：给定一些涉及操作与状态的**规则**，随着操作的演进，系统将一直**遵循这些规则**。我们把这样的规则称为**一致性模型**。

我们把对寄存器的规则用简单的英语来陈述，但它们也可以是任意复杂的数学结构。“读取两次写入之前的值，对值加三，如果结果为4，读操作可能返回cat或dog”，这也可以是一种一致性模型*（作者只是为了表名一致性模型的阐述原则，后同）*。也可以是“每次读操作都会返回0”。我们甚至可以说“根本没有什么规则，所有操作都是允许的”。这就是**最简单**的一致性模型，任何系统都能轻易满足。

更加正式地说，一致性模型是**所有被允许的操作记录**的集合。当我们运行一个程序，经过一系列集合中允许的操作，特定的执行结果总是**一致**的。如果程序意外地执行了**非**集合中的操作，我们就称执行记录是**非一致**的。如果**任意**可能的执行操作都在这个被允许的操作集合内，那么系统就**满足**一致性模型。我们希望实际的系统是满足这样“直观正确”的一致性模型的，这样我们才能写出可预测的程序。

## **并发记录（Concurrent histories）**

假设有一个用Node.js或Erlang写的并发程序。现有多个逻辑线程，我们称之为“多进程”。如果我们用2个进程运行这个并发程序，每个进程都对同一个寄存器进行访问（读和写），那么我们之前认为的寄存器系统的不变性*（指顺序不变性）*就会被**改写**。

![img](https://pic2.zhimg.com/80/v2-c80aa00f71237ae206031d19374b6a4d_hd.jpg)multiprocessor history

两个工作进程分别称为“top”和“bottom”。Top进程尝试执行`写a`，`读`，`读`。Bottom进程同时尝试执行`读`，`写b`，`读`。因为程序是并发的，所以两个进程之间互相交错的操作将导致**多个执行顺序**——而在单核场景下，执行顺序总是程序里指定的那一个逻辑顺序。图例中，top写入`a`，bottom读`a`，top读`a`，bottom写`b`，top读`b`，bottom读`b`。

但是并发会让一切表现的不同。我们可以**默认**地认为每个**并发**的程序——一旦执行，操作能以任意顺序发生。一个**线程**，或者说是一个**逻辑进程**，在执行记录层面的做了一个**约束**：属于同一个线程的操作**一定**会按顺序发生。逻辑线程对允许操作集合中的操作强加了部分顺序保证。*（一个逻辑线程即一个执行实体，即使编译器重排了指令，单个线程中的同步workflow顺序是不会颠倒的。但是不同线程之间的事件顺序无法保证。）*

即使有了以上的保证，从独立进程的角度来看，我们的寄存器不变性也被破坏了。Top写入`a`，读到`a`，接着读到`b`——这**不再**是它写入的值。 我们必须使一致性模型更**宽松**来有效描述并发。现在，进程可以从其他任意进程读到最近写入的值。寄存器变成了两个进程之间的**协调地**：它们共享了状态。

## **光锥（Light cones）**

> **读写不再是一个瞬时的过程，而是一个类似光传播->反射面->反向传播的过程。**

![img](https://pic4.zhimg.com/80/v2-beab502c56089454b146b5d85a6db737_hd.jpg)light cone history

现实往往没有那么理想化：在几乎每个实际的系统中，进程之间都有一定的**距离**。一个没有被缓存的值*（指没有被CPU的local cache缓存）*，通常在距离CPU**30厘米**的DIMM内存条上。光需要整整一个纳秒来传播这么长的距离，实际的内存访问会比光速慢得多。位于不同数据中心某台计算机上的值可以相距几千公里——意味着需要几百毫秒的传播时间。我们没有更快传播数据的方法，否则就违反了物理定律。*（物理定律都违反了，就更别谈什么现代计算机体系了。）*

这意味着我们的操作**不再是瞬时的**。某些操作也许快到可以被近乎认为是瞬时的，但是通常来说，操作是**耗时**的。我们**调用**对一个变量的写操作；写操作传播到内存，或其他计算机，或月球；内存改变状态；一个确认信息回传；这样我们才**知道**这个操作真实的发生了。

![img](https://pic1.zhimg.com/80/v2-315585d3ea20f55fa8f2354908a87f3c_hd.jpg)concurrent read

不同地点之间传送消息的延迟会在操作记录中造成**歧义**。消息传播的快慢会导致预期外的事件顺序发生。上图中，bottom发起一个读请求的时候，值为`a`，但在读请求的传播过程中，top将值写为`b`——写操作偶然地比读请求**先**到达寄存器。 Bottom最终读到了`b`而不是`a`。

这一记录破坏了我们的寄存器并发一致性模型。Bottom并没有读到它在发起读请求时的值。有人会考虑使用`完成时间`而不是`调用时间`作为操作的`真实时间`，但反过来想想，这同样行不通：当读请求比写操作先到达时，进程会在当前值为`b`时读到`a`。

在分布式系统中，操作的耗时被放大了，我们必须使一致性模型**更宽松**：允许这些有歧义的顺序发生。

我们该如何确定宽松的程度？我们必须允许**所有**可能的顺序吗？或许我们还是应该强加一些合理性约束？

## **线性一致性（Linearizability）**

![img](https://pic4.zhimg.com/80/v2-e33945f8d78833a0ff83c44f44d9b71b_hd.jpg)finite concurrency bounds

通过仔细的检查，我们发现事件的顺序是有边界的。在时间维度上，消息不能被逆向发送，因此**最先到达**的消息会即刻接触到数据源。一个操作不能在它**被调用之前**生效。

同样的，通知完成的消息也不能回传，这意味着一个操作**不能**在它**完成之后**生效。

如果我们假设有一个全局的状态与每个进程通信；继续假设与这个全局状态交互的操作都是**原子的**；那我们可以排除很多可能发生的记录。**每个操作会在它调用和完成之间的某个时间点原子地生效**。

我们把这样的一致性模型称为**线性一致性模型**。尽管操作都是并发且耗时的，但每一个操作都会在某地以严格的线性顺序发生。

![img](https://pic2.zhimg.com/80/v2-6fe2f21c945b3f89574ee8b57df87b41_hd.jpg)linearizability complete visibility

“全局单点状态”并不一定是一个单独的节点，同样的，操作也并不一定全是原子的，状态也可以被分片成横跨多台机器，或者分多步完成——只要从进程的角度看来，外部记录的表现与**一个原子的单点状态等效**。通常一个可线性化的系统由一些更小的协调进程组成，这些进程本身就是线性的，并且这些进程又是由更细粒度的协调进程组成，直到[硬件提供可线性化的操作](https://link.zhihu.com/?target=http%3A//en.wikipedia.org/wiki/Compare-and-swap)。

线性化是强大的武器。一旦一个操作完成，它或它之后的某一状态将对**所有参与者**可见。因为每个操作**一定**发生在它的`完成时间`之前，且任何之后被调用的操作**一定**发生在`调用时间`之后，也就是在原操作本身之后。 一旦我们成功写入`b`，每个之后调用的读请求都可以读到`b`，如果有更多的写操作发生的话，也可以是`b`之后的某个值。

我们可以利用线性一致性的原子性约束来**安全地修改状态**。我们定义一个类似`CAS（compare-and-set）`的操作，当且仅当寄存器持有某个值的时候，我们可以往它写入新值。 `CAS`操作可以作为互斥量，信号量，通道，计数器，列表，集合，映射，树等等的实现基础，使得这些共享数据结构变得可用。线性一致性保证了变更的**安全交错**。

此外，线性一致性的时间界限保证了操作完成后，所有变更都对其他参与者可见。于是线性一致性禁止了过时的读。每次读都会读到某一介于`调用时间`与`完成时间`的状态，但永远不会读到读请求调用之前的状态。线性一致性同样禁止了**非单调**的读，比如一个读请求先读到了一个新值，后读到一个旧值。

由于这些强约束条件的存在，可线性化的系统变得更容易推理，这也是很多并发编程模型构建的时候选择它作为基础的原因。Javascript中的所有变量都是（独立地）可线性化的，其他的还有Java中的volatile变量，Clojure中的atoms，Erlang中独立的process。大多数编程语言都实现了互斥量和信号量，它们也是可线性化的。强约束的假设通常会产生强约束的保证。

但如果我们无法满足这些假设会怎么办？

*（线性一致性模型提供了这样的保证：1.对于观察者来说，所有的读和写都在一个单调递增的时间线上串行地向前推进。 2.所有的读总能返回最近的写操作的值。）*

## **顺序一致性（Sequential consistency）**

![img](https://pic4.zhimg.com/80/v2-50027da38292395f07ca8dc348a79883_hd.jpg)sequencial history

如果我们允许进程在时间维度发生偏移，从而它们的操作可能会在调用之前或是完成之后生效，但仍然保证一个约束——任意进程中的操作必须按照进程中定义的顺序*（即编程的定义的逻辑顺序）*发生。这样我们就得到了一个稍弱的一致性模型：**顺序一致性**。

顺序一致性允许比线性一致性产生更多的记录，但它仍然是一个很有用的模型：我们每天都在使用它。举个例子，当一个用户上传一段视频到Youtube，Youtube把视频放入一个处理队列，并立刻返回一个此视频的网页。我们并不能立刻看到视频，上传的视频会在被充分处理后的几分钟内生效。队列会以入队的顺序**同步**地（取决于队列的具体实现）删除队列中的项。

很多缓存的行为和顺序一致性系统一直。如果我在Twitter上写了一条推文，或是在Facebook发布了一篇帖子，都会耗费一定的时间渗透进一层层的缓存系统。不同的用户将在不同的时间看到我的信息，但每个用户都以**同一个顺序**看到我的操作。一旦看到，这篇帖子便不会消失。如果我写了多条评论，其他人也会按顺序的看见，而非乱序。

*（顺序一致性放松了对一致性的要求：1.不要求操作按照真实的时间序发生。2.不同进程间的操作执行先后顺序也没有强制要求，但必须是原子的。3.单个进程内的操作顺序必须和编码时的顺序一直。）*

## **因果一致性（Casual consistency）**

我们不必对一个进程中的**每个**操作都施加顺序约束。只有**因果相关**的操作必须按顺序发生。同样拿帖子举例子：一篇帖子下的所有评论必须以同样的顺序展示给所有人，并且只有帖子可见**后**，帖子下的回复才可见*（也就是说帖子和帖子下的评论有因果关系）*。如果我们将这些因果关系编码成类似“我依赖于操作X”的形式，作为每个操作明确的一部分，数据库就可以将这些操作延迟直到它们的依赖都就绪后才可见。

因果一致性比同一进程下对每个操作严格排序的一致性*（即顺序一致性）*来的更宽松——属于同一进程但不同因果关系链的操作能以相对的顺序执行*（也就是说按因果关系隔离，无因果关系的操作可以并发执行）*，这能防止许多不直观的行为发生。

## **串行一致性（Serializable consistency）**

![img](https://pic1.zhimg.com/80/v2-f442af2ee4cbed0d52e4a13ab3e854e0_hd.jpg)serializable history

如果我们说操作记录的发生等效于某些单一的原子序，但和调用时间与完成时间无关，那么我们就得到了名为**串行一致性**的一致性模型。这一模型比你想象的更强大同时也更脆弱。

串行一致性是**弱约束**的，因为它能允许多种类型的记录发生，且对时间或顺序不设边界。在上面的示意图中，消息看似可以被任意地发送至过去和未来，因果关系也可以交错。在一个串行数据库中，即使在0时刻，`x`还没被初始化，一个类似`read x`的读事务也是允许执行的。 或者它也会被延迟到无限远的未来执行。类似`write 2 to x`的写事务可以立即执行，也可能永远都不会执行。

举个例子，在一个串行系统中，有这么一段程序

```text
x = 1
x = x + 1
puts x
```

这段程序可以输出`nil`，`1`或`2`，因为操作能以任意顺序发生。 这是十分弱的约束！这里可以把每一行代码看作是单个操作，所有操作都成功执行了。

另一方面，串行一致性也是**强约束**的，当它要求一个线性顺序时，它能拦截很大一部分操作记录。看以下程序

```text
print x if x = 3
x = 1 if x = nil
x = 2 if x = 1
x = 3 if x = 2
```

这段程序只有一种输出可能。它并不按我们**编写**的顺序输出，但`x`会从`nil`开始变化：`nil -> 1 -> 2 -> 3`，最终输出`3`。

因为串行一致性允许对操作顺序执行任意的重排（只要操作顺序是原子序的）， 它在实际的场景中并不是十分有用。大多数宣称提供了串行一致性的数据库实际上提供的是`强串行一致性`，它有着和线性一致性一样的时间边界。让事情更复杂的是，大多数SQL数据库宣称的串行一致性等级比[实际的更弱](https://link.zhihu.com/?target=http%3A//www.bailis.org/papers/hat-hotos2013.pdf)，比如可重复读，游标稳定性，或是快照隔离性。

*（关于线性一致性和串行一致性，看似十分相似，其实不然。串行一致性是数据库领域的概念，是针对事务而言的，描述对一组事务的执行效果等同于某种串行的执行，没有ordering的概念，而线性一致性来自并行计算领域，描述了针对某种数据结构的操作所表现出的顺序特征。**串行一致性是对多操作，多对象的保证，对总体的操作顺序无要求；线性一致性是对单操作，单对象的保证，所有操作遵循真实时间序**。详见*[Linearizability vs Serializability](https://link.zhihu.com/?target=http%3A//www.bailis.org/blog/linearizability-versus-serializability/)*）*

## **一致性的代价（Consistency comes with costs）**

之前说了“弱”一致性模型比“强”一致性模型**允许**更多的操作记录发生*（这里的强与弱是相对的）*。比如线性一致性保证操作在调用时间与完成时间之间发生。不管怎样，**需要协调来达成对顺序的强制约束**。不严格地说，执行越多的记录，系统中的参与者就必须越谨慎且通信频繁。

也许你听说过[CAP理论](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/CAP_theorem)，CAP理论声明给定一致性（**C**onsistency），可用性（**A**vailability）和分区容错性（**P**artition tolerance），任何系统**至多**能保证以上三项中的**两项**而不可能满足全部三项。这是Eric Brewer的CAP猜想的非正式说法，以下是准确的定义：

- **一致性（Consistency）**意味着线性化，具体说，可以是一个可线性化的寄存器。寄存器可以等效为集合，列表，映射，关系型数据库等等，因此该理论可以被拓展到各种可线性化的系统。
- **可用性（Availability）**意味着向非故障节点发出的每个请求都将成功完成。因为网络分区可以持续**任意长的时间**，因此节点不能简单地把响应推迟到分区结束。
- **分区容错性（Partition tolerance）**意味着分区很可能发生。当网络**可靠**的时候，提供一致性和可用性将变得十分**简单**，但当网络**不可靠**时，同时提供一致性和可用性将变得**几乎不可能**。然而网络总不是完美可靠的，所以我们不能选择CA。所有实际可用的商用分布式系统至多能提供AP或CP的保证。

![img](https://pic4.zhimg.com/80/v2-f752105247d9102c3eeeac69515ad2bf_hd.jpg)family tree

也许你会说：“等等！线性化并不是一致性模型的终极解决方案！我能围绕着CAP理论提供顺序一致性，串行一致性或是快照隔离性！”

没错，CAP理论只声称**我们不能构建一个完全可用的线性化系统**。但问题是我们又有了其他证据表明我们同样不能利用顺序化，串行化，可重复读，快照隔离，游标稳定或是其它任意一个比这些强的约束来构建完全可用的系统。在Peter Bailis的Highly Available Transactions这篇论文中，红色阴影标注的模型就不能是**完全**可用的。

如果我们**放松**对可用性的定义，只要求client节点能够一直与同一server保持通信，某种一致性就被达成了。我们能以此为基础提供因果一致性，PRAM*（pipelined random access memory）*一致性和“读你所写”一致性。

如果我们要求**完全可用**，那就能提供单调读，单调写，读的提交，单调且原子的视角等等。这些一致性模型是由如Riak和Cassandra这样的分布式存储系统，低隔离性设置的ANSI SQL数据库提供的。这些一致性模型并没有保证线性顺序，而是在批处理任务或网络场景下提供**部分**顺序保证。只能保证部分顺序是因为它们准许更丰富的记录。

## **一种混合方法（A hybrid approach）**

![img](https://pic2.zhimg.com/80/v2-396ed495b616202401fe9671f2521b4d_hd.jpg)weak not unsafe

一些算法依赖于线性化提供安全性。例如当我们想构建分布式锁的服务时，我们就需要线性化，如果没有硬性的时间边界的话，我们就可以持有一把将来的锁或是过去的锁。而另一方面，很多算法根本**不需要**线性化。即使仅提供“弱”一致性模型，比如有**最终一致性**保证的集合，列表，树，映射等结构也能被安全地表示为[CRDTs](https://link.zhihu.com/?target=https%3A//hal.inria.fr/file/index/docid/555588/filename/techreport.pdf)*（Commutative Replicated Data Types）*

更强约束的一致性模型需要更多的协调——需要更多的消息交互，确保操作在正确的顺序发生。这不仅意味着更低的可用性，还会被**导致更高的延迟**。这也是为什么现代CPU内存模型默认不是线性化的，除非显示指定。*（x86-64架构的CPU通常以Sequential consistency作为默认的memory order）*，现代CPU会在核之间重排内存操作，甚至更糟糕。虽然*（程序的行为）*更难以推理，但带来的性能提升是惊人的。在地理位置上零落的分布式系统，数据中心通常有几百毫秒的延迟，通常和CPU的场景类似，代价也相似。

因此在实践中，通常会用**混合**数据存储，在数据库之间混用不同的一致性模型来权衡冗余度，可用性，性能和安全性等目标。可能的话就为了可用性和性能选择“弱”一致性模型。必要的话就选择“强”一致性模型，因为某些算法对操作顺序有严格要求。我们可以向S3，Riak，Cassandra等数据库写入大量数据，然后线性地向Postgres，Zookeeper或Etcd写入指向数据的**指针**。一些数据库准许多种一致性模型共存，比如关系数据库中的可调节隔离等级，Cassandra和Riak中的线性化事务，减少了使用的系统数量。但底线是：**任何人宣称它的一致性模型是最优解，那么他一定是个大猪蹄子**。



## **接下来是精彩评论时间**

**Colin Scott：**当你提到*“属于同一进程但不同因果关系链的操作”*的时候，是否对潜在的因果关系*（happens before）*作了更保守的假设？我在苦想一个case，当来自同一台机器上的两个存在潜在因果关系*（A必须先于B发生）*的操作并发时，会发生什么？

**Aphyr（作者）：**尽管来自同一个进程的操作在某一节点上按顺序发生，但它们并不需要在**任何地方都**按序发生。顺序一致性*（Sequential consistency）*作了这样的约束，但因果一致性*（Casual consistency）*并没有。只有**显式**的因果关系在因果一致性系统中才是顺序不变的，而**隐式**的因果关系在顺序一致性系统中作了保证。*（因为都来自同一进程，通过pid区分）*



**Aurojit Panda：**实际上你对`Colin Scott`的回复和你在文章中的`一致性层级示意图`是自相矛盾的。 PRAM一致性模型约定：所有节点都能按同一顺序从一个给定节点观测到它的写操作*（Lipton, Sandberg 1988）*。而你描述的因果一致性是某一比PRAM一致性更弱的一致性模型，并不是经典的因果一致性模型。并且你描述中出现的隐式因果关系也是PRAM一致性模型约束中的一部分。如果因果一致性比PRAM一致性更强，PRAM一致性就应该用任何因果一致性系统来实现，利用因果一致性对单节点的写操作进行正确排序，使得其他节点的观测结果一致。

**Aphyr（作者）：**请参考[Survey on Consistency Conditions](https://link.zhihu.com/?target=http%3A//www.ics.forth.gr/tech-reports/2013/2013.TR439_Survey_on_Consistency_Conditions.pdf)获得为什么因果一致性比PRAM更强的详细解释。具体地说，有因果关系的操作可以在中间节点之间**传递**，但是PRAM并没有对因果一致性的传递性作任何定义。



**Prakash：**关于问题 - “在你串行一致性的示例中，各种`if`假设是如何对顺序进行约束的？”

你的回答中提到 - “这些操作发生的顺序有且只有一种可能。”

而我的问题是，当我们考虑在并发环境下执行某种操作时，是怎么做到“串行有且只有一种可能”的？举个例子，我们有不同的线程，其中一个检查`x`值是否为3并把它打印，另一个将`x`的值设为2。你能解释一下在这种场景中是如何维护顺序的？

**Aphyr（作者）：**串行的操作记录实际上会转化为单道线程下的操作记录，单道线程下的操作记录就是我们之前的讨论中发挥作用的**状态转移方程**。如果以**恒等函数**作为你的模型，操作记录中**任何**可能的路径都是有效的，它们并不会改变状态。为了让某些操作记录永远不可串行化*（限制那些非法的可能造成状态转移的操作发生）*，不得不声明一些等效于单线程执行的操作无效，这就是条件语句*（if）*强制部分操作有序的原因。

