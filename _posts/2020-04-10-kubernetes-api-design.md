# kubernetes中的api多版本机制实现

在web开发中随着版本的更新迭代,通常要在系统中维护多个版本的api,多个版本的api在数据结构上往往也各不相同,今天就来一起学习下kubernetes中的Scheme机制是如何解决这个问题的,如何借助HTTP请求里面的数据进行反序列化操作



# 1. web请求的处理流程



## 1.1 HTTP请求处理流程



![img](https://cdn.nlark.com/yuque/0/2020/png/97498/1583127480396-dd2011dd-37f8-4e25-9f38-911f54255dda.png?x-oss-process=image/resize,w_1500/watermark,type_d3F5LW1pY3JvaGVp,size_14,text_YmF4aWFvc2hpMjAyMA==,color_FFFFFF,shadow_50,t_80,g_se,x_10,y_10)

通常首先是webServer先进行Http协议的处理,然后解析成基础的webServer内部的一个Http请求对象, 通常该对象持有对应请求的请求头和底层对应的字节序列(从socket流中读取)

接着首先会通常根据对应的编码格式来进行反序列化，完成从字节序列到当前接口的业务模型的映射, 然后在交给业务逻辑处理，从而最终进行持久化存储, 本文的重点也就在**反序列化部分**



# 2.模型映射的实现



## 2.1 描述资源版本信息



```
/api/{version}/{resource}/{action}
```



上面是一个基础的web url通常我们都会为每个版本注册一个对应的URL, 其中会包含很关键的两个信息即version与resource,通过这两个信息,通常我们就可以知道这可能是某个资源的那个版本, 如果我们把后面的action也包裹进来,我们通常就可以知道对应的资源的那个具体操作



## 2.2 Group组信息



![img](https://cdn.nlark.com/yuque/0/2020/png/97498/1583127480428-c6253558-ecdf-4c6d-9374-eff27c72f431.png?x-oss-process=image/watermark,type_d3F5LW1pY3JvaGVp,size_14,text_YmF4aWFvc2hpMjAyMA==,color_FFFFFF,shadow_50,t_80,g_se,x_10,y_10)

在微服务流行的今天我们通常会为按照业务功能来进行微服务的切分，本质上一个微服务可能就是实现某个具体业务场景的功能集合，比如用户系统通常会包含用户的所有相关操作，在kubernetes中也有类似的概念就是所谓的Group



```
POST /apis/batch/v1beta1/namespaces/{namespace}/cronjobs
POST /apis/apps/v1/namespaces/{namespace}/daemonsets
```



我们来看一个实例这是一个创建daemonsets和cronjobs的url, 如果按照Group、resource、version来进行拆分可以拆成如下：batch、v1beta1、cronjobs和apps、v1、daemonsets，也就是大家尝试的GroupVersionKind,其中kind对应的就是resource



## 2.3 模型映射的实现



![img](https://cdn.nlark.com/yuque/0/2020/png/97498/1583127480454-2197c7fe-09de-4854-aa66-fc60d6a8ea0b.png)

我们通过url里面获取到资源的GroupVersionKind信息，如何将其映射为一个具体的类型呢？ 这貌似就很简单了结合反射和map来进行就可以了，我们通过url获取到对应想的GVK信息，然后在通过我们的映射表，就知道对应的模型是哪个，接下来就只需要进行转换就行了



```
gvkToType map[schema.GroupVersionKind]reflect.Type
```



# 3.反序列化实现



## 3.1 解码机制



那如何将对应的Http里面的数据流反序列化成内部的一个对象呢，别忘记了是Http协议, 肯定就是header头里面的信息了，我们通过header头里面的序列化就可以知道对应的编码格式，只需要调用对应格式的解码就可以完成了



```
Content-Type: "application/json"
```



## 3.2 默认对象



![img](https://cdn.nlark.com/yuque/0/2020/png/97498/1583127480395-07cbe03c-6627-41e6-9e85-cf935ee33c58.png?x-oss-process=image/watermark,type_d3F5LW1pY3JvaGVp,size_10,text_YmF4aWFvc2hpMjAyMA==,color_FFFFFF,shadow_50,t_80,g_se,x_10,y_10)

如果要将json格式的字节数组进行解码通常要进行如下操作，我们需要传入一个目标对象的指针，然后由json将对应的字节数据解析到目标对象中，我们也需要这样一个对象，用于存储反序列化的结果



```
func Unmarshal(data []byte, v interface{}) error {}
```



那只要我再提供一个当前版本对应的对象构造函数是不是就可以呢？答案是的



```
func() Object{ return 目标对象 },
```



# 4. 设计总结



![img](https://cdn.nlark.com/yuque/0/2020/png/97498/1583127480445-d11705c4-9ec8-4db8-946f-590673acb499.png?x-oss-process=image/resize,w_1500/watermark,type_d3F5LW1pY3JvaGVp,size_14,text_YmF4aWFvc2hpMjAyMA==,color_FFFFFF,shadow_50,t_80,g_se,x_10,y_10)

首先在进行url注册的时候，我们构造出对应url映射的资源的版本信息即GroupVersionKind,后续的很多操作我们可以通过对应的版本映射获取对应的目标操作或者对象,然后再通过Header里面的字段获取对应的解码器,并将Body里面的字节序列进行解码到目标对象，就可以实现多版本资源的映射和反序列化操作了

# api多版本中反序列化与转换

在之前的文章中分析过kubernetes是如何进行多版本管理中提到了一个关键的设计解码器, 负责将请求对象反序列化成一个具体的数据模型，今天一起来了解下其内部是如何实现多版本管理、转换的具体实现



# 1.版本化管理的关键设计



## 1.1 从拓扑转换到星状转换

在通常的web开发中更多的时候，大家都是断代向前兼容更新，大多数情况下当版本更新之后会独立演进，如果要在多版本之间转换通常则会出现如下的情况

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583209872005-93adbec1-431d-4f43-b00d-5b7b61343ee9.png)

如果我们要为每个版本都去适配其他所有的版本，则复杂度会指数级上升，而在kubernetes中则通过一个内部版本的设计来进行解决，内部版本是一个稳定的版本，所有的版本都只针对目标版本来进行转换的实现，而不关注其他版本



## 1.2 兼容设计之转换

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583210000899-992fa413-be3e-4efe-82d4-f0922335bafa.png)

那如果谋个版本需要独立的演进，或者增设一些新的字段，修改字段名称等破坏性更新的时候，则就需要一种转换机制，负责在当前版本和内部版本之间来进行字段或者数据的转换



## 1.3 转换的最终之反射

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583210217577-f866bea0-88bf-4681-800b-f9036b5329d4.png)

转换其实核心目标是完成从目标对象的字段中获取数据，然后经过一系列操作最终为目标对象的对应的字段进行赋值操作，要完成该操作，则就需要借助反射来实现，通过枚举字段，来获取对应的转换函数，执行转换函数，完成目标赋值



# 2. 关键设计的实现

为了实现上面的方案，kubernetes中设计了如下组件：Scheme(负责各个版本的注册和管理)、Convert(转换实现)、Serializer(实现对应版本的反序列化)，让我们依次看下其关键设计



## 2.1 Convert

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583210380283-6e538354-a5fd-40e0-8216-41af350eb3eb.png?x-oss-process=image/watermark,type_d3F5LW1pY3JvaGVp,size_10,text_YmF4aWFvc2hpMjAyMA==,color_FFFFFF,shadow_50,t_80,g_se,x_10,y_10)



Convert实现从一个目标对象到另外一个目标对象的转换, 为了实现这种转换，kubernetes里面主要是借助反射和不同版本的转换函数来共同完成



1.首先我们通过目标函数来获取对应的属性字段，然后针对该字段进行计算赋值操作

2.如果发现对应的字段需要来转换，则会调用对应的转换函数来进行赋值操作，如果不需要转换则会直接通过反射来进行赋值



## 2.2 外部版本到内部版本的转换

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583210632919-e5d25e2f-5028-4112-99d8-cca024834a05.png)

