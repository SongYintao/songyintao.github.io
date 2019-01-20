---
title: Linux 命令笔记
subtitle: 每天一点点，linux命令使用
tag: [linux,shell]
layout: post
---

​	每天一点点，总结学习linux命令。

### 1. sysctl

​	`sysctl`命令被用于**在内核运行时动态地修改内核的运行参数**，可用的内核参数在目录`/proc/sys`中。

​	它包含一些TCP/ip堆栈和虚拟内存系统的高级选项，这可以让有经验的管理员提高引人注目的系统性能。用`sysctl`可以读取设置超过五百个系统变量。

```shell
[root@docker1 sys]# ls
abi  crypto  debug  dev  fs  kernel  net  user  vm
[root@docker1 sys]# cd net
[root@docker1 net]# ls
bridge  core  ipv4  ipv6  netfilter  nf_conntrack_max  unix
[root@docker1 net]# cd bridge/
[root@docker1 bridge]# ls
bridge-nf-call-arptables  bridge-nf-call-iptables        bridge-nf-filter-vlan-tagged
bridge-nf-call-ip6tables  bridge-nf-filter-pppoe-tagged  bridge-nf-pass-vlan-input-dev
[root@docker1 bridge]#
```

#### 使用说明：

```shell
[root@docker1 bridge]# sysctl

Usage:
 sysctl [options] [variable[=value] ...]

Options:
  -a, --all            display all variables
  -A                   alias of -a
  -X                   alias of -a
      --deprecated     include deprecated parameters to listing
  -b, --binary         print value without new line
  -e, --ignore         ignore unknown variables errors
  -N, --names          print variable names without values
  -n, --values         print only values of a variables
  -p, --load[=<file>]  read values from file
  -f                   alias of -p
      --system         read values from all system directories
  -r, --pattern <expression>
                       select setting that match expression
  -q, --quiet          do not echo variable set
  -w, --write          enable writing a value to variable
  -o                   does nothing
  -x                   does nothing
  -d                   alias of -h

 -h, --help     display this help and exit
 -V, --version  output version information and exit

For more details see sysctl(8).
```

**实例:**

查看所有可读变量：

`sysctl -a` 读一个指定的变量，

例如kern.maxproc：

`sysctl kern.maxproc kern.maxproc`: 1044 要设置一个指定的变量，

直接用`variable=value`这样的语法：

`sysctl kern.maxfiles=5000 kern.maxfiles: 2088 -> 5000`



### 2. ssh-gen

`ssh`无密码登录，自然要用到`Linux`的基础命令`ssh`及`scp`.

##### 本机自身实现无密码登录：

**生成公钥、私钥对**

```shell
ssh-keygen
```

进入到生成密钥文件夹中，默认在用户的家目录下面，一个隐藏的.ssh文件夹中。

```shell
 cd /home/hzq/.ssh/
```

查看是否有“`authorized_keys`”文件，如果有，直接将公钥追加到“**authorized_keys**”文件中，如果没有，创建“`authorized_keys`”文件，并修改权限为“600”

```shell
 touch authorized_keys
 chmod 600 authorized_keys 
```

追加公钥到“`authorized_keys`”文件中

```shell
 cat id_rsa.pub >> authorized_keys 
```

### 3.查看当前linux内核版本

```shell
chen@mylinuxserver:~> cat /proc/version
Linux version 2.6.5-7.244-smp (geeko@buildhost) (gcc version 3.3.3 (SuSE Linux)) #1 SMP Mon Dec 12 18:32:25 UTC 2005
```



### 4. scp拷贝文件及文件夹

```shell
[root@harbor ~]# scp
usage: scp [-12346BCpqrv] [-c cipher] [-F ssh_config] [-i identity_file]
           [-l limit] [-o ssh_option] [-P port] [-S program]
           [[user@]host1:]file1 ... [[user@]host2:]file2
```

例如拷贝单个文件命令：

`scp file username@ip:filepath`

说明：`file`是要拷贝的文件名   

`username`:远程登录的用户名，

`ip`：远程服务器ip

`filepath`：远程文件路径

拷贝文件夹命令如下：

```shell
scp -r file username@ip:filepath
```

### 5. 端口号查看

查看端口使用情况，使用`netstat`命令。
查看已经连接的服务端口（ESTABLISHED

```shell
netstat -a
```


查看所有的服务端口（LISTEN，ESTABLISHED）

```shell
netstat -ap
```


查看8080端口，则可以结合`grep`命令：

```shell
netstat -ap | grep 8080
```

 


如查看8888端口，则在终端中输入：

```shell
lsof -i:8888
```

若要停止使用这个端口的程序，使用kill +对应的pid即可

### 6. Route &route trace

​	    Linux系统的`route`命令用于显示和操作IP路由表（`show / manipulate the IP routing table`）。要实现两个不同的子网之间的通信，需要一台连接两个网络的路由器，或者同时位于两个网络的网关来实现。在Linux系统中，设置路由通常是为了解决以下问题：该Linux系统在一个局域网中，局域网中有一个网关，能够让机器访问Internet，那么就需要将这台机器的IP地址设置为Linux机器的默认路由。要注意的是，直接在命令行下执行`route`命令来添加路由，不会永久保存，当网卡重启或者机器重启之后，该路由就失效了；可以在`/etc/rc.local`中添加`route`命令来保证该路由设置永久有效。

