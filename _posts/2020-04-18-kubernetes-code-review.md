---
title: Kubernetes源码分析计划
subtitle: 计划
layout: post
tags: [kubernetes]
---





为保证有序推进，代码的分析进度。特此作出以下计划，避免混乱。



任务：

- [ ]  apiserver实现

- [ ] scheduler实现

- [ ] watch-list informer的实现

- [ ] kubelet的实现

- [ ] controller-manager的实现

- [ ] kube-proxy的实现

## 2020年第16周

- [ ]  apiserver实现

- [ ] scheduler实现

- [ ] watch-list informer的实现

### 目标

上面的代码通读一遍，掌握运行的流程；

根据基本的流程，写一个demo示例。模仿他们的实现。相关的设计模式、并发编程等技术的学习。



### api-server



首先，看一下Server的相关配置信息。



```go
// ServerRunOptions runs a kubernetes api server.
type ServerRunOptions struct {
  //通用服务运行配置
   GenericServerRunOptions *genericoptions.ServerRunOptions
  
   Etcd                    *genericoptions.EtcdOptions
   SecureServing           *genericoptions.SecureServingOptionsWithLoopback
   InsecureServing         *genericoptions.DeprecatedInsecureServingOptionsWithLoopback
   Audit                   *genericoptions.AuditOptions
   Features                *genericoptions.FeatureOptions
   Admission               *kubeoptions.AdmissionOptions
   Authentication          *kubeoptions.BuiltInAuthenticationOptions
   Authorization           *kubeoptions.BuiltInAuthorizationOptions
   CloudProvider           *kubeoptions.CloudProviderOptions
   APIEnablement           *genericoptions.APIEnablementOptions
   EgressSelector          *genericoptions.EgressSelectorOptions

   AllowPrivileged           bool
   EnableLogsHandler         bool
   EventTTL                  time.Duration
  //Kubelet配置文件
   KubeletConfig             kubeletclient.KubeletClientConfig
   KubernetesServiceNodePort int
   MaxConnectionBytesPerSec  int64
   // ServiceClusterIPRange is mapped to input provided by user
   ServiceClusterIPRanges string
  
   //PrimaryServiceClusterIPRange and SecondaryServiceClusterIPRange are the results
   // of parsing ServiceClusterIPRange into actual values
   PrimaryServiceClusterIPRange   net.IPNet
   SecondaryServiceClusterIPRange net.IPNet

  //用户自定Srevice 的节点端口区间
   ServiceNodePortRange utilnet.PortRange
  //SSHKeyFile 文件
   SSHKeyfile           string
   SSHUser              string

  //代理Client的授权证书文件
   ProxyClientCertFile string
  //代理客户端的Key文件
   ProxyClientKeyFile  string

   EnableAggregatorRouting bool

   MasterCount            int
   EndpointReconcilerType string

   ServiceAccountSigningKeyFile     string
   ServiceAccountIssuer             serviceaccount.TokenGenerator
   ServiceAccountTokenMaxExpiration time.Duration

   ShowHiddenMetricsForVersion string
}
```



重点看一下，下面几个配置项：

- GenericServerRunOptions：*genericoptions.ServerRunOptions

- KubeletConfig ：kubeletclient.KubeletClientConfig

- Features ：*genericoptions.FeatureOptions

- ```go
  GenericAdmission *genericoptions.AdmissionOptions
  ```

#### 1. genericoptions.ServerRunOptions

```go
// ServerRunOptions contains the options while running a generic api server.
type ServerRunOptions struct {
   AdvertiseAddress net.IP
	 //允许跨域的配置
   CorsAllowedOriginList       []string
  //外部Host
   ExternalHost                string
   MaxRequestsInFlight         int
   MaxMutatingRequestsInFlight int
   RequestTimeout              time.Duration
   LivezGracePeriod            time.Duration
  //最小请求超时时间
   MinRequestTimeout           int
   ShutdownDelayDuration       time.Duration
   // We intentionally did not add a flag for this option. Users of the
   // apiserver library can wire it to a flag.
   JSONPatchMaxCopyBytes int64
   // The limit on the request body size that would be accepted and
   // decoded in a write request. 0 means no limit.
   // We intentionally did not add a flag for this option. Users of the
   // apiserver library can wire it to a flag.
   MaxRequestBodyBytes        int64
   TargetRAMMB                int
   EnableInflightQuotaHandler bool
}
```



