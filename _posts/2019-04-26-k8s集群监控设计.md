---
title: k8s集群监控方案
subtitle: 设计、分析、实施
layout: post
tags: [k8s,metric]
---

​		当前，公司已经在开发环境部署了k8s集群，但是集群的运维、管理能力还处于黑盒状态。docker daemon经常跑飞，管理人员往往需要开发通知才会发现。这个对于开发人员来讲是不能容忍的。因此，有必要逐渐健全集群的管理、监控能力。



### 一、需求

#### 综合监控

​		在Pass上运行的应用需要新的监控技术，因为有效的监控对于促进资源自动伸缩、维持高可用性、管理多租户、多种应用共享资源，以及识别应用和Pass系统中的性能缺陷是至关重要的。

​		为了从Pass平台的操作环境中提取充足的信息，我们必须有完整和全面的性能数据，并且进行有效的数据采集，因此，需要在容器云Paas平台上面提供全方位的应用性能监控（APM）。应用的性能数据包括应用的各项业务性能指标，以及应用运行环境（容器、虚拟机、硬件）的资源使用率指标。容器云Paas平台通过多样化采集、统一存储和展示，提供全面的监控、分析和可视化管理，至少从集群、主机和服务三个维度提供监控视图。

##### 1. 集群综合监控

在多集群、多租户模式下，每个租户都会关心自己的资源和服务的运行状况，容器云Paas平台为不同的租户模式下的租户展示不同的集群的综合监控信息，包含集群的资源使用情况、应用服务的资源使用情况、公共服务队资源的占用情况、系统关键事件等。

#####2. 主机监控视图

容器云Paas平台从主机维度提供对主机CPU、内存、存储资源使用情况的监控，提供实时综合监控视图和历史性性能数据监控视图。

#####3. 服务监控视图

从业务服务维度提供对业务服务（总体和各容器）的CPU、内存、网络IO等关键性能指标的监控，提供实时综合监控视图和历史性能数据监控视图。

#### 事件响应和处理

容器云Paas平台是一个综合多项目、多业务类型的大型基础设施平台，如何运维一个日趋复杂的大型分布式计算系统，以及如何快速响应和处理各类业务、系统、应用的事件和问题，是我们在实践工作中面临的巨大挑战。

对于容器云Paas平台的运维事件响应和处理过程，这里会给出具体的运维管理全景框架。

![](../img/运维框架.png)

​		对于容器云Paas监控平台产生的事件和告警，需要提供完备的事件响应流程、运维处理流程、告警处理机制和故障处理流程。

​		在Paas平台上管理着各类容器化和非容器化服务，相应的运维流程、涉及的运维角色和工作分工也不同。

- 对于在Paas平台上运行的容器化服务（包括企业的基础服务和自定义业务服务），其基础服务的日常监控、运维、告警处理由Paas平台的SRE人员来负责。其业务服务的日常监控、运维、告警处理则由各项目的业务运维人员来负责。
- 对于在Paas平台上运行的非容器话的基础服务（比如，Hadoop、Oracle），一般由Paas平台的SRE人员将监控告警排单给个项目的系统运维人员来进行告警和故障处理。



Paas平台承载的容器化业务服务的性能质量直接影响业务应用的好坏。因此，对容器化服务提供高效的运维指导是Paas平台的关键。Paas平台针对系统资源和日志进行监控，监控指标如下。

- 系统性能指标（cpu、内存、磁盘空间使用率等）。
- 采集分析系统日志，监控一场日志，统计分析业务处理的成功率。
- 配置告警策略，运维人员在收到告警短信后，第一时间处理。

告警机制如下图所示。

![]()

​		首先，进行原因定位，如果是主机、网络、存储故障，则第一时间升级到Iaas资源层去解决。如果为容器应用故障，则查看容器的运行时信息。其次，对于服务、容器问题，则迅速深入容器内部查看容器运行日志来定位具体的原因；对于平台上的某些复杂的调度等全局性问题，则需要结合集群中各节点的服务日志进行故障排查。

P241 todo



#### 数据分析和度量

DevOps**基于精益思想发展**而来，而**持续改进是精益思想的核心理念之一**。设立清晰可量化的度量指标，有助于衡量改进效果和实际的产出，并不断迭代后续的改进方向。根据容器云Paas平台系统自身的需求和用户关注的指标需求收集相关的数据，在收集数据的基础上进行数据的分析、度量、预测和反馈，因此，数据的采集是数据分析和度量的基础。

数据收集需要大量的工作，一般流程如下：

- 明确手机的目的，确定手机对象；
- 选择适合的数据收集方式；
- 在系统运行的过程中收集数据
- 整理数据，收集的数据结果比较乱，为了便于分析，需要对这些数据进行预处理；
- 分析数据，得出结论。

度量指标的选择和设定是度量和反馈的前提和基础。科学合理的设定度量指标有助于改进目标的达成。在选择度量指标的时候，需要关注两个方面：**度量指标的合理性**和**度量指标的有效性**。

**合理性方面**：依托于**当前业务价值流的分析**，从**过程指标**和**结果指标**两个维度来识别软件项目的实施结果，以及整个软件交付过程的改进方向。

**有效性方面**：遵循SMART原则，即指标必须是具体的、可衡量的、可达到的、同其他目标相关的、有明确的截止时间，这五大原则可以保证目标的科学有效。

度量指标的设定一般包括业务指标和系统指标。**业务指标**主要是根据具体业务的KPI的运营情况来具体分析和选择。

容器云Paas平台的系统运营指标的设定可以参考下面的表格。

| 分类         | 指标                             | 描述                                                         |
| ------------ | -------------------------------- | ------------------------------------------------------------ |
| 集群管理要求 | 单集群物理节点数量               | 单集群支持的物理节点数大于2000个                             |
|              | 可管理多集群数量                 | 支持至少10个集群数量                                         |
| 高可用性需求 | 支持集群环境的多样性             | 支持跨数据中心、跨广域网、跨VLAN的多集群管理                 |
|              | 应用服务的高可用                 | 服务访问成功率4个9                                           |
|              | Paas平台的高可用                 | 在发生不可预知事件时，平台整体对外的服务性能保持不变         |
|              | 数据高可用                       | 整个平台的配置管理数据必须提供备份和快速恢复的能力           |
|              | Paas平台的容灾                   | 各核心模块采用动态可扩展式设计，不会出现单点故障。当某个集群发生故障时，支持异地的容灾切换，由容灾集群自动接管所有故障的业务服务 |
|              | 灰度发布快速完成                 | 应用灰度发布能够做到快速完成，对业务无影响，让用户无感知     |
|              | 系统持续运行稳定性               | 在系统方案中应考虑硬件、数据库、应用服务器及Web服务器、管理平台等层面的高可用，保证7*24小时连续稳定的工作 |
| 扩展性要求   | 平台线性扩展能力                 | 即系统性能随着节点数量的增加能够同比例提升                   |
|              | 应用自动扩容所容的能力           | 在达到设定阈值时，系统能够实现基于CPU、内存或用户自定义策略的应用自动扩缩容 |
|              | 应用部署和更新的多集群支持       | 同一个应用可以实现在多集群同时部署和更新                     |
| 可管理性需求 | 和安全系统的对接                 |                                                              |
|              | 和网管系统的对接                 |                                                              |
| 性能要求     | 应用部署、更新、扩容缩容快速完成 | 1min内100个应用                                              |
|              | 网络IO时延                       | 比裸机损耗控制在10%                                          |
|              | 存储IO时延                       | 比裸机损耗控制在10%                                          |
|              | 业务吞吐量                       | 不低于裸机集群的95%                                          |
|              | 资源利用率                       | 比裸机集群高30%                                              |
|              | 容灾切换时长                     | 1分钟内完成                                                  |
|              |                                  |                                                              |
|              |                                  |                                                              |
|              |                                  |                                                              |



#### 反馈和优化





### 二、需求分析

主要监控的指标：

- 物理机性能监测
- kubernetes集群监测
- 应用服务、容器监测
- etcd集群监测
- docker daemon监测

### 三、技术方案

#### 1. 现有技术参考



#### 2. 物理机性能监测