1．**命令格式**：

```shell
route [-f][-p] [Command [Destination] [mask Netmask] [Gateway] [metric Metric]][if Interface]] 
```

2．**命令功能**：

​	Route命令是用于操作基于内核ip路由表，它的主要作用是创建一个静态路由让指定一个主机或者一个网络通过一个网络接口，如eth0。当使用"`add`"或者"`del`"参数时，路由表被修改，如果没有参数，则显示路由表当前的内容。

3．**命令参数**：

-c 显示更多信息

-n 不解析名字

-v 显示详细的处理信息

-F 显示发送信息

-C 显示路由缓存

-f 清除所有网关入口的路由表。 

-p 与 add 命令一起使用时使路由具有永久性。

 

add:添加一条新路由。

del:删除一条路由。

-net:目标地址是一个网络。

-host:目标地址是一个主机。

netmask:当添加一个网络路由时，需要使用网络掩码。

gw:路由数据包通过网关。注意，你指定的网关必须能够达到。

metric：设置路由跳数。

 

**Command** 指定您想运行的命令 (Add/Change/Delete/Print)。 

**Destination** 指定该路由的网络目标。 

**mask Netmask** 指定与网络目标相关的网络掩码（也被称作子网掩码）。 

**Gateway** 指定<u>网络目标</u>定义的<u>地址集和子网掩码可以到达的前进或下一跃</u>点 IP 地址。 

**metric Metric** 为路由指定一个整数成本值标（从 1 至 9999），当在路由表(与转发的数据包目标地址最匹配)的多个路由中进行选择时可以使用。 

**if Interface** 为可以访问目标的接口指定接口索引。若要获得一个接口列表和它们相应的接口索引，使用 route print 命令的显示功能。可以使用十进制或十六进制值进行接口索引。

 

4．使用实例：

**实例1：显示当前路由**

**命令**：

```shell
route

route -n
```

**输出**：

```shell
[root@localhost ~]# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
192.168.120.0   *               255.255.255.0   U     0      0        0 eth0
192.168.0.0     192.168.120.1   255.255.0.0     UG    0      0        0 eth0
10.0.0.0        192.168.120.1   255.0.0.0       UG    0      0        0 eth0
default         192.168.120.240 0.0.0.0         UG    0      0        0 eth0
[root@localhost ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
192.168.120.0   0.0.0.0         255.255.255.0   U     0      0        0 eth0
192.168.0.0     192.168.120.1   255.255.0.0     UG    0      0        0 eth0
10.0.0.0        192.168.120.1   255.0.0.0       UG    0      0        0 eth0
0.0.0.0         192.168.120.240 0.0.0.0         UG    0      0        0 eth0
```

**说明**：

第一行表示主机所在网络的地址为192.168.120.0，若数据传送目标是在本局域网内通信，则可直接通过eth0转发数据包;

第四行表示数据传送目的是访问Internet，则由接口eth0，将数据包发送到网关192.168.120.240

其中Flags为路由标志，标记当前网络节点的状态。

Flags标志说明：

**U** Up表示此路由当前为<u>启动状态</u>

**H** Host，表示<u>此网关为一主机</u>

**G** Gateway，表示<u>此网关为一路由器</u>

**R** Reinstate Route，<u>使用动态路由重新初始化的路</u>由

**D** Dynamically,此<u>路由是动态性地写入</u>

**M** Modified，此<u>路由是由路由守护程序或导向器动态修改</u>

**!** 表示此路由当前为<u>关闭状态</u>

 

备注：

***route -n (-n 表示不解析名字,列出速度会比route 快)***

 

**实例2：添加网关/设置网关**

命令：

```shell
route add -net 224.0.0.0 netmask 240.0.0.0 dev eth0
```

输出：

```shell
[root@localhost ~]# route add -net 224.0.0.0 netmask 240.0.0.0 dev eth0

[root@localhost ~]# route

Kernel IP routing table

Destination     Gateway         Genmask         Flags Metric Ref    Use Iface

192.168.120.0   *               255.255.255.0   U     0      0        0 eth0

192.168.0.0     192.168.120.1   255.255.0.0     UG    0      0        0 eth0

10.0.0.0        192.168.120.1   255.0.0.0       UG    0      0        0 eth0

224.0.0.0       *               240.0.0.0       U     0      0        0 eth0

default         192.168.120.240 0.0.0.0         UG    0      0        0 eth0

[root@localhost ~]#  
```



说明：

增加一条 到达244.0.0.0的路由

 

**实例3：屏蔽一条路由**

命令：

```shell
route add -net 224.0.0.0 netmask 240.0.0.0 reject
```

输出：

