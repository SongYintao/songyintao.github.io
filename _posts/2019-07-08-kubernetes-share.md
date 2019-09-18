---
title: Kubernetes 在xima的应用
subtitle: kubernetes在测试环境的落地实践
tags: [kuberenetes test]
layout: post
---

由于公司业务的扩张，测试环境容器化的需求越来越迫切，之前的基于mesos的marathon框架已经不能很好的支持未来的发展趋势。容器云的kubernetes化迫在眉睫。

本文结构如下：

- 背景知识：主要简单介绍什么是容器化，容器化的优点、和之前的对比。容器化的背景（测试环境发布费劲）、容器原理及其相关的知识。
- kubernetes简介，与marathon 的对比
- 实践：集群的部署、监控、报警、无损发布、日志、镜像构建、网络选型、存储、静态、应用容器化改造、对接mainstay、网关、灰度发布、上下文依赖
- 存在的问题
- 展望



## 背景

主要内容及其背景知识：

- 为什么需要容器化
- 容器基础（镜像构建、存储、分发）
- linux相关背景知识
- mesos、kubernetes简介



###  为什么需要容器化？

基于物理机的发布

- 需要维护一套测试环境的基础设施，运维负担较高
- 资源利用率不高
- 不能够一次构建到处运行
- 底层运行环境的差异

基于容器的发布

- 一次构建到处运行
- 资源利用率较高
- 灵活性较强、可以快速的扩容、缩容
- 屏蔽操作系统的底层差异
- 运维成本较低



### 容器基础知识

#### 1.容器的本质

一句话说明白容器是啥

所谓 Docker 镜像，其实就是一个压缩包。但是这个压缩包里的内容，比 PaaS 的应用可执行文件 + 启停脚本的组合就要丰富多了。实际上，大多数 Docker 镜像是直接由一个完整操作系统的所有文件和目录构成的，所以这个压缩包里的内容跟你本地开发和测试环境用的操作系统是完全一样的。

假设你的应用在本地运行时，能看见的环境是 CentOS 7.2 操作系统的所有文件和目录，那么只要用 CentOS 7.2 的 ISO 做一个压缩包，再把你的应用可执行文件也压缩进去，那么无论在哪里解压这个压缩包，都可以得到与你本地测试时一样的环境。当然，你的应用也在里面！

只要有这个压缩包在手，你就可以使用某种技术创建一个“沙盒”，在“沙盒”中解压这个压缩包，然后就可以运行你的程序了。 更重要的是，这个压缩包包含了完整的操作系统文件和目录，也就是包含了这个应用运行所需要的所有依赖，所以你可以先用这个压缩包在本地进行开发和测试，完成之后，再把这个压缩包上传到云端运行。 在这个过程中，你完全不需要进行任何配置或者修改，因为这个压缩包赋予了你一种极其宝贵的能力：本地环境和云端环境的高度一致！



你只需要提供一个下载好的操作系统文件与目录，然后使用它制作一个压缩包即可，这个命令就是：

docker build xxx

一旦镜像制作完成，用户就可以让 Docker 创建一个“沙盒”来解压这个镜像，然后在“沙盒”中运行自己的应用，这个命令就是：

```
docker run xxx
```

docker run 创建的“沙盒”，也是使用 Cgroups 和 Namespace 机制创建出来的隔离环境。

所以，Docker 项目给 PaaS 世界带来的“降维打击”，其实是提供了一种非常便利的打包机制。这种机制直接打包了应用运行所需要的整个操作系统，从而保证了本地环境和云端环境的高度一致，避免了用户通过“试错”来匹配两种不同运行环境之间差异的痛苦过程。

不过，Docker 项目固然解决了应用打包的难题，但正如前面所介绍的那样，它并不能代替 PaaS 完成大规模部署应用的职责。

- 容器技术的兴起源于 PaaS 技术的普及； 

- Docker 公司发布的 Docker 项目具有里程碑式的意义； 

- Docker 项目通过“容器镜像”，解决了应用打包这个根本性难题。

核心：

容器本身没有价值，有价值的是“容器编排”。

也正因为如此，容器技术生态才爆发了一场关于“容器编排”的“战争”。而这次战争，最终以 Kubernetes 项目和 CNCF 社区的胜利而告终。所以，这个专栏后面的内容，我会以 Docker 和 Kubernetes 项目为核心，为你详细介绍容器技术的各项实践与其中的原理。



#### 2. 资源隔离

容器，到底是怎么一回事儿？



容器技术的核心功能，就是通过约束和修改进程的动态表现，从而为其创造出一个“边界”。 

对于 Docker 等大多数 Linux 容器来说，Cgroups 技术是用来制造约束的主要手段，而Namespace 技术则是用来修改进程视图的主要方法。 

你可能会觉得 Cgroups 和 Namespace 这两个概念很抽象，别担心，接下来我们一起动手实践一下，你就很容易理解这两项技术了。

我们在 Docker 里最开始执行的 /bin/sh，就是这个容器内部的第 1 号进程（PID=1），而这个容器里一共只有两个进程在运行。这就意味着，前面执行的 /bin/sh，以及我们刚刚执行的 ps，已经被 Docker 隔离在了一个跟宿主机完全不同的世界当中。



