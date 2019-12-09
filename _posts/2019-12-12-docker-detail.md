# docker，containerd，runc，docker-shim



[https://www.jianshu.com/p/52c0f12b0294](https://www.jianshu.com/p/52c0f12b0294)

从[Docker 1.11](https://github.com/docker/docker/releases/tag/v1.11.1)开始，Docker容器运行已经不是简单的通过Docker daemon来启动，而是集成了containerd、runC等多个组件。Docker服务启动之后，我们也可以看见系统上启动了dockerd、docker-containerd等进程，本文主要介绍新版Docker（1.11以后）每个部分的功能和作用。

## Docker Daemon

作为Docker容器管理的守护进程，Docker Daemon从最初集成在`docker`命令中（1.11版本前），到后来的独立成单独二进制程序（1.11版本开始），其功能正在逐渐拆分细化，被分配到各个单独的模块中去。从Docker服务的启动脚本，也能看见守护进程的逐渐剥离：

在Docker 1.8之前，Docker守护进程启动的命令为：

```shell
docker -d
```

这个阶段，守护进程看上去只是Docker client的一个选项。

Docker 1.8开始，启动命令变成了：

```shell
docker daemon
```

这个阶段，守护进程看上去是`docker`命令的一个模块。

Docker 1.11开始，守护进程启动命令变成了：

```shell
dockerd
```

此时已经和Docker client分离，独立成一个二进制程序了。

当然，守护进程模块不停的在重构，其基本功能和定位没有变化。和一般的CS架构系统一样，守护进程负责和Docker client交互，并管理Docker镜像、容器。

下面就来介绍下独立分拆出来的其他几个模块。

## Containerd

[containerd](https://github.com/docker/containerd)是容器技术标准化之后的产物，为了能够兼容[OCI标准](https://www.opencontainers.org/)，将容器运行时及其管理功能从Docker Daemon剥离。理论上，即使不运行dockerd，也能够直接通过containerd来管理容器。（当然，containerd本身也只是一个守护进程，容器的实际运行时由后面介绍的runC控制。）



![img](https://upload-images.jianshu.io/upload_images/8911567-132e0c0087881517.png)

image



最近，Docker刚刚宣布[开源containerd](http://www.infoq.com/cn/news/2017/01/Docker-Containerd-OCI-1)。从其项目[介绍页面](https://github.com/docker/containerd/blob/master/design/architecture.md)可以看出，containerd主要职责是镜像管理（镜像、元信息等）、容器执行（调用最终运行时组件执行）。

containerd向上为Docker Daemon提供了gRPC接口，使得Docker Daemon屏蔽下面的结构变化，确保原有接口向下兼容。向下通过containerd-shim结合runC，使得引擎可以独立升级，避免之前Docker Daemon升级会导致所有容器不可用的问题。

Docker、containerd和containerd-shim之间的关系，可以通过启动一个Docker容器，观察进程之间的关联。首先启动一个容器，

```undefined
docker run -d busybox sleep 1000
```

然后通过`pstree`命令查看进程之间的父子关系（其中20708是`dockerd`的PID）：

```undefined
pstree -l -a -A 20708
```

输出结果如下：

```csharp
dockerd -H fd:// --storage-driver=overlay2
  |-docker-containe -l unix:///var/run/docker/libcontainerd/docker-containerd.sock --metrics-interval=0 --start-timeout 2m --state-dir /var/run/docker/libcontainerd/containerd --shim docker-containerd-shim --runtime docker-runc
  |   |-docker-containe b9a04a582b66206492d29444b5b7bc6ec9cf1eb83eff580fe43a039ad556e223 /var/run/docker/libcontainerd/b9a04a582b66206492d29444b5b7bc6ec9cf1eb83eff580fe43a039ad556e223 docker-runc
  |   |   |-sleep 1000
```

虽然`pstree`命令截断了命令，但我们还是能够看出，当Docker daemon启动之后，dockerd和docker-containerd进程一直存在。当启动容器之后，docker-containerd进程（也是这里介绍的containerd组件）会创建docker-containerd-shim进程，其中的参数b9a04a582b66206492d29444b5b7bc6ec9cf1eb83eff580fe43a039ad556e223就是要启动容器的id。最后docker-containerd-shim子进程，已经是实际在容器中运行的进程（既sleep 1000）。

docker-containerd-shim另一个参数，是一个和容器相关的目录/var/run/docker/libcontainerd/b9a04a582b66206492d29444b5b7bc6ec9cf1eb83eff580fe43a039ad556e223，里面的内容有：

```cpp
.
├── config.json
├── init-stderr
├── init-stdin
└── init-stdout
```

其中包括了容器配置和标准输入、标准输出、标准错误三个管道文件。

## docker-shim

docker-shim是一个真实运行的容器的真实垫片载体，每启动一个容器都会起一个新的docker-shim的一个进程，
他直接通过指定的三个参数：容器id，boundle目录（containerd的对应某个容器生成的目录，一般位于：/var/run/docker/libcontainerd/containerID），
运行是二进制（默认为runc）来调用runc的api创建一个容器（比如创建容器：最后拼装的命令如下：runc create 。。。。。）

## RunC

OCI定义了容器运行时标准，runC是Docker按照开放容器格式标准（OCF, Open Container Format）制定的一种具体实现。

runC是从Docker的libcontainer中迁移而来的，实现了容器启停、资源隔离等功能。Docker默认提供了docker-runc实现，事实上，通过containerd的封装，可以在Docker Daemon启动的时候指定runc的实现。

我们可以通过启动Docker Daemon时增加`--add-runtime`参数来选择其他的runC现。例如：

```csharp
docker daemon --add-runtime "custom=/usr/local/bin/my-runc-replacement"
```

下面就让我们看下这几个模块如何工作。

举个例子

这里通过Docker一些命令，实现不使用Docker Daemon直接启动一个镜像，以便了解Docker Daemon每个模块的作用。

首先，需要创建容器标准包，这部分实际上由containerd的bundle模块实现，将Docker镜像转换成容器标准包。

```bash
mkdir my_container
cd my_container
mkdir rootfs
docker export $(docker create busybox) | tar -C rootfs -xvf -
```

上述命令将busybox镜像解压缩到指定的rootfs目录中。如果本地不存在busybox镜像，containerd还会通过distribution模块去远程仓库拉取。

现在整个my_container目录结构如下：

```csharp
$ tree -d my_container/
my_container/
└── rootfs
    ├── bin
    ├── dev
    │   ├── pts
    │   └── shm
    ├── etc
    ├── home
    ├── proc
    ├── root
    ├── sys
    ├── tmp
    ├── usr
    │   └── sbin
    └── var
        ├── spool
        │   └── mail
        └── www
17 directories
```

此时，标准包所需的容器数据已经准备完毕，接下来我们需要创建配置文件：

```undefined
docker-runc spec
```

此时会生成一个名为`config.json`的配置文件，该文件和Docker容器的配置文件类似，主要包含容器挂载信息、平台信息、进程信息等容器启动依赖的所有数据。

最后，可以通过`runc`命令来启动容器：

```undefined
runc run busybox
```

注意，runc必须使用root权限启动。

执行之后，我们可以看见容器已经启动：

```bash
localhost my_container # runc run busybox
/ # ps aux
PID   USER     TIME   COMMAND
    1 root       0:00 sh
    9 root       0:00 ps aux
```

此时，事实上已经可以不依赖Docker本身，如果系统上安装了`runc`包，即可运行容器。对于Gentoo系统来说，安装`app-emulation/runc`包即可。

当然，也可以使用docker-runc命令来启动容器：

```bash
localhost my_container # docker-runc run busybox
/ # ps aux
PID   USER     TIME   COMMAND
    1 root       0:00 sh
    7 root       0:00 ps aux
```

从这里可以看到标准化的重要性。

总结

从Docker 1.11之后，Docker Daemon被分成了多个模块以适应OCI标准。拆分之后，结构分成了以下几个部分。

其中，containerd独立负责容器运行时和生命周期（如创建、启动、停止、中止、信号处理、删除等），其他一些如镜像构建、卷管理、日志等由Docker Daemon的其他模块处理。



![img](https://upload-images.jianshu.io/upload_images/8911567-a2909ee9253d3e1a.png)

image

Docker的模块块拥抱了开放标准，希望通过OCI的标准化，容器技术能够有很快的发展。