```shell
[root@localhost ~]# route add -net 224.0.0.0 netmask 240.0.0.0 reject
[root@localhost ~]# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
192.168.120.0   *               255.255.255.0   U     0      0        0 eth0
192.168.0.0     192.168.120.1   255.255.0.0     UG    0      0        0 eth0
10.0.0.0        192.168.120.1   255.0.0.0       UG    0      0        0 eth0
224.0.0.0       -               240.0.0.0       !     0      -        0 -
224.0.0.0       *               240.0.0.0       U     0      0        0 eth0
default         192.168.120.240 0.0.0.0         UG    0      0        0 eth0
```

说明：

增加一条屏蔽的路由，目的地址为 224.x.x.x 将被拒绝

 

**实例4：删除路由记录**

命令：

```shell
route del -net 224.0.0.0 netmask 240.0.0.0

route del -net 224.0.0.0 netmask 240.0.0.0 reject
```

输出：

```shell
[root@localhost ~]# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
192.168.120.0   *               255.255.255.0   U     0      0        0 eth0
192.168.0.0     192.168.120.1   255.255.0.0     UG    0      0        0 eth0
10.0.0.0        192.168.120.1   255.0.0.0       UG    0      0        0 eth0
224.0.0.0       -               240.0.0.0       !     0      -        0 -
224.0.0.0       *               240.0.0.0       U     0      0        0 eth0
default         192.168.120.240 0.0.0.0         UG    0      0        0 eth0
[root@localhost ~]# route del -net 224.0.0.0 netmask 240.0.0.0
[root@localhost ~]# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
192.168.120.0   *               255.255.255.0   U     0      0        0 eth0
192.168.0.0     192.168.120.1   255.255.0.0     UG    0      0        0 eth0
10.0.0.0        192.168.120.1   255.0.0.0       UG    0      0        0 eth0
224.0.0.0       -               240.0.0.0       !     0      -        0 -
default         192.168.120.240 0.0.0.0         UG    0      0        0 eth0
[root@localhost ~]# route del -net 224.0.0.0 netmask 240.0.0.0 reject
[root@localhost ~]# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
192.168.120.0   *               255.255.255.0   U     0      0        0 eth0
192.168.0.0     192.168.120.1   255.255.0.0     UG    0      0        0 eth0
10.0.0.0        192.168.120.1   255.0.0.0       UG    0      0        0 eth0
default         192.168.120.240 0.0.0.0         UG    0      0        0 eth0
[root@localhost ~]# 
```

说明：

 

**实例5：删除和添加设置默认网关**

命令：

```shell
route del default gw 192.168.120.240

route add default gw 192.168.120.240
```

输出：

```shell
[root@localhost ~]# route del default gw 192.168.120.240
[root@localhost ~]# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
192.168.120.0   *               255.255.255.0   U     0      0        0 eth0
192.168.0.0     192.168.120.1   255.255.0.0     UG    0      0        0 eth0
10.0.0.0        192.168.120.1   255.0.0.0       UG    0      0        0 eth0
[root@localhost ~]# route add default gw 192.168.120.240
[root@localhost ~]# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
192.168.120.0   *               255.255.255.0   U     0      0        0 eth0
192.168.0.0     192.168.120.1   255.255.0.0     UG    0      0        0 eth0
10.0.0.0        192.168.120.1   255.0.0.0       UG    0      0        0 eth0
default         192.168.120.240 0.0.0.0         UG    0      0        0 eth0
[root@localhost ~]# 
```



**traceroute命令**

​	通过`traceroute`我们可以知道信息从你的计算机到互联网另一端的主机是走的什么路径。当然每次数据包由某一同样的出发点（source）到达某一同样的目的地(destination)走的路径可能会不一样，但基本上来说大部分时候所走的路由是相同的。linux系统中，我们称之为traceroute,在MS Windows中为tracert。 traceroute通过发送小的数据包到目的设备直到其返回，来测量其需要多长时间。一条路径上的每个设备traceroute要测3次。输出结果中包括每次测试的时间(ms)和设备的名称（如有的话）及其IP地址。

在大多数情况下，我们会在**linux**主机系统下，直接执行命令行：

`traceroute hostname`

而在**Windows**系统下是执行tracert的命令：

`tracert hostname`

 

1.**命令格式**：

```traceroute[参数][主机]```

2.**命令功能**：

​	traceroute指令***让你追踪网络数据包的路由途径***，预设数据包大小是40Bytes，用户可另行设置。

具体参数格式：`traceroute [-dFlnrvx][-f<存活数值>][-g<网关>...][-i<网络界面>][-m<存活数值>][-p<通信端口>][-s<来源地址>][-t<服务类型>][-w<超时秒数>][主机名称或IP地址][数据包大小]`

3.**命令参数**：

-d 使用Socket层级的排错功能。

-f 设置第一个检测数据包的存活数值TTL的大小。

-F 设置勿离断位。

-g 设置来源路由网关，最多可设置8个。

-i 使用指定的网络界面送出数据包。

-I 使用ICMP回应取代UDP资料信息。

-m 设置检测数据包的最大存活数值TTL的大小。

-n 直接使用IP地址而非主机名称。

-p 设置UDP传输协议的通信端口。

-r 忽略普通的Routing Table，直接将数据包送到远端主机上。

-s 设置本地主机送出数据包的IP地址。

-t 设置检测数据包的TOS数值。

-v 详细显示指令的执行过程。