在构建rest接口的时候，每个rest接口都会持有一个生成当前版本对象的构造函数，当请求进入之后，会首先通过目标版本获取对应的decoder

decoder会利用当前的GroupVersionKind来进行第一步解析，首先将字节数组解析成当前版本，然后在解析成目标对象之后，又会根据目标版本进行转换



## 2.3 Scheme

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583213563730-17fa946e-beed-48ec-b247-98d9986054e5.png?x-oss-process=image/watermark,type_d3F5LW1pY3JvaGVp,size_14,text_YmF4aWFvc2hpMjAyMA==,color_FFFFFF,shadow_50,t_80,g_se,x_10,y_10)

Scheme负责各个资源版本的统一注册和管理，为其他组件提供根据GVK来获取对应的资源对象，也提供通过资源对象获取版本等操作，内部还持有convert对象，包装了源对象到目标对象的转换操作



Scheme对象是一个复合的数据结构，其实现了多种结果，诸如typer、defaulter、creater等,很多地方都是通过直接传递scheme来进行对应的参数的填充, 其内部关键数据结构如下



GVK与资源类型的映射，以及当前资源类型支持哪些GVK

```
    gvkToType map[schema.GroupVersionKind]reflect.Type

    typeToGVK map[reflect.Type][]schema.GroupVersionKind
```

