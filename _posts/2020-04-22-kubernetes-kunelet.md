## Kubelet分析



### Bootstrap ：Kubelet的启动接口

```go
// Bootstrap is a bootstrapping interface for kubelet, targets the initialization protocol
type Bootstrap interface {
   GetConfiguration() kubeletconfiginternal.KubeletConfiguration
   BirthCry()
   StartGarbageCollection()
   ListenAndServe(address net.IP, port uint, tlsOptions *server.TLSOptions, auth server.AuthInterface, enableCAdvisorJSONEndpoints, enableDebuggingHandlers, enableContentionProfiling bool)
   ListenAndServeReadOnly(address net.IP, port uint, enableCAdvisorJSONEndpoints bool)
   ListenAndServePodResources()
   Run(<-chan kubetypes.PodUpdate)
   RunOnce(<-chan kubetypes.PodUpdate) ([]RunPodResult, error)
}
```



### Run实现

初始化相应的模块

```
if err := kl.initializeModules(); err != nil {
   kl.recorder.Eventf(kl.nodeRef, v1.EventTypeWarning, events.KubeletSetupFailed, err.Error())
   klog.Fatal(err)
}
```





初始化内部模块，不依赖容器运行时启动。主要工作如下：

1. 初始化Prometheus配置，采集器配置 Prometheus metrics.
2. 初始化必备的文件系统目录 Setup filesystem directories.
3. 初始化容器日志目录 If the container logs directory does not exist, create it.
4. 启动**镜像管理器** Start the image manager.
5. 启动server认证管理 Start the certificate manager if it was enabled.
6. 物理机节点的**OOM监测**。 Start out of memory watcher.
7. 启动**资源分析器** Start resource analyzer

```go
// Note that the modules here must not depend on modules that are not initialized here.
func (kl *Kubelet) initializeModules() error {
   // 1. 初始化Prometheus配置，采集器配置 Prometheus metrics.
   metrics.Register(
      kl.runtimeCache,
      //数据卷状态采集器
      collectors.NewVolumeStatsCollector(kl),
     //Pod状态日志采集器
      collectors.NewLogMetricsCollector(kl.StatsProvider.ListPodStats),
   )
   metrics.SetNodeName(kl.nodeName)
   servermetrics.Register()

   // 2. 初始化必备的文件系统目录 Setup filesystem directories.
   if err := kl.setupDataDirs(); err != nil {
      return err
   }

   //3. 初始化容器日志目录 If the container logs directory does not exist, create it.
   if _, err := os.Stat(ContainerLogsDir); err != nil {
      if err := kl.os.MkdirAll(ContainerLogsDir, 0755); err != nil {
         klog.Errorf("Failed to create directory %q: %v", ContainerLogsDir, err)
      }
   }

   //4. 启动镜像管理器 Start the image manager.
   kl.imageManager.Start()

   //5. 启动server认证管理 Start the certificate manager if it was enabled.
   if kl.serverCertificateManager != nil {
      kl.serverCertificateManager.Start()
   }

   //6. 物理机节点的OOM监测。 Start out of memory watcher.
   if err := kl.oomWatcher.Start(kl.nodeRef); err != nil {
      return fmt.Errorf("failed to start OOM watcher %v", err)
   }

   //7. 启动资源分析器 Start resource analyzer
   kl.resourceAnalyzer.Start()

   return nil
}
```

我们重点分析一下：资源分析器、物理机节点的OOM监测、镜像管理器。

#### 资源分析器

接口定义如下：