-w 设置等待远端主机回报的时间。

-x 开启或关闭数据包的正确性检验。

 

4.**使用实例**：

实例1：traceroute 用法简单、最常用的用法

命令：

```traceroute www.baidu.com ```

输出：

```shell
[root@localhost ~]# traceroute www.baidu.com
traceroute to www.baidu.com (61.135.169.125), 30 hops max, 40 byte packets
 1  192.168.74.2 (192.168.74.2)  2.606 ms  2.771 ms  2.950 ms
 2  211.151.56.57 (211.151.56.57)  0.596 ms  0.598 ms  0.591 ms
 3  211.151.227.206 (211.151.227.206)  0.546 ms  0.544 ms  0.538 ms
 4  210.77.139.145 (210.77.139.145)  0.710 ms  0.748 ms  0.801 ms
 5  202.106.42.101 (202.106.42.101)  6.759 ms  6.945 ms  7.107 ms
 6  61.148.154.97 (61.148.154.97)  718.908 ms * bt-228-025.bta.net.cn (202.106.228.25)  5.177 ms
 7  124.65.58.213 (124.65.58.213)  4.343 ms  4.336 ms  4.367 ms
 8  202.106.35.190 (202.106.35.190)  1.795 ms 61.148.156.138 (61.148.156.138)  1.899 ms  1.951 ms
 9  * * *
30  * * *
[root@localhost ~]# 
```

说明：

​	记录按序列号从1开始，***每个纪录就是一跳 ，每跳表示一个网关***，我们看到每行有三个时间，单位是 ms，其实就是-q的默认参数。***探测数据包向每个网关发送三个数据包后，网关响应后返回的时间***；如果您用 `traceroute -q 4 www.58.com` ，表示向每个网关发送4个数据包。

​	***有时我们traceroute 一台主机时，会看到有一些行是以星号表示的。出现这样的情况，可能是防火墙封掉了ICMP的返回信息，所以我们得不到什么相关的数据包返回数据。***

​	有时我们在某一网关处延时比较长，有可能是某台网关比较阻塞，也可能是物理设备本身的原因。当然如果某台DNS出现问题时，不能解析主机名、域名时，也会 有延时长的现象；您可以加-n 参数来避免DNS解析，以IP格式输出数据。

​	如果在局域网中的不同网段之间，我们可以通过traceroute 来排查问题所在，是主机的问题还是网关的问题。如果我们通过远程来访问某台服务器遇到问题时，我们用到traceroute 追踪数据包所经过的网关，提交IDC服务商，也有助于解决问题；但目前看来在国内解决这样的问题是比较困难的，就是我们发现问题所在，IDC服务商也不可能帮助我们解决。

 

实例2：跳数设置

命令：

```traceroute -m 10 www.baidu.com```

输出：

```shell
[root@localhost ~]# traceroute -m 10 www.baidu.com
traceroute to www.baidu.com (61.135.169.105), 10 hops max, 40 byte packets
 1  192.168.74.2 (192.168.74.2)  1.534 ms  1.775 ms  1.961 ms
 2  211.151.56.1 (211.151.56.1)  0.508 ms  0.514 ms  0.507 ms
 3  211.151.227.206 (211.151.227.206)  0.571 ms  0.558 ms  0.550 ms
 4  210.77.139.145 (210.77.139.145)  0.708 ms  0.729 ms  0.785 ms
 5  202.106.42.101 (202.106.42.101)  7.978 ms  8.155 ms  8.311 ms
 6  bt-228-037.bta.net.cn (202.106.228.37)  772.460 ms bt-228-025.bta.net.cn (202.106.228.25)  2.152 ms 61.148.154.97 (61.148.154.97)  772.107 ms
 7  124.65.58.221 (124.65.58.221)  4.875 ms 61.148.146.29 (61.148.146.29)  2.124 ms 124.65.58.221 (124.65.58.221)  4.854 ms
 8  123.126.6.198 (123.126.6.198)  2.944 ms 61.148.156.6 (61.148.156.6)  3.505 ms 123.126.6.198 (123.126.6.198)  2.885 ms
 9  * * *
10  * * *
[root@localhost ~]#
```

说明：

实例3：**显示IP地址，不查主机名**

命令：

```traceroute -n www.baidu.com```

输出：

