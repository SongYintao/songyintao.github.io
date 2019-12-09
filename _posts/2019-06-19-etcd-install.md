---
title: etcd cluster install
layout: post
subtitle: etcd install
tags: [etcd]
---



etcd install

[安装包下载](https://github.com/etcd-io/etcd/releases/)



### 1.  解压安装

```shell
root@test-a4-60-89:~/songyintao# tar -zxvf etcd-v3.3.13-linux-amd64.tar.gz
root@test-a4-60-89:~/songyintao# mv etcd-v3.3.13-linux-amd64 /usr/local/bin/etcd
root@test-a4-60-89:~/songyintao# cd /usr/local/bin/etcd/
root@test-a4-60-89:/usr/local/bin/etcd# ls
Documentation  etcd  etcdctl  README-etcdctl.md  README.md  READMEv2-etcdctl.md
```



### 2.创建配置文件

```shell
root@test-a4-60-89:/etc/etcd# mkdir -p /etc/etcd
root@test-a4-60-89:/etc/etcd# mkdir -p /var/lib/etcd/default.etcd
root@test-a4-60-89:/etc/etcd# vim etcd.conf
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"  #etcd数据保存目录
ETCD_LISTEN_CLIENT_URLS="http://10.25.72.164:2379,http://localhost:2379"  #供外部客户端使用的url
ETCD_ADVERTISE_CLIENT_URLS="http://10.25.72.164:2379,http://localhost:2379" #广播给外部客户端使用的url
ETCD_NAME="etcd1"   #etcd实例名称

ETCD_LISTEN_PEER_URLS="http://10.25.72.164:2380"  #集群内部通信使用的URL
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.25.72.164:2380"  #广播给集群内其他成员访问的URL
ETCD_INITIAL_CLUSTER="etcd1=http://10.25.72.164:2380,etcd2=http://10.25.72.233:2380,etcd3=http://10.25.73.196:2380"    #初始集群成员列表
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster" #集群的名称
ETCD_INITIAL_CLUSTER_STATE="new"  #初始集群状态，new为新建集群


```

etcd2和etcd3为加入etcd-cluster集群的实例，需要将其ETCD_INITIAL_CLUSTER_STATE设置为"exist"

```yaml
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"  #etcd数据保存目录
ETCD_LISTEN_CLIENT_URLS="http://10.25.72.164:2379,http://localhost:2379"  #供外部客户端使用的url
ETCD_ADVERTISE_CLIENT_URLS="http://10.25.72.164:2379,http://localhost:2379" #广播给外部客户端使用的url
ETCD_NAME="etcd1"   #etcd实例名称

ETCD_LISTEN_PEER_URLS="http://10.25.72.164:2380"  #集群内部通信使用的URL
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.25.72.164:2380"  #广播给集群内其他成员访问的URL
ETCD_INITIAL_CLUSTER="etcd1=http://10.25.72.164:2380,etcd2=http://10.25.72.233:2380,etcd3=http://10.25.73.196:2380"    #初始集群成员列表
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster" #集群的名称
ETCD_INITIAL_CLUSTER_STATE="new"  #初始集群状态，new为新建集群
```



### 3. service 启动

```shell
root@test-a4-60-89:/usr/local/bin/etcd# cat /lib/systemd/system/etcd.service
[Unit]

Description=Etcd Server

Documentation=https://github.com/coreos/etcd

After=network.target

[Service]

User=root

Type=simple

EnvironmentFile=/etc/etcd/etcd.conf

ExecStart=/usr/local/bin/etcd/etcd

Restart=on-failure

RestartSec=10s

LimitNOFILE=40000

[Install]

WantedBy=multi-user.target
```



### 4. 查看集群状态

```shell
root@test-a4-60-89:/usr/local/bin/etcd# etcdctl member list
1056fd81d3c9576a: name=etcd1 peerURLs=http://192.168.60.89:2380 clientURLs=http://192.168.60.89:2379,http://localhost:2379 isLeader=true
```