```go
// ResourceAnalyzer provides statistics on node resource consumption
type ResourceAnalyzer interface {
  
   Start()

   fsResourceAnalyzerInterface
   SummaryProvider
}

// fsResourceAnalyzerInterface is for embedding fs functions into ResourceAnalyzer
//内嵌文件系统的功能到ResourceAnalyzer
type fsResourceAnalyzerInterface interface {
	GetPodVolumeStats(uid types.UID) (PodVolumeStats, bool)
}

// SummaryProvider provides summaries of the stats from Kubelet.
//Kubelet状态概述
type SummaryProvider interface {
	// Get provides a new Summary with the stats from Kubelet,
	// and will update some stats if updateStats is true
	Get(updateStats bool) (*statsapi.Summary, error)
	// GetCPUAndMemoryStats provides a new Summary with the CPU and memory stats from Kubelet,
	GetCPUAndMemoryStats() (*statsapi.Summary, error)
}

```



具体的实现：

```go
// resourceAnalyzer implements ResourceAnalyzer
type resourceAnalyzer struct {
   *fsResourceAnalyzer
   SummaryProvider
}

var _ ResourceAnalyzer = &resourceAnalyzer{}

// NewResourceAnalyzer returns a new ResourceAnalyzer
func NewResourceAnalyzer(statsProvider Provider, calVolumeFrequency time.Duration) ResourceAnalyzer {
   fsAnalyzer := newFsResourceAnalyzer(statsProvider, calVolumeFrequency)
   summaryProvider := NewSummaryProvider(statsProvider)
   return &resourceAnalyzer{fsAnalyzer, summaryProvider}
}

// Start starts background functions necessary for the ResourceAnalyzer to function
func (ra *resourceAnalyzer) Start() {
   ra.fsResourceAnalyzer.Start()
}
```

上面可以看出，具体依赖了`Provider`接口。并根据这个来初始化资源分析器里面核心的两个组件：

- fsResourceAnalyzer
- SummaryProvider



#### Provider接口定义

它的定义位于kubelet server的stats包下面，属于handler的范畴。

**下面所有的状态由CRI或者cAdvisor提供**

```go
// Provider hosts methods required by stats handlers.
type Provider interface {
   //1. 下面所有的状态由CRI或者cAdvisor提供
  
   // ListPodStats returns the stats of all the containers managed by pods.
  //PodStats 包含了:cpu、mem、网络、数据卷、存储
   ListPodStats() ([]statsapi.PodStats, error)
  
   // ListPodStatsAndUpdateCPUNanoCoreUsage updates the cpu nano core usage for
   // the containers and returns the stats for all the pod-managed containers.
   ListPodCPUAndMemoryStats() ([]statsapi.PodStats, error)
   // ListPodStatsAndUpdateCPUNanoCoreUsage returns the stats of all the
   // containers managed by pods and force update the cpu usageNanoCores.
   // This is a workaround for CRI runtimes that do not integrate with
   // cadvisor. See https://github.com/kubernetes/kubernetes/issues/72788
   // for more details.
   ListPodStatsAndUpdateCPUNanoCoreUsage() ([]statsapi.PodStats, error)
   // ImageFsStats returns the stats of the image filesystem.
   ImageFsStats() (*statsapi.FsStats, error)

   // The following stats are provided by cAdvisor.
   //
   // GetCgroupStats returns the stats and the networking usage of the cgroup
   // with the specified cgroupName.
   GetCgroupStats(cgroupName string, updateStats bool) (*statsapi.ContainerStats, *statsapi.NetworkStats, error)
   // GetCgroupCPUAndMemoryStats returns the CPU and memory stats of the cgroup with the specified cgroupName.
   GetCgroupCPUAndMemoryStats(cgroupName string, updateStats bool) (*statsapi.ContainerStats, error)

   // RootFsStats returns the stats of the node root filesystem.
   RootFsStats() (*statsapi.FsStats, error)

   // The following stats are provided by cAdvisor for legacy usage.
   //
   // GetContainerInfo returns the information of the container with the
   // containerName managed by the pod with the uid.
   GetContainerInfo(podFullName string, uid types.UID, containerName string, req *cadvisorapi.ContainerInfoRequest) (*cadvisorapi.ContainerInfo, error)
   // GetRawContainerInfo returns the information of the container with the
   // containerName. If subcontainers is true, this function will return the
   // information of all the sub-containers as well.
   GetRawContainerInfo(containerName string, req *cadvisorapi.ContainerInfoRequest, subcontainers bool) (map[string]*cadvisorapi.ContainerInfo, error)

   // The following information is provided by Kubelet.
   //
   // GetPodByName returns the spec of the pod with the name in the specified
   // namespace.
   GetPodByName(namespace, name string) (*v1.Pod, bool)
   // GetNode returns the spec of the local node.
   GetNode() (*v1.Node, error)
   // GetNodeConfig returns the configuration of the local node.
   GetNodeConfig() cm.NodeConfig
   // ListVolumesForPod returns the stats of the volume used by the pod with
   // the podUID.
   ListVolumesForPod(podUID types.UID) (map[string]volume.Volume, bool)
   // GetPods returns the specs of all the pods running on this node.
   GetPods() []*v1.Pod

   // RlimitStats returns the rlimit stats of system.
   RlimitStats() (*statsapi.RlimitStats, error)

   // GetPodCgroupRoot returns the literal cgroupfs value for the cgroup containing all pods
   GetPodCgroupRoot() string

   // GetPodByCgroupfs provides the pod that maps to the specified cgroup literal, as well
   // as whether the pod was found.
   GetPodByCgroupfs(cgroupfs string) (*v1.Pod, bool)
}
```