```shell
[root@localhost ~]# traceroute -n www.baidu.com
traceroute to www.baidu.com (61.135.169.125), 30 hops max, 40 byte packets
 1  211.151.74.2  5.430 ms  5.636 ms  5.802 ms
 2  211.151.56.57  0.627 ms  0.625 ms  0.617 ms
 3  211.151.227.206  0.575 ms  0.584 ms  0.576 ms
 4  210.77.139.145  0.703 ms  0.754 ms  0.806 ms
 5  202.106.42.101  23.683 ms  23.869 ms  23.998 ms
 6  202.106.228.37  247.101 ms * *
 7  61.148.146.29  5.256 ms 124.65.58.213  4.386 ms  4.373 ms
 8  202.106.35.190  1.610 ms 61.148.156.138  1.786 ms 61.148.3.34  2.089 ms
 9  * * *
30  * * *
[root@localhost ~]# traceroute www.baidu.com
traceroute to www.baidu.com (61.135.169.125), 30 hops max, 40 byte packets
 1  211.151.74.2 (211.151.74.2)  4.671 ms  4.865 ms  5.055 ms
 2  211.151.56.57 (211.151.56.57)  0.619 ms  0.618 ms  0.612 ms
 3  211.151.227.206 (211.151.227.206)  0.620 ms  0.642 ms  0.636 ms
 4  210.77.139.145 (210.77.139.145)  0.720 ms  0.772 ms  0.816 ms
 5  202.106.42.101 (202.106.42.101)  7.667 ms  7.910 ms  8.012 ms
 6  bt-228-025.bta.net.cn (202.106.228.25)  2.965 ms  2.440 ms 61.148.154.97 (61.148.154.97)  431.337 ms
 7  124.65.58.213 (124.65.58.213)  5.134 ms  5.124 ms  5.044 ms
 8  202.106.35.190 (202.106.35.190)  1.917 ms  2.052 ms  2.059 ms
 9  * * *
30  * * *
[root@localhost ~]# 
```

说明：

 

 

实例4：探测包使用的基本UDP端口设置6888

命令：

```traceroute -p 6888 www.baidu.com```

输出：

```shell
[root@localhost ~]# traceroute -p 6888 www.baidu.com
traceroute to www.baidu.com (220.181.111.147), 30 hops max, 40 byte packets
 1  211.151.74.2 (211.151.74.2)  4.927 ms  5.121 ms  5.298 ms
 2  211.151.56.1 (211.151.56.1)  0.500 ms  0.499 ms  0.509 ms
 3  211.151.224.90 (211.151.224.90)  0.637 ms  0.631 ms  0.641 ms
 4  * * *
 5  220.181.70.98 (220.181.70.98)  5.050 ms  5.313 ms  5.596 ms
 6  220.181.17.94 (220.181.17.94)  1.665 ms !X * *
[root@localhost ~]# 
```

说明：

 

实例5：把探测包的个数设置为值4

命令：

```traceroute -q 4 www.baidu.com```

输出：

```shell
[root@localhost ~]# traceroute -q 4 www.baidu.com
traceroute to www.baidu.com (61.135.169.125), 30 hops max, 40 byte packets
 1  211.151.74.2 (211.151.74.2)  40.633 ms  40.819 ms  41.004 ms  41.188 ms
 2  211.151.56.57 (211.151.56.57)  0.637 ms  0.633 ms  0.627 ms  0.619 ms
 3  211.151.227.206 (211.151.227.206)  0.505 ms  0.580 ms  0.571 ms  0.569 ms
 4  210.77.139.145 (210.77.139.145)  0.753 ms  0.800 ms  0.853 ms  0.904 ms
 5  202.106.42.101 (202.106.42.101)  7.449 ms  7.543 ms  7.738 ms  7.893 ms
 6  61.148.154.97 (61.148.154.97)  316.817 ms bt-228-025.bta.net.cn (202.106.228.25)  3.695 ms  3.672 ms *
 7  124.65.58.213 (124.65.58.213)  3.056 ms  2.993 ms  2.960 ms 61.148.146.29 (61.148.146.29)  2.837 ms
 8  61.148.3.34 (61.148.3.34)  2.179 ms  2.295 ms  2.442 ms 202.106.35.190 (202.106.35.190)  7.136 ms
 9  * * * *
30  * * * *
[root@localhost ~]# 
```

说明：

 

实例6：绕过正常的路由表，直接发送到网络相连的主机

命令：

``` traceroute -r www.baidu.com```

输出：

```shell
[root@localhost ~]# traceroute -r www.baidu.com
traceroute to www.baidu.com (61.135.169.125), 30 hops max, 40 byte packets
connect: 网络不可达
[root@localhost ~]#  
```

说明：

 

实例7：把对外发探测包的等待响应时间设置为3秒

命令：

```traceroute -w 3 www.baidu.com```

输出：

```shell
[root@localhost ~]# traceroute -w 3 www.baidu.com
traceroute to www.baidu.com (61.135.169.105), 30 hops max, 40 byte packets
 1  211.151.74.2 (211.151.74.2)  2.306 ms  2.469 ms  2.650 ms
 2  211.151.56.1 (211.151.56.1)  0.621 ms  0.613 ms  0.603 ms
 3  211.151.227.206 (211.151.227.206)  0.557 ms  0.560 ms  0.552 ms
 4  210.77.139.145 (210.77.139.145)  0.708 ms  0.761 ms  0.817 ms
 5  202.106.42.101 (202.106.42.101)  7.520 ms  7.774 ms  7.902 ms
 6  bt-228-025.bta.net.cn (202.106.228.25)  2.890 ms  2.369 ms 61.148.154.97 (61.148.154.97)  471.961 ms
 7  124.65.58.221 (124.65.58.221)  4.490 ms  4.483 ms  4.472 ms
 8  123.126.6.198 (123.126.6.198)  2.948 ms 61.148.156.6 (61.148.156.6)  7.688 ms  7.756 ms
 9  * * *
30  * * *
[root@localhost ~]# 
```