这究竟是怎么做到呢？ 

本来，每当我们在宿主机上运行了一个 /bin/sh 程序，操作系统都会给它分配一个进程编号，比如 PID=100。这个编号是进程的唯一标识，就像员工的工牌一样。所以 PID=100，可以粗略地理解为这个 /bin/sh 是我们公司里的第 100 号员工，而第 1 号员工就自然是比尔 · 盖茨这样统领全局的人物。 

而现在，我们要通过 Docker 把这个 /bin/sh 程序运行在一个容器当中。

这时候，Docker 就会在这个第 100 号员工入职时给他施一个“障眼法”，让他永远看不到前面的其他 99 个员工，更看不到比尔 · 盖茨。这样，他就会错误地以为自己就是公司里的第 1 号员工。

 这种机制，其实就是对被隔离应用的进程空间做了手脚，使得这些进程只能看到重新计算过的进程编号，比如 PID=1。

可实际上，他们在宿主机的操作系统里，还是原来的第 100 号进程。

 这种技术，就是 Linux 里面的 Namespace 机制。而 Namespace 的使用方式也非常有意思：它其实只是 Linux 创建新进程的一个可选参数。

我们知道，在 Linux 系统中创建线程的系统调用是 clone()，比如：

```
int pid = clone(main_function, stack_size, SIGCHLD, NULL); 

```

这个系统调用就会为我们创建一个新的进程，并且返回它的进程号 pid。 

而当我们用 clone() 系统调用创建一个新进程时，就可以在参数中指定 CLONE_NEWPID 参数，比如：

```
 int pid = clone(main_function, stack_size, CLONE_NEWPID | SIGCHLD, NULL); 
```

 这时，新创建的这个进程将会“看到”一个全新的进程空间，在这个进程空间里，它的 PID 是 1。

之所以说“看到”，是因为这只是一个“障眼法”，在宿主机真实的进程空间里，这个进程的 PID 还是真实的数值，比如 100。 

当然，我们还可以多次执行上面的 clone() 调用，**这样就会创建多个 PID Namespace**，而每个 Namespace 里的应用进程，都会认为自己是当前容器里的第 1 号进程，它们既看不到宿主机里真正的进程空间，也看不到其他 PID Namespace 里的具体情况。

 而除了我们刚刚用到的 **PID Namespace**，Linux 操作系统还提供了 **Mount、UTS、IPC、Network 和 User 这些 Namespace**，用来对各种不同的进程上下文进行“障眼法”操作。 

比如，Mount Namespace，用于让被隔离进程只看到当前 Namespace 里的挂载点信息；

Network Namespace，用于让被隔离进程看到当前 Namespace 里的网络设备和配置。 

这，就是 Linux 容器最基本的实现原理了。 所以，**Docker 容器这个听起来玄而又玄的概念，实际上是在创建容器进程时，指定了这个进程所需要启用的一组 Namespace 参数。这样，容器就只能“看”到当前 Namespace 所限定的资源、文件、设备、状态，或者配置。**而**对于宿主机以及其他不相关的程序，它就完全看不到了**。 所以说，**容器，其实是一种特殊的进程而已**。