则创建对象的时候可以直接通过New来实例化对应的对象



```
func (s *Scheme) New(kind schema.GroupVersionKind) (Object, error) {
    if t, exists := s.gvkToType[kind]; exists {
        // 利用反射来创建对象
        return reflect.New(t).Interface().(Object), nil
    }
    // 省略相关代码
}
```

默认初始化函数



```
    defaulterFuncs map[reflect.Type]func(interface{})
```

根据对应的类型来完成初始化操作



```
func (s *Scheme) Default(src Object) {
    if fn, ok := s.defaulterFuncs[reflect.TypeOf(src)]; ok {
        fn(src)
    }
}
```

提供转换函数注册接口, 注册到convert中

```
func (s *Scheme) AddConversionFunc(a, b interface{}, fn conversion.ConversionFunc) error {
    return s.converter.RegisterUntypedConversionFunc(a, b, fn)
}
```



# 3.学习总结

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583214613431-1b432b68-39e8-432f-8829-2288685fcf64.png?x-oss-process=image/watermark,type_d3F5LW1pY3JvaGVp,size_14,text_YmF4aWFvc2hpMjAyMA==,color_FFFFFF,shadow_50,t_80,g_se,x_10,y_10)

实现上无疑是复杂的作为一个工业设计有很多需要care的边缘情况，剖丝抽茧每个人看到的都不一样，这可能就是源码阅读的乐趣，从上面可以看到核心其实就三个点：将HTTP数据反序列化成为当前URL的资源对象，然后目标资源对象进行初始化默认值函数的执行，最后通过convert来处理不同版本之间的差异，最后统一操作内部版本，好了今天就到这了，希望对大家有所帮助



# api聚合机制实现原理

kubernetes中apiserver的设计无疑是复杂的,其自身内部就包含了三种角色的api服务,今天我们一起来臆测下其内部的设计,搞明白aggregator、apiserver、apiExtensionsServer(crd server)的设计精要

# 1.从web服务到web网关到CRD

apiserver还是蛮复杂的，今天我们只讨论其kube-aggregator/apiserver/apiextensions三者架构上的设计，而不关注诸如请求认证、准入控制、权限等等



## 1.1 最基础的REST服务

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583308414860-6f9878e0-eac7-4862-b572-5a908204ebe5.png)

一个最基础的Rest服务通常会包括一个resource资源和一组HTTP请求的方法, 在kubernetes中被称为一个REST，其内部还内嵌了一个Store(可以理解为继承)，其提供了针对某个具体资源的所有操作的集合，也就是我们常说的最终执行CRUD的具体操作的实现



## 1.2 Service

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583308438829-92598adb-4089-49be-8584-13d64f985743.png)

我们有了Rest就可以提供各种k8s中资源的管理，但是如果我要进行扩展呢，如果要支持一些外部的资源k8s中不存在， 最简单的方式肯定就是在外部独立一个服务了，由这个服务自己管理数据存储、变更、控制等等逻辑



## 1.3 APIAggregator



当通过外部服务来进行集群资源扩展的时候，针对这类资源我们如何集成到当前的apiserver中呢？为此k8s中设计了APIAggregator组件(其实APIAggreator组件还包括代理后端服务等功能)，来实现外部服务的集成，这样开发人员不用修改k8s代码，也可以来自定义服务信息



## 1.4 一个服务的基本功能



![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583308592572-b978209b-8526-4507-8eec-5c1d47405634.png)

一个基础的业务服务通常包含数据模型、控制逻辑、持久化存储、基础功能(认证、监控、日志等等)等等，为了要创建一个服务，我们通常需要如下操作(不包含设计阶段)：1)选择合适的框架(完成基础功能) 2)定义数据模型 3)选择数据存储 4)编写业务控制逻辑， 这里面除了业务控制逻辑，其余部分在大多数情况下可能都是通用，比如框架、数据存储这些，那能不能简化下？来看大招CRD