说明：



**Traceroute的工作原理：**

Traceroute最简单的基本用法是：```traceroute hostname```

Traceroute程序的设计是利用ICMP及IP header的TTL（Time To Live）栏位（field）。首先，traceroute送出一个TTL是1的IP datagram（其实，每次送出的为3个40字节的包，包括源地址，目的地址和包发出的时间标签）到目的地，当路径上的第一个路由器（router）收到这个datagram时，它将TTL减1。此时，TTL变为0了，所以该路由器会将此datagram丢掉，并送回一个「ICMP time exceeded」消息（包括发IP包的源地址，IP包的所有内容及路由器的IP地址），traceroute 收到这个消息后，便知道这个路由器存在于这个路径上，接着traceroute 再送出另一个TTL是2 的datagram，发现第2 个路由器...... traceroute 每次将送出的datagram的TTL 加1来发现另一个路由器，这个重复的动作一直持续到某个datagram 抵达目的地。当datagram到达目的地后，该主机并不会送回ICMP time exceeded消息，因为它已是目的地了，那么traceroute如何得知目的地到达了呢？

Traceroute在送出UDP datagrams到目的地时，它所选择送达的port number 是一个一般应用程序都不会用的号码（30000 以上），所以当此UDP datagram 到达目的地后该主机会送回一个「ICMP port unreachable」的消息，而当traceroute 收到这个消息时，便知道目的地已经到达了。所以traceroute 在Server端也是没有所谓的Daemon 程式。

Traceroute提取发 ICMP TTL到期消息设备的IP地址并作域名解析。每次 ，Traceroute都打印出一系列数据,包括所经过的路由设备的域名及 IP地址,三个包每次来回所花时间。

 

 我常用的route和traceroute命令；

`route`是查看路由表的，`traceroute`是跟踪路由路径的。

一般情况是不加参数，直接使用`route hostName/ traceroute hostName`



### 7. 环境变量

1. 显示环境变量HOME

```sh
　$ echo $HOME

　/home/redbooks
```

　

  　　2. 设置一个新的环境变量hello

```shell
$ export HELLO="Hello!"
$ echo $HELLO
Hello!
```



3. 使用env命令显示所有的环境变量

```shell
$ env

HOSTNAME=redbooks.safe.org

PVM_RSH=/usr/bin/rsh

Shell=/bin/bash

TERM=xterm

HISTSIZE=1000

　　...
```

　　

4. 使用set命令显示所有本地定义的Shell变量

```shell
$ set

　　BASH=/bin/bash

　　BASH_VERSINFO=([0]="2"[1]="05b"[2]="0"[3]="1"[4]="release"[5]="i386-redhat-[linux](http://www.chinabyte.com/keyword/Linux/)-gnu")

　　BASH_VERSION='2.05b.0(1)-release'

　　COLORS=/etc/DIR_COLORS.xterm

　　COLUMNS=80

　　DIRSTACK=()

　　DISPLAY=:0.0

　　...
```



5. 使用`unset`命令来清除环境变量

　　`set`可以设置某个环境变量的值。清除环境变量的值用`unset`命令。如果未指定值，则该变量值将被设为NULL。示例如下：

```shell
   $ export TEST="Test..." #增加一个环境变量TEST

　　$ env|grep TEST #此命令有输入，证明环境变量TEST已经存在了

　　TEST=Test...

　　$ unset $TEST #删除环境变量TEST

　　$ env|grep TEST #此命令没有输出，证明环境变量TEST已经存在了
```

　　

6. 使用readonly命令设置只读变量

　　如果使用了readonly命令的话，变量就不可以被修改或清除了。示例如下：

```shell
   $ export TEST="Test..." #增加一个环境变量TEST

　　$ readonly TEST #将环境变量TEST设为只读

　　$ unset TEST #会发现此变量不能被删除

　　-bash: unset: TEST: cannot unset: readonly variable

　　$ TEST="New" #会发现此也变量不能被修改

　　-bash: TEST: readonly variable
```

　　环境变量的设置位于/etc/profile文件

　　如果需要增加新的环境变量可以添加下属行

　　`export path=$path:/path1:/path2:/pahtN`



1.**Linux的变量种类**

按变量的生存周期来划分，Linux变量可分为两类：

　　1.1 **永久的**：<u>需要修改配置文件，变量永久生效</u>。

　　1.2 **临时的**：使用`export`命令声明即可，变量在`关闭shell时`失效。

2.**设置变量的三种方法**

　　2.1 **在/etc/profile文件中添加变量**【**对所有用户生效(永久的**)】

　　用VI在文件`/etc/profile`文件中增加变量，该变量将会对Linux下所有用户有效，并且是“永久的”。

　　例如：编辑`/etc/profile`文件，添加CLASSPATH变量

　　`# vi /etc/profile`

　　export CLASSPATH=./JAVA_HOME/lib;$JAVA_HOME/jre/lib

　　注：修改文件后要想马上生效还要运行`# source /etc/profile`不然只能在下次重进此用户时生效。

　　2.2 **在用户目录下的`.bash_profile`文件中增加变量【对单一用户生效(永久的)】**

