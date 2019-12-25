---
title: ceph原理、功能、特性
subtitle: 分布式文件系统ceph学习
layout: post
tags: [ceph]
---

最近工作中需要使用的ceph，用于实现测试环境的k8s中redis、mysql有状态服务的启动。保证数据的一致性，避免pod的重启数据丢失。

现在网络上找点资料科普一下，知道它是干什么的。

链接：https://juejin.im/post/5cf635066fb9a07ed911ae84

## Ceph 架构简介及使用场景介绍

Ceph 是一个统一的分布式存储系统，设计初衷是提供较好的性能、可靠性和可扩展性。

Ceph 项目最早起源于 Sage 就读博士期间的工作（最早的成果于2004年发表），并随后贡献给开源社区。在经过了数年的发展之后，目前已得到众多云计算厂商的支持并被广泛应用。

RedHat 及 OpenStack 都可与 Ceph 整合以支持虚拟机镜像的后端存储。

###  Ceph 特点

**高性能**

a. 摒弃了传统的集中式存储元数据寻址的方案，采用 CRUSH 算法，数据分布均衡，并行度高。

b. 考虑了容灾域的隔离，能够实现各类负载的副本放置规则，例如跨机房、机架感知等。

c. 能够支持上千个存储节点的规模，支持 TB 到 PB 级的数据。

**高可用性**

a. 副本数可以灵活控制。

b. 支持故障域分隔，数据强一致性。

c. 多种故障场景自动进行修复自愈。

d. 没有单点故障，自动管理。

**高可扩展性**

a. 去中心化。

b. 扩展灵活。

c. 随着节点增加而线性增长。

**特性丰富**

a. 支持三种存储接口：块存储、文件存储、对象存储。

b. 支持自定义接口，支持多种语言驱动。



### Ceph 架构

**支持三种接口：**

**Object**：有原生的 API，而且也兼容 Swift 和 S3 的 API。

**Block**：支持精简配置、快照、克隆。

**File**：Posix 接口，支持快照。

![img](../img/ceph-overview.png)

### Ceph 核心组件及概念介绍 

**Monitor**

一个 Ceph 集群需要多个 Monitor 组成的小集群，它们通过 Paxos 同步数据，用来保存 OSD 的元数据。



**OSD**

OSD 全称 Object Storage Device，也就是负责响应客户端请求返回具体数据的进程。一个 Ceph 集群一般都有很多个 OSD。



**MDS**

MDS 全称 Ceph Metadata Server，是 CephFS 服务依赖的元数据服务。



**Object**

Ceph 最底层的存储单元是 Object 对象，每个 Object 包含元数据和原始数据。

 

**PG**

PG 全称 Placement Grouops，是一个逻辑的概念，一个 PG 包含多个 OSD。引入 PG 这一层其实是为了更好的分配数据和定位数据。



**RADOS**

RADOS 全称 Reliable Autonomic Distributed Object Store，是 Ceph 集群的精华，用户实现数据分配、Failover 等集群操作。



**Libradio**

Librados 是 Rados 提供库，因为 RADOS 是协议很难直接访问，因此上层的 RBD、RGW 和 CephFS 都是通过 librados 访问的，目前提供 PHP、Ruby、Java、Python、C和C++支持。



**CRUSH**

CRUSH 是 Ceph 使用的数据分布算法，类似一致性哈希，让数据分配到预期的地方。



**RBD**

RBD 全称 RADOS block device，是 Ceph 对外提供的块设备服务。



**RGW**

RGW 全称 RADOS gateway，是 Ceph 对外提供的对象存储服务，接口与 S3 和 Swift 兼容。



**CephFS**

CephFS 全称 Ceph File System，是 Ceph 对外提供的文件系统服务。



### ▍1.5 三种存储类型-块存储 