## 1.5 CustomResourceDefinitions

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583308693752-9d159c00-c358-48d6-8af9-2190d3484fc4.png)

CRD中文被称为自定义资源类型，其核心在k8s中提供数据模型定义、数据存储、基础功能，这样如果我们要扩展服务就只需要编写一个业务逻辑控制器即可， 我们思考下其场景



通常web请求的处理流程都是反序列化、验证字段、业务逻辑处理、数据存储，而在k8s中业务控制逻辑大多数由controller来进行，那为了支持CRD剩余工作肯定也是在k8s中完成的



在我们完成定义模型之后，k8s的crd模块需要进行对应资源REST的构建、验证、转换、存储等操作这些无疑都是耗费资源的，而且在apiserver这种数据总线上，由此可以发行CRD并不支持大规模的数据存储 



## 1.6 CRD server

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583308821457-f4d0175f-1106-4f63-964b-aa50beec7874.png)

CRDServer主要就是负责CRD资源的管理，其会监听CRD资源的变更，并且为其创建对应的REST接口，完成对应的认证、转换、验证、存储等刀座



# 2. ServerChan

ServerChan从设计上更类似一种责任链的模式，简单来说如果我处理不了该请求，我就交给下个人处理，这种操作在k8s中被称为delegate(委托),接下来我们开始了解其关键实现



## 2.1 服务的角色划分



到目前我们已经有了三个server, 其中APIAggregator负责外部服务的集成和内部请求的转发，apiserver服务k8s汇总内部资源的控制，CRDServer则负责用户自定义资源的处理，然后我们就只需要将三者串联起来，就是我们最终的apiserver



## 2.2 责任链上的层层委托

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583308930526-6967fe76-09f8-4758-8ac1-bf7bdf8a3235.png)

当APIAggregator接收到请求之后，如果发现对应的是一个service的请求，则会直接转发到对应的服务上否则则委托给apiserver进行处理，apiserver中根据当前URL来选择对应的REST接口处理，如果未能找到对应的处理，则会交由CRD server处理， CRD server检测是否已经注册对应的CRD资源，如果注册就处理



## 2.3 APIAggregator上的服务注册

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583309006602-d52d9393-c5f8-4cde-bea9-38b7aa39cc05.png)

APIAggreagtor中会通过informer 监听后端Service的变化，如果发现有新的服务，就会创建对应的代理转发，从而实现对应的服务注册



## 2.4 CRD Server中的资源感知

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583309075683-c53304b6-dda8-4738-87fe-8009f90b97a1.png)

当在集群中创建了对应的CRD资源的时候，会通过内部的controller来感知对应的CRD资源信息，然后为其创建对应的REST处理接口，这样后续接收到对应的资源就可以进行处理了 





# 3. 基础概览图

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583309404523-d5452bc2-8acc-4c06-9af6-96d63b18ebaf.png?x-oss-process=image/watermark,type_d3F5LW1pY3JvaGVp,size_14,text_YmF4aWFvc2hpMjAyMA==,color_FFFFFF,shadow_50,t_80,g_se,x_10,y_10)

读k8s代码总是这样迷迷糊糊中又能灵光一现直接柳暗花明，流程真的重要嘛，貌似也不是很重要，了解其上层设计，然后直接关注感兴趣点即可，除非和我一样就是感兴趣，那就只能死磕了，我该去吃饭了，祝你好运





# 数据存储etcd增删改查操作

kubernetes中基于etcd实现集中的数据存储,今天来臆测基于etcd如何实现数据读取一致性、更新一致性、事务的具体实现



# 1. 数据的存储与版本



## 1.1 数据存储的转换

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583385013335-66b77efd-b881-4adf-be08-d6a344eedd6b.png)

在k8s中有部分数据的存储是需要经过处理之后才能存储的，比如secret这种加密的数据，既然要存储就至少包含两个操作，加密存储，解密读取，transformer就是为了完成该操作而实现的，其在进行etcd数据存储的时候回对数据进行加密，而在读取的时候，则会进行解密



## 1.2 资源版本revision

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583385085438-ed644e6f-8c8b-4b4c-aede-52613c191a60.png)

在etcd中进行修改(增删改)操作的时候，都会递增revision，而在k8s中也通过该值来作为k8s资源的ResourceVersion,该机制也是实现watch的关键机制，在操作etcd解码从etcd获取的数据的时候，会通过versioner组件来为资源动态的修改该值