　　用VI在用户目录下的`.bash_profile`文件中增加变量，改变量仅会对当前用户有效，并且是“永久的”。

　　例如：编辑guok用户目录(/home/guok)下的`.bash_profile`

　　`$ vi /home/guok/.bash.profile`

　　添加如下内容：

　　`export CLASSPATH=./JAVA_HOME/lib;$JAVA_HOME/jre/lib`

　　注：修改文件后要想马上生效还要运行`$ source /home/guok/.bash_profile`不然只能在下次重进此用户时生效。

　　2.3 **直接运行export命令定义变量**【*只对当前shell(BASH)有效(临时的)*】

　　在shell的命令行下直接使用[export 变量名=变量值] 定义变量，该变量只在当前的shell(BASH)或其子shell(BASH)下是有效的，shell关闭了，变量也就失效了，再打开新shell时就没有这个变量，需要使用的话还需要重新定义。

**3.环境变量的查看**

　　3.1 使用`echo`命令查看单个环境变量。例如：

　　`echo $PATH`

　　3.2 使用`env`查看所有环境变量。例如：

　　`env`

　　3.3 使用`set`查看所有本地定义的环境变量。

　　`unset`可以删除指定的环境变量。

　　4.常用的环境变量

　　`PATH` 决定了`shell`将到哪些目录中寻找命令或程序

　　**HOME** 当前用户主目录

　　**HISTSIZE**　历史记录数

　　**LOGNAME** 当前用户的登录名

　　**HOSTNAME**　指主机的名称

　　**SHELL** 　　当前用户Shell类型

　　**LANGUGE** 　语言相关的环境变量，多语言可以修改此环境变量

　　**MAIL**　　　当前用户的邮件存放目录

　　**PS1**　　　基本提示符，对于root用户是#，对于普通用户是$



### 8. sed  文本处理

sed编辑器被称为流编辑器。和vim交互式编辑器不同，流处理编辑器会在编辑器处理数据之前基于预先提供的一组规则来编辑数据流。

`sed`命令的格式如下：

```shell
sed options script file
用法: sed [选项]... {脚本(如果没有其他脚本)} [输入文件]...

  -n, --quiet, --silent
                 取消自动打印模式空间
  -e 脚本, --expression=脚本
                 添加“脚本”到程序的运行列表
  -f 脚本文件, --file=脚本文件
                 添加“脚本文件”到程序的运行列表
  --follow-symlinks
                 follow symlinks when processing in place; hard links
                 will still be broken.
  -i[SUFFIX], --in-place[=SUFFIX]
                 edit files in place (makes backup if extension supplied).
                 The default operation mode is to break symbolic and hard links.
                 This can be changed with --follow-symlinks and --copy.
  -c, --copy
                 use copy instead of rename when shuffling files in -i mode.
                 While this will avoid breaking links (symbolic or hard), the
                 resulting editing operation is not atomic.  This is rarely
                 the desired mode; --follow-symlinks is usually enough, and
                 it is both faster and more secure.
  -l N, --line-length=N
                 指定“l”命令的换行期望长度
  --posix
                 关闭所有 GNU 扩展
  -r, --regexp-extended
                 在脚本中使用扩展正则表达式
  -s, --separate
                 将输入文件视为各个独立的文件而不是一个长的连续输入
  -u, --unbuffered
                 从输入文件读取最少的数据，更频繁的刷新输出
      --help     打印帮助并退出
      --version  输出版本信息并退出

如果没有 -e, --expression, -f 或 --file 选项，那么第一个非选项参数被视为
sed脚本。其他非选项参数被视为输入文件，如果没有输入文件，那么程序将从标准
输入读取数据。
GNU sed home page: <http://www.gnu.org/software/sed/>.
General help using GNU software: <http://www.gnu.org/gethelp/>.
```

```shell
function：
a ：新增， a 的后面可以接字串，而这些字串会在新的一行出现(目前的下一行)～
c ：取代， c 的后面可以接字串，这些字串可以取代 n1,n2 之间的行！
d ：删除，因为是删除啊，所以 d 后面通常不接任何咚咚；
i ：插入， i 的后面可以接字串，而这些字串会在新的一行出现(目前的上一行)；
p ：列印，亦即将某个选择的数据印出。通常 p 会与参数 sed -n 一起运行～
s ：取代，可以直接进行取代的工作哩！通常这个 s 的动作可以搭配正规表示法！例如 1,20s/old/new/g 就是啦！
```

sed命令是一个很强大的文本编辑器，可以对来自文件、以及标准输入的文本进行编辑。

执行时，sed会从文件或者标准输入中读取一行，将其复制到缓冲区，对文本编辑完成之后，读取下一行直到所有的文本行都编辑完毕。

所以sed命令处理时只会改变缓冲区中文本的副本，如果想要直接编辑原文件，可以使用-i选项或者将结果重定向到新的文件中。



sed命令的基本语法如下：

```shell
sed [options] commands [inputfile...]
```

options表示sed命令的一些选项，常见的选项如下表：

