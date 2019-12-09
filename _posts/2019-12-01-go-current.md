---
title: 深入理解go——同步和并发设计模式
subtitle: go学习同步和并发设计模式
layout: post
tags: [go]
---

同步和并发设计模式

## 1. 基本同步原语

#### 1. 基本同步原语简介

什么是同步原语呢？ **同步原语就是方便我们进行并发编程的一些类库工具。**

首先第一个大家最常用到基本的原语就是Mutex，他是一个互斥锁.我们都知道**Goroutine中Mutex退出的时候，另外一个Goroutine才会可以操作**.使用的时候要避免死锁.

#### 2. 基本同步原语的演化历史

Mutex使用有演化的过程，从最初的不公平算法到现在相对公平的算法.

Mutex零值是未加锁的状态，Unlock未加锁会导致Panic，一般来说我们使用Mutex肯定在同一个方法进行Lock和Unlock，一般不会出现未加锁 .如果是在一个方法进行Lock，在另一个方法进行Unlock，就可能会出现Unlock未加锁的Mutex。

Mutex被一个Goroutine加锁，它不会对某一个Goroutine进行持有，所以另外一个Goroutine可以Unlock Goroutine加锁Mutex.另外一定要记住**Mutex是非重入的锁，在设计方法的时候，如果方法A和方法B都有对这个Mutex加锁，方法A调用方法B，因为方法A已经对Mutex进行加锁，如果进入方法B再进行加锁，会导致死锁的状态**.如果这里有JAVA工程师，JAVA有很重要特性就是可重入，有多次的lock是可以，但是在Mutex一定不要使用，肯定会导致死锁的状态。

#### 3. 2008年的基本同步原语

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SvYNbsJe3LEad8IoZHApYHFdfayiamul9bpxEy9YB6rFuLb9nvBaPiciaA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



2008年第一版的Mutex的实现很简单，第一个用Key看看是否加锁，**如果锁已经被Goroutine持有，如果再调用，会来阻塞调用者Goroutine**，Func会唤醒这个休眠的Goroutine.如果让我自己来写也很简单，我利用CS的原始操作，如果这个值里面没加锁，我就可以获取这个锁，然后这个交际被Goroutine已经持有了.如果已经获取锁就加入等待队列里面，暂时休眠，Unlock的时候我从这里唤醒.

我给大家看这个可以解决大部分功能，但是有问题：

第一，有可能每次都会被刚进来的Goroutine获取释放的锁，有可能很长的Goroutine没有机会获得锁，导致一个饥饿的状态.因为这里没有一个队列，也没有什么排队的顺序.谁抢得好就归谁.这个Mutex经过多次的修改.

#### 4. 2012年的基本同步原语

2012年的时候对锁的Key改成了State，**前面三位是Wait和饥饿的状态**.用一个字段做了至少三四个功能.后来加入**Spin的状态，如果当年在CPU运行的Goroutine，希望有更大机会获得这个锁**，这样避免了Key失效的功能，更好提高性能，但是不能一直死锁.**Spin一定次数之后就会进入休眠状态**.

#### 5. 2015年的基本同步原语

在Go1.5中Mutex实现为全协作式的,增加了spin机制.一旦有竞争,当前goroutine就会进入调度器.但是在临界区很短的情况下可能不是最好的解决方案.

#### 6. 2016年的基本同步原语

2016年增加了**饥饿模式**，让锁变得更公平.**不公平的等待时间限制在1毫秒。**

上一个月有一个新的Mutex修改，做了内联的优化.原来挨饿、饥饿的分享非常长，他抽取出来在fast保留，很长的检测出来方便内联，性能在这个锁竞争不是那么强烈的情况下，会有一些性能的提升.

## 2. 基本同步原语的实现

### 1. **互斥锁**

####  1.互斥锁的实现

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8S1zuSajUwuJu3c8xWMwReyicXf1SEQ8WCsfq6iazX2DiasD0J8alduian2A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



####  2. 互斥锁的状态