![img](https://user-gold-cdn.xitu.io/2019/6/4/16b21dae5c2ff993?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**典型设备：**

磁盘阵列，硬盘

主要是将裸磁盘空间映射给主机使用的。

**优点：**

a. 通过 Raid 与 LVM 等手段，对数据提供了保护。

b. 多块廉价的硬盘组合起来，提高容量。

c. 多块磁盘组合出来的逻辑盘，提升读写效率。   

**缺点：**

a. 采用 SAN 架构组网时，光纤交换机，造价成本高。

b. 主机之间无法共享数据。  

**使用场景：**

a. docker 容器、虚拟机磁盘存储分配。

b. 日志存储。

c. 文件存储。

d. …

### 1.6 三种存储类型-文件存储

![img](https://user-gold-cdn.xitu.io/2019/6/4/16b21dba83ce6b7e?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**典型设备：**

FTP、NFS 服务器

为了克服块存储文件无法共享的问题，所以有了文件存储。

在服务器上架设 FTP 与 NFS 服务，就是文件存储。

**优点：**

a. 造价低，随便一台机器就可以了。

b. 方便文件共享。

**缺点：**

a. 读写速率低。

b. 传输速率慢。

**使用场景：**

a. 日志存储。

b. 有目录结构的文件存储。

c. …

 

### ▍1.7 三种存储类型-对象存储

![img](https://user-gold-cdn.xitu.io/2019/6/4/16b21e1380a41200?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**典型设备：**

内置大容量硬盘的分布式服务器(swift, s3)

多台服务器内置大容量硬盘，安装上对象存储管理软件，对外提供读写访问功能。

**优点：**

a. 具备块存储的读写高速。

b. 具备文件存储的共享等特性。

**使用场景：**

(适合更新变动较少的数据)

a. 图片存储。

b. 视频存储。

c. …

## ▍2. Ceph IO 流程

![img](https://user-gold-cdn.xitu.io/2019/6/4/16b21e1f68ca8048?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### ▍2.1 正常 IO 流程图

##  

###  



![img](https://user-gold-cdn.xitu.io/2019/6/4/16b21f573c3bf7f5?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**步骤：**

- \1. client 创建 cluster handler。
- \2. client 读取配置文件。
- \3. client 连接上 monitor，获取集群 map 信息。
- \4. client 读写 io 根据 crshmap 算法请求对应的主 osd 数据节点。
- \5. 主 osd 数据节点同时写入另外两个副本节点数据。
- \6. 等待主节点以及另外两个副本节点写完数据状态。
- 

\7. 主节点及副本节点写入状态都成功后，返回给 client，io 写入完成。



### ▍2.2 新主 IO 流程图

**说明：**

如果新加入的 OSD1 取代了原有的 OSD4 成为 Primary OSD, 由于 OSD1 上未创建 PG , 不存在数据，那么 PG 上的 I/O 无法进行，怎样工作的呢？

![img](https://user-gold-cdn.xitu.io/2019/6/4/16b21f6e1a45798e?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**步骤：**

- \1. client 连接 monitor 获取集群 map 信息。
- \2. 同时新主 osd1 由于没有 pg 数据会主动上报 monitor 告知让 osd2 临时接替为主。
- \3. 临时主 osd2 会把数据全量同步给新主 osd1。
- \4. client IO 读写直接连接临时主 osd2 进行读写。
- \5. osd2 收到读写 io，同时写入另外两副本节点。
- \6. 等待 osd2 以及另外两副本写入成功。
- \7. osd2 三份数据都写入成功返回给 client, 此时 client io 读写完毕。
- \8. 如果 osd1 数据同步完毕，临时主 osd2 会交出主角色。
- \9. osd1 成为主节点，osd2 变成副本。

### ▍2.3 Ceph IO 算法流程

![img](https://user-gold-cdn.xitu.io/2019/6/4/16b21f79854f8365?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**1. File用户需要读写的文件。File->Object 映射：**

a. ino (File 的元数据，File 的唯一id)。

b. ono(File 切分产生的某个 object 的序号，默认以 4M 切分一个块大小)。

c. oid(object id: ino + ono)。

**2. Object 是 RADOS 需要的对象。Ceph 指定一个静态hash函数计算 oid 的值，将 oid 映射成一个近似均匀分布的伪随机值，然后和 mask 按位相与，得到 pgid。Object->PG 映射：**

a)  hash(oid) & mask-> pgid 。

b)  mask = PG 总数 m(m 为2的整数幂)-1 。

**3. PG(Placement Group),用途是对 object 的存储进行组织和位置映射, (类似于 redis cluster 里面的 slot 的概念) 一个 PG 里面会有很多 object。采用 CRUSH 算法，将 pgid 代入其中，然后得到一组 OSD。PG->OSD 映射：**

a)  CRUSH(pgid)->(osd1,osd2,osd3) 。



### ▍2.4 Ceph IO 伪代码流程







```
1 locator = object_name
2
3 obj_hash =  hash(locator)
4
5 pg = obj_hash % num_pg
6
7 osds_for_pg = crush(pg)    # returns a list of osds
8
9 primary = osds_for_pg[0]
10
11 replicas = osds_for_pg[1:]复制代码
```



###  ▍2.5 Ceph RBD IO 流程

**数据组织：**



![img](https://user-gold-cdn.xitu.io/2019/6/4/16b21f9ea0205dca?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**步骤：**

- \1.  客户端创建一个 pool，需要为这个 pool 指定 pg 的数量。
- \2.  创建 pool/image rbd 设备进行挂载。
- \3.  用户写入的数据进行切块，每个块的大小默认为4M，并且每个块都有一个名字，名字就是 object+序号。
- \4.  将每个 object 通过 pg 进行副本位置的分配。
- \5.  pg 根据 cursh 算法会寻找3个 osd，把这个 object 分别保存在这三个 osd 上。
- \6.  osd 上实际是把底层的 disk 进行了格式化操作，一般部署工具会将它格式化为 xfs 文件系统。
- \7.  object 的存储就变成了存储一个文 rbd0.object1.file。

### ▍2.6 Ceph RBD IO 框架图

![img](https://user-gold-cdn.xitu.io/2019/6/4/16b21fa86e9f4fc5?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**客户端写数据 osd 过程：**

- \1. 采用的是 librbd 的形式，使用 librbd 创建一个块设备，向这个块设备中写入数据。
- \2. 在客户端本地同过调用 librados 接口，然后经过 pool，rbd，object、pg 进行层层映射,在 PG 这一层中，可以知道数据保存在哪3个 OSD 上，这3个 OSD 分为主从的关系。
- \3. 客户端与 primay OSD 建立 SOCKET 通信，将要写入的数据传给 primary OSD，由primary OSD 再将数据发送给其他 replica OSD 数据节点。

### ▍2.7 Ceph Pool 和 PG 分布情况

![img](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="865" height="430"></svg>)

**说明：**

- a. pool 是 ceph 存储数据时的逻辑分区，它起到 namespace 的作用。
- b. 每个 pool 包含一定数量(可配置)的 PG。
- c. PG 里的对象被映射到不同的 OSD 上。
- d. pool 是分布到整个集群的。
- e. pool 可以做故障隔离域，根据不同的用户场景不一进行隔离。



### ▍2.8 Ceph 数据扩容 PG 分布

**场景数据迁移流程：**

a. 现状3个 OSD, 4个 PG

b. 扩容到4个 OSD, 4个 PG

**现状：**



![img](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="865" height="210"></svg>)

**扩容后：**



![img](https://user-gold-cdn.xitu.io/2019/6/4/16b21fc45fa1aaf2?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**说明：**

每个 OSD 上分布很多 PG, 并且每个 PG 会自动散落在不同的 OSD 上。如果扩容那么相应的 PG 会进行迁移到新的 OSD 上，保证 PG 数量的均衡。

## ▍3. Ceph 心跳机制

### ▍3.1 心跳介绍

心跳是用于节点间检测对方是否故障的，以便及时发现故障节点进入相应的故障处理流程。

**问题：**

a. 故障检测时间和心跳报文带来的负载之间做权衡。

b. 心跳频率太高则过多的心跳报文会影响系统性能。

c. 心跳频率过低则会延长发现故障节点的时间，从而影响系统的可用性。

**故障检测策略应该能够做到：**

**及时：**节点发生异常如宕机或网络中断时，集群可以在可接受的时间范围内感知。

**适当的压力：**包括对节点的压力，和对网络的压力。

**容忍网络抖动：**网络偶尔延迟。

**扩散机制：**节点存活状态改变导致的元信息变化需要通过某种机制扩散到整个集群。



### ▍3.2 Ceph 心跳检测

![img](https://user-gold-cdn.xitu.io/2019/6/4/16b21fd155062175?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**OSD 节点会监听 public、cluster、front 和 back 四个端口**

**·** public 端口：监听来自 Monitor 和 Client 的连接。

**·** cluster 端口：监听来自 OSD Peer 的连接。

**·** front 端口：供客户端连接集群使用的网卡, 这里临时给集群内部之间进行心跳。

**·** back 端口：供客集群内部使用的网卡。集群内部之间进行心跳。

**·** hbclient：发送 ping 心跳的 messenger。



### ▍3.3 Ceph OSD 之间相互心跳检测

![img](https://user-gold-cdn.xitu.io/2019/6/4/16b21fea7e22cf3d?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**步骤：**

a. 同一个 PG 内 OSD 互相心跳，他们互相发送 PING/PONG 信息。

b. 每隔6s检测一次(实际会在这个基础上加一个随机时间来避免峰值)。

c. 20s没有检测到心跳回复，加入 failure 队列。



### ▍3.4 Ceph OSD与Mon心跳检测

![img](https://user-gold-cdn.xitu.io/2019/6/4/16b220124a5be742?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**OSD 报告给 Monitor：**

- a. OSD 有事件发生时（比如故障、PG 变更）。
- b. 自身启动5秒内。
- c. OSD 周期性的上报给 Monito
- d. OSD 检查 failure_queue 中的伙伴 OSD 失败信息。
- e. 向 Monitor 发送失效报告，并将失败信息加入 failure_pending 队列，然后将其从 failure_queue 移除。
- f. 收到来自 failure_queue 或者 failure_pending 中的 OSD 的心跳时，将其从两个队列中移除，并告知 Monitor 取消之前的失效报告。
- g. 当发生与 Monitor 网络重连时，会将 failure_pending 中的错误报告加回到 failure_queue 中，并再次发送给 Monitor。
- h. Monitor 统计下线 OSD
- i. Monitor 收集来自 OSD 的伙伴失效报告。
- j. 当错误报告指向的 OSD 失效超过一定阈值，且有足够多的 OSD 报告其失效时，将该 OSD 下线。

### ▍3.5 Ceph 心跳检测总结

Ceph 通过伙伴 OSD 汇报失效节点和 Monitor 统计来自 OSD 的心跳两种方式判定 OSD 节点失效。

**及时：**

伙伴 OSD 可以在秒级发现节点失效并汇报 Monitor，并在几分钟内由 Monitor 将失效 OSD 下线。

**适当的压力：**

由于有伙伴 OSD 汇报机制，Monitor 与 OSD 之间的心跳统计更像是一种保险措施，因此 OSD 向 Monitor 发送心跳的间隔可以长达600秒，Monitor 的检测阈值也可以长达900秒。Ceph 实际上是将故障检测过程中中心节点的压力分散到所有的 OSD 上，以此提高中心节点 Monitor 的可靠性，进而提高整个集群的可扩展性。

**容忍网络抖动：**

Monitor 收到 OSD 对其伙伴 OSD 的汇报后，并没有马上将目标 OSD 下线，而是周期性的等待几个条件：

\1. 目标 OSD 的失效时间大于通过固定量 osd_heartbeat_grace 和历史网络条件动态确定的阈值。

\2. 来自不同主机的汇报达到 mon_osd_min_down_reporters。

\3. 满足前两个条件前失效汇报没有被源 OSD 取消。

**扩散**：

作为中心节点的 Monitor 并没有在更新 OSDMap 后尝试广播通知所有的 OSD 和 Client，而是惰性的等待 OSD 和 Client 来获取。以此来减少 Monitor 压力并简化交互逻辑。



## ▍4. Ceph 通信框架

### ▍4.1 Ceph 通信框架种类介绍

**网络通信框架三种不同的实现方式：**

**Simple 线程模式**

特点：每一个网络链接，都会创建两个线程，一个用于接收，一个用于发送。

缺点：大量的链接会产生大量的线程，会消耗 CPU 资源，影响性能。

**Async 事件的I/O多路复用模式**

特点：这种是目前网络通信中广泛采用的方式。k版默认已经使用 Asnyc 了。

**XIO 方式使用了开源的网络通信库 accelio 来实现**

特点：这种方式需要依赖第三方的库 accelio 稳定性，目前处于试验阶段。



### ▍4.2 Ceph 通信框架设计模式

**设计模式(Subscribe/Publish)：**

订阅发布模式又名观察者模式，它意图是“定义对象间的一种一对多的依赖关系，

当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新”。



### ▍4.3 Ceph 通信框架流程图

![img](https://user-gold-cdn.xitu.io/2019/6/4/16b22027a34ef998?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**步骤：**

- a. Accepter 监听 peer 的请求, 调用 SimpleMessenger::add_accept_pipe() 创建新的 Pipe 到 SimpleMessenger::pipes 来处理该请求。
- b. Pipe 用于消息的读取和发送。该类主要有两个组件，Pipe::Reader，Pipe::Writer 用来处理消息读取和发送。
- c. Messenger 作为消息的发布者, 各个 Dispatcher 子类作为消息的订阅者, Messenger 收到消息之后，通过 Pipe 读取消息，然后转给 Dispatcher 处理。
- d. Dispatcher 是订阅者的基类，具体的订阅后端继承该类,初始化的时候通过 Messenger::add_dispatcher_tail/head 注册到 Messenger::dispatchers. 收到消息。
- e. DispatchQueue 该类用来缓存收到的消息, 然后唤醒 DispatchQueue::dispatch_thread 线程找到后端的 Dispatch 处理消息。

![img](https://user-gold-cdn.xitu.io/2019/6/4/16b2202f1b1a4ea8?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### ▍4.4 Ceph 通信框架类图

![img](https://user-gold-cdn.xitu.io/2019/6/4/16b22035430ec76f?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### ▍4.5 Ceph 通信数据格式

通信协议格式需要双方约定数据格式。

**消息的内容主要分为三部分：**

· header              //消息头，类型消息的信封

· user data          //需要发送的实际数据

​        o payload     //操作保存元数据

​        o middle      //预留字段

​        o data          //读写数据

​        o footer             //消息的结束标记







```
1   class Message : public RefCountedObject {
2   protected:
3    ceph_msg_header  header;      // 消息头
4    ceph_msg_footer  footer;      // 消息尾
5    bufferlist       payload;  // "front" unaligned blob
6    bufferlist       middle;   // "middle" unaligned blob
7    bufferlist       data;     // data payload (page-alignment will be preserved where possible)
8 
9  /* recv_stamp is set when the Messenger starts reading the
10  * Message off the wire */
11 utime_t recv_stamp;       //开始接收数据的时间戳
12  /* dispatch_stamp is set when the Messenger starts calling dispatch() on
13    * its endpoints */
14   utime_t dispatch_stamp;   //dispatch 的时间戳
15  /* throttle_stamp is the point at which we got throttle */
16  utime_t throttle_stamp;   //获取throttle 的slot的时间戳
17   /* time at which message was fully read */
18   utime_t recv_complete_stamp;  //接收完成的时间戳
19
20  ConnectionRef connection;     //网络连接
21 
22  uint32_t magic = 0;           //消息的魔术字
23 
24  bi::list_member_hook<> dispatch_q;  //boost::intrusive 成员字段
25  };
26
27 struct ceph_msg_header {
28     __le64 seq;       // 当前session内 消息的唯一 序号
29     __le64 tid;       // 消息的全局唯一的 id
30     __le16 type;      // 消息类型
31     __le16 priority;  // 优先级
32     __le16 version;   // 版本号 
33
34     __le32 front_len; // payload 的长度
35     __le32 middle_len;// middle 的长度
36     __le32 data_len;  // data 的 长度
37     __le16 data_off;  // 对象的数据偏移量
38
39
40    struct ceph_entity_name src; //消息源
41
42    /* oldest code we think can decode this.  unknown if zero. */ 
43     __le16 compat_version;
44     __le16 reserved;
45     __le32 crc;       /* header crc32c */
46   } __attribute__ ((packed));
47
48   struct ceph_msg_footer {
49    __le32 front_crc, middle_crc, data_crc; //crc校验码
50    __le64  sig; //消息的64位signature
51    __u8 flags; //结束标志
52  } __attribute__ ((packed));复制代码
```



## ▍5. Ceph CRUSH 算法

### ▍5.1 数据分布算法挑战

**数据分布和负载均衡：**

\1. 数据分布均衡，使数据能均匀的分布到各个节点上。

\2. 负载均衡，使数据访问读写操作的负载在各个节点和磁盘的负载均衡。

**灵活应对集群伸缩：**

\1. 系统可以方便的增加或者删除节点设备，并且对节点失效进行处理。

\2. 增加或者删除节点设备后，能自动实现数据的均衡，并且尽可能少的迁移数据。

**支持大规模集群：**

\1. 要求数据分布算法维护的元数据相对较小，并且计算量不能太大。随着集群规模的增 加，数据分布算法开销相对比较小。



### ▍5.2 Ceph CRUSH 算法说明

CRUSH 算法的全称为：Controlled Scalable Decentralized Placement of Replicated Data，可控的、可扩展的、分布式的副本数据放置算法。

PG到OSD 的映射的过程算法叫做 CRUSH 算法。(一个 Object 需要保存三个副本，也就是需要保存在三个 osd 上)。

CRUSH 算法是一个伪随机的过程，他可以从所有的 OSD 中，随机性选择一个 OSD 集合，但是同一个 PG 每次随机选择的结果是不变的，也就是映射的 OSD 集合是固定的。



### ▍5.3 Ceph CRUSH 算法原理

**CRUSH 算法因子：**

**层次化的 Cluster Map**

反映了存储系统层级的物理拓扑结构。定义了 OSD 集群具有层级关系的静态拓扑结构。OSD 层级使得 CRUSH 算法在选择 OSD 时实现了机架感知能力，也就是通过规则定义， 使得副本可以分布在不同的机 架、不同的机房中、提供数据的安全性 。

**Placement Rules**

决定了一个 PG 的对象副本如何选择的规则，通过这些可以自己设定规则，用户可以自定义设置副本在集群中的分布。



**● 5.3.1 层级化的 Cluster Map**



![img](https://user-gold-cdn.xitu.io/2019/6/4/16b226b668fcdd89?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

CRUSH Map 是一个树形结构，OSDMap 更多记录的是 OSDMap 的属性(epoch/fsid/pool 信息以及 osd 的 ip 等等)。

叶子节点是 device（也就是 osd），其他的节点称为 bucket 节点，这些 bucket 都是虚构的节点，可以根据物理结构进行抽象，当然树形结构只有一个最终的根节点称之为 root 节点，中间虚拟的 bucket 节点可以是数据中心抽象、机房抽象、机架抽象、主机抽象等。

**● 5.3.2 数据分布策略 Placement Rules**

**数据分布策略 Placement Rules 主要有特点：**

\1. 从 CRUSH Map 中的哪个节点开始查找

\2. 使用那个节点作为故障隔离域

\3. 定位副本的搜索模式（广度优先 or 深度优先）







```
1 rule replicated_ruleset  #规则集的命名，创建pool时可以指定rule集
2 
3 {
4
5    ruleset 0                #rules集的编号，顺序编即可 
6
7    type replicated          #定义pool类型为replicated(还有erasure模式) 
8
9     min_size 1                #pool中最小指定的副本数量不能小1
10 
11    max_size 10               #pool中最大指定的副本数量不能大于10
12 
13    step take default         #查找bucket入口点，一般是root类型的bucket 
14 
15    step chooseleaf  firstn  0  type  host #选择一个host,并递归选择叶子节点osd 
16
17    step emit        #结束
18
19 }   复制代码
```





**● 5.3.3 Bucket 随机算法类型**

![img](https://user-gold-cdn.xitu.io/2019/6/4/16b2277dcc7827c7?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**一般的 buckets：**适合所有子节点权重相同，而且很少添加删除 item。

**list buckets：**适用于集群扩展类型。增加 item，产生最优的数据移动，查找 item，时间复杂度 O(n)。

**tree buckets：**查找负责度是 O (log n), 添加删除叶子节点时，其他节点 node_id 不变。

**straw buckets：**允许所有项通过类似抽签的方式来与其他项公平“竞争”。定位副本时，bucket 中的每一项都对应一个随机长度的 straw，且拥有最长长度的 straw 会获得胜利（被选中），添加或者重新计算，子树之间的数据移动提供最优的解决方案。

### ▍5.4 Ceph CRUSH 算法案例

**说明：**

集群中有部分 sas 和 ssd 磁盘，现在有个业务线性能及可用性优先级高于其他业务线，能否让这个高优业务线的数据都存放在 ssd 磁盘上。

**普通用户：**

![img](https://user-gold-cdn.xitu.io/2019/6/4/16b22791a066592d?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**高优用户：**

![img](https://user-gold-cdn.xitu.io/2019/6/4/16b22797ef5a5a4c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**配置规则：**

![img](https://user-gold-cdn.xitu.io/2019/6/4/16b2279f6deb9660?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

## ▍6. 定制化 Ceph RBD QOS

### ▍6.1 QOS 介绍

QoS （Quality of Service，服务质量）起源于网络技术，它用来解决网络延迟和阻塞等问题，能够为指定的网络通信提供更好的服务能力。

**问题：**

我们总的 Ceph 集群的 IO 能力是有限的，比如带宽，IOPS。如何避免用户争抢资源，如何保证集群所有用户资源的高可用性，以及如何保证高优用户资源的可用性。所以我们需要把有限的 IO 能力合理分配。

### ▍6.2 Ceph I

### O 操作类型

**ClientOp：**来自客户端的读写 I/O 请求。

**SubOp：**osd 之间的 I/O 请求。主要包括由客户端 I/O 产生的副本间数据读写请求，以及由数据同步、数据扫描、负载均衡等引起的 I/O 请求。

**SnapTrim：**快照数据删除。从客户端发送快照删除命令后，删除相关元数据便直接返回，之后由后台线程删除真实的快照数据。通过控制 snaptrim 的速率间接控制删除速率。

**Scrub：**用于发现对象的静默数据错误，扫描元数据的 Scrub 和对象整体扫描的 deep Scrub。

**Recovery：**数据恢复和迁移。集群扩/缩容、osd 失效/从新加入等过程。

 

### ▍6.3 Ceph 官方 QOS 原理

![img](https://user-gold-cdn.xitu.io/2019/6/4/16b227aece6c7be3?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

mClock 是一种基于时间标签的 I/O 调度算法，最先被 Vmware 提出来的用于集中式管理的存储系统。(目前官方 QOS 模块属于半成品)。

**基本思想：**

**·** reservation 预留，表示客户端获得的最低 I/O 资源。

**·** weight 权重，表示客户端所占共享 I/O 资源的比重。

**·** limit 上限，表示客户端可获得的最高 I/O 资源。

## ▍6.4 定制化 QOS 原理

### ● 6.4.1 令牌桶算法介绍

![img](https://user-gold-cdn.xitu.io/2019/6/4/16b227bb0e76e904?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

基于令牌桶算法(TokenBucket)实现了一套简单有效的 qos 功能，满足了云平台用户的核心需求。

**基本思想：**

- 按特定的速率向令牌桶投放令牌。
- 根据预设的匹配规则先对报文进行分类，不符合匹配规则的报文不需要经过令牌桶的处理，直接发送。
- 符合匹配规则的报文，则需要令牌桶进行处理。当桶中有足够的令牌则报文可以被继续发送下去，同时令牌桶中的令牌量按报文的长度做相应的减少。
- 当令牌桶中的令牌不足时，报文将不能被发送，只有等到桶中生成了新的令牌，报文才可以发送。这就可以限制报文的流量只能是小于等于令牌生成的速度，达到限制流量的目的。

 

### ● 6.4.2 RBD 令牌桶算法流程

![img](https://user-gold-cdn.xitu.io/2019/6/4/16b227c38cbfad70?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**步骤：**

- 用户发起请求异步IO到达Image中。
- 请求到达ImageRequestWQ队列中。
- 在ImageRequestWQ出队列的时候加入令牌桶算法TokenBucket。
- 通过令牌桶算法进行限速，然后发送给ImageRequest进行处理。

 

### ● 6.4.3 RBD令牌桶算法框架图

 

**现有框架图：**

![img](https://user-gold-cdn.xitu.io/2019/6/4/16b227cb3a4de191?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**令牌图算法框架图：**

![img](https://user-gold-cdn.xitu.io/2019/6/4/16b227d183922475?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

## 

作者：滴滴技术

链接：https://juejin.im/post/5cf635066fb9a07ed911ae84

来源：掘金

著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。