| 选项名 | 作用                                                         |
| ------ | ------------------------------------------------------------ |
| -n     | 取消默认输出                                                 |
| -e     | 多点编辑，可以执行多个子命令                                 |
| -f     | 从脚本文件中读取命令（sed操作可以事先写入脚本，然后通过-f读取并执行） |
| -i     | 直接编辑原文件                                               |
| -l     | 指定行的长度                                                 |
| -r     | 在脚本中使用扩展表达式                                       |

## 2. 应用场景

sed命令比较适用于大的文本文件，用普通文本编辑器难以胜任的情况。下面分别介绍直接打印、插入、删除、替换等编辑操作。
​    实验用文件内容

```shell
#===================test1.txt======================
letitia
mail
uuencode
1003605091
01566
```

（1）行打印，输出缓冲区内容，使用sed的`p子命令`

```shell
sed '1,3 p' test1.txt
echo "====================="
sed -n '1,3 p' test1.txt

#输出结果
letitia
letitia
mail
mail
uuencode
uuencode
1003605091
01566
=====================
letitia
mail
uuencode
```

p子命令代表print，可以打印出sed缓冲区内的内容。
 sed命令中，直接采用数字代表某个特定的文本行：`'1 p'`代表打印第一行；`'1,3 p'`代表打印1到3行；特别的，最后一行的行号为$。

观察输出结果，不使用-n选项时，sed命令把1到3行输出了两次。这是因为不使用-n时，sed首先读取一行，并默认将缓冲区内的文本输出出来，之后p子命令再次输出。使用-n时，默认输出取消，只有p子命令的输出结果。

```shell
sed -n '/^ma/,5 p' test1.txt

#输出结果
mail
uuencode
1003605091
01566
```

sed命令支持正则表达式定位。语法为`/re/`，re表示正则表达式。
 本例表示打印出从匹配正则表达式的地方到第5行，也就是从匹配以ma开头的文本行处开始。

```shell
sed -n '1~2 p' test1.txt

#输出结果
letitia
uuencode
01566
```

`1~2`表示从第一行开始，行号递增2输出，即输出奇数行。语法格式为`first~step`。

（2）插入文本行，追加文本行
 这两种情况很类似。插入文本使用`i子命令`，表示在指定位置前面插入文本；追加文本使用`a子命令`，表示在指定位置之后插入文本。观察一下两个的区别：

```shell
sed -n -e '2 i insert' -e '1,4 p' test1.txt 

#-e选项表示多个子命令，本例执行i子命令之后执行了p子命令
#输出结果
letitia
insert
mail
uuencode
1003605091
sed -n -e '2 a insert' -e '1,4 p' test1.txt

#输出结果
letitia
mail
insert
uuencode
1003605091
```

（3）删除文本行，使用`d子命令`

```shell
sed -n -e '2 d' -e '1,$ p' test1.txt

#输出结果
letitia
uuencode
1003605091
01566
```

（4）替换文本行，使用`c子命令`

```shell
sed -n -e '2 c newmail' -e '1,$ p' test1.txt

#输出结果
letitia
newmail
uuencode
1003605091
01566
```

【注】以上均未使用-i选项，所以更改的只是副本。

<b>（5）替换指定文本，使用`s子命令`</b>
 这一个命令实用性很广，并且灵活。语法也比之上面特别一些：

```shell
sed '位置参数 s/pattern/replaced/[flag]'
```

pattern为要替换的文本，支持正则表达式，replaced表示用来替换的一般字符串（不支持正则表达式）。

flag是替换标志，用来影响匹配替换的规则：

| flag    | 用法                                                   |
| ------- | ------------------------------------------------------ |
| g       | 全局匹配，会替换文本行中所有匹配的字符串               |
| 十进制n | 替换文本行中第n个匹配的字符串                          |
| p       | 替换第一个匹配的字符串，并且将缓冲区输出到标准输出     |
| w       | 替换第一个匹配的字符串，并且将改动的行输出到磁盘文件中 |
| 缺省    | 替换第一个匹配的字符串                                 |

```shell
sed -n -e 's/[0-9]\{10\}/miss letitia/g' -e '1,$ p' test1.txt
#{}要转义，因为此处使用的不是扩展正则表达式

#输出结果
letitia
mail
uuencode
miss letitia
01566
sed -n -e '1,/^ma/ s/l/L/g' -e '1,$ p' test1.txt

#输出结果
Letitia
maiL
uuencode
miss letitia
01566
#可以看到，本例将前两行里的l替换为L。
sed -n '1,3{
            s/l/L/g
            s/e/E/g
            2 i tyrone
            p
            }' test1.txt

#输出结果
LEtitia
tyrone
maiL
uuEncodE
```

最后这个例子比较复杂。使用大括号，表示对1到3行做了一组操作。

## 3. 其他的小事

- 以上都是采用了文件输入做实验，也可以采用其他方式，例如

```shell
sed -i "s/letitia/hello world/g" `grep "letitia" -rl test1.txt`
#将grep的结果作为输入，注意要用反引号括起来，将括号内部分解释为linux命令
```

- 当用户的编辑操作比较复杂时，建议使用sed脚本文件。
- 同正则表达式一样，匹配元字符时要用转义。使用基本正则表达式时，{}等也要转义。