#### fsResourceAnalyzer：文件资源分析器的具体实现

```go
type statCache map[types.UID]*volumeStatCalculator

// fsResourceAnalyzerInterface is for embedding fs functions into ResourceAnalyzer
type fsResourceAnalyzerInterface interface {
   GetPodVolumeStats(uid types.UID) (PodVolumeStats, bool)
}

// fsResourceAnalyzer provides stats about fs resource usage
type fsResourceAnalyzer struct {
   statsProvider     Provider
   calcPeriod        time.Duration
   cachedVolumeStats atomic.Value
   startOnce         sync.Once
}

var _ fsResourceAnalyzerInterface = &fsResourceAnalyzer{}

// newFsResourceAnalyzer returns a new fsResourceAnalyzer implementation
func newFsResourceAnalyzer(statsProvider Provider, calcVolumePeriod time.Duration) *fsResourceAnalyzer {
   r := &fsResourceAnalyzer{
      statsProvider: statsProvider,
      calcPeriod:    calcVolumePeriod,
   }
   r.cachedVolumeStats.Store(make(statCache))
   return r
}

// Start eager background caching of volume stats.
func (s *fsResourceAnalyzer) Start() {
   s.startOnce.Do(func() {
      if s.calcPeriod <= 0 {
         klog.Info("Volume stats collection disabled.")
         return
      }
      klog.Info("Starting FS ResourceAnalyzer")
      go wait.Forever(func() { s.updateCachedPodVolumeStats() }, s.calcPeriod)
   })
}

// updateCachedPodVolumeStats calculates and caches the PodVolumeStats for every Pod known to the kubelet.
func (s *fsResourceAnalyzer) updateCachedPodVolumeStats() {
   oldCache := s.cachedVolumeStats.Load().(statCache)
   newCache := make(statCache)

   // Copy existing entries to new map, creating/starting new entries for pods missing from the cache
   for _, pod := range s.statsProvider.GetPods() {
      if value, found := oldCache[pod.GetUID()]; !found {
         newCache[pod.GetUID()] = newVolumeStatCalculator(s.statsProvider, s.calcPeriod, pod).StartOnce()
      } else {
         newCache[pod.GetUID()] = value
      }
   }

   // Stop entries for pods that have been deleted
   for uid, entry := range oldCache {
      if _, found := newCache[uid]; !found {
         entry.StopOnce()
      }
   }

   // Update the cache reference
   s.cachedVolumeStats.Store(newCache)
}

// GetPodVolumeStats returns the PodVolumeStats for a given pod.  Results are looked up from a cache that
// is eagerly populated in the background, and never calculated on the fly.
func (s *fsResourceAnalyzer) GetPodVolumeStats(uid types.UID) (PodVolumeStats, bool) {
   cache := s.cachedVolumeStats.Load().(statCache)
   statCalc, found := cache[uid]
   if !found {
      // TODO: Differentiate between stats being empty
      // See issue #20679
      return PodVolumeStats{}, false
   }
   return statCalc.GetLatest()
}
```