## 1.3 数据模型的映射

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583385187120-941dcef4-d51d-4d27-bcb1-0ace37472980.png?x-oss-process=image/watermark,type_d3F5LW1pY3JvaGVp,size_10,text_YmF4aWFvc2hpMjAyMA==,color_FFFFFF,shadow_50,t_80,g_se,x_10,y_10)

将数据从etcd中读取后，数据本身就是一个字节数组，如何将对应的数据转换成我们真正的运行时对象呢？还记得我们之前的scheme与codec么，在这里我们知道对应的数据编码格式，也知道资源对象的类型，则通过codec、字节数组、目标类型，我们就可以完成对应数据的反射



# 2. 查询接口一致性 

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583385339721-c288266a-42a6-4198-96e2-c99beffcabb8.png)

etcd中的数据写入是基于leader单点写入和集群quorum机制实现的，并不是一个强一致性的数据写入，则如果如果我们访问的节点不存在quorum的半数节点内，则可能造成短暂的数据不一致，针对一些强一致的场景，我们可以通过其revision机制来进行数据的读取, 保证我们读取到更新之后的数据

```
// 省略非核心代码
func (s *store) Get(ctx context.Context, key string, resourceVersion string, out runtime.Object, ignoreNotFound bool) error {
    // 获取key
    getResp, err := s.client.KV.Get(ctx, key, s.getOps...)

    // 检测当前版本，是否达到最小版本的
    if err = s.ensureMinimumResourceVersion(resourceVersion, uint64(getResp.Header.Revision)); err != nil {
        return err
    }

    // 执行数据转换
    data, _, err := s.transformer.TransformFromStorage(kv.Value, authenticatedDataString(key))
    if err != nil {
        return storage.NewInternalError(err.Error())
    }
    // 解码数据
    return decode(s.codec, s.versioner, data, out, kv.ModRevision)
}
```

# 3. 创建接口实现![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583385558145-f8e8cc59-b8a6-4706-befb-73989c2ebc3f.png)

创建一个接口数据则会首先进行资源对象的检查，避免重复创建对象，此时会先通过资源对象的version字段来进行初步检查，然后在利用etcd的事务机制来保证资源创建的原子性操作

```
// 省略非核心代码
func (s *store) Create(ctx context.Context, key string, obj, out runtime.Object, ttl uint64) error {
    if version, err := s.versioner.ObjectResourceVersion(obj); err == nil && version != 0 {
        return errors.New("resourceVersion should not be set on objects to be created")
    }
    if err := s.versioner.PrepareObjectForStorage(obj); err != nil {
        return fmt.Errorf("PrepareObjectForStorage failed: %v", err)
    }
    // 将数据编码
    data, err := runtime.Encode(s.codec, obj)
    if err != nil {
        return err
    }
    
    // 转换数据
    newData, err := s.transformer.TransformToStorage(data, authenticatedDataString(key))
    if err != nil {
        return storage.NewInternalError(err.Error())
    }

    startTime := time.Now()
    // 事务操作
    txnResp, err := s.client.KV.Txn(ctx).If(
        notFound(key), // 如果之前不存在 这里是利用的etcd的ModRevision即修改版本为0， 寓意着对应的key不存在
    ).Then(
        clientv3.OpPut(key, string(newData), opts...), // put修改数据
    ).Commit()
    metrics.RecordEtcdRequestLatency("create", getTypeName(obj), startTime)
    if err != nil {
        return err
    }
    if !txnResp.Succeeded {
        return storage.NewKeyExistsError(key, 0)
    }

    if out != nil {
        // 获取对应的Revision
        putResp := txnResp.Responses[0].GetResponsePut()
        return decode(s.codec, s.versioner, data, out, putResp.Header.Revision)
    }
    return nil
}

func notFound(key string) clientv3.Cmp {
    return clientv3.Compare(clientv3.ModRevision(key), "=", 0)
}
```

# 4. 删除接口的实现

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583386116901-3ecbce49-b1f5-41d5-ab33-a60e273b29af.png)

删除接口主要是通过CAS和事务机制来共同实现，确保在etcd不发生异常的情况，即使并发对同个资源来进行删除操作也能保证至少有一个节点成功