#### 2. KubeletConfig ：kubeletclient.KubeletClientConfig

kubelet相关的配置。

```go
// KubeletClientConfig defines config parameters for the kubelet client
type KubeletClientConfig struct {
   // Port specifies the default port - used if no information about Kubelet port can be found in Node.NodeStatus.DaemonEndpoints.
   Port uint

   // ReadOnlyPort specifies the Port for ReadOnly communications.
   ReadOnlyPort uint

   // EnableHTTPs specifies if traffic should be encrypted.
   EnableHTTPS bool

   // PreferredAddressTypes - used to select an address from Node.NodeStatus.Addresses
   PreferredAddressTypes []string

   // TLSClientConfig contains settings to enable transport layer security
   restclient.TLSClientConfig

   // Server requires Bearer authentication。最重要的Token，很好用。针对不同的用户搞一个
   BearerToken string

   // HTTPTimeout is used by the client to timeout http requests to Kubelet.
   HTTPTimeout time.Duration

   // Dial is a custom dialer used for the client  客户端使用的自定义拨号器
   Dial utilnet.DialFunc

   // Lookup will give us a dialer if the egress selector is configured for it
   Lookup egressselector.Lookup
}
```



这里面比较重要的：

- BearerToken：http权限校验的token
- Dial：自定义拨号器
- Lookup：如果配置了外出selector的Dialer拨号器。



仔细研究一下这个Dial的拨号器到底是如何实现的呢？看看？

本质上是对网络的封装，配置成net.Conn。比如，将http的请求封装一下。

```go
type RoundTripperWrapper interface {
   http.RoundTripper
   WrappedRoundTripper() http.RoundTripper
}

type DialFunc func(ctx context.Context, net, addr string) (net.Conn, error)

func DialerFor(transport http.RoundTripper) (DialFunc, error) {
   if transport == nil {
      return nil, nil
   }

   switch transport := transport.(type) {
   case *http.Transport:
      // transport.DialContext takes precedence over transport.Dial
      if transport.DialContext != nil {
         return transport.DialContext, nil
      }
      // adapt transport.Dial to the DialWithContext signature
      if transport.Dial != nil {
         return func(ctx context.Context, net, addr string) (net.Conn, error) {
            return transport.Dial(net, addr)
         }, nil
      }
      // otherwise return nil
      return nil, nil
   case RoundTripperWrapper:
      return DialerFor(transport.WrappedRoundTripper())
   default:
      return nil, fmt.Errorf("unknown transport type: %T", transport)
   }
}
```





```go
// DialerFunc implements Dialer for the provided function.
type DialerFunc func(req *http.Request) (net.Conn, error)

func (fn DialerFunc) Dial(req *http.Request) (net.Conn, error) {
   return fn(req)
}

// Dialer dials a host and writes a request to it.
type Dialer interface {
   // Dial connects to the host specified by req's URL, writes the request to the connection, and
   // returns the opened net.Conn.
   Dial(req *http.Request) (net.Conn, error)
}
```



#### 3. Features ：*genericoptions.FeatureOptions

```go
type FeatureOptions struct {
   EnableProfiling           bool
   EnableContentionProfiling bool
}
```

#### 4. GenericAdmission *genericoptions.AdmissionOptions

初始化准入控制器相关插件的配置

```go
// AdmissionOptions holds the admission准入配置 options
type AdmissionOptions struct {
   // RecommendedPluginOrder holds an ordered list of plugin names we recommend to use by default
   RecommendedPluginOrder []string
   // DefaultOffPlugins is a set of plugin names that is disabled by default
   DefaultOffPlugins sets.String

   // EnablePlugins indicates plugins to be enabled passed through `--enable-admission-plugins`.
   EnablePlugins []string
   // DisablePlugins indicates plugins to be disabled passed through `--disable-admission-plugins`.
   DisablePlugins []string
   // ConfigFile is the file path with admission control configuration.
   ConfigFile string
   // Plugins contains all registered plugins.
  //Plugins 包含了所有注册的插件
   Plugins *admission.Plugins
   // Decorators is a list of admission decorator to wrap around the admission plugins
   Decorators admission.Decorators
}
```