#### kubelet具体的实现

核心：数据的提供者还得靠CRI或者CADvisor

具体干活的**web service**主要如下：

```go
type handler struct {
   provider        Provider
   summaryProvider SummaryProvider
}

// CreateHandlers creates the REST handlers for the stats.
//todo 创建kubelet的server的web service。对外暴露一个接口
func CreateHandlers(rootPath string, provider Provider, summaryProvider SummaryProvider, enableCAdvisorJSONEndpoints bool) *restful.WebService {
   h := &handler{provider, summaryProvider}

   ws := &restful.WebService{}
   ws.Path(rootPath).
      Produces(restful.MIME_JSON)

   type endpoint struct {
      path    string
      handler restful.RouteFunction
   }

   //todo 概括描述：cpu和memde使用
   endpoints := []endpoint{
      {"/summary", h.handleSummary},
   }

   //todo 如果打开了相关的CAdvisor 。暴露这些功能，主要用于容器资源的查询
   if enableCAdvisorJSONEndpoints {
      endpoints = append(endpoints,
         endpoint{"", h.handleStats},
         endpoint{"/container", h.handleSystemContainer},
         endpoint{"/{podName}/{containerName}", h.handlePodContainer},
         endpoint{"/{namespace}/{podName}/{uid}/{containerName}", h.handlePodContainer},
      )
   }

   //将具体的处理，注入到webService里面
   for _, e := range endpoints {
      for _, method := range []string{"GET", "POST"} {
         ws.Route(ws.
            Method(method).
            Path(e.path).
            To(e.handler))
      }
   }

   return ws
}
```

#### kubelet server的具体实现

```go
// Server is a http.Handler which exposes kubelet functionality over HTTP.
type Server struct {
   auth                       AuthInterface
   host                       HostInterface
   restfulCont                containerInterface
   resourceAnalyzer           stats.ResourceAnalyzer
   redirectContainerStreaming bool
}
```



```go
// AuthInterface contains all methods required by the auth filters
type AuthInterface interface {
   authenticator.Request
   authorizer.RequestAttributesGetter
   authorizer.Authorizer
}

// HostInterface contains all the kubelet methods required by the server.
// For testability.
type HostInterface interface {
   stats.Provider
   GetVersionInfo() (*cadvisorapi.VersionInfo, error)
   GetCachedMachineInfo() (*cadvisorapi.MachineInfo, error)
   GetRunningPods() ([]*v1.Pod, error)
   RunInContainer(name string, uid types.UID, container string, cmd []string) ([]byte, error)
   GetKubeletContainerLogs(ctx context.Context, podFullName, containerName string, logOptions *v1.PodLogOptions, stdout, stderr io.Writer) error
   ServeLogs(w http.ResponseWriter, req *http.Request)
   ResyncInterval() time.Duration
   GetHostname() string
   LatestLoopEntryTime() time.Time
   GetExec(podFullName string, podUID types.UID, containerName string, cmd []string, streamOpts remotecommandserver.Options) (*url.URL, error)
   GetAttach(podFullName string, podUID types.UID, containerName string, streamOpts remotecommandserver.Options) (*url.URL, error)
   GetPortForward(podName, podNamespace string, podUID types.UID, portForwardOpts portforward.V4Options) (*url.URL, error)
}
```



####  restful容器功能，基于根容器