```
// 省略非核心代码
func (s *store) conditionalDelete(ctx context.Context, key string, out runtime.Object, v reflect.Value, preconditions *storage.Preconditions, validateDeletion storage.ValidateObjectFunc) error {
    startTime := time.Now()
    // 获取当前的key的数据
    getResp, err := s.client.KV.Get(ctx, key)
    for {
        // 获取当前的状态
        origState, err := s.getState(getResp, key, v, false)
        if err != nil {
            return err
        }
        txnResp, err := s.client.KV.Txn(ctx).If(
            clientv3.Compare(clientv3.ModRevision(key), "=", origState.rev), // 如果修改版本等于当前状态，就尝试删除
        ).Then(
            clientv3.OpDelete(key), // 删除
        ).Else(
            clientv3.OpGet(key),    // 获取
        ).Commit()
        if !txnResp.Succeeded {
            // 获取最新的数据重试事务操作
            getResp = (*clientv3.GetResponse)(txnResp.Responses[0].GetResponseRange())
            klog.V(4).Infof("deletion of %s failed because of a conflict, going to retry", key)
            continue
        }
        // 将最后一个版本的数据解码到out里面，然后返回
        return decode(s.codec, s.versioner, origState.data, out, origState.rev)
    }
}
```

# 5. 更新接口的实现

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583386286561-35f4435b-0925-46ff-9a40-fd0f07acfa67.png)

更新接口实现上与删除接口并无本质上的差别，但是如果多个节点同时进行更新，CAS并发操作必然会有一个节点成功，当发现已经有节点操作成功，则当前节点其实并不需要再做过多的操作，直接返回即可

```
// 省略非核心代码
func (s *store) GuaranteedUpdate(
    ctx context.Context, key string, out runtime.Object, ignoreNotFound bool,
    preconditions *storage.Preconditions, tryUpdate storage.UpdateFunc, suggestion ...runtime.Object) error {
    // 获取当前key的最新数据
    getCurrentState := func() (*objState, error) {
        startTime := time.Now()
        getResp, err := s.client.KV.Get(ctx, key, s.getOps...)
        metrics.RecordEtcdRequestLatency("get", getTypeName(out), startTime)
        if err != nil {
            return nil, err
        }
        return s.getState(getResp, key, v, ignoreNotFound)
    }

    // 获取当前数据
    var origState *objState
    var mustCheckData bool
    if len(suggestion) == 1 && suggestion[0] != nil {
        // 如果提供了建议的数据，则会使用，
        origState, err = s.getStateFromObject(suggestion[0])
        if err != nil {
            return err
        }
        //但是需要检测数据
        mustCheckData = true
    } else {
        // 尝试重新获取数据
        origState, err = getCurrentState()
        if err != nil {
            return err
        }
    }

    transformContext := authenticatedDataString(key)
    for {
        // 检查对象是否已经更新, 主要是通过检测uuid/revision来实现
        if err := preconditions.Check(key, origState.obj); err != nil {
            // If our data is already up to date, return the error
            if !mustCheckData {
                return err
            }
            // 如果检查数据一致性错误，则需要重新获取
            origState, err = getCurrentState()
            if err != nil {
                return err
            }
            mustCheckData = false
            // Retry
            continue
        }

        // 删除当前的版本数据revision
        ret, ttl, err := s.updateState(origState, tryUpdate)
        if err != nil {
            // If our data is already up to date, return the error
            if !mustCheckData {
                return err
            }

            // It's possible we were working with stale data
            // Actually fetch
            origState, err = getCurrentState()
            if err != nil {
                return err
            }
            mustCheckData = false
            // Retry
            continue
        }

        // 编码数据
        data, err := runtime.Encode(s.codec, ret)
        if err != nil {
            return err
        }
        if !origState.stale && bytes.Equal(data, origState.data) {
            // 如果我们发现我们当前的数据与获取到的数据一致，则会直接跳过
            if mustCheckData {
                origState, err = getCurrentState()
                if err != nil {
                    return err
                }
                mustCheckData = false
                if !bytes.Equal(data, origState.data) {
                    // original data changed, restart loop
                    continue
                }
            }
            if !origState.stale {
                // 直接返回数据
                return decode(s.codec, s.versioner, origState.data, out, origState.rev)
            }
        }

        // 砖汉数据
        newData, err := s.transformer.TransformToStorage(data, transformContext)
        if err != nil {
            return storage.NewInternalError(err.Error())
        }

        opts, err := s.ttlOpts(ctx, int64(ttl))
        if err != nil {
            return err
        }
        trace.Step("Transaction prepared")

        startTime := time.Now()
        // 事务更新数据
        txnResp, err := s.client.KV.Txn(ctx).If(
            clientv3.Compare(clientv3.ModRevision(key), "=", origState.rev),
        ).Then(
            clientv3.OpPut(key, string(newData), opts...),
        ).Else(
            clientv3.OpGet(key),
        ).Commit()
        metrics.RecordEtcdRequestLatency("update", getTypeName(out), startTime)
        if err != nil {
            return err
        }
        trace.Step("Transaction committed")
        if !txnResp.Succeeded {
            // 重新获取数据
            getResp := (*clientv3.GetResponse)(txnResp.Responses[0].GetResponseRange())
            klog.V(4).Infof("GuaranteedUpdate of %s failed because of a conflict, going to retry", key)
            origState, err = s.getState(getResp, key, v, ignoreNotFound)
            if err != nil {
                return err
            }
            trace.Step("Retry value restored")
            mustCheckData = false
            continue
        }
        // 获取put响应
        putResp := txnResp.Responses[0].GetResponsePut()

        return decode(s.codec, s.versioner, data, out, putResp.Header.Revision)
    }
}
```