##### 1. admission.Plugins

```go
// Factory is a function that returns an Interface for admission decisions.
// The config parameter provides an io.Reader handler to the factory in
// order to load specific configurations. If no configuration is provided
// the parameter is nil.
type Factory func(config io.Reader) (Interface, error)

//注册的所有插件信息
type Plugins struct {
   lock     sync.Mutex
   registry map[string]Factory
}
```

##### 2. Interface：an abstract, pluggable interface for Admission Control decisions

准入控制决策接口（抽象化、插件化）

```go
// Interface is an abstract, pluggable interface for Admission Control decisions.
type Interface interface {
   // Handles returns true if this admission controller（权限准入控制器） can handle the given operation
   // where operation can be one of CREATE, UPDATE, DELETE, or CONNECT
  //如果admission权限准入控制器对相应的操作处理之后，允许之后返回true
   Handles(operation Operation) bool
}
```

##### 3. Decorator装饰器

```go
type Decorator interface {
	Decorate(handler Interface, name string) Interface
}

type DecoratorFunc func(handler Interface, name string) Interface

func (d DecoratorFunc) Decorate(handler Interface, name string) Interface {
	return d(handler, name)
}

type Decorators []Decorator

// Decorate applies the decorator in inside-out order, i.e. the first decorator in the slice is first applied to the given handler.
func (d Decorators) Decorate(handler Interface, name string) Interface {
	result := handler
	for _, d := range d {
		result = d.Decorate(result, name)
	}

	return result
}
```



### 背景知识

#### clientset