![](https://static001.geekbang.org/resource/image/80/40/8089934bedd326703bf5fa6cf70f9740.png)



谈到为“进程划分一个独立空间”的思想，相信你一定会联想到虚拟机。

而且，你应该还看过一张虚拟机和容器的对比图。 

这幅图的左边，画出了虚拟机的工作原理。其中，名为 Hypervisor 的软件是虚拟机最主要的部分。

它通过硬件虚拟化功能，模拟出了运行一个操作系统需要的各种硬件，比如 CPU、内存、I/O 设备等等。然后，它在这些虚拟的硬件上安装了一个新的操作系统，即 Guest OS。 

这样，用户的应用进程就可以运行在这个虚拟的机器中，它能看到的自然也只有 Guest OS 的文件和目录，以及这个机器里的虚拟设备。

这就是为什么虚拟机也能起到将不同的应用进程相互隔离的作用。 

而这幅图的右边，则用一个名为 Docker Engine 的软件替换了 Hypervisor。这也是为什么，很多人会把 Docker 项目称为“轻量级”虚拟化技术的原因，实际上就是把虚拟机的概念套在了容器上。

在理解了 Namespace 的工作方式之后，你就会明白，跟真实存在的虚拟机不同，在使用 Docker 的时候，并没有一个真正的“Docker 容器”运行在宿主机里面。

**Docker 项目帮助用户启动的，还是原来的应用进程，只不过在创建这些进程时，Docker 为它们加上了各种各样的 Namespace 参数**。 这时，这些进程就会觉得自己是各自 PID Namespace 里的第 1 号进程，只能看到各自 Mount Namespace 里挂载的目录和文件，只能访问到各自 Network Namespace 里的网络设备，就仿佛运行在一个个“容器”里面，与世隔绝。 不过，相信你此刻已经会心一笑：这些不过都是“障眼法”罢了。

在这个对比图里，我们应该把 Docker 画在跟应用同级别并且靠边的位置。这意味着，用户运行在容器里的应用进程，跟宿主机上的其他进程一样，都由宿主机操作系统统一管理，只不过这些被隔离的进程拥有额外设置过的 Namespace 参数。而 Docker 项目在这里扮演的角色，更多的是旁路式的辅助和管理工作。

![](https://static001.geekbang.org/resource/image/9f/59/9f973d5d0faab7c6361b2b67800d0e59.jpg)

不过，有利就有弊，基于 Linux Namespace 的隔离机制相比于虚拟化技术也有很多不足之处，其中最主要的问题就是：隔离得不彻底。 

首先，既然容器只是运行在宿主机上的一种特殊的进程，那么多个容器之间使用的就还是同一个宿主机的操作系统内核。

其次，在 Linux 内核中，有很多资源和对象是不能被 Namespace 化的，最典型的例子就是：时间。 

这就意味着，如果你的容器中的程序使用 settimeofday(2) 系统调用修改了时间，整个宿主机的时间都会被随之修改，这显然不符合用户的预期。相比于在虚拟机里面可以随便折腾的自由度，在容器里部署应用的时候，“什么能做，什么不能做”，就是用户必须考虑的一个问题。 

此外，由于上述问题，尤其是共享宿主机内核的事实，容器给应用暴露出来的攻击面是相当大的，应用“越狱”的难度自然也比虚拟机低得多。 

更为棘手的是，尽管在实践中我们确实可以使用 Seccomp 等技术，对容器内部发起的所有系统调用进行过滤和甄别来进行安全加固，但这种方法因为多了一层对系统调用的过滤，一定会拖累容器的性能。

何况，默认情况下，谁也不知道到底该开启哪些系统调用，禁止哪些系统调用。

 所以，在生产环境中，没有人敢把运行在物理机上的 Linux 容器直接暴露到公网上。

当然，我后续会讲到的基于虚拟化或者独立内核技术的容器实现，则可以比较好地在隔离与性能之间做出平衡。

**在介绍完容器的“隔离”技术之后，我们再来研究一下容器的“限制”问题。** 

也许你会好奇，我们不是已经通过 Linux Namespace 创建了一个“容器”吗，为什么还需要对容器做“限制”呢？

 我还是以 PID Namespace 为例，来给你解释这个问题。

 虽然容器内的第 1 号进程在“障眼法”的干扰下只能看到容器里的情况，但是宿主机上，它作为第 100 号进程与其他所有进程之间依然是平等的竞争关系。

这就意味着，虽然第 100 号进程表面上被隔离了起来，但是它所能够使用到的资源（比如 CPU、内存），却是可以随时被宿主机上的其他进程（或者其他容器）占用的。

当然，这个 100 号进程自己也可能把所有资源吃光。

这些情况，显然都不是一个“沙盒”应该表现出来的合理行为。

 而**Linux Cgroups 就是 Linux 内核中用来为进程设置资源限制的一个重要功能**。

 有意思的是，Google 的工程师在 2006 年发起这项特性的时候，曾将它命名为“进程容器”（process container）。实际上，在 Google 内部，“容器”这个术语长期以来都被用于形容被 Cgroups 限制过的进程组。后来 Google 的工程师们说，他们的 KVM 虚拟机也运行在 Borg 所管理的“容器”里，其实也是运行在 Cgroups“容器”当中。这和我们今天说的 Docker 容器差别很大。

 **Linux Cgroups 的全称是 Linux Control Group。它最主要的作用，就是限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等。**

此外，Cgroups 还能够对进程进行优先级设置、审计，以及将进程挂起和恢复等操作。在今天的分享中，我只和你重点探讨它与容器关系最紧密的“限制”能力，并通过一组实践来带你认识一下 Cgroups。 在 Linux 中，Cgroups 给用户暴露出来的操作接口是文件系统，即它以文件和目录的方式组织在操作系统的 /sys/fs/cgroup 路径下。在 Ubuntu 16.04 机器里，我可以用 mount 指令把它们展示出来，这条命令是：

```shell
root@test-a4-60-89:~# mount -t cgroup
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/lib/systemd/systemd-cgroups-agent,name=systemd)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_cls,net_prio)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
```



可以看到，在 /sys/fs/cgroup 下面有很多诸如 cpuset、cpu、 memory 这样的子目录，也叫子系统。

这些都是我这台机器当前可以被 Cgroups 进行限制的资源种类。

而在子系统对应的资源种类下，你就可以看到该类资源具体可以被限制的方法。

比如，对 CPU 子系统来说，我们就可以看到如下几个配置文件，这个指令是：

```shell
ls /sys/fs/cgroup/cpu
cgroup.clone_children cpu.cfs_period_us cpu.rt_period_us  cpu.shares notify_on_release
cgroup.procs      cpu.cfs_quota_us  cpu.rt_runtime_us cpu.stat  tasks

```

如果熟悉 Linux CPU 管理的话，你就会在它的输出里注意到 cfs_period 和 cfs_quota 这样的关键词。这两个参数需要组合使用，可以用来限制进程在长度为 cfs_period 的一段时间内，只能被分配到总量为 cfs_quota 的 CPU 时间。

而这样的配置文件又如何使用呢？ 你需要在对应的子系统下面创建一个目录，比如，我们现在进入 /sys/fs/cgroup/cpu 目录下：

```
root@ubuntu:/sys/fs/cgroup/cpu$ mkdir container
root@ubuntu:/sys/fs/cgroup/cpu$ ls container/
cgroup.clone_children cpu.cfs_period_us cpu.rt_period_us  cpu.shares notify_on_release
cgroup.procs      cpu.cfs_quota_us  cpu.rt_runtime_us cpu.stat  tasks

```

这个目录就称为一个“控制组”。你会发现，操作系统会在你新创建的 container 目录下，自动生成该子系统对应的资源限制文件。 现在，我们在后台执行这样一条脚本：

```
while : ; do : ; done &
[1] 226

```

显然，它执行了一个死循环，可以把计算机的 CPU 吃到 100%，根据它的输出，我们可以看到这个脚本在后台运行的进程号（PID）是 226。

在输出里可以看到，CPU 的使用率已经 100% 了（%Cpu0 :100.0 us）。 而此时，我们可以通过查看 container 目录下的文件，看到 container 控制组里的 CPU quota 还没有任何限制（即：-1），CPU period 则是默认的 100 ms（100000 us）：

```
cat /sys/fs/cgroup/cpu/container/cpu.cfs_quota_us 
-1
$ cat /sys/fs/cgroup/cpu/container/cpu.cfs_period_us 
100000

```

接下来，我们可以通过修改这些文件的内容来设置限制。 比如，向 container 组里的 cfs_quota 文件写入 20 ms（20000 us）：

```
echo 20000 > /sys/fs/cgroup/cpu/container/cpu.cfs_quota_us
```

结合前面的介绍，你应该能明白这个操作的含义，它意味着在每 100 ms 的时间里，被该控制组限制的进程只能使用 20 ms 的 CPU 时间，也就是说这个进程只能使用到 20% 的 CPU 带宽。 接下来，我们把被限制的进程的 PID 写入 container 组里的 tasks 文件，上面的设置就会对该进程生效了：

```
echo 226 > /sys/fs/cgroup/cpu/container/tasks 

```

可以看到，计算机的 CPU 使用率立刻降到了 20%（%Cpu0 : 20.3 us）。 

除 CPU 子系统外，Cgroups 的每一项子系统都有其独有的资源限制能力，比如： blkio，为块设备设定I/O 限制，一般用于磁盘等设备； cpuset，为进程分配单独的 CPU 核和对应的内存节点； memory，为进程设定内存使用的限制。 Linux Cgroups 的设计还是比较易用的，简单粗暴地理解呢，它就是一个子系统目录加上一组资源限制文件的组合。而对于 Docker 等 Linux 容器项目来说，它们只需要在每个子系统下面，为每个容器创建一个控制组（即创建一个新目录），然后在启动容器进程之后，把这个进程的 PID 填写到对应控制组的 tasks 文件中就可以了。 而至于在这些控制组下面的资源文件里填上什么值，就靠用户执行 docker run 时的参数指定了，比如这样一条命令：

```
docker run -it --cpu-period=100000 --cpu-quota=20000 ubuntu /bin/bash

```

在启动这个容器后，我们可以通过查看 Cgroups 文件系统下，CPU 子系统中，“docker”这个控制组里的资源限制文件的内容来确认：

```
cat /sys/fs/cgroup/cpu/docker/5d5c9f67d/cpu.cfs_period_us 
100000
$ cat /sys/fs/cgroup/cpu/docker/5d5c9f67d/cpu.cfs_quota_us 
20000

```

这就意味着这个 Docker 容器，只能使用到 20% 的 CPU 带宽。



#### 3. 容器间通信

同一个物理机、跨物理机



#### 4. 容器镜像

Namespace 的作用是“隔离”，它让应用进程只能看到该 Namespace 内的“世界”；而 Cgroups 的作用是“限制”，它给这个“世界”围上了一圈看不见的墙。

这么一折腾，进程就真的被“装”在了一个与世隔绝的房间里，而这些房间就是 PaaS 项目赖以生存的应用“沙盒”。 可是，还有一个问题不知道你有没有仔细思考过：这个房间四周虽然有了墙，但是如果容器进程低头一看地面，又是怎样一副景象呢？ 

换句话说，容器里的进程看到的文件系统又是什么样子的呢？

 可能你立刻就能想到，这一定是一个关于 Mount Namespace 的问题：容器里的应用进程，理应看到一份完全独立的文件系统。这样，它就可以在自己的容器目录（比如 /tmp）下进行操作，而完全不会受宿主机以及其他容器的影响。



即使开启了 Mount Namespace，容器进程看到的文件系统也跟宿主机完全一样。 

这是怎么回事呢？

 仔细思考一下，你会发现这其实并不难理解：Mount Namespace 修改的，是容器进程对文件系统“挂载点”的认知。

但是，这也就意味着，**只有在“挂载”这个操作发生之后，进程的视图才会被改变**。

而**在此之前，新创建的容器会直接继承宿主机的各个挂载点**。

 这时，你可能已经想到了一个解决办法：**创建新进程时，除了声明要启用 Mount Namespace 之外，我们还可以告诉容器进程，有哪些目录需要重新挂载，就比如这个 /tmp 目录。于是，我们在容器进程执行前可以添加一步重新挂载 /tmp 目录的操作：**

```
int container_main(void* arg)
{
  printf("Container - inside the container!\n");
  // 如果你的机器的根目录的挂载类型是 shared，那必须先重新挂载根目录
  // mount("", "/", NULL, MS_PRIVATE, "");
  mount("none", "/tmp", "tmpfs", 0, "");
  execv(container_args[0], container_args);
  printf("Something's wrong!\n");
  return 1;
}

```

可以看到，在修改后的代码里，我在容器进程启动之前，加上了一句 mount(“none”, “/tmp”, “tmpfs”, 0, “”) 语句。就这样，我告诉了容器以 tmpfs（内存盘）格式，重新挂载了 /tmp 目录。

可以看到，这次 /tmp 变成了一个空目录，这意味着重新挂载生效了。

可以看到，**容器里的 /tmp 目录是以 tmpfs 方式单独挂载的**。 

更重要的是，**因为我们创建的新进程启用了 Mount Namespace，所以这次重新挂载的操作，只在容器进程的 Mount Namespace 中有效**。

如果在宿主机上用 mount -l 来检查一下这个挂载，你会发现它是不存在的。

这就是 Mount Namespace 跟其他 Namespace 的使用略有不同的地方：**它对容器进程视图的改变，一定是伴随着挂载操作（mount）才能生效**。 

可是，作为一个普通用户，我们希望的是一个更友好的情况：**每当创建一个新容器时，我希望容器进程看到的文件系统就是一个独立的隔离环境，而不是继承自宿主机的文件系统**。

怎么才能做到这一点呢？ 不难想到，**我们可以在容器进程启动之前重新挂载它的整个根目录“/”**。

而**由于 Mount Namespace 的存在，这个挂载对宿主机不可见**，**所以容器进程就可以在里面随便折腾了**。 

在 Linux 操作系统里，**有一个名为 chroot 的命令可以帮助你在 shell 中方便地完成这个工作**。顾名思义，它的作用就是帮你“change root file system”，**即改变进程的根目录到你指定的位置**。它的用法也非常简单。 

假设，我们现在有一个 $HOME/test 目录，想要把它作为一个 /bin/bash 进程的根目录。 首先，创建一个 test 目录和几个 lib 文件夹：

```
mkdir -p $HOME/test
$ mkdir -p $HOME/test/{bin,lib64,lib}
$ cd $T

```

然后，把 bash 命令拷贝到 test 目录对应的 bin 路径下：

```
cp -v /bin/{bash,ls} $HOME/test/bin
```

接下来，把 bash 命令需要的所有 so 文件，也拷贝到 test 目录对应的 lib 路径下。找到 so 文件可以用 ldd 命令：

```
T=$HOME/test
$ list="$(ldd /bin/ls | egrep -o '/lib.*\.[0-9]')"
$ for i in $list; do cp -v "$i" "${T}${i}"; done

```

最后，执行 chroot 命令，告诉操作系统，我们将使用 $HOME/test 目录作为 /bin/bash 进程的根目录：

```
chroot $HOME/test /bin/bash

```

这时，你如果执行 "ls /"，就会看到，它返回的都是HOME/test 目录下面的内容，而不是宿主机的内容。 

更重要的是，对于被 chroot 的进程来说，它并不会感受到自己的根目录已经被“修改”成 HOME/test 了。 

这种视图被修改的原理，是不是跟我之前介绍的 Linux Namespace 很类似呢？ 

没错！ 实际上，Mount Namespace 正是基于对 chroot 的不断改良才被发明出来的，它也是 Linux 操作系统里的第一个 Namespace。 

当然，**为了能够让容器的这个根目录看起来更“真实”，我们一般会在这个容器的根目录下挂载一个完整操作系统的文件系统，比如 Ubuntu16.04 的 ISO。**这样，在容器启动之后，**我们在容器里通过执行 "ls /" 查看根目录下的内容，就是 Ubuntu 16.04 的所有目录和文件**。 

而这个**挂载在容器根目录上、用来为容器进程提供隔离后执行环境的文件系统，就是所谓的“容器镜像”**。它还有一个更为专业的名字，叫作：**rootfs（根文件系统）**。 所以，一个最常见的 rootfs，或者说容器镜像，会包括如下所示的一些目录和文件，比如 /bin，/etc，/proc 等等：

现在，你应该可以理解，**对 Docker 项目来说，它最核心的原理实际上就是为待创建的用户进程**：

-  启用 Linux Namespace 配置； 
- 设置指定的 Cgroups 参数； 
- 切换进程的根目录（Change Root）。



需要明确的是，rootfs **只是一个操作系统所包含的文件、配置和目录，并不包括操作系统内核**。

在 Linux 操作系统中，这两部分是分开存放的，**操作系统只有在开机启动时才会加载指定版本的内核镜像**。 

所以说，**rootfs 只包括了操作系统的“躯壳”，并没有包括操作系统的“灵魂”。** 

那么，对于容器来说，这个操作系统的“灵魂”又在哪里呢？ 

实际上，**同一台机器上的所有容器，都共享宿主机操作系统的内核**。

 这就意味着**，如果你的应用程序需要配置内核参数、加载额外的内核模块，以及跟内核进行直接的交互，你就需要注意了：这些操作和依赖的对象，都是宿主机操作系统的内核，它对于该机器上的所有容器来说是一个“全局变量”，牵一发而动全身**。



不过，**正是由于 rootfs 的存在，容器才有了一个被反复宣传至今的重要特性：一致性**。 

什么是容器的“一致性”呢？

**由于云端与本地服务器环境不同，应用的打包过程，一直是使用 PaaS 时最“痛苦”的一个步骤**。

 但有了容器之后，更准确地说，有了容器镜像（即 rootfs）之后，这个问题被非常优雅地解决了。 

由于 **rootfs 里打包的不只是应用，而是整个操作系统的文件和目录，也就意味着，应用以及它运行所需要的所有依赖，都被封装在了一起**。

**这种深入到操作系统级别的运行环境一致性，打通了应用在本地开发和远端执行环境之间难以逾越的鸿沟**。

 不过，这时你可能已经发现了另一个非常棘手的问题：难道我每开发一个应用，或者升级一下现有的应用，都要重复制作一次 rootfs 吗？

那么，**既然这些修改都基于一个旧的 rootfs，我们能不能以增量的方式去做这些修改呢**？这样做的好处是，所有人都只需要维护相对于 base rootfs 修改的增量内容，而不是每次修改都制造一个“fork”。

Docker 在镜像的设计中，引入了层（layer）的概念。也就是说，用户制作镜像的每一步操作，都会生成一个层，也就是一个增量 rootfs。

这里用到了联合文件系统（Union File System）， UnionFS 的最主要的功能是将多个不同位置的目录联合挂载（union mount）到同一个目录下。

一个容器的 rootfs 主要由三部分组成，即只读层，Init 层，可读写层：



![](https://static001.geekbang.org/resource/image/8a/5f/8a7b5cfabaab2d877a1d4566961edd5f.png)



**第一部分，只读层**。 它是这个容器的 rootfs 最下面的五层，对应的正是 ubuntu:latest 镜像的五层。可以看到，它们的挂载方式都是只读的（ro+wh，即 readonly+whiteout，至于什么是 whiteout，我下面马上会讲到）。

**第二部分，可读写层**。 它是这个容器的 rootfs 最上面的一层（6e3be5d2ecccae7cc），它的挂载方式为：rw，即 read write。

在没有写入文件之前，这个目录是空的。而一旦在容器里做了写操作，你修改产生的内容就会以增量的方式出现在这个层中。 

可是，你有没有想到这样一个问题：如果我现在要做的，是删除只读层里的一个文件呢？ 为了实现这样的删除操作，**AuFS 会在可读写层创建一个 whiteout 文件，把只读层里的文件“遮挡”起来。 比如，你要删除只读层里一个名叫 foo 的文件，那么这个删除操作实际上是在可读写层创建了一个名叫.wh.foo 的文件。这样，当这两个层被联合挂载之后，foo 文件就会被.wh.foo 文件“遮挡”起来，“消失”了**。

这个功能，就是“ro+wh”的挂载方式，即只读 +whiteout 的含义。

我喜欢把 **whiteout 形象地翻译为：“白障”**。 所以，**最上面这个可读写层的作用，就是专门用来存放你修改 rootfs 后产生的增量，无论是增、删、改，都发生在这里**。

而当我们使用完了这个被修改过的容器之后，还可以使用 docker commit 和 push 指令，保存这个被修改过的可读写层，并上传到 Docker Hub 上，供其他人使用；

而与此同时，原先的只读层里的内容则不会有任何变化。这，就是增量 rootfs 的好处。 

**第三部分，Init 层**。 它是一个以“-init”结尾的层，夹在只读层和读写层之间。**Init 层是 Docker 项目单独生成的一个内部层，专门用来存放 /etc/hosts、/etc/resolv.conf 等信息**。 

需要这样一层的原因是，**这些文件本来属于只读的 Ubuntu 镜像的一部分，但是用户往往需要在启动容器时写入一些指定的值比如 hostname，所以就需要在可读写层对它们进行修改**。 

可是，这些修改往往只对当前的容器有效，我们并不希望执行 docker commit 时，把这些信息连同可读写层一起提交掉。 所以，Docker 做法是，在修改了这些文件之后，以一个单独的层挂载了出来。而用户执行 **docker commit 只会提交可读写层，所以是不包含这些内容的。 最终，这 7 个层都被联合挂载到 /var/lib/docker/aufs/mnt 目录下，表现为一个完整的 Ubuntu 操作系统供容器使用**。



![](https://static001.geekbang.org/resource/image/0d/cb/0da944e5bac4fe1d00d3f01a747e86cb.jpg)

**总的来说，一个“容器”，实际上是一个由 Linux Namespace、Linux Cgroups 和 rootfs 三种技术构建出来的进程的隔离环境。**

一个正在运行的 Linux 容器，其实可以被“一分为二”地看待：

- 一组联合挂载在 /var/lib/docker/aufs/mnt 上的 rootfs，这一部分我们称为“**容器镜像**”（Container Image），是**容器的静态视图**
- 一个由 Namespace + Cgroups 构成的隔离环境，这一部分我们称为“**容器运行时**”（Container Runtime），是**容器的动态视图**





### linux相关的知识

#### 1、Cgroups



#### 2、网桥



## Mesos、kubernetes简介

这个重要假设，正是容器技术圈在 Docker 项目成功后不久，就迅速走向了“容器编排”这个“上层建筑”的主要原因：作为一家云服务商或者基础设施提供商，我只要能够将用户提交的 Docker 镜像以容器的方式运行起来，就能成为这个非常热闹的容器生态图上的一个承载点，从而将整个容器技术栈上的价值，沉淀在我的这个节点上。

**从一个开发者和单一的容器镜像，到无数开发者和庞大的容器集群，容器技术实现了从“容器”到“容器云”的飞跃，标志着它真正得到了市场和生态的认可**。 

这样，**容器就从一个开发者手里的小工具，一跃成为了云计算领域的绝对主角；而能够定义容器组织和管理规范的“容器编排”技术，则当仁不让地坐上了容器技术领域的“头把交椅”。** 

这其中，最具代表性的容器编排工具，当属 Docker 公司的 **Compose+Swarm 组合**，以及 Google 与 RedHat 公司**共同主导的 Kubernetes 项目**。

跟很多基础设施领域先有工程实践、后有方法论的发展路线不同，Kubernetes 项目的理论基础则要比工程实践走得靠前得多，这当然要归功于 Google 公司在 2015 年 4 月发布的 Borg 论文了。 

**Borg 系统，一直以来都被誉为 Google 公司内部最强大的“秘密武器”。**

虽然略显夸张，但这个说法倒不算是吹牛。 因为，相比于 Spanner、BigTable 等相对上层的项目，Borg 要承担的责任，是承载 Google 公司整个基础设施的核心依赖。在 Google 公司已经公开发表的基础设施体系论文中，Borg 项目当仁不让地位居整个基础设施技术栈的最底层。



![](https://static001.geekbang.org/resource/image/c7/bd/c7ed0043465bccff2efc1a1257e970bd.png)

正是由于这样的定位，Borg 可以说是 Google 最不可能开源的一个项目。

而幸运地是，得益于 Docker 项目和容器技术的风靡，它却终于得以以另一种方式与开源社区见面，这个方式就是 Kubernetes 项目。 

所以，相比于“小打小闹”的 Docker 公司、“旧瓶装新酒”的 Mesos 社区，Kubernetes 项目从一开始就比较幸运地站上了一个他人难以企及的高度：在它的成长阶段，这个项目每一个核心特性的提出，几乎都脱胎于 Borg/Omega 系统的设计与经验。

更重要的是，这些特性在开源社区落地的过程中，又在整个社区的合力之下得到了极大的改进，修复了很多当年遗留在 Borg 体系中的缺陷和问题。 

所以**，尽管在发布之初被批评是“曲高和寡”，但是在逐渐觉察到 Docker 技术栈的“稚嫩”和 Mesos 社区的“老迈”之后**，这个社区很快就明白了：**Kubernetes 项目在 Borg 体系的指导下，体现出了一种独有的“先进性”与“完备性”，而这些特质才是一个基础设施领域开源项目赖以生存的核心价值**。 

为了更好地理解这两种特质，我们不妨从 Kubernetes 的顶层设计说起。



![](https://static001.geekbang.org/resource/image/8e/67/8ee9f2fa987eccb490cfaa91c6484f67.png)

Kubernetes 项目的架构，跟它的原型项目 Borg 非常类似，都由 Master 和 Node 两种节点组成，而这两种角色分别对应着控制节点和计算节点。

**控制节点，即 Master 节点**，**由三个紧密协作的独立组件组合而成**，它们分别是**负责 API 服务的 kube-apiserver**、**负责调度的 kube-scheduler**，以及**负责容器编排的 kube-controller-manager**。整个集群的持久化数据，则**由 kube-apiserver 处理后保存在 Etcd** 中。

而**计算节点上最核心的部分**，则是一个叫作 **kubelet 的组件**：
在 Kubernetes 项目中，kubelet **主要负责同容器运行时（比如 Docker 项目）打交道**。而这个交互所依赖的，是一个称作 **CRI（Container Runtime Interface）的远程调用接口**，**这个接口定义了容器运行时的各项核心操作**，**比如：启动一个容器需要的所有参数**。Kubernetes 项目并不关心你部署的是什么容器运行时、使用的什么技术实现，**只要你的这个容器运行时能够运行标准的容器镜像，它就可以通过实现 CRI 接入到 Kubernetes 项目当中**。

kubelet 还通过 gRPC 协议同一个叫作 Device Plugin 的插件进行交互。这个插件，是 Kubernetes 项目用来管理 GPU 等宿主机物理设备的主要组件。这使得kebuernetes 能够提供对机器学习训练、高性能作业支持。

kubelet 的另一个重要功能，则是**调用网络插件和存储插件为容器配置网络和持久化存储**。这两个插件与 kubelet 进行交互的接口，分别是 CNI（Container Networking Interface）和 CSI（Container Storage Interface）。

**运行在大规模集群中的各种任务之间，实际上存在着各种各样的关系。这些关系的处理，才是作业编排和管理系统最困难的地方。**

Kubernetes 项目最主要的设计思想是，**从更宏观的角度，以统一的方式来定义任务之间的各种关系，并且为将来支持更多种类的关系留有余地**。

比如说，Kubernetes 项目对容器间的“访问”进行了分类，首先总结出了一类非常常见的“紧密交互”的关系，即：这些应用之间需要非常频繁的交互和访问；又或者，它们会直接通过本地文件进行信息交换。在常规环境下，这些应用往往会被直接部署在同一台机器上，通过 Localhost 通信，通过本地磁盘目录交换文件。而在 Kubernetes 项目中，这些容器则会被划分为一个“Pod”，Pod 里的容器共享同一个 Network Namespace、同一组数据卷，从而达到高效率交换信息的目的。

而对于另外一种更为常见的需求，比如 Web 应用与数据库之间的访问关系，Kubernetes 项目则提供了一种叫作“Service”的服务。像这样的两个应用，往往故意不部署在同一台机器上，这样即使 Web 应用所在的机器宕机了，数据库也完全不受影响。可是，我们知道，对于一个容器来说，它的 IP 地址等信息不是固定的，那么 Web 应用又怎么找到数据库容器的 Pod 呢？所以，Kubernetes 项目的做法是给 Pod 绑定一个 Service 服务，而 Service 服务声明的 IP 地址等信息是“终生不变”的。这个Service 服务的主要作用，就是作为 Pod 的代理入口（Portal），从而代替 Pod 对外暴露一个固定的网络地址。这样，对于 Web 应用的 Pod 来说，它需要关心的就是数据库 Pod 的 Service 信息。不难想象，Service 后端真正代理的 Pod 的 IP 地址、端口等信息的自动更新、维护，则是 Kubernetes 项目的职责。

**围绕着容器和 Pod 不断向真实的技术场景扩展，我们就能够摸索出一幅如下所示的 Kubernetes 项目核心功能的“全景图”：**

![](http://blog.jrwang.me/img/container-and-kubernetes/15482124182636.png)



按照这幅图的线索，我们从容器这个最基础的概念出发，**首先遇到了容器间“紧密协作”关系的难题，于是就扩展到了 Pod；有了 Pod 之后，我们希望能一次启动多个应用的实例，这样就需要 Deployment 这个 Pod 的多实例管理器；而有了这样一组相同的 Pod 后，我们又需要通过一个固定的 IP 地址和端口以负载均衡的方式访问它，于是就有了 Service。此外，还有用于配制文件读取的 ConfigMap，访问授权信息的 Secret 等。**

**除了应用与应用之间的关系外，应用运行的形态是影响“如何容器化这个应用”的第二个重要因素**。为此，Kubernetes 定义了新的、基于 Pod 改进后的对象。比如 Job，用来描述一次性运行的 Pod（比如，大数据任务）；再比如 DaemonSet，用来描述每个宿主机上必须且只能运行一个副本的守护进程服务；又比如 CronJob，则用于描述定时任务等等。



在 Kubernetes 项目中，所推崇的使用方法是：

- 首先，通过一个“编排对象”，比如 Pod、Job、CronJob 等，来描述你试图管理的应用；
- 然后，再为它定义一些“服务对象”，比如 Service、Secret、Horizontal Pod Autoscaler（自动水平扩展器）等。这些对象，会负责具体的平台级功能。
  这种使用方法，就是所谓的“声明式 API”。这种 API 对应的“编排对象”和“服务对象”，都是 Kubernetes 项目中的 API 对象（API Object）。 Kubernetes 项目所擅长的，是按照用户的意愿和整个系统的规则，完全自动化地处理好容器之间的各种关系。这种功能，就是我们所说的“编排”。







## Kubernetes实践

主要内容：

- 技术的选型（网络方案对比）
- 自定义容器（僵尸进程的避免）
- cmdb-docker的整合（nile原理、使用、资源分配策略、）
- coreDns+依赖图
- 隔离组实现原理
- 监控、报警





## 不足、未来、感谢







## Q&A

自己提出或者收集一些问题

### 1. 为什么选择macvlan

网络对比图，结合公司的实际来讲解





### 2. 高级特性什么时候能用上





### 3. 如何保证集群的稳定性、容灾处理、一键拉起

多集群、数据备份、运维脚本



### 4.什么时候支持redis、mysql存储相关的容器实例

PV相关内容



### 5. 集群内部的负载均衡、流量控制

Ingress、service机制



### 6. kubernetes的优势是什么？

Cloud native

### 7. 日志问题





## 表达方式

- 先整体宏观的介绍一下它是什么、能干什么
- 详细的描述它背后的实现原理
- 基于它我们做了什么、和之前比有什么优势
- 在实践中，我们遇到了什么困难、存在的不足、怎么解决的
- 未来计划