# 6. 未曾讲到的地方

transformer的实现和注册地方我并没有找到，只看到了几个覆盖资源类型的地方，还有list/watch接口，后续再继续学习，今天就先到这里，下次再见





# 基于etcd的watch机制实现原理

本文介绍了针对etcd的watch场景,k8s在性能优化上面的一些设计，逐个介绍缓存、定时器、序列化缓存、bookmark机制、forget机制、针对数据的索引与ringbuffer等组件的场景以及解决的问题，希望能帮助到那些对apiserver中的listwatch感兴趣的朋友

# 1. 事件驱动与控制器

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583498400622-1a50cc6c-45f9-4da7-bb46-ed6f6221ea7e.png)

k8s中并没有将业务的具体处理逻辑耦合在rest接口中，rest接口只负责数据的存储，通过控制器模式，分离数据存储与业务逻辑的耦合，保证apiserver业务逻辑的简洁。

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583498598298-f8f7e3b2-2e35-47ca-8cae-f49d57c4d7f8.png)

控制器通过watch接口来感知对应的资源的数据变更，从而根据资源对象中的期望状态与当前状态之间的差异，来决策业务逻辑的控制，watch本质上做的事情其实就是将感知到的事件发生给关注该事件的控制器



# 2.Watch的核心机制

这里我们先介绍基于etcd实现的基础的watch模块

## 2.1 事件类型与etcd

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583498739811-7a7ac782-e337-4f00-bd1f-d90efd275315.png)

一个数据变更本质上无非就是三种类型：新增、更新和删除，其中新增和删除都比较容易因为都可以通过当前数据获取，而更新则可能需要获取之前的数据，这里其实就是借助了etcd中revision和mvcc机制来实现，这样就可以获取到之前的状态和更新后的状态，并且获取后续的通知



## 2.2 事件管道 

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583498862637-948ebc4a-c64d-4e28-a0ef-8ab2a1983ad2.png)

事件管道则是负责事件的传递，在watch的实现中通过两级管道来实现消息的分发，首先通过watch etcd中的key获取感兴趣的事件，并进行数据的解析，完成从bytes到内部事件的转换并且发送到输入管道(incomingEventChan)中，然后后台会有线程负责输入管道中获取数据，并进行解析发送到输出管道(resultChan)中，后续会从该管道来进行事件的读取发送给对应的客户端



## 2.3 事件缓冲区

事件缓冲区是指的如果对应的事件处理程序与当前事件发生的速率不匹配的时候，则需要一定的buffer来暂存因为速率不匹配的事件, 在go里面大家通常使用一个有缓冲的chan构建

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583498973111-53b35a4b-eaad-4d4f-ab4c-31bf28be8e3a.png)

到这里基本上就实现了一个基本可用的watch服务，通过etcd的watch接口监听数据，然后启动独立goroutine来进行事件的消费，并且发送到事件管道供其他接口调用



# 3. Cacher

kubernetes中所有的数据和系统都基于etcd来实现，如何减轻访问压力呢，答案就是缓存，watch也是这样，本节我们来看看如何实现watch缓存机制的实现，这里的cacher是针对



## 3.1 Reflector

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583499027952-481c144a-82f6-489b-95ea-281dbc942c4f.png)

Reflector是client-go中的一个组件，其通过listwatch接口获取数据存储在自己内部的store中，cacher中通过该组件从etcd进行watch操作，避免为每个组件都创建一个etcd的watch



## 3.2 watchCache

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583499172186-a16b681f-be80-4b02-bb57-ff9b8db41f27.png)