- 互斥锁有两个状态:**正常状态**、**饥饿状态**.
- **正常状态**：等待Goroutine按照FIFO顺序等待**.唤醒的Goroutine不会直接拥有锁，而会和新请求的锁Goroutine竞争**，谁拥有谁就获取这个锁.**如果一个等待的Goroutine超过1毫秒的情况下，<u>Mutex会转成饥饿状态</u>，会在第三位标记为1，说明这个锁是进入饥饿状态**.
- **在饥饿状态下，当一个持有的Unlock的Goroutine直接交给等待队列中的第一个**.**饥饿的Goroutine处理结束之后，发觉后面没有等待的Goroutine了，或者等待的Goroutine时间小于1毫秒，又转入正常状态**.这样避免饥饿的状态.



#### 3. 互斥锁的扩展

- TryLock,Count

另外对Mutex做一个扩展，如果有特殊情况下可以试一下.有一个Trylock和Count，如果获取锁就获取锁，如果没有获取就转而做其他事情.在Go有专门讨论，Go开发者认为Trylock名字不好听，到底是获取锁还是不获取锁这个反馈结果不明确.

另外认为这个场景还是比较少的.如果大家要有这样的需求，可以进行扩展，另外获取当前等待的Goroutine有多少数量，本来没有这种功能，但是可以通过这种方式获取过来.因为State往后开始是计数等待的Goroutine有多少个.通过指定的方式可以获取这个状态.查找是正常状态还是饥饿状态，都可以扩展。

- Mutex  IsWoken,IsStarving



