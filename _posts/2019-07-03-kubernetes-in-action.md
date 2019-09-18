---
title: Kubernetes 无损发布
layout: post
tags: [kubernetes]
---



### 背景

当前测试环境经常会遇到测试服务不稳定的情况。我们已经提供了隔离组、DNS的解决方案。但是，在细节方面好需要再优化一下，保证服务发布时，对使用方无感知、如丝般顺滑（即用户的请求不会被中断）。

结合我们当前的情况，主要有两类服务需要进行无损发布：

- web服务，主要是在应用下线时，正常的客户端请求不要再打到了这台机器上面；
- mainstay服务，和web一样。服务关闭时，我们需要在注册中心将服务下线。

总结一下，在我们的容器实例被销毁的时候，我们需要通知相关的组件，将该服务进行下线，流量不要打过来。

另外，需要介绍一下当前容器实例在kubernetes上面的启动背景。我们的每个服务都会配置一个健康检查的机制：http、tcp或者自定义脚本。在我们重新发布应用的时候，之前的容器不会马上删除，它还在继续提供服务。等新发布的实例启动成功，健康检查结果为health的时候，此时旧的容器才会被删除。

### 需求、分析

#### 1. 需求

实例销毁时：

- mainstay服务对应的host+port，进行下线；
- web服务在网关进行机器的下线，禁止流量打过来；

#### 2. 分析

**针对mainstay服务**，mainstay支持host+port的上下线接口。因此，我们只要在容器销毁之前将相关的信息提交到mainstay的管理中心，就可以将相关的 mainstay实例下线。不会导致服务方的调用失败。

**针对web服务**，需要交代一下背景。当前我们的web服务大部分都是接入的网关。主要是通过ZK上面节点的监听来进行服务的上线、下线。因此，我们不需要对其进行处理。



### 实现方案

针对进程的销毁，我们可以监听信号量SIGTERM、SIGKILL。但是，由于容器里面运行了不止一个进程，操作起来复杂度比较高。下线强耦合到了进程里面，修改起来很不方便。因此，这个方案被pass掉。

Kubernetes提供了pod生命周期的hook功能。我们可以在pod销毁之前，执行相关的操作prestop。（执行脚本，下线服务）