主要采用[Node Exporter](https://github.com/prometheus/node_exporter) 组件搜集物理机上面的性能指标。

#### 3. 应用服务、容器监测

主要采用 [cAdvisor](https://github.com/google/cadvisor) 采集相关的容器运行时性能指标。

#### 4. kubernetes集群监测

主要采用 [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)采集相关的kubernetes。

#### 5. etcd集群监测

etcd集群本身支持prometheus的metrics监测。

#### 6. docker daemon监测

采用Docker 自身的metrics的功能。



### 四、技术实现

#### 0.参考文档

[Prometheus学习文档](https://yunlzheng.gitbook.io/prometheus-book/)

[Prometheus+Kubernetes 部署参考](https://blog.csdn.net/liukuan73/article/details/78881008#)

[Webhook对接](https://theo.im/blog/2017/10/16/release-prometheus-alertmanager-webhook-for-dingtalk/)

#### 1. Prometheus架构

![](../img/promithus-1.png)



#### 2.  kubernetes集群监测: kube-state-metrics 部署

   通过kube-state-metrics采集k8s资源对象的状态指标数据，并通过暴露的/metrics接口用prometheus抓取。

​	kube-state-metrics is a simple service that listens to the Kubernetes API server and generates metrics about the state of the objects. 

[kube-state-metrics ](https://github.com/kubernetes/kube-state-metrics)

- `quay.io/coreos/kube-state-metrics:v1.5.0`



##### 2.1部署文件  state-metric.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-state-metrics
  namespace: kube-system
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: kube-state-metrics
  namespace: kube-system
  labels:
    app: kube-state-metrics
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: kube-state-metrics
    spec:
      serviceAccountName: kube-state-metrics
      nodeName: 192.168.33.12
      containers:
      - name: kube-state-metrics
        image: quay.io/coreos/kube-state-metrics:v1.5.0
        ports:
        - containerPort: 8080
        args:
        - --apiserver=https://192.168.60.83:6443
        - --kubeconfig=/etc/kubernetes/kubeconfig.conf
        volumeMounts:
        - mountPath: /etc/kubernetes
          name: kubeconfig
      volumes:
      - name: kubeconfig
        hostPath:
          path: /etc/kubernetes
          type: Directory
      restartPolicy: Always
 #     nodeSelector:
#        node-role.kubernetes.io/master: "true"
      tolerations:
      - key: "node-role.kubernetes.io/master"
        effect: "NoSchedule"
---
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/scrape: 'true'
    prometheus.io/http-probe: 'true'
    prometheus.io/http-probe-path: '/healthz'
    prometheus.io/http-probe-port: '8080'
  name: kube-state-metrics
  namespace: kube-system
  labels:
    app: kube-state-metrics
spec:
  type: NodePort
  ports:
  - name: kube-state-metrics
    port: 8080
    targetPort: 8080
    nodePort: 30005
  selector:
    app: kube-state-metrics
```



##### 2.2 Metric-Server 启动配置

```shell
Usage:
   [flags]

Flags:
      --alsologtostderr                                         log to standard error as well as files
      --authentication-kubeconfig string                        kubeconfig file pointing at the 'core' kubernetes server with enough rights to create tokenaccessreviews.authentication.k8s.io.
      --authentication-skip-lookup                              If false, the authentication-kubeconfig will be used to lookup missing authentication configuration from the cluster.
      --authentication-token-webhook-cache-ttl duration         The duration to cache responses from the webhook token authenticator. (default 10s)
      --authorization-kubeconfig string                         kubeconfig file pointing at the 'core' kubernetes server with enough rights to create  subjectaccessreviews.authorization.k8s.io.
      --authorization-webhook-cache-authorized-ttl duration     The duration to cache 'authorized' responses from the webhook authorizer. (default 10s)
      --authorization-webhook-cache-unauthorized-ttl duration   The duration to cache 'unauthorized' responses from the webhook authorizer. (default 10s)
      --bind-address ip                                         The IP address on which to listen for the --secure-port port. The associated interface(s) must be reachable by the rest of the cluster, and by CLI/web clients. If blank, all interfaces will be used (0.0.0.0 for all IPv4 interfaces and :: for all IPv6 interfaces). (default 0.0.0.0)
      --cert-dir string                                         The directory where the TLS certs are located. If --tls-cert-file and --tls-private-key-file are provided, this flag will be ignored. (default "apiserver.local.config/certificates")
      --client-ca-file string                                   If set, any request presenting a client certificate signed by one of the authorities in the client-ca-file is authenticated with an identity corresponding to the CommonName of the client certificate.
      --contention-profiling                                    Enable lock contention profiling, if profiling is enabled
      --enable-swagger-ui                                       Enables swagger ui on the apiserver at /swagger-ui
  -h, --help                                                    help for this command
      --http2-max-streams-per-connection int                    The limit that the server gives to clients for the maximum number of streams in an HTTP/2 connection. Zero means to use golang's default.
      --kubeconfig string                                       The path to the kubeconfig used to connect to the Kubernetes API server and the Kubelets (defaults to in-cluster config)
      --kubelet-certificate-authority string                    Path to the CA to use to validate the Kubelet's serving certificates.
      --kubelet-insecure-tls                                    Do not verify CA of serving certificates presented by Kubelets.  For testing purposes only.
      --kubelet-port int                                        The port to use to connect to Kubelets. (default 10250)
      --kubelet-preferred-address-types strings                 The priority of node address types to use when determining which address to use to connect to a particular node (default [Hostname,InternalDNS,InternalIP,ExternalDNS,ExternalIP])
      --log-flush-frequency duration                            Maximum number of seconds between log flushes (default 5s)
      --log_backtrace_at traceLocation                          when logging hits line file:N, emit a stack trace (default :0)
      --log_dir string                                          If non-empty, write log files in this directory
      --logtostderr                                             log to standard error instead of files (default true)
      --metric-resolution duration                              The resolution at which metrics-server will retain metrics. (default 1m0s)
      --profiling                                               Enable profiling via web interface host:port/debug/pprof/ (default true)
      --requestheader-allowed-names strings                     List of client certificate common names to allow to provide usernames in headers specified by --requestheader-username-headers. If empty, any client certificate validated by the authorities in --requestheader-client-ca-file is allowed.
      --requestheader-client-ca-file string                     Root certificate bundle to use to verify client certificates on incoming requests before trusting usernames in headers specified by --requestheader-username-headers. WARNING: generally do not depend on authorization being already done for incoming requests.
      --requestheader-extra-headers-prefix strings              List of request header prefixes to inspect. X-Remote-Extra- is suggested. (default [x-remote-extra-])
      --requestheader-group-headers strings                     List of request headers to inspect for groups. X-Remote-Group is suggested. (default [x-remote-group])
      --requestheader-username-headers strings                  List of request headers to inspect for usernames. X-Remote-User is common. (default [x-remote-user])
      --secure-port int                                         The port on which to serve HTTPS with authentication and authorization. If 0, don't serve HTTPS at all. (default 443)
      --stderrthreshold severity                                logs at or above this threshold go to stderr (default 2)
      --tls-cert-file string                                    File containing the default x509 Certificate for HTTPS. (CA cert, if any, concatenated after server cert). If HTTPS serving is enabled, and --tls-cert-file and --tls-private-key-file are not provided, a self-signed certificate and key are generated for the public address and saved to the directory specified by --cert-dir.
      --tls-cipher-suites strings                               Comma-separated list of cipher suites for the server. If omitted, the default Go cipher suites will be use.  Possible values: TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA,TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256,TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_RC4_128_SHA,TLS_ECDHE_RSA_WITH_3DES_EDE_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_RC4_128_SHA,TLS_RSA_WITH_3DES_EDE_CBC_SHA,TLS_RSA_WITH_AES_128_CBC_SHA,TLS_RSA_WITH_AES_128_CBC_SHA256,TLS_RSA_WITH_AES_128_GCM_SHA256,TLS_RSA_WITH_AES_256_CBC_SHA,TLS_RSA_WITH_AES_256_GCM_SHA384,TLS_RSA_WITH_RC4_128_SHA
      --tls-min-version string                                  Minimum TLS version supported. Possible values: VersionTLS10, VersionTLS11, VersionTLS12
      --tls-private-key-file string                             File containing the default x509 private key matching --tls-cert-file.
      --tls-sni-cert-key namedCertKey                           A pair of x509 certificate and private key file paths, optionally suffixed with a list of domain patterns which are fully qualified domain names, possibly with prefixed wildcard segments. If no domain patterns are provided, the names of the certificate are extracted. Non-wildcard matches trump over wildcard matches, explicit domain patterns trump over extracted names. For multiple key/certificate pairs, use the --tls-sni-cert-key multiple times. Examples: "example.crt,example.key" or "foo.crt,foo.key:*.foo.com,foo.com". (default [])
  -v, --v Level                                                 log level for V logs
      --vmodule moduleSpec                                      comma-separated list of pattern=N settings for file-filtered logging


```



#### 3.  应用服务、容器监测：cadvisor部署



##### 3.1 部署配置文件

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cadvisor2

---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: cadvisor
rules:
  - apiGroups: ['policy']
    resources: ['podsecuritypolicies']
    verbs:     ['use']
    resourceNames:
    - cadvisor

---

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: cadvisor
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cadvisor
subjects:
- kind: ServiceAccount
  name: cadvisor
  namespace: cadvisor2

---

apiVersion: apps/v1 # for Kubernetes versions before 1.9.0 use apps/v1beta2
kind: DaemonSet
metadata:
  name: cadvisor
  namespace: cadvisor2
  annotations:
      seccomp.security.alpha.kubernetes.io/pod: 'docker/default'
spec:
  selector:
    matchLabels:
      name: cadvisor
  template:
    metadata:
      labels:
        name: cadvisor
    spec:
      serviceAccountName: cadvisor
      containers:
      - name: cadvisor
        image: registry.cn-hangzhou.aliyuncs.com/k8s_lijun/cadvisor:v0.30.2
        resources:
          requests:
            memory: 200Mi
            cpu: 150m
          limits:
            cpu: 300m
        volumeMounts:
        - name: rootfs
          mountPath: /rootfs
          readOnly: true
        - name: var-run
          mountPath: /var/run
          readOnly: true
        - name: sys
          mountPath: /sys
          readOnly: true
        - name: docker
          mountPath: /var/lib/docker
          readOnly: true
        - name: disk
          mountPath: /dev/disk
          readOnly: true
        ports:
         # - name: http
         #   containerPort: 8080
         #   protocol: TCP
         #   hostPort: 4194
         #   hostIP: 0.0.0.0
          - name: cadvisor
            containerPort: 8080
            hostPort: 8080
      hostNetwork: true
      hostPID: true
      hostIPC: true
      restartPolicy: Always
      automountServiceAccountToken: false
      terminationGracePeriodSeconds: 30
      volumes:
      - name: rootfs
        hostPath:
          path: /
      - name: var-run
        hostPath:
          path: /var/run
      - name: sys
        hostPath:
          path: /sys
      - name: docker
        hostPath:
          path: /var/lib/docker
      - name: disk
        hostPath:
          path: /dev/disk

---

apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: cadvisor
spec:
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  volumes:
  - '*'
  allowedHostPaths:
  - pathPrefix: "/"
  - pathPrefix: "/var/run"
  - pathPrefix: "/sys"
  - pathPrefix: "/var/lib/docker"
  - pathPrefix: "/dev/disk"

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: cadvisor
  namespace: cadvisor2
```





#### 4. Etcd metric

```shell
root@test-a1-60-83:/etc/kubernetes/pki/etcd# pwd
/etc/kubernetes/pki/etcd
root@test-a1-60-83:/etc/kubernetes/pki/etcd# curl -k --cert ca.crt --key ca.key https://127.0.0.1:2379/metrics
```

metric输出结果：

```shell
# HELP etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds Bucketed histogram of db compaction pause duration.
# TYPE etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds histogram
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="1"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="2"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="4"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="8"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="16"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="32"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="64"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="128"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="256"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="512"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="1024"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="2048"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="4096"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="+Inf"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_sum 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_count 0
# HELP etcd_debugging_mvcc_db_compaction_total_duration_milliseconds Bucketed histogram of db compaction total duration.
# TYPE etcd_debugging_mvcc_db_compaction_total_duration_milliseconds histogram
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="100"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="200"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="400"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="800"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="1600"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="3200"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="6400"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="12800"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="25600"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="51200"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="102400"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="204800"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="409600"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="819200"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="+Inf"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_sum 0
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_count 40765
# HELP etcd_debugging_mvcc_db_total_size_in_bytes Total size of the underlying database in bytes.
# TYPE etcd_debugging_mvcc_db_total_size_in_bytes gauge
etcd_debugging_mvcc_db_total_size_in_bytes 2.0455424e+07
# HELP etcd_debugging_mvcc_delete_total Total number of deletes seen by this member.
# TYPE etcd_debugging_mvcc_delete_total counter
etcd_debugging_mvcc_delete_total 40893
# HELP etcd_debugging_mvcc_events_total Total number of events sent by this member.
# TYPE etcd_debugging_mvcc_events_total counter
etcd_debugging_mvcc_events_total 2.887695e+07
# HELP etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds Bucketed histogram of index compaction pause duration.
# TYPE etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds histogram
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="0.5"} 0
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="1"} 0
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="2"} 0
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="4"} 0
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="8"} 0
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="16"} 0
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="32"} 69
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="64"} 22163
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="128"} 38584
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="256"} 40741
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="512"} 40765
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="1024"} 40765
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="+Inf"} 40765
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_sum 2.890765e+06
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_count 40765
# HELP etcd_debugging_mvcc_keys_total Total number of keys.
# TYPE etcd_debugging_mvcc_keys_total gauge
etcd_debugging_mvcc_keys_total 2366
# HELP etcd_debugging_mvcc_pending_events_total Total number of pending events to be sent.
# TYPE etcd_debugging_mvcc_pending_events_total gauge
etcd_debugging_mvcc_pending_events_total 0
# HELP etcd_debugging_mvcc_put_total Total number of puts seen by this member.
# TYPE etcd_debugging_mvcc_put_total counter
etcd_debugging_mvcc_put_total 2.9558814e+07
# HELP etcd_debugging_mvcc_range_total Total number of ranges seen by this member.
# TYPE etcd_debugging_mvcc_range_total counter
etcd_debugging_mvcc_range_total 1.09487112e+08
# HELP etcd_debugging_mvcc_slow_watcher_total Total number of unsynced slow watchers.
# TYPE etcd_debugging_mvcc_slow_watcher_total gauge
etcd_debugging_mvcc_slow_watcher_total 0
# HELP etcd_debugging_mvcc_txn_total Total number of txns seen by this member.
# TYPE etcd_debugging_mvcc_txn_total counter
etcd_debugging_mvcc_txn_total 3
# HELP etcd_debugging_mvcc_watch_stream_total Total number of watch streams.
# TYPE etcd_debugging_mvcc_watch_stream_total gauge
etcd_debugging_mvcc_watch_stream_total 60
# HELP etcd_debugging_mvcc_watcher_total Total number of watchers.
# TYPE etcd_debugging_mvcc_watcher_total gauge
etcd_debugging_mvcc_watcher_total 60
# HELP etcd_debugging_server_lease_expired_total The total number of expired leases.
# TYPE etcd_debugging_server_lease_expired_total counter
etcd_debugging_server_lease_expired_total 1.34057e+06
# HELP etcd_debugging_snap_save_marshalling_duration_seconds The marshalling cost distributions of save called by snapshot.
# TYPE etcd_debugging_snap_save_marshalling_duration_seconds histogram
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="0.001"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="0.002"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="0.004"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="0.008"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="0.016"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="0.032"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="0.064"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="0.128"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="0.256"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="0.512"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="1.024"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="2.048"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="4.096"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="8.192"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="+Inf"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_sum 0.08750034600000012
etcd_debugging_snap_save_marshalling_duration_seconds_count 3228
# HELP etcd_debugging_snap_save_total_duration_seconds The total latency distributions of save called by snapshot.
# TYPE etcd_debugging_snap_save_total_duration_seconds histogram
etcd_debugging_snap_save_total_duration_seconds_bucket{le="0.001"} 0
etcd_debugging_snap_save_total_duration_seconds_bucket{le="0.002"} 0
etcd_debugging_snap_save_total_duration_seconds_bucket{le="0.004"} 0
etcd_debugging_snap_save_total_duration_seconds_bucket{le="0.008"} 0
etcd_debugging_snap_save_total_duration_seconds_bucket{le="0.016"} 0
etcd_debugging_snap_save_total_duration_seconds_bucket{le="0.032"} 106
etcd_debugging_snap_save_total_duration_seconds_bucket{le="0.064"} 2458
etcd_debugging_snap_save_total_duration_seconds_bucket{le="0.128"} 3164
etcd_debugging_snap_save_total_duration_seconds_bucket{le="0.256"} 3227
etcd_debugging_snap_save_total_duration_seconds_bucket{le="0.512"} 3228
etcd_debugging_snap_save_total_duration_seconds_bucket{le="1.024"} 3228
etcd_debugging_snap_save_total_duration_seconds_bucket{le="2.048"} 3228
etcd_debugging_snap_save_total_duration_seconds_bucket{le="4.096"} 3228
etcd_debugging_snap_save_total_duration_seconds_bucket{le="8.192"} 3228
etcd_debugging_snap_save_total_duration_seconds_bucket{le="+Inf"} 3228
etcd_debugging_snap_save_total_duration_seconds_sum 175.96491270099995
etcd_debugging_snap_save_total_duration_seconds_count 3228
# HELP etcd_debugging_store_expires_total Total number of expired keys.
# TYPE etcd_debugging_store_expires_total counter
etcd_debugging_store_expires_total 0
# HELP etcd_debugging_store_reads_total Total number of reads action by (get/getRecursive), local to this member.
# TYPE etcd_debugging_store_reads_total counter
etcd_debugging_store_reads_total{action="get"} 1
etcd_debugging_store_reads_total{action="getRecursive"} 1
# HELP etcd_debugging_store_watch_requests_total Total number of incoming watch requests (new or reestablished).
# TYPE etcd_debugging_store_watch_requests_total counter
etcd_debugging_store_watch_requests_total 0
# HELP etcd_debugging_store_watchers Count of currently active watchers.
# TYPE etcd_debugging_store_watchers gauge
etcd_debugging_store_watchers 0
# HELP etcd_debugging_store_writes_total Total number of writes (e.g. set/compareAndDelete) seen by this member.
# TYPE etcd_debugging_store_writes_total counter
etcd_debugging_store_writes_total{action="create"} 1
etcd_debugging_store_writes_total{action="set"} 2
# HELP etcd_disk_backend_commit_duration_seconds The latency distributions of commit called by backend.
# TYPE etcd_disk_backend_commit_duration_seconds histogram
etcd_disk_backend_commit_duration_seconds_bucket{le="0.001"} 0
etcd_disk_backend_commit_duration_seconds_bucket{le="0.002"} 0
etcd_disk_backend_commit_duration_seconds_bucket{le="0.004"} 0
etcd_disk_backend_commit_duration_seconds_bucket{le="0.008"} 0
etcd_disk_backend_commit_duration_seconds_bucket{le="0.016"} 1
etcd_disk_backend_commit_duration_seconds_bucket{le="0.032"} 525626
etcd_disk_backend_commit_duration_seconds_bucket{le="0.064"} 2.207691e+07
etcd_disk_backend_commit_duration_seconds_bucket{le="0.128"} 2.8184157e+07
etcd_disk_backend_commit_duration_seconds_bucket{le="0.256"} 2.8406397e+07
etcd_disk_backend_commit_duration_seconds_bucket{le="0.512"} 2.8408558e+07
etcd_disk_backend_commit_duration_seconds_bucket{le="1.024"} 2.8408567e+07
etcd_disk_backend_commit_duration_seconds_bucket{le="2.048"} 2.8408568e+07
etcd_disk_backend_commit_duration_seconds_bucket{le="4.096"} 2.8408568e+07
etcd_disk_backend_commit_duration_seconds_bucket{le="8.192"} 2.8408568e+07
etcd_disk_backend_commit_duration_seconds_bucket{le="+Inf"} 2.8408568e+07
etcd_disk_backend_commit_duration_seconds_sum 1.4916583291146925e+06
etcd_disk_backend_commit_duration_seconds_count 2.8408568e+07
# HELP etcd_disk_backend_snapshot_duration_seconds The latency distribution of backend snapshots.
# TYPE etcd_disk_backend_snapshot_duration_seconds histogram
etcd_disk_backend_snapshot_duration_seconds_bucket{le="0.01"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="0.02"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="0.04"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="0.08"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="0.16"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="0.32"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="0.64"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="1.28"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="2.56"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="5.12"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="10.24"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="20.48"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="40.96"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="81.92"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="163.84"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="327.68"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="655.36"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="+Inf"} 0
etcd_disk_backend_snapshot_duration_seconds_sum 0
etcd_disk_backend_snapshot_duration_seconds_count 0
# HELP etcd_disk_wal_fsync_duration_seconds The latency distributions of fsync called by wal.
# TYPE etcd_disk_wal_fsync_duration_seconds histogram
etcd_disk_wal_fsync_duration_seconds_bucket{le="0.001"} 0
etcd_disk_wal_fsync_duration_seconds_bucket{le="0.002"} 2
etcd_disk_wal_fsync_duration_seconds_bucket{le="0.004"} 240885
etcd_disk_wal_fsync_duration_seconds_bucket{le="0.008"} 1.270288e+06
etcd_disk_wal_fsync_duration_seconds_bucket{le="0.016"} 5.605456e+06
etcd_disk_wal_fsync_duration_seconds_bucket{le="0.032"} 1.7382572e+07
etcd_disk_wal_fsync_duration_seconds_bucket{le="0.064"} 2.8135092e+07
etcd_disk_wal_fsync_duration_seconds_bucket{le="0.128"} 3.2043837e+07
etcd_disk_wal_fsync_duration_seconds_bucket{le="0.256"} 3.2169407e+07
etcd_disk_wal_fsync_duration_seconds_bucket{le="0.512"} 3.2170953e+07
etcd_disk_wal_fsync_duration_seconds_bucket{le="1.024"} 3.2170959e+07
etcd_disk_wal_fsync_duration_seconds_bucket{le="2.048"} 3.217096e+07
etcd_disk_wal_fsync_duration_seconds_bucket{le="4.096"} 3.217096e+07
etcd_disk_wal_fsync_duration_seconds_bucket{le="8.192"} 3.217096e+07
etcd_disk_wal_fsync_duration_seconds_bucket{le="+Inf"} 3.217096e+07
etcd_disk_wal_fsync_duration_seconds_sum 1.168266875368499e+06
etcd_disk_wal_fsync_duration_seconds_count 3.217096e+07
# HELP etcd_grpc_proxy_cache_hits_total Total number of cache hits
# TYPE etcd_grpc_proxy_cache_hits_total gauge
etcd_grpc_proxy_cache_hits_total 0
# HELP etcd_grpc_proxy_cache_keys_total Total number of keys/ranges cached
# TYPE etcd_grpc_proxy_cache_keys_total gauge
etcd_grpc_proxy_cache_keys_total 0
# HELP etcd_grpc_proxy_cache_misses_total Total number of cache misses
# TYPE etcd_grpc_proxy_cache_misses_total gauge
etcd_grpc_proxy_cache_misses_total 0
# HELP etcd_grpc_proxy_events_coalescing_total Total number of events coalescing
# TYPE etcd_grpc_proxy_events_coalescing_total counter
etcd_grpc_proxy_events_coalescing_total 0
# HELP etcd_grpc_proxy_watchers_coalescing_total Total number of current watchers coalescing
# TYPE etcd_grpc_proxy_watchers_coalescing_total gauge
etcd_grpc_proxy_watchers_coalescing_total 0
# HELP etcd_network_client_grpc_received_bytes_total The total number of bytes received from grpc clients.
# TYPE etcd_network_client_grpc_received_bytes_total counter
etcd_network_client_grpc_received_bytes_total 8.4254157317e+10
# HELP etcd_network_client_grpc_sent_bytes_total The total number of bytes sent to grpc clients.
# TYPE etcd_network_client_grpc_sent_bytes_total counter
etcd_network_client_grpc_sent_bytes_total 2.28265292191e+11
# HELP etcd_server_has_leader Whether or not a leader exists. 1 is existence, 0 is not.
# TYPE etcd_server_has_leader gauge
etcd_server_has_leader 1
# HELP etcd_server_leader_changes_seen_total The number of leader changes seen.
# TYPE etcd_server_leader_changes_seen_total counter
etcd_server_leader_changes_seen_total 1
# HELP etcd_server_proposals_applied_total The total number of consensus proposals applied.
# TYPE etcd_server_proposals_applied_total gauge
etcd_server_proposals_applied_total 3.2286263e+07
# HELP etcd_server_proposals_committed_total The total number of consensus proposals committed.
# TYPE etcd_server_proposals_committed_total gauge
etcd_server_proposals_committed_total 3.2286263e+07
# HELP etcd_server_proposals_failed_total The total number of failed proposals seen.
# TYPE etcd_server_proposals_failed_total counter
etcd_server_proposals_failed_total 0
# HELP etcd_server_proposals_pending The current number of pending proposals to commit.
# TYPE etcd_server_proposals_pending gauge
etcd_server_proposals_pending 0
# HELP go_gc_duration_seconds A summary of the GC invocation durations.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 7.5042e-05
go_gc_duration_seconds{quantile="0.25"} 0.000132326
go_gc_duration_seconds{quantile="0.5"} 0.000157816
go_gc_duration_seconds{quantile="0.75"} 0.000211393
go_gc_duration_seconds{quantile="1"} 0.00208058
go_gc_duration_seconds_sum 30.314232696
go_gc_duration_seconds_count 155023
# HELP go_goroutines Number of goroutines that currently exist.
# TYPE go_goroutines gauge
go_goroutines 702
# HELP go_memstats_alloc_bytes Number of bytes allocated and still in use.
# TYPE go_memstats_alloc_bytes gauge
go_memstats_alloc_bytes 8.9751928e+07
# HELP go_memstats_alloc_bytes_total Total number of bytes allocated, even if freed.
# TYPE go_memstats_alloc_bytes_total counter
go_memstats_alloc_bytes_total 5.326682560512e+12
# HELP go_memstats_buck_hash_sys_bytes Number of bytes used by the profiling bucket hash table.
# TYPE go_memstats_buck_hash_sys_bytes gauge
go_memstats_buck_hash_sys_bytes 2.2585e+06
# HELP go_memstats_frees_total Total number of frees.
# TYPE go_memstats_frees_total counter
go_memstats_frees_total 2.2123286875e+10
# HELP go_memstats_gc_sys_bytes Number of bytes used for garbage collection system metadata.
# TYPE go_memstats_gc_sys_bytes gauge
go_memstats_gc_sys_bytes 7.110656e+06
# HELP go_memstats_heap_alloc_bytes Number of heap bytes allocated and still in use.
# TYPE go_memstats_heap_alloc_bytes gauge
go_memstats_heap_alloc_bytes 8.9751928e+07
# HELP go_memstats_heap_idle_bytes Number of heap bytes waiting to be used.
# TYPE go_memstats_heap_idle_bytes gauge
go_memstats_heap_idle_bytes 9.2512256e+07
# HELP go_memstats_heap_inuse_bytes Number of heap bytes that are in use.
# TYPE go_memstats_heap_inuse_bytes gauge
go_memstats_heap_inuse_bytes 9.9704832e+07
# HELP go_memstats_heap_objects Number of allocated objects.
# TYPE go_memstats_heap_objects gauge
go_memstats_heap_objects 184060
# HELP go_memstats_heap_released_bytes_total Total number of heap bytes released to OS.
# TYPE go_memstats_heap_released_bytes_total counter
go_memstats_heap_released_bytes_total 7.8872576e+07
# HELP go_memstats_heap_sys_bytes Number of heap bytes obtained from system.
# TYPE go_memstats_heap_sys_bytes gauge
go_memstats_heap_sys_bytes 1.92217088e+08
# HELP go_memstats_last_gc_time_seconds Number of seconds since 1970 of last garbage collection.
# TYPE go_memstats_last_gc_time_seconds gauge
go_memstats_last_gc_time_seconds 1.5574549875435476e+09
# HELP go_memstats_lookups_total Total number of pointer lookups.
# TYPE go_memstats_lookups_total counter
go_memstats_lookups_total 1.144142e+07
# HELP go_memstats_mallocs_total Total number of mallocs.
# TYPE go_memstats_mallocs_total counter
go_memstats_mallocs_total 2.2123470935e+10
# HELP go_memstats_mcache_inuse_bytes Number of bytes in use by mcache structures.
# TYPE go_memstats_mcache_inuse_bytes gauge
go_memstats_mcache_inuse_bytes 9600
# HELP go_memstats_mcache_sys_bytes Number of bytes used for mcache structures obtained from system.
# TYPE go_memstats_mcache_sys_bytes gauge
go_memstats_mcache_sys_bytes 16384
# HELP go_memstats_mspan_inuse_bytes Number of bytes in use by mspan structures.
# TYPE go_memstats_mspan_inuse_bytes gauge
go_memstats_mspan_inuse_bytes 1.049864e+06
# HELP go_memstats_mspan_sys_bytes Number of bytes used for mspan structures obtained from system.
# TYPE go_memstats_mspan_sys_bytes gauge
go_memstats_mspan_sys_bytes 1.523712e+06
# HELP go_memstats_next_gc_bytes Number of heap bytes when next garbage collection will take place.
# TYPE go_memstats_next_gc_bytes gauge
go_memstats_next_gc_bytes 1.10755856e+08
# HELP go_memstats_other_sys_bytes Number of bytes used for other system allocations.
# TYPE go_memstats_other_sys_bytes gauge
go_memstats_other_sys_bytes 1.503924e+06
# HELP go_memstats_stack_inuse_bytes Number of bytes in use by the stack allocator.
# TYPE go_memstats_stack_inuse_bytes gauge
go_memstats_stack_inuse_bytes 4.849664e+06
# HELP go_memstats_stack_sys_bytes Number of bytes obtained from system for stack allocator.
# TYPE go_memstats_stack_sys_bytes gauge
go_memstats_stack_sys_bytes 4.849664e+06
# HELP go_memstats_sys_bytes Number of bytes obtained by system. Sum of all system allocations.
# TYPE go_memstats_sys_bytes gauge
go_memstats_sys_bytes 2.09479928e+08
# HELP grpc_server_handled_total Total number of RPCs completed on the server, regardless of success or failure.
# TYPE grpc_server_handled_total counter
grpc_server_handled_total{grpc_code="Canceled",grpc_method="Watch",grpc_service="etcdserverpb.Watch",grpc_type="bidi_stream"} 1
grpc_server_handled_total{grpc_code="OK",grpc_method="Compact",grpc_service="etcdserverpb.KV",grpc_type="unary"} 40765
grpc_server_handled_total{grpc_code="OK",grpc_method="LeaseGrant",grpc_service="etcdserverpb.Lease",grpc_type="unary"} 1.3407e+06
grpc_server_handled_total{grpc_code="OK",grpc_method="Range",grpc_service="etcdserverpb.KV",grpc_type="unary"} 5.100508e+07
grpc_server_handled_total{grpc_code="OK",grpc_method="Txn",grpc_service="etcdserverpb.KV",grpc_type="unary"} 2.9564223e+07
grpc_server_handled_total{grpc_code="Unavailable",grpc_method="Watch",grpc_service="etcdserverpb.Watch",grpc_type="bidi_stream"} 27400
grpc_server_handled_total{grpc_code="Unknown",grpc_method="Range",grpc_service="etcdserverpb.KV",grpc_type="unary"} 1
# HELP grpc_server_msg_received_total Total number of RPC stream messages received on the server.
# TYPE grpc_server_msg_received_total counter
grpc_server_msg_received_total{grpc_method="Compact",grpc_service="etcdserverpb.KV",grpc_type="unary"} 40765
grpc_server_msg_received_total{grpc_method="LeaseGrant",grpc_service="etcdserverpb.Lease",grpc_type="unary"} 1.3407e+06
grpc_server_msg_received_total{grpc_method="Range",grpc_service="etcdserverpb.KV",grpc_type="unary"} 5.1005081e+07
grpc_server_msg_received_total{grpc_method="Txn",grpc_service="etcdserverpb.KV",grpc_type="unary"} 2.9564223e+07
grpc_server_msg_received_total{grpc_method="Watch",grpc_service="etcdserverpb.Watch",grpc_type="bidi_stream"} 27461
# HELP grpc_server_msg_sent_total Total number of gRPC stream messages sent by the server.
# TYPE grpc_server_msg_sent_total counter
grpc_server_msg_sent_total{grpc_method="Compact",grpc_service="etcdserverpb.KV",grpc_type="unary"} 40765
grpc_server_msg_sent_total{grpc_method="LeaseGrant",grpc_service="etcdserverpb.Lease",grpc_type="unary"} 1.3407e+06
grpc_server_msg_sent_total{grpc_method="Range",grpc_service="etcdserverpb.KV",grpc_type="unary"} 5.100508e+07
grpc_server_msg_sent_total{grpc_method="Txn",grpc_service="etcdserverpb.KV",grpc_type="unary"} 2.9564223e+07
grpc_server_msg_sent_total{grpc_method="Watch",grpc_service="etcdserverpb.Watch",grpc_type="bidi_stream"} 2.8904411e+07
# HELP grpc_server_started_total Total number of RPCs started on the server.
# TYPE grpc_server_started_total counter
grpc_server_started_total{grpc_method="Compact",grpc_service="etcdserverpb.KV",grpc_type="unary"} 40765
grpc_server_started_total{grpc_method="LeaseGrant",grpc_service="etcdserverpb.Lease",grpc_type="unary"} 1.3407e+06
grpc_server_started_total{grpc_method="Range",grpc_service="etcdserverpb.KV",grpc_type="unary"} 5.1005081e+07
grpc_server_started_total{grpc_method="Txn",grpc_service="etcdserverpb.KV",grpc_type="unary"} 2.9564223e+07
grpc_server_started_total{grpc_method="Watch",grpc_service="etcdserverpb.Watch",grpc_type="bidi_stream"} 27461
# HELP http_request_duration_microseconds The HTTP request latencies in microseconds.
# TYPE http_request_duration_microseconds summary
http_request_duration_microseconds{handler="prometheus",quantile="0.5"} 1213.773
http_request_duration_microseconds{handler="prometheus",quantile="0.9"} 1213.773
http_request_duration_microseconds{handler="prometheus",quantile="0.99"} 1213.773
http_request_duration_microseconds_sum{handler="prometheus"} 13162.734
http_request_duration_microseconds_count{handler="prometheus"} 4
# HELP http_request_size_bytes The HTTP request sizes in bytes.
# TYPE http_request_size_bytes summary
http_request_size_bytes{handler="prometheus",quantile="0.5"} 63
http_request_size_bytes{handler="prometheus",quantile="0.9"} 63
http_request_size_bytes{handler="prometheus",quantile="0.99"} 63
http_request_size_bytes_sum{handler="prometheus"} 252
http_request_size_bytes_count{handler="prometheus"} 4
# HELP http_requests_total Total number of HTTP requests made.
# TYPE http_requests_total counter
http_requests_total{code="200",handler="prometheus",method="get"} 4
# HELP http_response_size_bytes The HTTP response sizes in bytes.
# TYPE http_response_size_bytes summary
http_response_size_bytes{handler="prometheus",quantile="0.5"} 26842
http_response_size_bytes{handler="prometheus",quantile="0.9"} 26842
http_response_size_bytes{handler="prometheus",quantile="0.99"} 26842
http_response_size_bytes_sum{handler="prometheus"} 107152
http_response_size_bytes_count{handler="prometheus"} 4
# HELP process_cpu_seconds_total Total user and system CPU time spent in seconds.
# TYPE process_cpu_seconds_total counter
process_cpu_seconds_total 293885.59
# HELP process_max_fds Maximum number of open file descriptors.
# TYPE process_max_fds gauge
process_max_fds 1.048576e+06
# HELP process_open_fds Number of open file descriptors.
# TYPE process_open_fds gauge
process_open_fds 79
# HELP process_resident_memory_bytes Resident memory size in bytes.
# TYPE process_resident_memory_bytes gauge
process_resident_memory_bytes 1.59084544e+08
# HELP process_start_time_seconds Start time of the process since unix epoch in seconds.
# TYPE process_start_time_seconds gauge
process_start_time_seconds 1.54521978131e+09
# HELP process_virtual_memory_bytes Virtual memory size in bytes.
# TYPE process_virtual_memory_bytes gauge
process_virtual_memory_bytes 1.0965065728e+10
root@test-a1-60-83:/etc/kubernetes/pki/etcd# ls
ca.crt  ca.key  healthcheck-client.crt  healthcheck-client.key  peer.crt  peer.key  server.crt  server.key  tmp
root@test-a1-60-83:/etc/kubernetes/pki/etcd# cat tmp
# HELP etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds Bucketed histogram of db compaction pause duration.
# TYPE etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds histogram
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="1"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="2"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="4"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="8"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="16"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="32"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="64"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="128"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="256"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="512"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="1024"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="2048"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="4096"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="+Inf"} 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_sum 0
etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_count 0
# HELP etcd_debugging_mvcc_db_compaction_total_duration_milliseconds Bucketed histogram of db compaction total duration.
# TYPE etcd_debugging_mvcc_db_compaction_total_duration_milliseconds histogram
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="100"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="200"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="400"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="800"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="1600"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="3200"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="6400"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="12800"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="25600"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="51200"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="102400"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="204800"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="409600"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="819200"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket{le="+Inf"} 40765
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_sum 0
etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_count 40765
# HELP etcd_debugging_mvcc_db_total_size_in_bytes Total size of the underlying database in bytes.
# TYPE etcd_debugging_mvcc_db_total_size_in_bytes gauge
etcd_debugging_mvcc_db_total_size_in_bytes 2.0455424e+07
# HELP etcd_debugging_mvcc_delete_total Total number of deletes seen by this member.
# TYPE etcd_debugging_mvcc_delete_total counter
etcd_debugging_mvcc_delete_total 40893
# HELP etcd_debugging_mvcc_events_total Total number of events sent by this member.
# TYPE etcd_debugging_mvcc_events_total counter
etcd_debugging_mvcc_events_total 2.887695e+07
# HELP etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds Bucketed histogram of index compaction pause duration.
# TYPE etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds histogram
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="0.5"} 0
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="1"} 0
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="2"} 0
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="4"} 0
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="8"} 0
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="16"} 0
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="32"} 69
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="64"} 22163
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="128"} 38584
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="256"} 40741
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="512"} 40765
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="1024"} 40765
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_bucket{le="+Inf"} 40765
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_sum 2.890765e+06
etcd_debugging_mvcc_index_compaction_pause_duration_milliseconds_count 40765
# HELP etcd_debugging_mvcc_keys_total Total number of keys.
# TYPE etcd_debugging_mvcc_keys_total gauge
etcd_debugging_mvcc_keys_total 2366
# HELP etcd_debugging_mvcc_pending_events_total Total number of pending events to be sent.
# TYPE etcd_debugging_mvcc_pending_events_total gauge
etcd_debugging_mvcc_pending_events_total 0
# HELP etcd_debugging_mvcc_put_total Total number of puts seen by this member.
# TYPE etcd_debugging_mvcc_put_total counter
etcd_debugging_mvcc_put_total 2.9558814e+07
# HELP etcd_debugging_mvcc_range_total Total number of ranges seen by this member.
# TYPE etcd_debugging_mvcc_range_total counter
etcd_debugging_mvcc_range_total 1.09487112e+08
# HELP etcd_debugging_mvcc_slow_watcher_total Total number of unsynced slow watchers.
# TYPE etcd_debugging_mvcc_slow_watcher_total gauge
etcd_debugging_mvcc_slow_watcher_total 0
# HELP etcd_debugging_mvcc_txn_total Total number of txns seen by this member.
# TYPE etcd_debugging_mvcc_txn_total counter
etcd_debugging_mvcc_txn_total 3
# HELP etcd_debugging_mvcc_watch_stream_total Total number of watch streams.
# TYPE etcd_debugging_mvcc_watch_stream_total gauge
etcd_debugging_mvcc_watch_stream_total 60
# HELP etcd_debugging_mvcc_watcher_total Total number of watchers.
# TYPE etcd_debugging_mvcc_watcher_total gauge
etcd_debugging_mvcc_watcher_total 60
# HELP etcd_debugging_server_lease_expired_total The total number of expired leases.
# TYPE etcd_debugging_server_lease_expired_total counter
etcd_debugging_server_lease_expired_total 1.34057e+06
# HELP etcd_debugging_snap_save_marshalling_duration_seconds The marshalling cost distributions of save called by snapshot.
# TYPE etcd_debugging_snap_save_marshalling_duration_seconds histogram
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="0.001"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="0.002"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="0.004"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="0.008"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="0.016"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="0.032"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="0.064"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="0.128"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="0.256"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="0.512"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="1.024"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="2.048"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="4.096"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="8.192"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_bucket{le="+Inf"} 3228
etcd_debugging_snap_save_marshalling_duration_seconds_sum 0.08750034600000012
etcd_debugging_snap_save_marshalling_duration_seconds_count 3228
# HELP etcd_debugging_snap_save_total_duration_seconds The total latency distributions of save called by snapshot.
# TYPE etcd_debugging_snap_save_total_duration_seconds histogram
etcd_debugging_snap_save_total_duration_seconds_bucket{le="0.001"} 0
etcd_debugging_snap_save_total_duration_seconds_bucket{le="0.002"} 0
etcd_debugging_snap_save_total_duration_seconds_bucket{le="0.004"} 0
etcd_debugging_snap_save_total_duration_seconds_bucket{le="0.008"} 0
etcd_debugging_snap_save_total_duration_seconds_bucket{le="0.016"} 0
etcd_debugging_snap_save_total_duration_seconds_bucket{le="0.032"} 106
etcd_debugging_snap_save_total_duration_seconds_bucket{le="0.064"} 2458
etcd_debugging_snap_save_total_duration_seconds_bucket{le="0.128"} 3164
etcd_debugging_snap_save_total_duration_seconds_bucket{le="0.256"} 3227
etcd_debugging_snap_save_total_duration_seconds_bucket{le="0.512"} 3228
etcd_debugging_snap_save_total_duration_seconds_bucket{le="1.024"} 3228
etcd_debugging_snap_save_total_duration_seconds_bucket{le="2.048"} 3228
etcd_debugging_snap_save_total_duration_seconds_bucket{le="4.096"} 3228
etcd_debugging_snap_save_total_duration_seconds_bucket{le="8.192"} 3228
etcd_debugging_snap_save_total_duration_seconds_bucket{le="+Inf"} 3228
etcd_debugging_snap_save_total_duration_seconds_sum 175.96491270099995
etcd_debugging_snap_save_total_duration_seconds_count 3228
# HELP etcd_debugging_store_expires_total Total number of expired keys.
# TYPE etcd_debugging_store_expires_total counter
etcd_debugging_store_expires_total 0
# HELP etcd_debugging_store_reads_total Total number of reads action by (get/getRecursive), local to this member.
# TYPE etcd_debugging_store_reads_total counter
etcd_debugging_store_reads_total{action="get"} 1
etcd_debugging_store_reads_total{action="getRecursive"} 1
# HELP etcd_debugging_store_watch_requests_total Total number of incoming watch requests (new or reestablished).
# TYPE etcd_debugging_store_watch_requests_total counter
etcd_debugging_store_watch_requests_total 0
# HELP etcd_debugging_store_watchers Count of currently active watchers.
# TYPE etcd_debugging_store_watchers gauge
etcd_debugging_store_watchers 0
# HELP etcd_debugging_store_writes_total Total number of writes (e.g. set/compareAndDelete) seen by this member.
# TYPE etcd_debugging_store_writes_total counter
etcd_debugging_store_writes_total{action="create"} 1
etcd_debugging_store_writes_total{action="set"} 2
# HELP etcd_disk_backend_commit_duration_seconds The latency distributions of commit called by backend.
# TYPE etcd_disk_backend_commit_duration_seconds histogram
etcd_disk_backend_commit_duration_seconds_bucket{le="0.001"} 0
etcd_disk_backend_commit_duration_seconds_bucket{le="0.002"} 0
etcd_disk_backend_commit_duration_seconds_bucket{le="0.004"} 0
etcd_disk_backend_commit_duration_seconds_bucket{le="0.008"} 0
etcd_disk_backend_commit_duration_seconds_bucket{le="0.016"} 1
etcd_disk_backend_commit_duration_seconds_bucket{le="0.032"} 525626
etcd_disk_backend_commit_duration_seconds_bucket{le="0.064"} 2.207691e+07
etcd_disk_backend_commit_duration_seconds_bucket{le="0.128"} 2.8184157e+07
etcd_disk_backend_commit_duration_seconds_bucket{le="0.256"} 2.8406397e+07
etcd_disk_backend_commit_duration_seconds_bucket{le="0.512"} 2.8408558e+07
etcd_disk_backend_commit_duration_seconds_bucket{le="1.024"} 2.8408567e+07
etcd_disk_backend_commit_duration_seconds_bucket{le="2.048"} 2.8408568e+07
etcd_disk_backend_commit_duration_seconds_bucket{le="4.096"} 2.8408568e+07
etcd_disk_backend_commit_duration_seconds_bucket{le="8.192"} 2.8408568e+07
etcd_disk_backend_commit_duration_seconds_bucket{le="+Inf"} 2.8408568e+07
etcd_disk_backend_commit_duration_seconds_sum 1.4916583291146925e+06
etcd_disk_backend_commit_duration_seconds_count 2.8408568e+07
# HELP etcd_disk_backend_snapshot_duration_seconds The latency distribution of backend snapshots.
# TYPE etcd_disk_backend_snapshot_duration_seconds histogram
etcd_disk_backend_snapshot_duration_seconds_bucket{le="0.01"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="0.02"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="0.04"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="0.08"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="0.16"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="0.32"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="0.64"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="1.28"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="2.56"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="5.12"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="10.24"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="20.48"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="40.96"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="81.92"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="163.84"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="327.68"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="655.36"} 0
etcd_disk_backend_snapshot_duration_seconds_bucket{le="+Inf"} 0
etcd_disk_backend_snapshot_duration_seconds_sum 0
etcd_disk_backend_snapshot_duration_seconds_count 0
# HELP etcd_disk_wal_fsync_duration_seconds The latency distributions of fsync called by wal.
# TYPE etcd_disk_wal_fsync_duration_seconds histogram
etcd_disk_wal_fsync_duration_seconds_bucket{le="0.001"} 0
etcd_disk_wal_fsync_duration_seconds_bucket{le="0.002"} 2
etcd_disk_wal_fsync_duration_seconds_bucket{le="0.004"} 240885
etcd_disk_wal_fsync_duration_seconds_bucket{le="0.008"} 1.270288e+06
etcd_disk_wal_fsync_duration_seconds_bucket{le="0.016"} 5.605456e+06
etcd_disk_wal_fsync_duration_seconds_bucket{le="0.032"} 1.7382572e+07
etcd_disk_wal_fsync_duration_seconds_bucket{le="0.064"} 2.8135092e+07
etcd_disk_wal_fsync_duration_seconds_bucket{le="0.128"} 3.2043837e+07
etcd_disk_wal_fsync_duration_seconds_bucket{le="0.256"} 3.2169407e+07
etcd_disk_wal_fsync_duration_seconds_bucket{le="0.512"} 3.2170953e+07
etcd_disk_wal_fsync_duration_seconds_bucket{le="1.024"} 3.2170959e+07
etcd_disk_wal_fsync_duration_seconds_bucket{le="2.048"} 3.217096e+07
etcd_disk_wal_fsync_duration_seconds_bucket{le="4.096"} 3.217096e+07
etcd_disk_wal_fsync_duration_seconds_bucket{le="8.192"} 3.217096e+07
etcd_disk_wal_fsync_duration_seconds_bucket{le="+Inf"} 3.217096e+07
etcd_disk_wal_fsync_duration_seconds_sum 1.168266875368499e+06
etcd_disk_wal_fsync_duration_seconds_count 3.217096e+07
# HELP etcd_grpc_proxy_cache_hits_total Total number of cache hits
# TYPE etcd_grpc_proxy_cache_hits_total gauge
etcd_grpc_proxy_cache_hits_total 0
# HELP etcd_grpc_proxy_cache_keys_total Total number of keys/ranges cached
# TYPE etcd_grpc_proxy_cache_keys_total gauge
etcd_grpc_proxy_cache_keys_total 0
# HELP etcd_grpc_proxy_cache_misses_total Total number of cache misses
# TYPE etcd_grpc_proxy_cache_misses_total gauge
etcd_grpc_proxy_cache_misses_total 0
# HELP etcd_grpc_proxy_events_coalescing_total Total number of events coalescing
# TYPE etcd_grpc_proxy_events_coalescing_total counter
etcd_grpc_proxy_events_coalescing_total 0
# HELP etcd_grpc_proxy_watchers_coalescing_total Total number of current watchers coalescing
# TYPE etcd_grpc_proxy_watchers_coalescing_total gauge
etcd_grpc_proxy_watchers_coalescing_total 0
# HELP etcd_network_client_grpc_received_bytes_total The total number of bytes received from grpc clients.
# TYPE etcd_network_client_grpc_received_bytes_total counter
etcd_network_client_grpc_received_bytes_total 8.4254157317e+10
# HELP etcd_network_client_grpc_sent_bytes_total The total number of bytes sent to grpc clients.
# TYPE etcd_network_client_grpc_sent_bytes_total counter
etcd_network_client_grpc_sent_bytes_total 2.28265292191e+11
# HELP etcd_server_has_leader Whether or not a leader exists. 1 is existence, 0 is not.
# TYPE etcd_server_has_leader gauge
etcd_server_has_leader 1
# HELP etcd_server_leader_changes_seen_total The number of leader changes seen.
# TYPE etcd_server_leader_changes_seen_total counter
etcd_server_leader_changes_seen_total 1
# HELP etcd_server_proposals_applied_total The total number of consensus proposals applied.
# TYPE etcd_server_proposals_applied_total gauge
etcd_server_proposals_applied_total 3.2286263e+07
# HELP etcd_server_proposals_committed_total The total number of consensus proposals committed.
# TYPE etcd_server_proposals_committed_total gauge
etcd_server_proposals_committed_total 3.2286263e+07
# HELP etcd_server_proposals_failed_total The total number of failed proposals seen.
# TYPE etcd_server_proposals_failed_total counter
etcd_server_proposals_failed_total 0
# HELP etcd_server_proposals_pending The current number of pending proposals to commit.
# TYPE etcd_server_proposals_pending gauge
etcd_server_proposals_pending 0
# HELP go_gc_duration_seconds A summary of the GC invocation durations.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 7.5042e-05
go_gc_duration_seconds{quantile="0.25"} 0.000132326
go_gc_duration_seconds{quantile="0.5"} 0.000157816
go_gc_duration_seconds{quantile="0.75"} 0.000211393
go_gc_duration_seconds{quantile="1"} 0.00208058
go_gc_duration_seconds_sum 30.314232696
go_gc_duration_seconds_count 155023
# HELP go_goroutines Number of goroutines that currently exist.
# TYPE go_goroutines gauge
go_goroutines 702
# HELP go_memstats_alloc_bytes Number of bytes allocated and still in use.
# TYPE go_memstats_alloc_bytes gauge
go_memstats_alloc_bytes 8.9751928e+07
# HELP go_memstats_alloc_bytes_total Total number of bytes allocated, even if freed.
# TYPE go_memstats_alloc_bytes_total counter
go_memstats_alloc_bytes_total 5.326682560512e+12
# HELP go_memstats_buck_hash_sys_bytes Number of bytes used by the profiling bucket hash table.
# TYPE go_memstats_buck_hash_sys_bytes gauge
go_memstats_buck_hash_sys_bytes 2.2585e+06
# HELP go_memstats_frees_total Total number of frees.
# TYPE go_memstats_frees_total counter
go_memstats_frees_total 2.2123286875e+10
# HELP go_memstats_gc_sys_bytes Number of bytes used for garbage collection system metadata.
# TYPE go_memstats_gc_sys_bytes gauge
go_memstats_gc_sys_bytes 7.110656e+06
# HELP go_memstats_heap_alloc_bytes Number of heap bytes allocated and still in use.
# TYPE go_memstats_heap_alloc_bytes gauge
go_memstats_heap_alloc_bytes 8.9751928e+07
# HELP go_memstats_heap_idle_bytes Number of heap bytes waiting to be used.
# TYPE go_memstats_heap_idle_bytes gauge
go_memstats_heap_idle_bytes 9.2512256e+07
# HELP go_memstats_heap_inuse_bytes Number of heap bytes that are in use.
# TYPE go_memstats_heap_inuse_bytes gauge
go_memstats_heap_inuse_bytes 9.9704832e+07
# HELP go_memstats_heap_objects Number of allocated objects.
# TYPE go_memstats_heap_objects gauge
go_memstats_heap_objects 184060
# HELP go_memstats_heap_released_bytes_total Total number of heap bytes released to OS.
# TYPE go_memstats_heap_released_bytes_total counter
go_memstats_heap_released_bytes_total 7.8872576e+07
# HELP go_memstats_heap_sys_bytes Number of heap bytes obtained from system.
# TYPE go_memstats_heap_sys_bytes gauge
go_memstats_heap_sys_bytes 1.92217088e+08
# HELP go_memstats_last_gc_time_seconds Number of seconds since 1970 of last garbage collection.
# TYPE go_memstats_last_gc_time_seconds gauge
go_memstats_last_gc_time_seconds 1.5574549875435476e+09
# HELP go_memstats_lookups_total Total number of pointer lookups.
# TYPE go_memstats_lookups_total counter
go_memstats_lookups_total 1.144142e+07
# HELP go_memstats_mallocs_total Total number of mallocs.
# TYPE go_memstats_mallocs_total counter
go_memstats_mallocs_total 2.2123470935e+10
# HELP go_memstats_mcache_inuse_bytes Number of bytes in use by mcache structures.
# TYPE go_memstats_mcache_inuse_bytes gauge
go_memstats_mcache_inuse_bytes 9600
# HELP go_memstats_mcache_sys_bytes Number of bytes used for mcache structures obtained from system.
# TYPE go_memstats_mcache_sys_bytes gauge
go_memstats_mcache_sys_bytes 16384
# HELP go_memstats_mspan_inuse_bytes Number of bytes in use by mspan structures.
# TYPE go_memstats_mspan_inuse_bytes gauge
go_memstats_mspan_inuse_bytes 1.049864e+06
# HELP go_memstats_mspan_sys_bytes Number of bytes used for mspan structures obtained from system.
# TYPE go_memstats_mspan_sys_bytes gauge
go_memstats_mspan_sys_bytes 1.523712e+06
# HELP go_memstats_next_gc_bytes Number of heap bytes when next garbage collection will take place.
# TYPE go_memstats_next_gc_bytes gauge
go_memstats_next_gc_bytes 1.10755856e+08
# HELP go_memstats_other_sys_bytes Number of bytes used for other system allocations.
# TYPE go_memstats_other_sys_bytes gauge
go_memstats_other_sys_bytes 1.503924e+06
# HELP go_memstats_stack_inuse_bytes Number of bytes in use by the stack allocator.
# TYPE go_memstats_stack_inuse_bytes gauge
go_memstats_stack_inuse_bytes 4.849664e+06
# HELP go_memstats_stack_sys_bytes Number of bytes obtained from system for stack allocator.
# TYPE go_memstats_stack_sys_bytes gauge
go_memstats_stack_sys_bytes 4.849664e+06
# HELP go_memstats_sys_bytes Number of bytes obtained by system. Sum of all system allocations.
# TYPE go_memstats_sys_bytes gauge
go_memstats_sys_bytes 2.09479928e+08
# HELP grpc_server_handled_total Total number of RPCs completed on the server, regardless of success or failure.
# TYPE grpc_server_handled_total counter
grpc_server_handled_total{grpc_code="Canceled",grpc_method="Watch",grpc_service="etcdserverpb.Watch",grpc_type="bidi_stream"} 1
grpc_server_handled_total{grpc_code="OK",grpc_method="Compact",grpc_service="etcdserverpb.KV",grpc_type="unary"} 40765
grpc_server_handled_total{grpc_code="OK",grpc_method="LeaseGrant",grpc_service="etcdserverpb.Lease",grpc_type="unary"} 1.3407e+06
grpc_server_handled_total{grpc_code="OK",grpc_method="Range",grpc_service="etcdserverpb.KV",grpc_type="unary"} 5.100508e+07
grpc_server_handled_total{grpc_code="OK",grpc_method="Txn",grpc_service="etcdserverpb.KV",grpc_type="unary"} 2.9564223e+07
grpc_server_handled_total{grpc_code="Unavailable",grpc_method="Watch",grpc_service="etcdserverpb.Watch",grpc_type="bidi_stream"} 27400
grpc_server_handled_total{grpc_code="Unknown",grpc_method="Range",grpc_service="etcdserverpb.KV",grpc_type="unary"} 1
# HELP grpc_server_msg_received_total Total number of RPC stream messages received on the server.
# TYPE grpc_server_msg_received_total counter
grpc_server_msg_received_total{grpc_method="Compact",grpc_service="etcdserverpb.KV",grpc_type="unary"} 40765
grpc_server_msg_received_total{grpc_method="LeaseGrant",grpc_service="etcdserverpb.Lease",grpc_type="unary"} 1.3407e+06
grpc_server_msg_received_total{grpc_method="Range",grpc_service="etcdserverpb.KV",grpc_type="unary"} 5.1005081e+07
grpc_server_msg_received_total{grpc_method="Txn",grpc_service="etcdserverpb.KV",grpc_type="unary"} 2.9564223e+07
grpc_server_msg_received_total{grpc_method="Watch",grpc_service="etcdserverpb.Watch",grpc_type="bidi_stream"} 27461
# HELP grpc_server_msg_sent_total Total number of gRPC stream messages sent by the server.
# TYPE grpc_server_msg_sent_total counter
grpc_server_msg_sent_total{grpc_method="Compact",grpc_service="etcdserverpb.KV",grpc_type="unary"} 40765
grpc_server_msg_sent_total{grpc_method="LeaseGrant",grpc_service="etcdserverpb.Lease",grpc_type="unary"} 1.3407e+06
grpc_server_msg_sent_total{grpc_method="Range",grpc_service="etcdserverpb.KV",grpc_type="unary"} 5.100508e+07
grpc_server_msg_sent_total{grpc_method="Txn",grpc_service="etcdserverpb.KV",grpc_type="unary"} 2.9564223e+07
grpc_server_msg_sent_total{grpc_method="Watch",grpc_service="etcdserverpb.Watch",grpc_type="bidi_stream"} 2.8904411e+07
# HELP grpc_server_started_total Total number of RPCs started on the server.
# TYPE grpc_server_started_total counter
grpc_server_started_total{grpc_method="Compact",grpc_service="etcdserverpb.KV",grpc_type="unary"} 40765
grpc_server_started_total{grpc_method="LeaseGrant",grpc_service="etcdserverpb.Lease",grpc_type="unary"} 1.3407e+06
grpc_server_started_total{grpc_method="Range",grpc_service="etcdserverpb.KV",grpc_type="unary"} 5.1005081e+07
grpc_server_started_total{grpc_method="Txn",grpc_service="etcdserverpb.KV",grpc_type="unary"} 2.9564223e+07
grpc_server_started_total{grpc_method="Watch",grpc_service="etcdserverpb.Watch",grpc_type="bidi_stream"} 27461
# HELP http_request_duration_microseconds The HTTP request latencies in microseconds.
# TYPE http_request_duration_microseconds summary
http_request_duration_microseconds{handler="prometheus",quantile="0.5"} 1213.773
http_request_duration_microseconds{handler="prometheus",quantile="0.9"} 1213.773
http_request_duration_microseconds{handler="prometheus",quantile="0.99"} 1213.773
http_request_duration_microseconds_sum{handler="prometheus"} 13162.734
http_request_duration_microseconds_count{handler="prometheus"} 4
# HELP http_request_size_bytes The HTTP request sizes in bytes.
# TYPE http_request_size_bytes summary
http_request_size_bytes{handler="prometheus",quantile="0.5"} 63
http_request_size_bytes{handler="prometheus",quantile="0.9"} 63
http_request_size_bytes{handler="prometheus",quantile="0.99"} 63
http_request_size_bytes_sum{handler="prometheus"} 252
http_request_size_bytes_count{handler="prometheus"} 4
# HELP http_requests_total Total number of HTTP requests made.
# TYPE http_requests_total counter
http_requests_total{code="200",handler="prometheus",method="get"} 4
# HELP http_response_size_bytes The HTTP response sizes in bytes.
# TYPE http_response_size_bytes summary
http_response_size_bytes{handler="prometheus",quantile="0.5"} 26842
http_response_size_bytes{handler="prometheus",quantile="0.9"} 26842
http_response_size_bytes{handler="prometheus",quantile="0.99"} 26842
http_response_size_bytes_sum{handler="prometheus"} 107152
http_response_size_bytes_count{handler="prometheus"} 4
# HELP process_cpu_seconds_total Total user and system CPU time spent in seconds.
# TYPE process_cpu_seconds_total counter
process_cpu_seconds_total 293885.59
# HELP process_max_fds Maximum number of open file descriptors.
# TYPE process_max_fds gauge
process_max_fds 1.048576e+06
# HELP process_open_fds Number of open file descriptors.
# TYPE process_open_fds gauge
process_open_fds 79
# HELP process_resident_memory_bytes Resident memory size in bytes.
# TYPE process_resident_memory_bytes gauge
process_resident_memory_bytes 1.59084544e+08
# HELP process_start_time_seconds Start time of the process since unix epoch in seconds.
# TYPE process_start_time_seconds gauge
process_start_time_seconds 1.54521978131e+09
# HELP process_virtual_memory_bytes Virtual memory size in bytes.
# TYPE process_virtual_memory_bytes gauge
process_virtual_memory_bytes 1.0965065728e+10
```



#### 5. Docker Daemon

使用docker本身的metric接口



#### 6. 物理机数据采集：Node-exporter

部署Node-Exporter，略。



### 五、Prometheus 部署





配置文件prometheus.yml

```yaml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
       - localhost:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
   - "/usr/local/bin/prometheus/node_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']

  - job_name: 'prometheus-node-exporter'
    scrape_interval: 5s
    static_configs:
      - targets: ['192.168.62.212:9100','192.168.60.83:9100','192.168.33.12:9100','192.168.33.13:9100','192.168.33.14:9100','192.168.33.15:9100','192.168.33.16:9100','192.168.33.18:9100','192.168.33.19:9100','192.168.33.20:9100','192.168.33.21:9100']
  - job_name: 'docker'
    static_configs:
      - targets: ['192.168.62.212:1334','192.168.33.19:1334']
  - job_name: 'k8s-state-metric'
    static_configs:
      - targets: ['192.168.33.12:30005']
  - job_name: 'k8s-schedule'
    static_configs:
      - targets: ['192.168.60.83:10251']

  - job_name: 'cadvisor'
    scrape_interval: 80s
    scrape_timeout: 80s
    static_configs:
      - targets: ['192.168.33.12:8080','192.168.33.13:8080','192.168.33.14:8080','192.168.33.15:8080','192.168.33.16:8080','192.168.33.18:8080','192.168.33.19:8080','192.168.33.20:8080','192.168.33.21:8080']
  - job_name: 'etcd-metric'
    scheme: 'https'
    tls_config:
      ca_file: /etc/kubernetes/pki/etcd/server.crt
      cert_file: /etc/kubernetes/pki/etcd/ca.crt
      key_file: /etc/kubernetes/pki/etcd/ca.key
    static_configs:
      - targets: ['127.0.0.1:2379']
```





报警规则文件：node_rules.yml

```yaml
groups:
- name: Memory Alert
  rules:
  - alert: 节点内存使用报警
    expr: (node_memory_MemTotal_bytes-node_memory_MemFree_bytes-node_memory_Cached_bytes)/node_memory_MemTotal_bytes*100 > 90
    for: 1m
    annotations:
      summary: " Instance {{ $labels.instance }} memory 使用 over 90%"
      description: " {{ $labels.instance }} of {{ $labels.job }} job has been over 90%"

- name: docker daemon Alert
  rules:
  - alert: Docker Daemon is  dowm!
    expr: count_values('instance',engine_daemon_engine_info) < 2
    for: 1m
    annotations:
      summary: "Docker 守护进程 Go Die"
      description: "Instances: {{ $labels.instance }}"

- name: CPU Alert
  rules:
  - alert: 节点CPU负载报警
    expr: (1-avg(irate(node_cpu_seconds_total{mode="idle"}[1m] ) )by (instance))*100 > 80
    for: 1m
    annotations:
      summary: "Instance {{ $labels.instance }} CPU 负载 over 80%"
      description: "Instance {{ $labels.instance }} CPU 负载过高"

- name: Disk Alert
  rules:
  - alert: 节点Disk负载报警
    expr: (1-node_filesystem_free_bytes{fstype!~"rootfs|selinuxfs|autofs|rpc_pipefs|tmpfs|udev|none|devpts|sysfs|debugfs|fuse.*"}/node_filesystem_size_bytes{fstype!~"rootfs|selinuxfs|autofs|rpc_pipefs|tmpfs|udev|none|devpts|sysfs|debugfs|fuse.*"})*100 > 80
    for: 5m
    annotations:
      summary: "Instance {{ $labels.instance }} Disk 使用超过80%！"
      description: "Instance {{ $labels.instance }} Disk 使用负载过高"

- name: 容器CPU Alert
  rules:
  - alert: 容器 CPU 使用负载报警
    expr: sum(irate(container_cpu_usage_seconds_total{container_label_io_kubernetes_container_name!~"POD",container_label_io_kubernetes_docker_type="container"}[2m])) by (instance,container_label_io_kubernetes_container_name) > 1
    for: 3s
    annotations:
      summary: "Instance  {{ $labels.instance }} 容器CPU负载超过1 core"
      description: "Container {{ $labels.container_label_io_kubernetes_container_name }} in {{ $labels.instance }} 容器CPU负载：{{ $value }} cores"
```



#### prometheus-webhook-dingtalk启动参数

```shell
[Unit]
Description=Prometheus-DingTalk
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/prometheus-webhook-dingtalk/prometheus-webhook-dingtalk \
    --ding.profile="webhook1=https://oapi.dingtalk.com/robot/send?access_token=8604c1318892702ff7b5d7b7ba8ceba79d0407df5362414a9e0a4b9d0a47552b"

[Install]
WantedBy=multi-user.target
```

消息模版：

```toml
{{ define "__subject" }}[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .GroupLabels.SortedPairs.Values | join " " }} {{ if gt (len .CommonLabels) (len .GroupLabels) }}({{ with .CommonLabels.Remove .GroupLabels.Names }}{{ .Values | join " " }}{{ end }}){{ end }}{{ end }}
{{ define "__alertmanagerURL" }}{{ .ExternalURL }}/#/alerts?receiver={{ .Receiver }}{{ end }}

{{ define "__text_alert_list" }}{{ range . }}
**Labels**
{{ range .Labels.SortedPairs }}> - {{ .Name }}: {{ .Value | markdown | html }}
{{ end }}
**Annotations**
{{ range .Annotations.SortedPairs }}> - {{ .Name }}: {{ .Value | markdown | html }}
{{ end }}
**Source:** [{{ .GeneratorURL }}]({{ .GeneratorURL }})

{{ end }}{{ end }}

{{ define "ding.link.title" }}{{ template "__subject" . }}{{ end }}
{{ define "ding.link.content" }}#### \[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}\] **[{{ index .GroupLabels "alertname" }}]({{ template "__alertmanagerURL" . }})**
{{ template "__text_alert_list" .Alerts.Firing }}
{{ end }}
```



#### Q&A

##### 1. 为啥127.0.0.1:2379/metrics返回空的数据，没有结果？

操作方式不对，没有添加相关的证书校验信息。

##### 2. Etcd的metric功能是干啥的呢？

主要用来监控当前etcd集群相关的性能，防止数据的丢失，提前预警。