```go

type Interface interface {
	Discovery() discovery.DiscoveryInterface
	AdmissionregistrationV1() admissionregistrationv1.AdmissionregistrationV1Interface
	AdmissionregistrationV1beta1() admissionregistrationv1beta1.AdmissionregistrationV1beta1Interface
	AppsV1() appsv1.AppsV1Interface
	AppsV1beta1() appsv1beta1.AppsV1beta1Interface
	AppsV1beta2() appsv1beta2.AppsV1beta2Interface
	AuditregistrationV1alpha1() auditregistrationv1alpha1.AuditregistrationV1alpha1Interface
	AuthenticationV1() authenticationv1.AuthenticationV1Interface
	AuthenticationV1beta1() authenticationv1beta1.AuthenticationV1beta1Interface
	AuthorizationV1() authorizationv1.AuthorizationV1Interface
	AuthorizationV1beta1() authorizationv1beta1.AuthorizationV1beta1Interface
	AutoscalingV1() autoscalingv1.AutoscalingV1Interface
	AutoscalingV2beta1() autoscalingv2beta1.AutoscalingV2beta1Interface
	AutoscalingV2beta2() autoscalingv2beta2.AutoscalingV2beta2Interface
	BatchV1() batchv1.BatchV1Interface
	BatchV1beta1() batchv1beta1.BatchV1beta1Interface
	BatchV2alpha1() batchv2alpha1.BatchV2alpha1Interface
	CertificatesV1beta1() certificatesv1beta1.CertificatesV1beta1Interface
	CoordinationV1beta1() coordinationv1beta1.CoordinationV1beta1Interface
	CoordinationV1() coordinationv1.CoordinationV1Interface
	CoreV1() corev1.CoreV1Interface
	DiscoveryV1alpha1() discoveryv1alpha1.DiscoveryV1alpha1Interface
	DiscoveryV1beta1() discoveryv1beta1.DiscoveryV1beta1Interface
	EventsV1beta1() eventsv1beta1.EventsV1beta1Interface
	ExtensionsV1beta1() extensionsv1beta1.ExtensionsV1beta1Interface
	FlowcontrolV1alpha1() flowcontrolv1alpha1.FlowcontrolV1alpha1Interface
	NetworkingV1() networkingv1.NetworkingV1Interface
	NetworkingV1beta1() networkingv1beta1.NetworkingV1beta1Interface
	NodeV1alpha1() nodev1alpha1.NodeV1alpha1Interface
	NodeV1beta1() nodev1beta1.NodeV1beta1Interface
	PolicyV1beta1() policyv1beta1.PolicyV1beta1Interface
	RbacV1() rbacv1.RbacV1Interface
	RbacV1beta1() rbacv1beta1.RbacV1beta1Interface
	RbacV1alpha1() rbacv1alpha1.RbacV1alpha1Interface
	SchedulingV1alpha1() schedulingv1alpha1.SchedulingV1alpha1Interface
	SchedulingV1beta1() schedulingv1beta1.SchedulingV1beta1Interface
	SchedulingV1() schedulingv1.SchedulingV1Interface
	SettingsV1alpha1() settingsv1alpha1.SettingsV1alpha1Interface
	StorageV1beta1() storagev1beta1.StorageV1beta1Interface
	StorageV1() storagev1.StorageV1Interface
	StorageV1alpha1() storagev1alpha1.StorageV1alpha1Interface
}

// Clientset contains the clients for groups. Each group has exactly one
// version included in a Clientset.
type Clientset struct {
	*discovery.DiscoveryClient
	admissionregistrationV1      *admissionregistrationv1.AdmissionregistrationV1Client
	admissionregistrationV1beta1 *admissionregistrationv1beta1.AdmissionregistrationV1beta1Client
	appsV1                       *appsv1.AppsV1Client
	appsV1beta1                  *appsv1beta1.AppsV1beta1Client
	appsV1beta2                  *appsv1beta2.AppsV1beta2Client
	auditregistrationV1alpha1    *auditregistrationv1alpha1.AuditregistrationV1alpha1Client
	authenticationV1             *authenticationv1.AuthenticationV1Client
	authenticationV1beta1        *authenticationv1beta1.AuthenticationV1beta1Client
	authorizationV1              *authorizationv1.AuthorizationV1Client
	authorizationV1beta1         *authorizationv1beta1.AuthorizationV1beta1Client
	autoscalingV1                *autoscalingv1.AutoscalingV1Client
	autoscalingV2beta1           *autoscalingv2beta1.AutoscalingV2beta1Client
	autoscalingV2beta2           *autoscalingv2beta2.AutoscalingV2beta2Client
	batchV1                      *batchv1.BatchV1Client
	batchV1beta1                 *batchv1beta1.BatchV1beta1Client
	batchV2alpha1                *batchv2alpha1.BatchV2alpha1Client
	certificatesV1beta1          *certificatesv1beta1.CertificatesV1beta1Client
	coordinationV1beta1          *coordinationv1beta1.CoordinationV1beta1Client
	coordinationV1               *coordinationv1.CoordinationV1Client
	coreV1                       *corev1.CoreV1Client
	discoveryV1alpha1            *discoveryv1alpha1.DiscoveryV1alpha1Client
	discoveryV1beta1             *discoveryv1beta1.DiscoveryV1beta1Client
	eventsV1beta1                *eventsv1beta1.EventsV1beta1Client
	extensionsV1beta1            *extensionsv1beta1.ExtensionsV1beta1Client
	flowcontrolV1alpha1          *flowcontrolv1alpha1.FlowcontrolV1alpha1Client
	networkingV1                 *networkingv1.NetworkingV1Client
	networkingV1beta1            *networkingv1beta1.NetworkingV1beta1Client
	nodeV1alpha1                 *nodev1alpha1.NodeV1alpha1Client
	nodeV1beta1                  *nodev1beta1.NodeV1beta1Client
	policyV1beta1                *policyv1beta1.PolicyV1beta1Client
	rbacV1                       *rbacv1.RbacV1Client
	rbacV1beta1                  *rbacv1beta1.RbacV1beta1Client
	rbacV1alpha1                 *rbacv1alpha1.RbacV1alpha1Client
	schedulingV1alpha1           *schedulingv1alpha1.SchedulingV1alpha1Client
	schedulingV1beta1            *schedulingv1beta1.SchedulingV1beta1Client
	schedulingV1                 *schedulingv1.SchedulingV1Client
	settingsV1alpha1             *settingsv1alpha1.SettingsV1alpha1Client
	storageV1beta1               *storagev1beta1.StorageV1beta1Client
	storageV1                    *storagev1.StorageV1Client
	storageV1alpha1              *storagev1alpha1.StorageV1alpha1Client
}
```



### 详细的实现细节

#### 1. 创建流程





#### 2. 代码实现的技巧

- 设计模式
- 并发实现方案



### Demo模仿实现