```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "21"
  creationTimestamp: 2019-06-13T03:19:04Z
  generation: 21
  name: fm-pns-hdfs-rpc-provider-stable
  namespace: default
  resourceVersion: "45735270"
  selfLink: /apis/extensions/v1beta1/namespaces/default/deployments/fm-pns-hdfs-rpc-provider-stable
  uid: ff602647-8d89-11e9-96bf-3417eb9eb8af
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      appName: pns-hdfs-rpc-provider
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        appName: pns-hdfs-rpc-provider
        isolation: stable
        type: mainstay3
      name: pns-hdfs-rpc-provider-stable
    spec:
      containers:
      - env:
        - name: JVM_PERM_SIZE
          value: "819"
        - name: PORT
          value: "9959"
        - name: IMAGE_NAME
          value: harbor102.test.ximalaya.com/test/pns-hdfs-rpc-provider:20190703-101514
        - name: MAINSTAY_GROUP
          value: hdfs-provider
        - name: HEALTH_CHECK_PORT
          value: "9959"
        - name: JVM_START
          value: ' -Xms128m -Xmx2457m -XX:MetaspaceSize=64m -XX:MaxMetaspaceSize=819m '
        - name: APP_NAME
          value: pns-hdfs-rpc-provider
        - name: JVM_HEAP_MIN_SIZE
          value: "128"
        - name: JVM_PERM_MIN_SIZE
          value: "64"
        - name: ISOLATION_GROUP
          value: stable
        - name: PUBLISH_BILL_ID
          value: "60805"
        - name: APP_TYPES
          value: '[mainstay3]'
        - name: DEBUG
          value: "false"
        - name: PACKAGE_TYPE
          value: war
        - name: JVM_HEAP_SIZE
          value: "2457"
        - name: HEALTH_CHECK_TYPE
          value: tcp
        - name: CREATOR
          value: ops
        - name: HOST
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: spec.nodeName
        image: harbor102.test.ximalaya.com/test/pns-hdfs-rpc-provider:20190703-101514
        imagePullPolicy: IfNotPresent
        lifecycle:
          preStop:
            exec:
              command:
              - sh
              - /srv/stop/pre-stop.sh
        livenessProbe:
          failureThreshold: 3
          initialDelaySeconds: 5
          periodSeconds: 10
          successThreshold: 1
          tcpSocket:
            port: 8888
          timeoutSeconds: 10
        name: pns-hdfs-rpc-provider
        readinessProbe:
          failureThreshold: 3
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          tcpSocket:
            port: 9959
          timeoutSeconds: 1800
        resources:
          limits:
            cpu: "1"
            memory: 4096e6
          requests:
            cpu: 100m
            memory: 4096e6
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /logs
          name: logs
        - mountPath: /etc/resolv.conf
          name: resolv
          readOnly: true
        - mountPath: /srv/stop
          name: prestop-config
      dnsPolicy: ClusterFirst
      hostAliases:
      - hostnames:
        - storage2.test.lan
        ip: 192.168.3.222
      - hostnames:
        - storage3.test.lan
        ip: 192.168.1.118
      - hostnames:
        - storage4.test.lan
        ip: 192.168.3.117
      - hostnames:
        - sh-bs-3-i7-clickhouse-18-57
        - sh-bs-3-i10-clickhouse-18-14
        - sh-bs-3-i10-clickhouse-18-12
        - sh-bs-3-i7-clickhouse-18-24
        - sh-bs-3-i8-clickhouse-18-59
        - sh-bs-3-i13-clickhouse-18-66
        - sh-bs-3-i13-clickhouse-18-63
        - sh-bs-3-i12-clickhouse-18-71
        - sh-bs-3-j9-clickhouse-35-73
        - sh-bs-3-i12-clickhouse-18-53
        - sh-bs-3-j2-clickhouse-33-49
        - sh-bs-3-j2-clickhouse-33-48
        - sh-bs-3-j2-clickhouse-33-47
        - sh-bs-3-j2-clickhouse-33-46
        - sh-bs-3-i8-clickhouse-18-33
        - sh-bs-3-j2-clickhouse-33-45
        ip: 61.172.194.131
      - hostnames:
        - tracker.test.lan
        ip: 192.168.3.7
      - hostnames:
        - storage1.test.lan
        ip: 192.168.3.166
      - hostnames:
        - ares.com
        ip: 192.168.62.24
      - hostnames:
        - test.9nali.com
        ip: 192.168.1.171
      - hostnames:
        - storage7.test.lan
        ip: 192.168.3.48
      - hostnames:
        - storage5.test.lan
        ip: 192.168.3.46
      - hostnames:
        - storage8.test.lan
        ip: 192.168.3.68
      - hostnames:
        - storage9.test.lan
        ip: 192.168.3.69
      - hostnames:
        - storage6.test.lan
        ip: 192.168.3.47
      - hostnames:
        - storage10.test.lan
        ip: 192.168.3.70
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - hostPath:
          path: /var/log/app
          type: DirectoryOrCreate
        name: logs
      - hostPath:
          path: /etc/resolv.conf
          type: FileOrCreate
        name: resolv
      - configMap:
          defaultMode: 420
          name: prestop-config
        name: prestop-config
status:
  availableReplicas: 1
  conditions:
  - lastTransitionTime: 2019-07-03T03:19:57Z
    lastUpdateTime: 2019-07-03T03:19:57Z
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: 2019-07-03T02:07:23Z
    lastUpdateTime: 2019-07-03T03:47:31Z
    message: ReplicaSet "fm-pns-hdfs-rpc-provider-stable-59f6bfb94b" has successfully
      progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  observedGeneration: 21
  readyReplicas: 1
  replicas: 1
  updatedReplicas: 1
```



```shell
#!/bin/bash

if [ $MAINSTAY_GROUP ] ;then
	curl -d "host=$HOSTNAME&port=$PORT" "http://cms.test.9nali.com/mainstay-admin/api/cmdbunapprove"
fi
```





## 服务依赖图相关