![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8STdk9L4cEXcJ0iah7tAGzdw0dRbcIbgIibbYow7G8rhxZib7sfvbfcYnvQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

### 2. 读写锁

#### 1. 读写锁(RWMutex)适用的场景

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SiaicdvhdZ4PyXm3PrlkFyt3mZZ7ibEjnT10OVBfTbK2zscPaMZwqmUVBQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

另外`RWMutex`对应特定场景，我们读的时候比较少的情况下，要用读写锁可能提高性能，这个锁如果被持有只有两种情况，一种是被一堆的Reader持有，另外是被一个Writer持有.适合大并发Read场景，如果写一次以后经常读，大家可以用读写锁的同步的情况.在JAVA也有类似这样一个数据结果.零值也是未加锁的状态.另外很重要特点就是WRITER的Lock相对后续的Reader的Rlock有优先级.同时有新的Reader想调用Rlock想持有这个锁，当前面Writer释放这个锁之后，谁获取这个锁呢？是Reader优先获取这个锁，Reader释放之后Writer才获取这个锁.对写的情况有优先级.禁止递归读锁.虽然这个锁可以被多个Reader调用，但是写代码一定要避免在同一个Goroutine多次调用这个锁.假设有一个Reader有一个方法A，在A调用读锁，方法B也调用读锁，避免A调用B，B调用C，C调用D，D里面也有一个读锁Lock的方法，为了避免死锁的状态.根据前面第四条，后续Writer的Lock相对后续的Reader的Rlock优先级高.

RWMutex这里有控制Writer的设置，还有一个Reader的变量，这个Reader是读的状态.ReaderWait前面有多少Read获得这个锁没有释放，记录这个状态.

#### 2. 错误的读写锁(RWMutex)递归

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SE3CMDLr2KOMiaySeibcewkibPU1arIIdgTw6XjhgAGYKBgRIoN5OCWg3g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

这是一个递归的问题，这是一个错误的例子，禁止递归调用.这个RR是读的状态，Lock对读锁进行请求.接着又Defer可以读这个锁，问题最后一句是递归，递归调到RR，RR里面又会有读锁获取，里面多次读锁获取，碰巧递归过程中有一个Reader点Lock进行调用，就会进入死锁的状态.

#### 3. 获取锁的数量

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SdCEIjU3xic6rNaNTfNtLotd65nXGrbwicCBRMh7gS3ETANUXdjUhCqAw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

如果你想获取有多少读锁的数量还有写的锁数量，可以通过这种方式获得.

### 3. **条件变量Cond**

Mutex在有些情况不太适用.比如说有一个消息通知机制，释放锁以后，不是等待任何一个去获取下一次的获取不获取，他是想我可能通知一个，也可能通知所有的，我这个状态已经满足了，你可以继续下一个动作了.从这个描述来说，比较类似于Monitor同步言语，Monitor内部也用Mutex加条件的Goroutine的容器.对于这种情况基本同步原语提供了Cond来满足.每个Cond和一个Locker相关联，改变条件或者调用Wait需要获取所，控制并发的问题.

```
上面这个例子看起来乱七八糟，但是我可以说你可以比喻奥运会比赛的时候，十个运动员站在起跑线正在做热身运动，然后有个裁判指发令枪，看大家准备好了没有，如果准备好了发令枪一响，所有运动员开始起跑.
```

我喜欢跟JAVA比较，JAVA有两个数据结构，他有点类似.下面一个数据也类似，我们一开始定义了一个Mutex这一个Lock的接口具体的实践，相当于十个运动员准备好了，可以继续往下进行了，在这个过程当中，启动十个Goroutine，Goroutine开始的时候会调用MLock，做一些准备，或者初始化，准备好以后，就释放信号至少我这个Goroutine准备好了，下面判断这个条件是不是真的准备好了，里面调用一个CWait等待这个条件，如果没有准备好会等在这儿，等待信号的发生，如果信号被触发以后，条件准备好了，大家可以一起做下一步的动作.前面有两个假动作，之所以是假的，因为在条件没有准备好的情况下，你可以调用这个方法，但是你这个方法只会给前面的Goroutine一个假信号.等都设置好了以后，发一个准备好的信号，下面Goroutine可以继续执行.

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SCBFyaDRnWMGW70KPWNd0bBY9aX3mGVwXGO3CEeUTFnzbn3VAft9lFg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

Cond最主要三个方面，一是Broadcast，我给所有的Goroutine信号，Signal给所有的Goroutine信号，Wait是等待条件是否满足.前面两个不要求加锁，调用Wait的时候一定要加锁.而且里面有一个先解锁后再加锁，中间有时间间隔，如果你对其他变量进行更改，可能有数据不一致的状态，所以前面用FOR判断条件是否真的满足了.

### 4. **Waitgroup**

另外很常用是Waitgroup，它更像Java的Countdown、Lach的结构，CyclicBarrier），比如说百米赛跑可能20秒就解决完了，就统计结果，我会用Barrier统计运动员是否完成了，统一做下一步的事情.这两个数据过程中可以支一个Waitgroup实现.Waitgroup有一个ADD方法，一开始是多少，可以一次设置好，也可以中间多次进行设置.这个ADD方法可以设置负值，相当于减相应的数目，如果计数期小于零会Panic.所以避免.前面如果不为零，所有调用Wait方法会被阻塞住，等待所有的Waitgroup进行下一步工作.

Waitgroup可以重用，但是重用的时候要避免下面几种情况.

ADD的时候一定要在Wait方法之前完成.

第一个Waitgroup方法之前要把Add增加上，后面错误的例子是说，因为Waitgroup和下面Wait是并行执行，有可能add方法没有执行，Wait方法已经执行了，前面的Goroutine没有执行，后面Waitgroup已经执行过了.

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SP8aEw6MCeediaJCicjmSicFxSnCL3NdRvrf6NwnehiamezAQtqjVXcjczg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SsiasHia0UyWAww5xVozhaqZFf89xSb0Khac3U3eOLiamcqNIQ95d2tr6Q/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

另外，虽然说Waitgroup是可以重用的，但是避免在Done分子，Done是add的附值，这个是有一个并行.我把错误的贴在这儿，因为Waitgroup检测很多状态不一致的情况，一旦一个Wait没有执行完，前面的Done已经执行结束，接着再add，状态就不知所措了，到时候看到一个错误的状态.这种情况是说，等于Wait可以执行多次，但是Done一定避免超过设置的数，前面设置10，后面Wait调用几次没有关系，但是右边的例子我设置是10，里面执行10次，如果外面5，如果我执行多一次，Done会导致Panic.这是一个Wait、Done还有Add执行当中避免的错误.

所以我们在Waitgroup使用的时候，保持这样一个原理，你前面一开始要执行Wait接受器设置好，然后等Wait，等Wait执行之后再重用，再进行新的Add方法增加，这样至少保证避免Add并发错误.

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8STh9TVB5AMMGy4YHICMz1mxEXWN78ia33SVFl9yO7Uy6q3K9AllNwErQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8Sjjm7lHJI1hNsibMHK4ZZsrqJEzzzJVWWyKarMrkpItmYfq3dxFhZXVw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

### 5.**Once**

once 的Do只执行一次.

避免死锁.

即使func panic,once 也认为他完成了.

once是执行单锁,可以执行一次初始化.单例有多种方法，一个是常量,再就是变量,可以用初始化操作,另外可以使用Getlnstance里面加一个锁，每一次获取一个变量有一个锁请求，这个会导致性能问题.通过Sync.once实现一次性初始化，性能不会带来特别的牺牲.once实现了一个done的功能，一开始检查Done完了，就返回了，这样性能不会带来太多牺牲.

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SgzFeSCVrtdztxoOzRHbE0ubvMkddwibOLJj8evs7xssrfzdWWLW8QNw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

所有性能框架，Go下面你会看到这样一句话，第一次使用不能被Copy，因为一开始我们所有的零值都是无锁的，使用后有状态的.原来有机会释放锁，但是Copy过来没有机会释放锁.

### 6.**临时对象池 POOL**

**POOL临时对象池**是暂时的，而不是长久的，因为可能在任何时候任意的对象可能被移除.但是可以安全并发访问，本身有一个装箱和开箱的操作.所以使用POOL一定避免有一些情况导致内存泄露的问题.这是Go标准库里面的Bug，本身的例子就是内存泄露了.

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SBibvicfDxlPa1ib4iaXY82QVG73XFl6Oib6YAxJln78JwiaibX3Q3HjG3O1vg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

这个是一个BufPOOL池的例子.

这下面是我们标准包里面的两个错误的使用的例子，一个是FMT在格式化输出的时候有一个池，一开始放又回过来BufPOOL真相，放在这个池里.但是现在改与小于一定数量的BufPOOL才放到池子里，否则就丢掉了.在新的版本中会算已经修复的新版本中.

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SbMzU3Cluoic2Sqy1PJp9RKVXr0JMo103TQFb2eafZ2BjCib2NSpvAfiag/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SbMzU3Cluoic2Sqy1PJp9RKVXr0JMo103TQFb2eafZ2BjCib2NSpvAfiag/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**MAP**

另外有一个MAP，我们知道Java的MAP不是安全，大家踩过这个坑.为了提供并发的访问，Go的MAP适用两个Cases，第一是设置一次多次读.另外如果多个Goroutine并发读写，更新不同的Key也是适用于这个MAP.这个MAP是内部实现了两个方面，用时间换空间的思想，使用两个MAP，MAP其中一个假如说这是一个MAP，他这里叫做带锁的功能，内部叫Derta，如果你增加新的数据放在DertaMAP，DertaMAP一直往这儿放，因为有锁所以性能不好，如果访问不好每次访问都是访问只读的MAP，就要开一个Mees转到Derta查询，查询一定次数，会把DertaMAP给只读MAP，Dertamap是空值.再增加新的MAP，Derta可以放在新的MAP里面.DertaMAP里面总是最新的数据.Range有可能新增加的不存在，现在整个MAP获取，如果没有再进行进一步加锁到Derta里面去.Range性能不好.



## 2. 扩展同步原语

前面说了Mutex不能重入.但是我们编写的时候可能会有重入的方法，写的东西太多了没有办法解决这个问题.可以用标记，导致这个锁是谁持有，另外他已经重入多少次，通过两个变量的情况下，我们就可以来判断获取这个锁我们到底是请Goroutine是否持有这个锁，如果持有就加一个锁，如果不持有就让他等待.获取GoID有两个方法，一个是通过自己的方法解析，第二是通过Go获取ID.第二个方法性能更好.

- **ReentrantLock**

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8S8N9iawE7kXpD3ScwPlluED3bFEVTjolokKhaOmUOV1YxyBTgbOGdQJw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8Sfke6VwiawvviapicDZLWXMicFp039Mtwrcic9kdF2nj70QR2SklbkBrJdfw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

- **ReentrantLock**

**![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SnlFU60TrpibEHO5ZgOicWGuiazcpZht17jJnTLF1Qj31e0NpRv0Dz7d6w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)**



我们学过操作系统都知道，他有两个概念，一个是二进制实现Mutex的功能，另外可以通过技术器的方式实现多个情况.

**GoSemaphore**可以初始化**Semaphore**量是多少，用这个量获取，释放这个信号量，信号量可以是一组资源，我有十台打印机，把这个信号量设置成十.等第十一就没有了，就等待，前面如果有等待的Goroutine会重新化解.

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8Sx27bTPiah1zA0t3OkcbymT9tHlGozveEs7WMv3PhLBJh51XC2icxpYgQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

- **SingleFlight**

**单飞模式**:有一组任务过来的时候，我可以挑出其中一个让他执行任务，等他执行结束把结果告诉等待剩下一系列的人让他返回，我选择一个执行，有好处是避免使用的状态.

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8S73P9lUdjGpLtZgbb4RqOOzoFT4v4BRdcHNQ6HaA9qVOriakC5dtrGBw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

- **ErrGroup**

ErrGroup刚才有老师讲了，我说两个问题，**Wait方法是等待所有的并发函数执行结束才能返回**.即使其中的调用方法有错误之后，他也不会立马返回，必须等到所有的函数一起返回.通过方法返回的Error只保持第一个Error，其他后续请求函数也是错误的话，也没有办法找到你，除非用额外的变量退出.如果不是等待所有的函数返回，可以用Context，因为Context是内部机制，等所有完成之后，Context可以执行完，可以监测ErrorContext检测，到底是第一个错误请求，还是说函数都执行完了.

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SQTHykAvmV9DAk8eHAicJUDNIG1dwuajIWcjrV1aZJkoibY9MWQlHKyXQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



- **Spinlock**

另外Spinlock用自旋锁，性能非常高.

**在压力不是特别大的情况下性能非常好**.另外可以保护多个性能进行加锁，这个进程有共享的对象在操作系统上，可以通过稳健锁进行锁控制.

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SQTHykAvmV9DAk8eHAicJUDNIG1dwuajIWcjrV1aZJkoibY9MWQlHKyXQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

- **Concurrent  map**

Concurrent map和Java非常相似的，一个是访问Key，另外不断增加性能建设情况下，我可以把这个锁进行细化，可以产生32个锁，然后分成32个Shared，这样可以快速进行锁的读取.

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8Seva1VmRtrPQUcCfIEThoutX2Jx7YEKWkL8QE3dG1jvzcWfcfrxNoKw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

## **原子操作**

### **原子操作及数据类型**

另外atomic提供了很多类似的方法，我把这些函数、数据内容和方法进行分类，这样通过两页PPT把所有方法都记住了.一个是所有执行的类型是32、64、还有Uint32、Uint64，还有无指征.方法是提供Add方法，另外提供CAS的方法.如果读取就是Load，存储用Store，Swap和原来交换，返回原来的指征.

- atomic 数据类型

1. int32
2. int64
3. uint32
4. uint64
5. uintptr
6. unsafe.Pointer 

- atomic 方法                

 AddXXX (整数类)    

1. CompareAndSwapXXX（cas）
2. LoadXXX（读取）
3. StoreXXX (存储）
4. SwapXXX （交换）

**Subtract方法**  

有Add没有Substract方法？

有符号类型    可以使用ADD负数

无符号类型 AddUint32(&x, ^uint32(c-1)),AddUint64(&x, ^uint64(c-1))

无符号类型减1 AddUint32(&x, ^uint32(0))􏰁 AddUint64(&x, ^uint64(0) 

**value 方法**

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SKDxW4mY3YrPX4ankNoMAoJWCXNATC4sibsuNVI7F50HzfSz68QQdic9g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)





## 3. channel

### 1. **channel 功能**

Channel主要是用于信号通知，数据的交流，利用Channel可以实现锁.

### 2. **channel特殊情况**

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SK859YG3PIgrX0Z4CrkqxaKqvJIsWmVMQRofZXtUk1DNxE7mIOxlNYA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

Channel一些特殊的类型，这个表大家在用的时候一定要记住，如果记不住中间使用Channel的时候经常会犯一些错误.我把比较重要的点标出来，在某些行为会被Blok住，在某些情况会Panic.

**Locker**



![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SbZrUWpka49qzicevhlKC7anW24QicwFJruiba75gWWSQKmKj6sksPBTWA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8Sk13UAS6cn32csoOgpH9hNw87ibJKgsRbtENbMDfndNSO1y9G5IRC33A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SCZ2RZ1GleX6OknDPXSWbngBpMzK9VqSRMIwMIoD5ATjSThmnic8mS5A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SKZI3YckTVXa4mm68kBlibBUibW5EMq9XZhywaRiaAtZSshNGw4h8ibXnNQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

这是利用Locker锁的结构.

在后面我们**Go数据模型**会介绍，为什么使用Channel的方式，怎么实现锁的功能，主要是有一个规则在里面.

这是另外利用Channel，这里利用Channel有两种写法.

### 3. **channel vs mutex**

刚才说Channel实现锁，Mutex也可以实现锁，到底我们使用Channel还是使用Mutex.从G因为Channel因为Go有一篇文章说还是根据场景选择不同的言语.

#### 1. channel 适用场景

Channel用在什么情况下呢？

- 传递的数据的时候,类似Rast把数据传递给你就不管了，以后传递给下一个Goroutine处理.
- 分发任务单元，做一个应用池的时候，前面把任务包装好，仍给其中某一个Worker就行了，Worker用Channel实现了任务列表，他从Channel取任务就OK.
- 另外交流异步的结果，可以利用Channel进行传递
- 还可以进行复杂编排功能.简单可以用RerderGroup就行了.

#### 2.  Mutex 适用场景

**Mutex是对Cache和带状态的对象进行加锁，主要是对临界区的保护.**

**channel 使用的错误率**

上个月在一次大会上有一个留美中国人分析了当前比较流行的Go框架的一些并发的错误，做了一个总结.这个文章应该是几个月之前已经登在网上了，上个月大会的时候正式宣讲.大家可以找找看，网上都有这个文章，很有意思的例子.流行的Go处理他们并发的错误非常多，包括标准库里面都有一些错误的使用，使用Channel和比使用标准的传统的基本同步言语错误率还要高，因为Channel刚才列出表有很多异常情况，这些异常情况，万一没注意，或者根本没有处理，会导致你会有意外的Panic的情况发生.

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8Sj2hiaouJfrFzTc1GdjOYJBNl4B8Y5qkTBTiav0YHZ64RQQcxicNDPicibYw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**利用channe来实现一些设计模式**

利用Channel可以实践一些Done的设计模式，可以用二分法递归的方式，或者用Fan—in的方式.另外使用Channel可以实现多个数据源取一个数据源，从一个数据源出来.同样还有善后的功能，多个数据源选一组输出.另外可以写Pipeline的功能.写软件具体不讲了，我只是给大家提供这样一个Channel的设计模式.另外还有写流动功能，比如说Reduce的功能.

- orDone



![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SWcFdMtrvtFtkpLgUzNU4SYkc2dR5X6XgMibet4RjB6v6iclrAwm98KPg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8S4DYUb3rFKibQZQx9akENeyceMB2Eiaq9icaG0wz6IcAM82ItHicCFXOwGQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

- or 

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SkmNpwCvJ2kVqkz67e27D4400xChbEMZk0eib36AZ2jVTfuOvBjp1NQw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

- or 二分法递归

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

- or 反射

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

- fan-in递归

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8S8wY5tlVQMmsvZJSkB2s3VIc8wObZUkU3pcxq7bVoHh0HoBAM5EC2ag/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SOIDicErW798cWxg97ic9eM7vm8pMWnY4QyyVyO4bdPruzSIqkWHTtdPg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

- fan-in 反射



![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SDaEe3o4nQHK8yX2IdDyWd4qicRmqNib5uHEko3iaF20oiaaMFmF0icBpwhw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SB3tnIYBP2bgZVdS1VyN5kQkribPWlBAKJ4kBvjNUknf7o23hUKRDLyw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

- fan-out



![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SsicNNiazkdgfJ5qDKeJQFHk0jjDRE9iclT8jnQgZibKgxJBE09uMaH0Hgw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)





- tee



![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)





- pipeline

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)



- stream-skip

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SianicxPy1g7gFadX453jJs53DfG2zib6k4dTTVksGPlKeSjIrrZRe95TQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SomrL7bVxIiastRMw8gOm2NibuNARKicOTxlBvrpUjJUGSlh4cTTUDeic5g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



- stream-take

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8SWqvF0O5hUrVFHlUv9OkDib7UV5es13Cl94S99G3qWuwh8iaqI2T2PFwA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



- stream-map

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8S72RbMJNV5F2F2AVm8PCXuFgMMokFTZz061p0FUtaLGBMQQOGYEqMSw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



- stream-reduce

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8StsfoP8ibNrsOH03qiaJkXib4VMVUibWLcKmiaTlvRkDCB0kM9GwoUQvNs5Q/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

04

内存模型



Go内存模型所提供的就是Go内存模型的文章，内存模型描述了线程通过内存的交互以及数据的共享使用.

**历史**

最早是Java，C++也定义了内存模型.Go。

**go 内存模型**

在文章中定义了内存模型是这样定义的

- 对一同一个变量如何保证在一个Goroutine对此变量读的时候能观察到其他Goroutine对此变量的写.
- 修改一个同时被多个goroutine并行访问的变量时，需要进行串行化访问
- 通过channel或者其他同步原语进行串行化访问







**happen-before**

**同一个 Goroutine**

同一个Goroutine他的执行是和代码定义的顺序是一样的.这个里面会有一些重复的问题，但是不影响你的理解，你可以认为是和你编写的顺序是按照编写的顺序执行的.

**strict partial order**

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

**init函数**

对于Init函数，如果在PackageP引入了定义的PackageQ，就是这个Q的init函数一定要在Happens  beforeP的Init之前执行，Main函数一定在其他引用函数之后才执行.

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

**go语句**

Go运行其中一个函数，在Go的语句之前定义的变量，一定是对Goroutine函数是可见的，里面如果打印前面的变量一定能正面打印出来.不会打印空的，还没赋值的状态

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

**channel**

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

**mutex/rwmutex**

Mutex和RWMutex保证你在设计你的代码的时候，你如果不确定你代码的执行顺序是什么样，可以对照规则排查你的代码是不是写的按照你所设想的顺序执行.

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8S32PxCMYXg3KuRNn2uw6ibrXJgx3IJEcvn0gtXjajibr8ebryePkTNoPA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)**waitgroup**

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibDrQhDD5yrAl26pHTkZYD8ShEazUllQ4LGGibwspuE2rBg6yGNO5GvJLtbPOibefiavNmhS6e5yuUibAA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**once**

once.do 方法的返回一定happen before 任何一个方法 once.do 的返回

**atomic Atomic**

没有官方的保证，建议不要依赖atomic保证内存的顺序.