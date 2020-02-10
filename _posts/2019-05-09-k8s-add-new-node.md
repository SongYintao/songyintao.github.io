---
title: kubernetes新增节点
subtitle: kubernetes add new node
layout: post
tags: [kubernetes,kubeadm]
---

随着应用不断增加，公司里面的k8s集群越来越不够用。只能增加新的机器啊。



###  前置条件

- 已有kubernetes集群；

- 物理机要求：

  - 操作系统：ubuntu16.04。
  - docker：最新版本安装，开启metric服务，镜像仓库配置。
  - 网络配置：开启macvlan功能，vlan：28
  - node-exporter service

  

### 实施步骤

#### 1. 裸机操作系统安装

运维完成，除了运维需要的工具。其他的啥都不要装。

#### 2. Docker 安装

安装最新版本的Docker，并以Service的方式运行。配置相关的harbor仓库地址。添加daemon.json打开metric功能。

「具体的安装步骤。。。」

[Docker安装官方文档](https://docs.docker.com/install/linux/docker-ce/ubuntu/#install-docker-ce)

##### 1. Uninstall old versions：卸载老的版本

```shell
$ sudo apt-get remove docker docker-engine docker.io containerd runc
```

It’s OK if `apt-get` reports that none of these packages are installed.

The contents of `/var/lib/docker/`, including images, containers, volumes, and networks, are preserved. The Docker CE package is now called `docker-ce`.

##### 2. Supported storage drivers 支持的存储驱动

Docker CE on Ubuntu supports `overlay2`, `aufs` and `btrfs` storage drivers.

> **Note**: In Docker Engine - Enterprise, `btrfs` is only supported on SLES. See the documentation on [btrfs](https://docs.docker.com/engine/userguide/storagedriver/btrfs-driver/) for more details.

For new installations on version 4 and higher of the Linux kernel, `overlay2` is supported and preferred over `aufs`. Docker CE uses the `overlay2` storage driver by default. If you need to use `aufs` instead, you need to configure it manually. See [aufs](https://docs.docker.com/engine/userguide/storagedriver/aufs-driver/)

##### 3. Install Docker CE

### Install using the repository

Before you install Docker CE for the first time on a new host machine, you need to set up the Docker repository. Afterward, you can install and update Docker from the repository.

#### SET UP THE REPOSITORY

1. Update the `apt` package index:

   ```
   $ sudo apt-get update
   ```

2. Install packages to allow `apt` to use a repository over HTTPS:

   ```
   $ sudo apt-get install \
       apt-transport-https \
       ca-certificates \
       curl \
       gnupg-agent \
       software-properties-common
   ```

3. Add Docker’s official GPG key:

   ```
   $ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
   ```

   Verify that you now have the key with the fingerprint `9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88`, by searching for the last 8 characters of the fingerprint.

   ```
   $ sudo apt-key fingerprint 0EBFCD88
       
   pub   rsa4096 2017-02-22 [SCEA]
         9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
   uid           [ unknown] Docker Release (CE deb) <docker@docker.com>
   sub   rsa4096 2017-02-22 [S]
   ```







#### 3. node-exporter 服务安装



「具体的安装步骤」



#### 4. 网络配置

打开macvlan功能



#### 5. kubelet组件安装



#### 6. CNI安装



#### 7. 加入集群





### 脚本聚合

将上面的步骤以校本的形式构建，部署。便于后续的持续添加







# Marathon迁移到Kubernetes具体方案

由于运维装机比较尴尬，我们无法获取一个纯净的机器。那么，退而求其次，只能在原来的基础上面搞事情。



### 1.手动

当前需要暂停的服务：

- mesos-master
- mesos-slave
- xipam



#### 1.首先，暂停上面的服务。

```shell
root@test-a3-60-14:~# cd /lib/systemd/system
root@test-a3-60-14:/lib/systemd/system# ls
#暂停服务
root@test-a3-60-14:/lib/systemd/system# systemctl stop mesos-master.service
root@test-a3-60-14:/lib/systemd/system# systemctl stop mesos-slave.service
root@test-a3-60-14:/lib/systemd/system# systemctl stop xipam.service
root@test-a3-60-14:/lib/systemd/system# systemctl stop docker
#删除服务
root@test-a3-60-14:/lib/systemd/system# rm mesos-slave.service
root@test-a3-60-14:/lib/systemd/system# rm mesos-master.service
root@test-a3-60-14:/lib/systemd/system# rm xipam.service
#service reload
root@test-a3-60-14:/lib/systemd/system# systemctl daemon-reload

```

#### 2. 修改docker service文件

##### docker.service

```yaml
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.com
After=network.target docker.socket

[Service]
Type=notify
WorkingDirectory=/usr/local/bin
ExecStart=/usr/bin/dockerd \
                --registry-mirror=http://192.168.60.8 \
                --insecure-registry 192.168.60.8 \
                -H tcp://127.0.0.1:4243 \
                -H unix:///var/run/docker.sock \
                --selinux-enabled=false \
                $DOCKER_STORAGE \
                $DOCKER_LOG \
                $DOCKER_OTHERS

ExecReload=/bin/kill -s HUP $MAINPID
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
#TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process
Restart=on-failure

[Install]
WantedBy=multi-user.target
```



##### 新建daemon.json  docker配置文件,打开metric功能

```json
{
"metrics-addr":"192.168.33.17:1334",
"experimental" : true
}
```



#### 3.关闭Swap功能

```shell
root@test-a3-60-14:/etc/docker# free -h
              total        used        free      shared  buff/cache   available
Mem:            62G        2.9G         46G        143M         13G         58G
Swap:          191G        589M        190G
root@test-a3-60-14:/etc/docker#
#暂时关闭交换区功能
root@test-a3-60-14:/etc/docker# swapoff -a
root@test-a3-60-14:/etc/docker# free -h
              total        used        free      shared  buff/cache   available
Mem:            62G        2.8G         45G        611M         14G         58G
Swap:            0B          0B          0B
root@test-a3-60-14:/etc/docker# vim /etc/fstab
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/sda3 during installation
UUID=776b4bf7-49e7-41ed-b09d-4d14813e62a7 /               ext4    errors=remount-ro 0       1
# /boot was on /dev/sda2 during installation
UUID=142ffa79-8e9a-460f-a5a2-b3a551f7c798 /boot           ext4    defaults        0       2
# swap was on /dev/sda4 during installation  注释掉，关闭交换区
#UUID=8b603d4a-01f0-4a81-8318-becfb7f90f9e none            swap    sw              0       0
```



#### 4. 关闭防火墙

```shell
root@test-a3-60-14:~# apt-get install ufw
Reading package lists... Done
Building dependency tree
Reading state information... Done
The following NEW packages will be installed:
  ufw
0 upgraded, 1 newly installed, 0 to remove and 128 not upgraded.
Need to get 149 kB of archives.
After this operation, 838 kB of additional disk space will be used.
Get:1 http://mirrors.163.com/ubuntu xenial/main amd64 ufw all 0.35-0ubuntu2 [149 kB]
Fetched 149 kB in 0s (1,164 kB/s)
Preconfiguring packages ...
Selecting previously unselected package ufw.
(Reading database ... 95948 files and directories currently installed.)
Preparing to unpack .../ufw_0.35-0ubuntu2_all.deb ...
Unpacking ufw (0.35-0ubuntu2) ...
Processing triggers for systemd (229-4ubuntu19) ...
Processing triggers for ureadahead (0.100.0-19) ...
ureadahead will be reprofiled on next reboot
Setting up ufw (0.35-0ubuntu2) ...

Creating config file /etc/ufw/before.rules with new version

Creating config file /etc/ufw/before6.rules with new version

Creating config file /etc/ufw/after.rules with new version

Creating config file /etc/ufw/after6.rules with new version
Processing triggers for systemd (229-4ubuntu19) ...
Processing triggers for ureadahead (0.100.0-19) ...
root@test-a3-60-14:~# ufw disable
ERROR: Invalid syntax

Usage: ufw COMMAND

Commands:
 enable                          enables the firewall
 disable                         disables the firewall
 default ARG                     set default policy
 logging LEVEL                   set logging to LEVEL
 allow ARGS                      add allow rule
 deny ARGS                       add deny rule
 reject ARGS                     add reject rule
 limit ARGS                      add limit rule
 delete RULE|NUM                 delete RULE
 insert NUM RULE                 insert RULE at NUM
 route RULE                      add route RULE
 route delete RULE|NUM           delete route RULE
 route insert NUM RULE           insert route RULE at NUM
 reload                          reload firewall
 reset                           reset firewall
 status                          show firewall status
 status numbered                 show firewall status as numbered list of RULES
 status verbose                  show verbose firewall status
 show ARG                        show firewall report
 version                         display version information

Application profile commands:
 app list                        list application profiles
 app info PROFILE                show information on PROFILE
 app update PROFILE              update PROFILE
 app default ARG                 set default application policy

root@test-a3-60-14:~# ufw disable
Firewall stopped and disabled on system startup
```

#### 5. 打开网桥功能

```shell
root@test-a3-60-14:~# sysctl net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-iptables = 1
root@test-a3-60-14:~# sysctl net.bridge.bridge-nf-call-ip6tables=1
net.bridge.bridge-nf-call-ip6tables = 1
root@test-a1-60-32:~# sysctl vm.swappiness=0
vm.swappiness = 0
root@test-a1-60-32:~# sysctl -p
fs.inotify.max_user_instances = 10240
```

#### 6. 清理iptables

```shell
#!/bin/bash

/sbin/iptables --policy INPUT ACCEPT
/sbin/iptables --policy FORWARD ACCEPT
/sbin/iptables --policy OUTPUT ACCEPT
/sbin/iptables --flush

iptables -X
iptables -F
iptables -Z
iptables -X -t nat
iptables -F -t nat
iptables -Z -t nat
```

至此，reboot一下计算机。

#### 7. 安装kubelet kubeadm kubectl  kubernetes-cni

在harbor机器上面，把相关的镜像和apt安装包复制到待安装的机器上面：

```shell
[root@harbor files]# pwd
/root/songyintao/kube-install/files
[root@harbor files]# scp -r kube/ root@192.168.60.14:~/k8s
pause-amd64-3.1.tar.gz                                                                                                                                                                                                                        100%  308KB 307.5KB/s   00:00
kube-scheduler-amd64-v1.11.0.tar.gz                                                                                                                                                                                                           100%   14MB  13.8MB/s   00:00
pause-3.1.tar.gz                                                                                                                                                                                                                              100%  308KB 307.5KB/s   00:00
kube-controller-manager-amd64-v1.11.0.tar.gz                                                                                                                                                                                                  100%   31MB  30.5MB/s   00:00
kube-apiserver-amd64-v1.11.0.tar.gz                                                                                                                                                                                                           100%   33MB  33.1MB/s   00:01
etcd-amd64-3.2.18.tar.gz                                                                                                                                                                                                                      100%   60MB  59.7MB/s   00:01
coredns-1.1.3.tar.gz                                                                                                                                                                                                                          100%   15MB  14.6MB/s   00:00
kube-proxy-amd64-v1.11.0.tar.gz                                                                                                                                                                                                               100%   30MB  29.6MB/s   00:00
kubelet_1.11.0-00_amd64.deb                                                                                                                                                                                                                   100%   22MB  22.2MB/s   00:00
apt-transport-https_1.6.2_all.deb                                                                                                                                                                                                             100% 1692     1.7KB/s   00:00
cri-tools_1.11.0-00_amd64.deb                                                                                                                                                                                                                 100% 5185KB   5.1MB/s   00:00
socat_1.7.3.1-1_amd64.deb                                                                                                                                                                                                                     100%  314KB 313.5KB/s   00:00
ebtables_2.0.10.4-3.5ubuntu2.18.04.3_amd64.deb                                                                                                                                                                                                100%   78KB  78.0KB/s   00:00
apt-transport-https_1.2.29_amd64.deb                                                                                                                                                                                                          100%   26KB  25.6KB/s   00:00
kubeadm_1.11.0-00_amd64.deb                                                                                                                                                                                                                   100% 9201KB   9.0MB/s   00:00
ethtool_1%3a4.15-0ubuntu1_amd64.deb                                                                                                                                                                                                           100%  112KB 111.7KB/s   00:00
kubectl_1.11.0-00_amd64.deb                                                                                                                                                                                                                   100% 9178KB   9.0MB/s   00:00
kubernetes-cni_0.6.0-00_amd64.deb                                                                                                                                                                                                             100% 5771KB   5.6MB/s   00:00
socat_1.7.3.2-2ubuntu2_amd64.deb                                                                                                                                                                                                              100%  334KB 333.8KB/s   00:00
[root@harbor files]#
[root@harbor files]# pwd
/root/songyintao/kube-install/files
[root@harbor files]#
```





##### 安装相关deb

```shell
root@test-a3-60-14:~/k8s/kube/deb# dpkg -i cri-tools_1.11.0-00_amd64.deb
&dpkg -i ethtool_1%3a4.15-0ubuntu1_amd64.deb
&dpkg -i ebtables_2.0.10.4-3.5ubuntu2.18.04.3_amd64.deb
&dpkg -i socat_1.7.3.1-1_amd64.deb
root@test-a3-60-14:~/k8s/kube/deb# dpkg -i kubectl_1.11.0-00_amd64.deb
root@test-a3-60-14:~/k8s/kube/deb# dpkg -i kubeadm_1.11.0-00_amd64.deb
root@test-a3-60-14:~/k8s/kube/deb# dpkg -i kubelet_1.11.0-00_amd64.deb
root@test-a3-60-14:~/k8s/kube/deb# dpkg -i kubernetes-cni_0.6.0-00_amd64.deb

```

##### 安装必备image

```shell
root@test-a3-60-14:~/k8s/kube/image# docker load -i pause-amd64-3.1.tar.gz
e17133b79956: Loading layer [==================================================>]  744.4kB/744.4kB
Loaded image: k8s.gcr.io/pause-amd64:3.1
root@test-a3-60-14:~/k8s/kube/image# docker load -i pause-3.1.tar.gz
Loaded image: k8s.gcr.io/pause:3.1
root@test-a3-60-14:~/k8s/kube/image# docker load -i kube-proxy-amd64-v1.11.0.tar.gz
582b548209e1: Loading layer [==================================================>]   44.2MB/44.2MB
e20569a478ed: Loading layer [==================================================>]  3.358MB/3.358MB
91ac6787ccdf: Loading layer [==================================================>]  52.06MB/52.06MB
Loaded image: k8s.gcr.io/kube-proxy-amd64:v1.11.0
```



##### enable kubelet服务



```shell

root@test-a3-60-14:/etc/cni/net.d# systemctl daemon-reload
root@test-a3-60-14:/etc/cni/net.d#
root@test-a3-60-14:/etc/cni/net.d# systemctl enable kubelet
root@test-a3-60-14:/etc/cni/net.d# systemctl start kubelet
```



#### 8. 配置macvlan cni

测试环境的cni，我们采用的是macvlan+host

##### 创建cni启动目录

```shell
root@test-a3-60-14:~/k8s/kube/image# mkdir -p /etc/cni/net.d
root@test-a3-60-14:~/k8s/kube/image# cd /etc/cni/net.d/
root@test-a3-60-14:/etc/cni/net.d#
```

##### 创建本地的macvlan

```shell
root@test-a3-60-14:/etc/cni/net.d# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eno3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 18:66:da:6e:5d:3e brd ff:ff:ff:ff:ff:ff
    inet 192.168.60.14/24 brd 192.168.60.255 scope global eno3
       valid_lft forever preferred_lft forever
    inet6 fe80::1a66:daff:fe6e:5d3e/64 scope link
       valid_lft forever preferred_lft forever
3: eno4: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 18:66:da:6e:5d:3f brd ff:ff:ff:ff:ff:ff
4: enp5s0f0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether a0:36:9f:d3:ef:de brd ff:ff:ff:ff:ff:ff
5: enp5s0f1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether a0:36:9f:d3:ef:df brd ff:ff:ff:ff:ff:ff
6: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:2e:53:a0:d1 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:2eff:fe53:a0d1/64 scope link
       valid_lft forever preferred_lft forever
7: eno3.96@eno3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 18:66:da:6e:5d:3e brd ff:ff:ff:ff:ff:ff
    inet6 fe80::1a66:daff:fe6e:5d3e/64 scope link
       valid_lft forever preferred_lft forever
8: eno3.95@eno3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 18:66:da:6e:5d:3e brd ff:ff:ff:ff:ff:ff
    inet6 fe80::1a66:daff:fe6e:5d3e/64 scope link
       valid_lft forever preferred_lft forever
       
##选择eno3,创建vlan 96,启动macvlan eno3.9

root@test-a3-60-14:/etc/cni/net.d# ip link set bond0 promisc on
root@test-a3-60-14:/etc/cni/net.d# ip link add link bond0 name bond0.96 type vlan id 96
root@test-a3-60-14:~/k8s# ip link set bond0.96 up
root@test-a3-60-14:~/k8s#
root@test-a3-60-14:~/k8s# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eno3: <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 18:66:da:6e:5d:3e brd ff:ff:ff:ff:ff:ff
    inet 192.168.60.14/24 brd 192.168.60.255 scope global eno3
       valid_lft forever preferred_lft forever
    inet6 fe80::1a66:daff:fe6e:5d3e/64 scope link
       valid_lft forever preferred_lft forever
3: eno4: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 18:66:da:6e:5d:3f brd ff:ff:ff:ff:ff:ff
4: enp5s0f0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether a0:36:9f:d3:ef:de brd ff:ff:ff:ff:ff:ff
5: enp5s0f1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether a0:36:9f:d3:ef:df brd ff:ff:ff:ff:ff:ff
6: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:2e:53:a0:d1 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:2eff:fe53:a0d1/64 scope link
       valid_lft forever preferred_lft forever
12: eno3.96@eno3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 18:66:da:6e:5d:3e brd ff:ff:ff:ff:ff:ff
    inet6 fe80::1a66:daff:fe6e:5d3e/64 scope link
       valid_lft forever preferred_lft forever
```



##### 创建cni文件：

```shell
 root@test-a3-60-14:/etc/cni/net.d# vim 10-macvlannet.conf
 {
	"cniVersion": "0.3.1",
	"name": "macnet",
	"type": "macvlan",
	"master": "eno1.96",
	"ipam": {

	"type": "host-local",
		"ranges": [
			[
				{
					"subnet": "172.31.1.0/16",
					"rangeStart": "172.31.230.20",
					"rangeEnd": "172.31.230.255",
					"gateway": "172.31.0.1"
				}
			]

		],
		"routes": [
			{ "dst": "0.0.0.0/0" }
		]
	}
}
```

#### 9.加入集群



##### 获取master节点kubeadm的token

```shell
root@test-a1-60-83:~# kubeadm token list
TOKEN                     TTL         EXPIRES   USAGES                   DESCRIPTION   EXTRA GROUPS
c37yve.mi8qkk388dytv6ct   <forever>   <never>   authentication,signing   <none>        system:bootstrappers:kubeadm:default-node-token

mheww8.c3j2kxze2ckyi8aa   <invalid>   2019-05-17T16:41:12+08:00   authentication,signing   <none>    system:bootstrappers:kubeadm:default-node-token

ycc3ql.herhckrvzzyce992   <invalid>   2019-05-17T15:08:39+08:00   authentication,signing   <none>    system:bootstrappers:kubeadm:default-node-token
```

##### 修改本地host

```shell
::1 localhost
# END ANSIBLE MANAGED BLOCK localhost
# BEGIN ANSIBLE MANAGED BLOCK test-a1-60-83
192.168.60.83 test-a1-60-83
# END ANSIBLE MANAGED BLOCK test-a1-60-83
# BEGIN ANSIBLE MANAGED BLOCK test-10-33-11
192.168.33.11 test-10-33-11
# END ANSIBLE MANAGED BLOCK test-10-33-11
# BEGIN ANSIBLE MANAGED BLOCK test-10-33-12
192.168.33.12 test-10-33-12
# END ANSIBLE MANAGED BLOCK test-10-33-12
# BEGIN ANSIBLE MANAGED BLOCK test-10-33-13
192.168.33.13 test-10-33-13
# END ANSIBLE MANAGED BLOCK test-10-33-13
# BEGIN ANSIBLE MANAGED BLOCK test-10-33-14
192.168.33.14 test-10-33-14
# END ANSIBLE MANAGED BLOCK test-10-33-14
# BEGIN ANSIBLE MANAGED BLOCK test-10-33-15
192.168.33.15 test-10-33-15
# END ANSIBLE MANAGED BLOCK test-10-33-15
# BEGIN ANSIBLE MANAGED BLOCK test-10-33-16
192.168.33.16 test-10-33-16
# END ANSIBLE MANAGED BLOCK test-10-33-16
# BEGIN ANSIBLE MANAGED BLOCK test-10-33-17
192.168.33.17 test-10-33-17
# END ANSIBLE MANAGED BLOCK test-10-33-17
# BEGIN ANSIBLE MANAGED BLOCK test-10-33-18
192.168.33.18 test-10-33-18
# END ANSIBLE MANAGED BLOCK test-10-33-18
# BEGIN ANSIBLE MANAGED BLOCK test-10-33-19
192.168.33.19 test-10-33-19
# END ANSIBLE MANAGED BLOCK test-10-33-19
# BEGIN ANSIBLE MANAGED BLOCK test-10-33-20
192.168.33.20 test-10-33-20
# END ANSIBLE MANAGED BLOCK test-10-33-20
# BEGIN ANSIBLE MANAGED BLOCK test-10-33-21
192.168.33.21 test-10-33-21
# END ANSIBLE MANAGED BLOCK test-10-33-21
# BEGIN ANSIBLE MANAGED BLOCK test-10-33-22
192.168.33.22 test-10-33-22
# END ANSIBLE MANAGED BLOCK test-10-33-22
# BEGIN ANSIBLE MANAGED BLOCK test-10-33-23
192.168.33.23 test-10-33-23
# END ANSIBLE MANAGED BLOCK test-10-33-23
# BEGIN ANSIBLE MANAGED BLOCK test-10-33-24
192.168.33.24 test-10-33-24
# END ANSIBLE MANAGED BLOCK test-10-33-24
# BEGIN ANSIBLE MANAGED BLOCK test-10-33-25
192.168.33.25 test-10-33-25
# END ANSIBLE MANAGED BLOCK test-10-33-25
```

ip link del flannel.1

ip link del cni0

##### 待加入节点加入集群

```shell
kubeadm join --node-name 192.168.60.130 --token c37yve.mi8qkk388dytv6ct 192.168.60.83:6443 --discovery-token-unsafe-skip-ca-verification





/sbin/iptables --policy INPUT ACCEPT
/sbin/iptables --policy FORWARD ACCEPT
/sbin/iptables --policy OUTPUT ACCEPT
/sbin/iptables --flush

iptables -X
iptables -F
iptables -Z
iptables -X -t nat
iptables -F -t nat
iptables -Z -t nat
```

##### master 节点查看

```shell
root@test-a1-60-83:~# kubectl get nodes
NAME            STATUS     ROLES     AGE       VERSION
192.168.33.11   NotReady   <none>    148d      v1.11.0
192.168.33.12   Ready      <none>    148d      v1.11.0
192.168.33.13   Ready      <none>    148d      v1.11.0
192.168.33.14   Ready      <none>    148d      v1.11.0
192.168.33.15   Ready      <none>    148d      v1.11.0
192.168.33.16   Ready      <none>    148d      v1.11.0
192.168.33.17   NotReady   <none>    148d      v1.11.0
192.168.33.18   Ready      <none>    148d      v1.11.0
192.168.33.19   Ready      <none>    148d      v1.11.0
192.168.33.20   Ready      <none>    148d      v1.11.0
192.168.33.21   Ready      <none>    148d      v1.11.0
192.168.33.22   NotReady   <none>    36d       v1.11.0
192.168.33.23   NotReady   <none>    148d      v1.11.0
192.168.60.13   Ready      <none>    1h        v1.11.0
192.168.60.14   Ready      <none>    57s       v1.11.0
192.168.60.83   Ready      master    148d      v1.11.0
```



##### 查看相关pod是否正常运行

```shell
root@test-a1-60-83:~# kubectl get pods
NAME                                                              READY     STATUS             RESTARTS   AGE
ad-ad-bidding-stable-54d6c9c974-m75vx                             1/1       Running            0          7d
ad-ad-query-stable-7f5766f4f9-vkf4b                               1/1       Running            0          8d
ad-ad-stock-stable-6b476dff4f-v2n4w                               1/1       Running            0          8d
anchor-achor-skill-package-web-stable-5c4bc7f5f4-wmblx            1/1       Running            0          23d
anchor-anchor-nickname-mainstay-stable-648d5fd767-ssfbt           1/1       Running            0          32d
anchor-anchor-rap-stable-5b84cd7c68-8kd8s                         1/1       Running            0          9d
anchor-anchor-read-backend-stable-549f886958-6vqgc                1/1       Running            0          1h
```

#### 10. node-exporter 安装

以service的方式运行。

```shell
root@test-a3-60-14:/lib/systemd/system# vim node_exporter.service
root@test-a3-60-14:/lib/systemd/system# cat node_exporter.service
[Unit]
Description=Node Exporter
After=network.target

[Service]
Restart=always
StartLimitInterval=0
RestartSec=10
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
root@test-a3-60-14:/lib/systemd/system#
root@test-a3-60-14:/lib/systemd/system# systemctl daemon-reload
root@test-a3-60-14:/lib/systemd/system# systemctl start node_exporter.service
```

### 2. Ansible脚本









### kubernetes 节点

| 物理节点       | 网段         | 是否可用       | 静态ip |
| -------------- | ------------ | -------------- | ------ |
| 192.168.60.13  | 172.31.200.x |                |        |
| 192.168.60.14  | 172.31.201.x |                |        |
| 192.168.60.15  | 172.31.202.x |                |        |
| 192.168.60.16  | 172.31.203.x |                |        |
| 192.168.60.17  | 172.30.200.x |                | Y      |
| 192.168.60.18  | 172.31.204.x | N              |        |
| 192.168.60.19  | 172.31.205.x |                |        |
| 192.168.60.20  | 172.31.206.x | N              |        |
| 192.168.60.21  | 172.31.207.x |                |        |
| 192.168.60.22  | 198          |                |        |
| 192.168.60.27  | 208          |                |        |
| 192.168.60.28  | 209          |                |        |
| 192.168.60.29  | 199          | N              |        |
|                |              |                |        |
| 192.168.60.30  | 172.31.210.x |                |        |
| 192.168.60.31  | 172.31.211.x |                |        |
| 192.168.60.32  | 172.31.212.x |                |        |
| 192.168.60.33  | 172.31.213.x |                |        |
| 192.168.60.34  | 172.31.214.x |                |        |
| 192.168.60.35  | 172.31.215.x |                |        |
| 192.168.60.36  | 172.31.216.x | N  linux有问题 |        |
| 192.168.60.    |              | eno3.96        |        |
| 192.168.60.83  | master       |                |        |
| 192.168.60.86  | 172.31.217   | eno1.96        |        |
| 192.168.60.87  | 172.31.218   |                |        |
| 192.168.60.88  | 172.31.219   |                |        |
| 192.168.60.89  | coredns      |                |        |
| 192.168.60.90  | 172.31.220   |                |        |
| 192.168.60.91  | 172.31.221   |                |        |
| 192.168.60.92  | 172.31.222   |                |        |
| 192.168.60.93  | 172.31.223   |                |        |
| 192.168.60.94  | 172.31.224   |                |        |
| 192.168.60.95  | 172.31.225   | yunxiao        |        |
| 192.168.60.96  | 172.31.226   | yunxiao        |        |
| 192.168.60.97  | 172.31.227   | xdcs           |        |
| 192.168.60.98  | 172.31.228   | xdcs           |        |
| 192.168.60.99  | 172.31.229   | xdcs           |        |
| 100            | 230          |                |        |
| 192.168.33.12  | 172.28.4.x   |                |        |
| 192.168.33.13  | 172.28.2.x   |                |        |
| 192.168.33.14  | 172.28.3     |                |        |
| 192.168.33.15  | 172.28.5     |                |        |
| 192.168.33.16  | 172.28.8     |                |        |
| 192.168.33.17  | 172.28.6     | n              |        |
| 192.168.33.18  | 172.28.10    |                |        |
| 192.168.33.19  | 172.28.7     |                |        |
| 192.168.33.20  | 172.28.9     |                |        |
| 192.168.33.21  | 172.28.11    |                |        |
| 192.168.33.22  | 172.28.22    |                |        |
| 192.168.33.23  | 172.28.23    |                |        |
| 192.168.33.24  | 172.28.24    |                |        |
| 192.168.33.25  | 172.28.25    |                |        |
| 192.168.60.127 | 172.31.231   |                |        |
| 192.168.60.129 | 172.31.233   |                |        |
| 192.168.60.130 | 172.31.234   |                |        |
| 192.168.60.128 | 172.31.232   |                |        |
| 192.168.60.131 | 172.31.235   |                |        |
| 192.168.60.134 | 172.31.238   |                |        |





```
{
	"cniVersion": "0.3.1",
	"name": "macnet",
	"type": "macvlan",
	"master": "enp175s0f0.96",
	"ipam": {

	"type": "host-local",
		"ranges": [
			[
				{
					"subnet": "172.31.1.0/16",
					"rangeStart": "172.31.238.20",
					"rangeEnd": "172.31.238.255",
					"gateway": "172.31.0.1"
				}
			]

		],
		"routes": [
			{ "dst": "0.0.0.0/0" }
		]
	}
}
```



```
ip link set eno1 promisc on
ip link add link eno1 name eno1.96 type vlan id 96
ip link set eno1.96 up
ip addr


ip link set eno3 promisc on
ip link add link eno3 name eno3.96 type vlan id 96
ip link set eno3.96 up
ip addr
```





2] failed to read pod IP from plugin/docker: NetworkPlugin cni failed on the status hook for pod "fm-pns-hdfs-rpc-provider-test-84d465544b-hj8qd_default": CNI failed to retr
Jun 25 12:21:38 test-a3-60-19 kubelet[6583]: W0625 12:21:38.818712





```shell
# step 1: 安装必要的一些系统工具
sudo apt-get update
sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common
# step 2: 安装GPG证书
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# Step 3: 写入软件源信息
sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# Step 4: 更新并安装 Docker-CE
sudo apt-get -y update
sudo apt-get -y install docker-ce

# 安装指定版本的Docker-CE:
# Step 1: 查找Docker-CE的版本:
# apt-cache madison docker-ce
# docker-ce | 17.03.1~ce-0~ubuntu-xenial | http://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
# docker-ce | 17.03.0~ce-0~ubuntu-xenial | http://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
# Step 2: 安装指定版本的Docker-CE: (VERSION 例如上面的 17.03.1~ce-0~ubuntu-xenial)
# sudo apt-get -y install docker-ce=[VERSION]
```

deb [arch=amd64] https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu xenial stable









```
scp macvlan.sh root@192.168.33.23:/root
scp macvlan.sh root@192.168.33.22:/root
scp macvlan.sh root@192.168.33.21:/root
scp macvlan.sh root@192.168.33.20:/root
scp macvlan.sh root@192.168.33.19:/root
scp macvlan.sh root@192.168.33.18:/root
scp macvlan.sh root@192.168.33.17:/root
scp macvlan.sh root@192.168.33.16:/root
scp macvlan.sh root@192.168.33.15:/root
scp macvlan.sh root@192.168.33.14:/root
scp macvlan.sh root@192.168.33.13:/root
scp macvlan.sh root@192.168.33.12:/root
scp macvlan.sh root@192.168.33.11:/root


scp macvlan.sh root@192.168.60.13:/root
scp macvlan.sh root@192.168.60.14:/root
scp macvlan.sh root@192.168.60.15:/root
scp macvlan.sh root@192.168.60.16:/root
scp macvlan.sh root@192.168.60.17:/root
scp macvlan.sh root@192.168.60.19:/root
scp macvlan.sh root@192.168.60.20:/root
scp macvlan.sh root@192.168.60.21:/root
scp macvlan.sh root@192.168.60.22:/root
scp macvlan.sh root@192.168.60.27:/root
scp macvlan.sh root@192.168.60.28:/root
scp macvlan.sh root@192.168.60.29:/root
scp macvlan.sh root@192.168.60.30:/root
scp macvlan.sh root@192.168.60.31:/root
scp macvlan.sh root@192.168.60.32:/root
scp macvlan.sh root@192.168.60.33:/root
scp macvlan.sh root@192.168.60.34:/root
scp macvlan.sh root@192.168.60.35:/root
scp macvlan.sh root@192.168.60.36:/root

scp macvlan.sh root@192.168.60.83:/root
scp macvlan.sh root@192.168.60.86:/root
scp macvlan.sh root@192.168.60.87:/root
scp macvlan.sh root@192.168.60.88:/root
scp macvlan.sh root@192.168.60.89:/root
scp macvlan.sh root@192.168.60.90:/root
scp macvlan.sh root@192.168.60.91:/root
scp macvlan.sh root@192.168.60.92:/root
scp macvlan.sh root@192.168.60.93:/root
scp macvlan.sh root@192.168.60.94:/root
scp macvlan.sh root@192.168.60.95:/root
scp macvlan.sh root@192.168.60.96:/root
scp macvlan.sh root@192.168.60.97:/root
scp macvlan.sh root@192.168.60.98:/root
scp macvlan.sh root@192.168.60.99:/root

```







# #



```yaml
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: flannel
rules:
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes/status
    verbs:
      - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-system
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "type": "flannel",
      "delegate": {
        "isDefaultGateway": true
      }
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: kube-flannel-ds
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      hostNetwork: true
      nodeSelector:
        beta.kubernetes.io/arch: amd64
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: quay.io/coreos/flannel:v0.9.1-amd64
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-nflannel.conf
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.9.1-amd64
        command: [ "/opt/bin/flanneld", "--ip-masq", "--kube-subnet-mgr" ]
        securityContext:
          privileged: true
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg
```







### genie：pod 多接口解决方案

[Git地址](https://github.com/cni-genie/CNI-Genie/blob/master/docs/GettingStarted.md)

```

$ kubectl apply -f https://raw.githubusercontent.com/cni-genie/CNI-Genie/master/conf/1.8/genie-plugin.yaml



```





```
ip route del 172.28.0.0/24 via 172.28.0.0 dev flannel.1 onlink
ip route del 172.28.1.0/24 via 172.28.1.0 dev flannel.1 onlink
ip route del 172.28.2.0/24 via 172.28.2.0 dev flannel.1 onlink
ip route del 172.28.3.0/24 via 172.28.3.0 dev flannel.1 onlink
ip route del 172.28.4.0/24 via 172.28.4.0 dev flannel.1 onlink
ip route del 172.28.5.0/24 via 172.28.5.0 dev flannel.1 onlink
ip route del 172.28.6.0/24 via 172.28.6.0 dev flannel.1 onlink
ip route del 172.28.7.0/24 via 172.28.7.0 dev flannel.1 onlink
ip route del 172.28.8.0/24 via 172.28.8.0 dev flannel.1 onlink
ip route del 172.28.9.0/24 via 172.28.9.0 dev flannel.1 onlink
ip route del 172.28.10.0/24 via 172.28.10.0 dev flannel.1 onlink
ip route del 172.28.11.0/24 via 172.28.11.0 dev flannel.1 onlink
ip route del 172.28.12.0/24 via 172.28.12.0 dev flannel.1 onlink
ip route del 172.28.13.0/24 via 172.28.13.0 dev flannel.1 onlink
ip route del 172.28.14.0/24 via 172.28.14.0 dev flannel.1 onlink
ip route del 172.28.18.0/24 via 172.28.18.0 dev flannel.1 onlink
ip route del 172.28.21.0/24 via 172.28.21.0 dev flannel.1 onlink
ip route del 172.28.22.0/24 via 172.28.22.0 dev flannel.1 onlink
ip route del 172.28.23.0/24 via 172.28.23.0 dev flannel.1 onlink
ip route del 172.28.25.0/24 via 172.28.25.0 dev flannel.1 onlink
ip route del 172.28.26.0/24 via 172.28.26.0 dev flannel.1 onlink
ip route del 172.28.27.0/24 via 172.28.27.0 dev flannel.1 onlink
ip route del 172.28.28.0/24 via 172.28.28.0 dev flannel.1 onlink
ip route del 172.28.31.0/24 via 172.28.31.0 dev flannel.1 onlink
ip route del 172.28.32.0/24 via 172.28.32.0 dev flannel.1 onlink
ip route del 172.28.33.0/24 via 172.28.33.0 dev flannel.1 onlink
ip route del 172.28.34.0/24 via 172.28.34.0 dev flannel.1 onlink
ip route del 172.28.35.0/24 via 172.28.35.0 dev flannel.1 onlink
ip route del 172.28.36.0/24 via 172.28.36.0 dev flannel.1 onlink
ip route del 172.28.37.0/24 via 172.28.37.0 dev flannel.1 onlink
ip route del 172.28.38.0/24 via 172.28.38.0 dev flannel.1 onlink
ip route del 172.28.39.0/24 via 172.28.39.0 dev flannel.1 onlink
ip route del 172.28.40.0/24 via 172.28.40.0 dev flannel.1 onlink
ip route del 172.28.41.0/24 via 172.28.41.0 dev flannel.1 onlink
ip route del 172.28.42.0/24 via 172.28.42.0 dev flannel.1 onlink
ip route del 172.28.43.0/24 via 172.28.43.0 dev flannel.1 onlink
ip route del 172.28.44.0/24 via 172.28.44.0 dev flannel.1 onlink
ip route del 172.28.45.0/24 via 172.28.45.0 dev flannel.1 onlink
ip route del 172.28.46.0/24 via 172.28.46.0 dev flannel.1 onlink
ip route del 172.28.47.0/24 via 172.28.47.0 dev flannel.1 onlink
ip route del 172.28.50.0/24 via 172.28.50.0 dev flannel.1 onlink
ip route
exit



```





```
kubectl taint nodes 192.168.60.17 ipam=static-ipam:NoSchedule
```





## CentOs



修改yum.repo

```shell
mkdir /opt/centos-yum.bak
mv /etc/yum.repos.d/* /opt/centos-yum.bak/


cat /etc/redhat-release
cd /etc/yum.repos.d/

vim CentOS-Base.repo

# CentOS-Base.repo
#
# The mirror system uses the connecting IP address of the client and the
# update status of each mirror to pick mirrors that are updated to and
# geographically close to the client.  You should use this for CentOS updates
# unless you are manually picking other mirrors.
#
# If the mirrorlist= does not work for you, as a fall back you can try the
# remarked out baseurl= line instead.
#
#

[base]
name=CentOS-$releasever - Base - mirrors.aliyun.com
failovermethod=priority
baseurl=http://mirrors.aliyun.com/centos/7/os/$basearch/
        http://mirrors.aliyuncs.com/centos/7/os/$basearch/
        http://mirrors.cloud.aliyuncs.com/centos/7/os/$basearch/
gpgcheck=1
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7

#released updates
[updates]
name=CentOS-$releasever - Updates - mirrors.aliyun.com
failovermethod=priority
baseurl=http://mirrors.aliyun.com/centos/7/updates/$basearch/
        http://mirrors.aliyuncs.com/centos/7/updates/$basearch/
        http://mirrors.cloud.aliyuncs.com/centos/7/updates/$basearch/
gpgcheck=1
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7

#additional packages that may be useful
[extras]
name=CentOS-$releasever - Extras - mirrors.aliyun.com
failovermethod=priority
baseurl=http://mirrors.aliyun.com/centos/7/extras/$basearch/
        http://mirrors.aliyuncs.com/centos/7/extras/$basearch/
        http://mirrors.cloud.aliyuncs.com/centos/7/extras/$basearch/
gpgcheck=1
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7

#additional packages that extend functionality of existing packages
[centosplus]
name=CentOS-$releasever - Plus - mirrors.aliyun.com
failovermethod=priority
baseurl=http://mirrors.aliyun.com/centos/7/centosplus/$basearch/
        http://mirrors.aliyuncs.com/centos/7/centosplus/$basearch/
        http://mirrors.cloud.aliyuncs.com/centos/7/centosplus/$basearch/
gpgcheck=1
enabled=0
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7

#contrib - packages by Centos Users
[contrib]
name=CentOS-$releasever - Contrib - mirrors.aliyun.com
failovermethod=priority
baseurl=http://mirrors.aliyun.com/centos/7/contrib/$basearch/
        http://mirrors.aliyuncs.com/centos/7/contrib/$basearch/
        http://mirrors.cloud.aliyuncs.com/centos/7/contrib/$basearch/
gpgcheck=1
enabled=0
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7



vim docker-ce.repo

[docker-ce-stable]
name=Docker CE Stable - $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/$basearch/stable
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-stable-debuginfo]
name=Docker CE Stable - Debuginfo $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/debug-$basearch/stable
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-stable-source]
name=Docker CE Stable - Sources
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/source/stable
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-edge]
name=Docker CE Edge - $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/$basearch/edge
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-edge-debuginfo]
name=Docker CE Edge - Debuginfo $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/debug-$basearch/edge
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-edge-source]
name=Docker CE Edge - Sources
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/source/edge
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-test]
name=Docker CE Test - $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/$basearch/test
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-test-debuginfo]
name=Docker CE Test - Debuginfo $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/debug-$basearch/test
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-test-source]
name=Docker CE Test - Sources
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/source/test
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-nightly]
name=Docker CE Nightly - $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/$basearch/nightly
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-nightly-debuginfo]
name=Docker CE Nightly - Debuginfo $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/debug-$basearch/nightly
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-nightly-source]
name=Docker CE Nightly - Sources
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/source/nightly
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg



wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
yum clean all
yum makecache
yum list
yum update
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum update
yum install docker-ce-17.12.1.ce-1.el7.centos -y

cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
  yum update -y
    yum install -y  kubelet-1.11.0
    yum install -y  kubectl-1.11.0
    yum install -y  kubeadm-1.11.0
    kubeadm reset
  systemctl stop firewalld
    systemctl disable firewalld
    setenforce 0
    vi /etc/selinux/config
    vi /etc/sysctl.d/k8s.conf
    modprobe br_netfilter
    sysctl -p /etc/sysctl.d/k8s.conf
   swapoff -a
   vim /etc/fstab
   vi /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness=0

   
sysctl -p /etc/sysctl.d/k8s.conf
systemctl enable docker.service
systemctl enable kubelet.service
systemctl daemon-reload
systemctl start docker
   
   
    /sbin/iptables --policy INPUT ACCEPT
    /sbin/iptables --policy FORWARD ACCEPT
    /sbin/iptables --policy OUTPUT ACCEPT
    /sbin/iptables --flush
    iptables -X
    iptables -F
    iptables -Z
    iptables -X -t nat
    iptables -F -t nat
    iptables -Z -t nat
    
    
    
    ip a
    ip link set enp175s0f0 promisc on
    ip a
    ip link add link enp175s0f0 name enp175s0f0.96 type vlan id 96
    ip link set enp175s0f0.96 up
```



```
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.com
After=network.target docker.socket

[Service]
Type=notify
WorkingDirectory=/usr/local/bin
ExecStart=/usr/bin/dockerd \
                --registry-mirror=http://192.168.60.8 \
                --insecure-registry 192.168.60.8 \
                -H tcp://127.0.0.1:4243 \
                -H unix:///var/run/docker.sock \
                --selinux-enabled=false \
                $DOCKER_STORAGE \
                $DOCKER_LOG \
                $DOCKER_OTHERS

ExecReload=/bin/kill -s HUP $MAINPID
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
#TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process
Restart=on-failure

[Install]
WantedBy=multi-user.target
```



```
kubeadm join --node-name 192.168.60.134 --token c37yve.mi8qkk388dytv6ct 192.168.60.83:6443 --discovery-token-unsafe-skip-ca-verification
```