wacthCache负责存储watch到的事件，并且将watch的事件建立对应的本地索引缓存，同时在构建watchCache还负责将事件的传递，其将watch到的事件通过eventHandler来传递给上层的Cacher组件



## 3.3 cacheWatcher

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583499208108-122c2a29-d572-4671-b55b-c5c259cc739b.png)



cacheWatcher顾名思义其是就是针对cache的一个watcher(watch.Interface)实现, 前端的watchServer负责从起ResultChan里面获取事件进行转发



## 3.4 Cacher

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583499437719-34b3ebd9-7144-45f0-9eab-7f42565d2a93.png?x-oss-process=image/watermark,type_d3F5LW1pY3JvaGVp,size_10,text_YmF4aWFvc2hpMjAyMA==,color_FFFFFF,shadow_50,t_80,g_se,x_10,y_10)

Cacher基于etcd的store结合上面的watchCache和Reflector共同构建带缓存的REST store, 针对普通的增删改功能其直接转发给etcd的store来进行底层的操作，而对于watch操作则进行拦截，构建并返回cacheWatcher组件





# 4. Cacher的优化

看完基础组件的实现，接着我们看下针对watch这个场景k8s中还做了那些优化，学习针对类似场景的优化方案



## 4.1 序列化缓存

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583499581771-bb5fc56d-f9e7-452f-9c8c-a6efa0118477.png)

如果我们有多个watcher都wacth同一个事件，在最终的时候我们都需要进行序列化，cacher中在分发的时候，如果发现超过指定数量的watcher, 则会在进行dispatch的时候，为其构建构建一个缓存函数，针对多个watcher只会进行一次的序列化



## 4.2 nonblocking

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583499684987-176c0090-1d2c-4848-9965-31d769d8945c.png)

在上面我们提到过事件缓冲区，但是如果某个watcher消费过慢依然会影响事件的分发，为此cacher中通过是否阻塞(是否可以直接将数据写入到管道中)来将watcher分为两类，针对不能立即投递事件的watcher, 则会在后续进行重试



## 4.3 TimeBudget

针对阻塞的watcher在进行重试的时候，会通过dispatchTimeoutBudget构建一个定时器来进行超时控制, 那什么叫Budget呢，其实如果在这段时间内，如果重试立马就成功，则本次剩余的时间，在下一次进行定时的时候，则可以使用之前剩余的余额，但是后台也还有个线程，用于周期性重置



## 4.4 forget机制

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583499809964-00c03a8c-3a09-49ef-8d85-bf9ce1a1c069.png)

针对上面的TimeBudget如果在给定的时间内依旧无法进行重试成功，则就会通过forget来删除对应的watcher, 由此针对消费特别缓慢的watcher则可以通过后续的重试来重新建立watch，从而减小对apiserver的watch压力



## 4.5 bookmark机制

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583499925108-1d92f690-47a5-4b28-be51-b1a4cf0d600e.png)

bookmark机制是大阿里提供的一种优化方案，其核心是为了避免单个某个资源一直没有对应的事件，此时对应的informer的revision会落后集群很大，bookmark通过构建一种BookMark类型的事件来进行revision的传递，从而让informer再重启后不至于落后特别多



## 4.6 watchCache中的ringbuffer

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583500025520-5aa73889-1ad4-4076-a599-5f793a7be4e9.png?x-oss-process=image/watermark,type_d3F5LW1pY3JvaGVp,size_10,text_YmF4aWFvc2hpMjAyMA==,color_FFFFFF,shadow_50,t_80,g_se,x_10,y_10)

watchCache中通过store来构建了对应的索引缓存，但是在listwatch操作的时候，则通常需要获取某个revision后的所有数据，针对这类数据watchCache中则构建了一个ringbuffer来进行历史数据的缓存

# 5.设计总结

![image.png](https://cdn.nlark.com/yuque/0/2020/png/97498/1583501103452-70c9fcd3-e469-42d3-bc6f-14ae3a23d0a1.png?x-oss-process=image/watermark,type_d3F5LW1pY3JvaGVp,size_14,text_YmF4aWFvc2hpMjAyMA==,color_FFFFFF,shadow_50,t_80,g_se,x_10,y_10)

本文介绍了针对etcd的watch场景,k8s在性能优化上面的一些设计，逐个介绍缓存、定时器、序列化缓存、bookmark机制、forget机制、针对数据的索引与ringbuffer等组件的场景以及解决的问题，希望能帮助到那些对apiserver中的listwatch感兴趣的朋友