```go
// containerInterface defines the restful.Container functions used on the root container
type containerInterface interface {
   Add(service *restful.WebService) *restful.Container
   Handle(path string, handler http.Handler)
   Filter(filter restful.FilterFunction)
   ServeHTTP(w http.ResponseWriter, r *http.Request)
   RegisteredWebServices() []*restful.WebService

   // RegisteredHandlePaths returns the paths of handlers registered directly with the container (non-web-services)
   // Used to test filters are being applied on non-web-service handlers
   RegisteredHandlePaths() []string
}

// filteringContainer delegates all Handle(...) calls to Container.HandleWithFilter(...),
// so we can ensure restful.FilterFunctions are used for all handlers
type filteringContainer struct {
   *restful.Container
   registeredHandlePaths []string
}

func (a *filteringContainer) Handle(path string, handler http.Handler) {
   a.HandleWithFilter(path, handler)
   a.registeredHandlePaths = append(a.registeredHandlePaths, path)
}
func (a *filteringContainer) RegisteredHandlePaths() []string {
   return a.registeredHandlePaths
}
```





### 对于流请求进行的处理



```go
// Server is the library interface to serve the stream requests.
type Server interface {
   http.Handler

   // Get the serving URL for the requests.
   // Requests must not be nil. Responses may be nil iff an error is returned.
   GetExec(*runtimeapi.ExecRequest) (*runtimeapi.ExecResponse, error)
   GetAttach(req *runtimeapi.AttachRequest) (*runtimeapi.AttachResponse, error)
   GetPortForward(*runtimeapi.PortForwardRequest) (*runtimeapi.PortForwardResponse, error)

   // Start the server.
   // addr is the address to serve on (address:port) stayUp indicates whether the server should
   // listen until Stop() is called, or automatically stop after all expected connections are
   // closed. Calling Get{Exec,Attach,PortForward} increments the expected connection count.
   // Function does not return until the server is stopped.
   Start(stayUp bool) error
   // Stop the server, and terminate any open connections.
   Stop() error
}

// Runtime is the interface to execute the commands and provide the streams.
type Runtime interface {
   Exec(containerID string, cmd []string, in io.Reader, out, err io.WriteCloser, tty bool, resize <-chan remotecommand.TerminalSize) error
   Attach(containerID string, in io.Reader, out, err io.WriteCloser, tty bool, resize <-chan remotecommand.TerminalSize) error
   PortForward(podSandboxID string, port int32, stream io.ReadWriteCloser) error
}
```

### 重要的包——remotecommand

```
Package remotecommand contains functions related to executing commands in and attaching to pods.
```



exec.go

```go
// Executor knows how to execute a command in a container in a pod.
type Executor interface {
   // ExecInContainer executes a command in a container in the pod, copying data
   // between in/out/err and the container's stdin/stdout/stderr.
   ExecInContainer(name string, uid types.UID, container string, cmd []string, in io.Reader, out, err io.WriteCloser, tty bool, resize <-chan remotecommand.TerminalSize, timeout time.Duration) error
}

// ServeExec handles requests to execute a command in a container. After
// creating/receiving the required streams, it delegates the actual execution
// to the executor.
func ServeExec(w http.ResponseWriter, req *http.Request, executor Executor, podName string, uid types.UID, container string, cmd []string, streamOpts *Options, idleTimeout, streamCreationTimeout time.Duration, supportedProtocols []string) {
   ctx, ok := createStreams(req, w, streamOpts, supportedProtocols, idleTimeout, streamCreationTimeout)
   if !ok {
      // error is handled by createStreams
      return
   }
   defer ctx.conn.Close()

   err := executor.ExecInContainer(podName, uid, container, cmd, ctx.stdinStream, ctx.stdoutStream, ctx.stderrStream, ctx.tty, ctx.resizeChan, 0)
   if err != nil {
      if exitErr, ok := err.(utilexec.ExitError); ok && exitErr.Exited() {
         rc := exitErr.ExitStatus()
         ctx.writeStatus(&apierrors.StatusError{ErrStatus: metav1.Status{
            Status: metav1.StatusFailure,
            Reason: remotecommandconsts.NonZeroExitCodeReason,
            Details: &metav1.StatusDetails{
               Causes: []metav1.StatusCause{
                  {
                     Type:    remotecommandconsts.ExitCodeCauseType,
                     Message: fmt.Sprintf("%d", rc),
                  },
               },
            },
            Message: fmt.Sprintf("command terminated with non-zero exit code: %v", exitErr),
         }})
      } else {
         err = fmt.Errorf("error executing command in container: %v", err)
         runtime.HandleError(err)
         ctx.writeStatus(apierrors.NewInternalError(err))
      }
   } else {
      ctx.writeStatus(&apierrors.StatusError{ErrStatus: metav1.Status{
         Status: metav1.StatusSuccess,
      }})
   }
}
```