[参考文档](https://github.com/google/guava/wiki/GraphsExplained)

### Guava Graph

Guava's `common.graph` is a library for modeling [graph](https://en.wikipedia.org/wiki/Graph_(discrete_mathematics))-structured data, that is, entities and the relationships between them. Examples include webpages and hyperlinks; scientists and the papers that they write; airports and the routes between them; and people and their family ties (family trees). I**ts purpose is to provide a common and extensible language for working with such data.**

## Definitions

A graph consists of a set of **nodes** (also called vertices) and a set of **edges** (also called links, or arcs); each edge connects nodes to each other. The nodes incident to an edge are called its **endpoints**.

(While we introduce an interface called `Graph` below, we will use "graph" (lower case "g") as a general term referring to this type of data structure. When we want to refer to a specific type in this library, we capitalize it.)

An edge is **directed** if it has a defined start (its **source**) and end (its **target**, also called its destination). Otherwise, it is **undirected**. Directed edges are suitable for modeling asymmetric relations ("descended from", "links to", "authored by"), while undirected edges are suitable for modeling symmetric relations ("coauthored a paper with", "distance between", "sibling of").

A graph is directed if each of its edges are directed, and undirected if each of its edges are undirected. (`common.graph` does not support graphs that have both directed and undirected edges.)

Given this example:

```
graph.addEdge(nodeU, nodeV, edgeUV);
```

- `nodeU` and `nodeV` are mutually **adjacent**
- `edgeUV` is **incident** to `nodeU` and to `nodeV` (and vice versa)

If `graph` is directed, then:

- `nodeU` is a **predecessor** of `nodeV`
- `nodeV` is a **successor** of `nodeU`
- `edgeUV` is an **outgoing** edge (or out-edge) of `nodeU`
- `edgeUV` is an **incoming** edge (or in-edge) of `nodeV`
- `nodeU` is a **source** of `edgeUV`
- `nodeV` is a **target** of `edgeUV`

If `graph` is undirected, then:

- `nodeU` is a predecessor and a successor of `nodeV`
- `nodeV` is a predecessor and a successor of `nodeU`
- `edgeUV` is both an incoming and an outgoing edge of `nodeU`
- `edgeUV` is both an incoming and an outgoing edge of `nodeV`

All of these relationships are with respect to `graph`.

A **self-loop** is an edge that connects a node to itself; equivalently, it is an edge whose endpoints are the same node. If a self-loop is directed, it is both an outgoing and incoming edge of its incident node, and its incident node is both a source and a target of the self-loop edge.

Two edges are **parallel** if they connect the same nodes in the same order (if any), and **antiparallel**if they connect the same nodes in the opposite order. (Undirected edges cannot be antiparallel.)

Given this example:

```
directedGraph.addEdge(nodeU, nodeV, edgeUV_a);
directedGraph.addEdge(nodeU, nodeV, edgeUV_b);
directedGraph.addEdge(nodeV, nodeU, edgeVU);

undirectedGraph.addEdge(nodeU, nodeV, edgeUV_a);
undirectedGraph.addEdge(nodeU, nodeV, edgeUV_b);
undirectedGraph.addEdge(nodeV, nodeU, edgeVU);
```

In `directedGraph`, `edgeUV_a` and `edgeUV_b` are mutually parallel, and each is antiparallel with `edgeVU`.

In `undirectedGraph`, each of `edgeUV_a`, `edgeUV_b`, and `edgeVU` is mutually parallel with the other two.



## Capabilities

`common.graph` is focused on providing interfaces and classes to support working with graphs. It does not provide functionality such as I/O or visualization support, and it has a very limited selection of utilities.

`common.graph` is focused on providing interfaces and classes to support working with graphs. It **does not provide functionality such as I/O or visualization support**, and it has a very limited selection of utilities. See the [FAQ](https://github.com/google/guava/wiki/GraphsExplained#faq) for more on this topic.

As a whole, `common.graph` supports graphs of the following varieties:

- directed graphs
- undirected graphs
- nodes and/or edges with associated values (weights, labels, etc.)
- graphs that do/don't allow self-loops
- graphs that do/don't allow parallel edges (graphs with parallel edges are sometimes called multigraphs)
- graphs whose nodes/edges are insertion-ordered, sorted, or unordered

The library is agnostic as to the choice of underlying data structures: relationships can be stored as matrices, adjacency lists, adjacency maps, etc. depending on what use cases the implementor wants to optimize for.



## Graph Types

There are three top-level graph interfaces, that are distinguished by their representation of edges: `Graph`, `ValueGraph`, and `Network`. These are sibling types, i.e., none is a subtype of any of the others.

Each of these "top-level" interfaces extends [`SuccessorsFunction`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/graph/SuccessorsFunction.html) and [`PredecessorsFunction`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/graph/PredecessorsFunction.html). These interfaces are meant to be used as the type of a parameter to graph algorithms (such as breadth first traversal) that only need a way of accessing the successors/predecessors of a node in a graph. This is especially useful in cases where the owner of a graph already has a representation that works for them and doesn't particularly want to serialize their representation into a `common.graph` type just to run one graph algorithm.

### Graph

[`Graph`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/graph/Graph.html) is the simplest and most fundamental graph type. It defines the low-level operators for dealing with node-to-node relationships, such as `successors(node)`, `adjacentNodes(node)`, and `inDegree(node)`. Its nodes are first-class unique objects; you can think of them as analogous to `Map` keys into the `Graph` internal data structures.

The edges of a `Graph` are completely anonymous; they are defined only in terms of their endpoints.

Example use case: `Graph<Airport>`, whose edges connect the airports between which one can take a direct flight.

### ValueGraph

[`ValueGraph`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/graph/ValueGraph.html) has all the node-related methods that [`Graph`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/graph/Graph.html) does, but adds a couple of methods that retrieve a value for a specified edge.

The edges of a `ValueGraph` each have an associated user-specified value. These values need not be unique (as nodes are); the relationship between a `ValueGraph` and a `Graph` is analogous to that between a `Map` and a `Set`; a `Graph`'s edges are a set of pairs of endpoints, and a `ValueGraph`'s edges are a map from pairs of endpoints to values.)

[`ValueGraph`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/graph/ValueGraph.html) provides an `asGraph()` method which returns a `Graph` view of the `ValueGraph`. This allows methods which operate on `Graph` instances to function for `ValueGraph` instances as well.

Example use case: `ValueGraph<Airport, Integer>`, whose edges values represent the time required to travel between the two `Airport`s that the edge connects.

### Network

[`Network`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/graph/Network.html) has all the node-related methods that `Graph` does, but adds methods that work with edges and node-to-edge relationships, such as `outEdges(node)`, `incidentNodes(edge)`, and `edgesConnecting(nodeU, nodeV)`.

The edges of a `Network` are first-class (unique) objects, just as nodes are in all graph types. The uniqueness constraint for edges allows [`Network`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/graph/Network.html) to natively support parallel edges, as well as the methods relating to edges and node-to-edge relationships.

[`Network`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/graph/Network.html) provides an `asGraph()` method which returns a `Graph` view of the `Network`. This allows methods which operate on `Graph` instances to function for `Network` instances as well.

Example use case: `Network<Airport, Flight>`, in which the edges represent the specific flights that one can take to get from one airport to another.



根据上面的介绍，我们可以采用NetWork的图类型，来存储应用的相关信息，其中节点为容器实例的信息。边为隔离组的信息。前继节点stable可以连向非stable、stable的节点。如果非stable节点有自己的同类型的下家，那么连上他，没有的话，连接默认的stable。

缺点：guava graph不支持可视化。没有持久化存储，需要动态实时的生成相关节点的链路，可能会导致处理时间比较长。需要前端支持。每次客户端查询的时候，我们将相关的数据查询出来，结合容器的实例信息，返回给前端渲染。

[前端可视化插件](https://antv.alipay.com/zh-cn/index.html)



### Plan B

采用图数据库neo4j，使用方便、可视化、支持数据库里面的所有操作（查询、删除、索引。。。）。



[参考文档](https://neo4j.com/docs/getting-started/current/graphdb-concepts/)

和guava graph的概念一样，都是节点、边。

可以根据目标应用，查找出其相关的依赖节点路线。屏蔽不必要的项目干扰。有了这条调用链之后，结合容器信息返回给前端，展示。



### 总结

能不能使用neo4j来持久化相关的应用依赖的基本信息，等需要的时候直接查询，提供刷行接口，实时更新图数据。然后使用guava构建动态的包含容器实例信息的依赖图。