#### exec的实现主要还得依赖 CRI

封装了容器运行时的接口

```go
// criAdapter wraps the Runtime functions to conform to the remotecommand interfaces.
// The adapter binds the container ID to the container name argument, and the pod sandbox ID to the pod name.
type criAdapter struct {
  //具体看实际的运行时
   Runtime
}

var _ remotecommandserver.Executor = &criAdapter{}
var _ remotecommandserver.Attacher = &criAdapter{}
var _ portforward.PortForwarder = &criAdapter{}

func (a *criAdapter) ExecInContainer(podName string, podUID types.UID, container string, cmd []string, in io.Reader, out, err io.WriteCloser, tty bool, resize <-chan remotecommand.TerminalSize, timeout time.Duration) error {
   return a.Runtime.Exec(container, cmd, in, out, err, tty, resize)
}

func (a *criAdapter) AttachContainer(podName string, podUID types.UID, container string, in io.Reader, out, err io.WriteCloser, tty bool, resize <-chan remotecommand.TerminalSize) error {
   return a.Runtime.Attach(container, in, out, err, tty, resize)
}

func (a *criAdapter) PortForward(podName string, podUID types.UID, port int32, stream io.ReadWriteCloser) error {
   return a.Runtime.PortForward(podName, port, stream)
}
```

#### Attach：如何将一个运行的容器添加到一个pod

```go
// Attacher knows how to attach to a running container in a pod.
type Attacher interface {
   // AttachContainer attaches to the running container in the pod, copying data between in/out/err
   // and the container's stdin/stdout/stderr.
   AttachContainer(name string, uid types.UID, container string, in io.Reader, out, err io.WriteCloser, tty bool, resize <-chan remotecommand.TerminalSize) error
}

// ServeAttach handles requests to attach to a container. After creating/receiving the required
// streams, it delegates the actual attaching to attacher.
func ServeAttach(w http.ResponseWriter, req *http.Request, attacher Attacher, podName string, uid types.UID, container string, streamOpts *Options, idleTimeout, streamCreationTimeout time.Duration, supportedProtocols []string) {
   ctx, ok := createStreams(req, w, streamOpts, supportedProtocols, idleTimeout, streamCreationTimeout)
   if !ok {
      // error is handled by createStreams
      return
   }
   defer ctx.conn.Close()

   err := attacher.AttachContainer(podName, uid, container, ctx.stdinStream, ctx.stdoutStream, ctx.stderrStream, ctx.tty, ctx.resizeChan)
   if err != nil {
      err = fmt.Errorf("error attaching to container: %v", err)
      runtime.HandleError(err)
      ctx.writeStatus(apierrors.NewInternalError(err))
   } else {
      ctx.writeStatus(&apierrors.StatusError{ErrStatus: metav1.Status{
         Status: metav1.StatusSuccess,
      }})
   }
}
```

#### HttpStream 双向处理

```go
// Stream represents a bidirectional communications channel that is part of an
// upgraded connection.
type Stream interface {
   io.ReadWriteCloser
   // Reset closes both directions of the stream, indicating that neither client
   // or server can use it any more.
   Reset() error
   // Headers returns the headers used to create the stream.
   Headers() http.Header
   // Identifier returns the stream's ID.
   Identifier() uint32
}
```

