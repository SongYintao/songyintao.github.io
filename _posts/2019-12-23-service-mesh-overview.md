---
title: Service Mesh：蚂蚁的实践
subtitle: service mesh 在蚂蚁的实现，使用场景，设计的考量
layout: post
tags: [service-mesh, sofa-mesh,sofa-moson]
---

service mesh是未来cloud native的趋势，有必要花点时间和精力来学习一下他的实现原理，已经最新的大厂技术动态。国内相对比较厉害的蚂蚁金服开源了sofa-mesh、sofa-moson以及他们在支付宝的实践。看了之后，不觉明厉。内容真的很是丰富，特此学习、转发、分享、总结一下。

[博客地址](https://www.sofastack.tech/blog/)

[系列文章之一](https://www.sofastack.tech/blog/service-mesh-practice-in-production-at-ant-financial-part5-gateway/)



## 背景知识

Envoy快速入门：[入门文档](https://www.servicemesher.com/envoy/)











### Mosn

![](../img/mosn-modules-arch.jpg)

​	





## 一些问题

- 和现有的服务注册中心如何整合
- 监听的链接是怎么管理的
- 是怎么热更新listener的、热重启mosn的
- 请求的数据流路径



带着这些问题，我们来深入探究一下mosn这个数据平面的实现原理。







### 代码

#### listener

```go
func (l *listener) Start(lctx context.Context, restart bool) {
	defer func() {
		if r := recover(); r != nil {
			log.DefaultLogger.Errorf("[network] [listener start] panic %v\n%s", r, string(debug.Stack()))
		}
	}()

	if l.bindToPort {
		ignore := func() bool {
			l.mutex.Lock()
			defer l.mutex.Unlock()
			switch l.state {
			case ListenerRunning:
				// if listener is running, ignore start
				return true
			case ListenerStopped:
				if !restart {
					return true
				}
				if err := l.listen(lctx); err != nil {
					// TODO: notify listener callbacks
					return true
				}
			default:
				// try start listener
				//call listen if not inherit
				if l.rawl == nil {
					if err := l.listen(lctx); err != nil {
						// TODO: notify listener callbacks
					}
				}
			}
			l.state = ListenerRunning
			return false
		}()
		if ignore {
			return
		}

		for {
			if err := l.accept(lctx); err != nil {
				if nerr, ok := err.(net.Error); ok && nerr.Timeout() {
					return
				} else if ope, ok := err.(*net.OpError); ok {
					// not timeout error and not temporary, which means the error is non-recoverable
					// stop accepting loop and log the event
					if !(ope.Timeout() && ope.Temporary()) {
						// accept error raised by sockets closing
						if ope.Op == "accept" {
							log.DefaultLogger.Infof("[network] [listener start] [accept] listener %s %s closed", l.name, l.Addr())
						} else {
			
						}
						return
					}
				} else {
				
				}
			}
		}
	}
}

if ignore {
			return
		}
		for {
			if err := l.accept(lctx); err != nil {
				if nerr, ok := err.(net.Error); ok && nerr.Timeout() {
					log.DefaultLogger.Infof("[network] [listener start] [accept] listener %s stop accepting connections by deadline", l.name)
					return
				} else if ope, ok := err.(*net.OpError); ok {
					// not timeout error and not temporary, which means the error is non-recoverable
					// stop accepting loop and log the event
					if !(ope.Timeout() && ope.Temporary()) {
						// accept error raised by sockets closing
						if ope.Op == "accept" {
							log.DefaultLogger.Infof("[network] [listener start] [accept] listener %s %s closed", l.name, l.Addr())
						} else {
							log.DefaultLogger.Errorf("[network] [listener start] [accept] listener %s occurs non-recoverable error, stop listening and accepting:%s", l.name, err.Error())
						}
						return
					}
				} else {
					log.DefaultLogger.Errorf("[network] [listener start] [accept] listener %s occurs unknown error while accepting:%s", l.name, err.Error())
				}
			}
		}
```



处理流程：

1. 判断当前的listener状态，决定是否start
2. 打开listen
3. 开始accept，loop循环监听

```go
func (l *listener) listen(lctx context.Context) error {
	var err error
	var rawl *net.TCPListener
	if rawl, err = net.ListenTCP("tcp", l.localAddress.(*net.TCPAddr)); err != nil {
		return err
	}
	l.rawl = rawl
	return nil
}
```



Accept获取当前listener的连接Conn，之后异步触发`ListenerEventListener`的Accept的监听事件，根据获取到的rawConn进行相关的处理：

```go
func (l *listener) accept(lctx context.Context) error {
   rawc, err := l.rawl.Accept()

   if err != nil {
      return err
   }

   // TODO: use thread pool
   utils.GoWithRecover(func() {
      l.cb.OnAccept(rawc, l.useOriginalDst, nil, nil, nil)
   }, nil)

   return nil
}
```







```go
//   The bunch of interfaces are structure skeleton to build a high performance, extensible network framework.
//
//   In mosn, we have 4 layers to build a mesh, net/io layer is the fundamental layer to support upper level's functionality.
//	 -----------------------
//   |        PROXY          |
//    -----------------------
//   |       STREAMING       |
//    -----------------------
//   |        PROTOCOL       |
//    -----------------------
//   |         NET/IO        |
//    -----------------------
//


//Stream layer leverages protocol's ability to do binary-model conversation. In detail, Stream uses Protocols's encode/decode facade method and DecodeFilter to receive decode event call.


//   Listener:
//   	- Event listener
// 			- ListenerEventListener
//      - Filter
// 			- ListenerFilter

//   Connection:
//		- Event listener
// 			- ConnectionEventListener
//		- Filter
// 			- ReadFilter
//			- WriteFilter

//
//   Below is the basic relation on listener and connection:
//    --------------------------------------------------
//   |                                      			      |
//   | 	  EventListener       EventListener     		    |
//   |        *|                   |*          		      |
//   |         |                   |       				      |
//   |        1|     1      *      |1          		    	|
// 	 |	    Listener --------- Connection      		    	|
//   |        1|      [accept]     |1          	    		|
//	 |         |                   |-----------         |
//   |        *|                   |*          |*       |
//   |	 ListenerFilter       ReadFilter  WriteFilter   |
//   |                                                  |
//    --------------------------------------------------

```



Core model in network layer are **listener** and **connection**. Listener listens specified port, waiting for new connections.

Both **listener** and **connection** have **a extension mechanism, implemented as listener and filter chain, which are used to fill in customized logic.**

**Event listeners** are used to **subscribe important event of Listener and Connection**. Method in listener will be called on event occur, but **not effect the control flow**.

**Filters are called on event occurs, it also returns a status to effect control flow.** Currently 2 states are used: `Continue` to let it go, `Stop` to stop the control flow.

**Filter** has **a callback handler** to interactive with core model. For example, `ReadFilterCallbacks` can be used to continue filter chain in connection, on which is in a stopped state.



**RequestInfo** has information for a request, include the basic information, the request's downstream information, ,the request's upstream information and the router information.

**Routers** defines and manages all router
**RouterManager** is a manager for all routers' config





![](../img/mosn-stream.png)

**Core model in stream layer is stream, <u>which manages process of a round-trip, a request and a corresponding response</u>.**

**Event listeners** can be installed into a stream to monitor event.

**Stream** has two related models, **encoder** and **decoder**:

- **StreamSender**: a sender encodes request/response to binary and sends it out, flag 'endStream' means data is ready to sendout, no need to wait for further input.
- **StreamReceiveListener**: It's more like a decode listener to get called on a receiver receives binary and decodes to a request/response.
- **Stream** does not have a predetermined direction, so `StreamSender` could be a request encoder as a client or a response encoder as a server. It's just about the scenario, so does `StreamReceiveListener`.

//   Stream:
//      - Encoder
//           - StreamSender
//      - Decoder
//           - StreamReceiveListener

Event listeners:

- `StreamEventListener`: listen **stream** event: `reset`, `destroy`.

- `StreamConnectionEventListener`: listen **stream connection** event: `goaway`.

In order to meet the expansion requirements in the stream processing, `StreamSenderFilter` and `StreamReceiverFilter` are introduced as **a filter chain in encode/decode process**.
**Filter's method will be called on corresponding stream process stage and returns a status(Continue/Stop) to effect the control flow.**

**From an abstract perspective, stream represents a virtual process on underlying connection**. To make stream interactive with connection, some intermediate object can be used.
**StreamConnection** is the core model to connect connection system to stream system. 

**As a example, when proxy reads binary data from connection, it dispatches data to StreamConnection to do protocol decode.** 流主要用于协议的编解码。
Specifically, `ClientStreamConnection` uses a `NewStream` to exchange `StreamReceiveListener` with `StreamSender`.
`Engine` provides a callbacks(`StreamSenderFilterHandler`/`StreamReceiverFilterHandler`) to let filter interact with stream engine.
As a example, a encoder filter stopped the encode process, it can continue it by `StreamSenderFilterHandler`.`ContinueSending` later. Actually, a filter engine is a encoder/decoder itself.

#### 核心Model 接口定义

`Stream` 是一个通用的协议流，它是stream层里面的核心Model。

```go
// Stream is a generic protocol stream, it is the core model in stream layer
type Stream interface {
	// ID returns unique stream id during one connection life-cycle
  //ID表示一个连接生命周期里面的唯一表示
	ID() uint64

	// AddEventListener adds stream event listener
  // 添加流事件监听器
	AddEventListener(streamEventListener StreamEventListener)

	// RemoveEventListener removes stream event listener
  // 删除流事件的监听器
	RemoveEventListener(streamEventListener StreamEventListener)

	// ResetStream rests and destroys stream, called on exception cases like connection close.
	// Any registered StreamEventListener.OnResetStream and OnDestroyStream will be called.
  //根据流重置的原因，重置并销毁流 StreamEventListener.OnResetStream OnDestroyStream会被调用
	ResetStream(reason StreamResetReason)

	// DestroyStream destroys stream, called after stream process in client/server cases.
	// Any registered StreamEventListener.OnDestroyStream will be called.
  // 在client／server流处理完成之后，销毁流
	DestroyStream()
}
```

#### 1. 发送接口

`StreamSender` 编码并发送协议流。

- 在服务端场景，`StreamSender`发送返回响应
- 在客户端场景，`StreamSender`发送请求

```go
// StreamSender encodes and sends protocol stream
// On server scenario, StreamSender sends response
// On client scenario, StreamSender sends request
type StreamSender interface {
	// Append headers
	// endStream supplies whether this is a header only request/response
  //添加请求、返回的头
	AppendHeaders(ctx context.Context, headers HeaderMap, endStream bool) error

	// Append data
	// endStream supplies whether this is the last data frame
  //添加请求、返回的数据体，endStream表示是否是最后的数据帧
	AppendData(ctx context.Context, data IoBuffer, endStream bool) error

	// Append trailers, implicitly ends the stream.
	AppendTrailers(ctx context.Context, trailers HeaderMap) error

	// Get related stream
  //获取相关的流
	GetStream() Stream
}
```

#### 2. 接收接口

`StreamReceiveListener` 当数据接收时被调用，并进行解码。

- 在服务端场景，StreamReceiveListener 被用来处理请求参数。
- 在客户端场景，StreamReceiveListener被用来处理返回参数。

```go
// StreamReceiveListener is called on data received and decoded
// On server scenario, StreamReceiveListener is called to handle request
// On client scenario, StreamReceiveListener is called to handle response
type StreamReceiveListener interface {
	// OnReceive is called with decoded request/response
  //被解码的请求／返回参数调用
	OnReceive(ctx context.Context, headers HeaderMap, data IoBuffer, trailers HeaderMap)

	// OnDecodeError is called with when exception occurs
  //当异常发生时调用
	OnDecodeError(ctx context.Context, err error, headers HeaderMap)
}
```





### StreamConnection 

`StreamConnection` 本质上是一个connection连接，跑了很多的stream。

```go
// StreamConnection is a connection runs multiple streams
type StreamConnection interface {
	// Dispatch incoming data
	// On data read scenario, it connects connection and stream by dispatching read buffer to stream,
	// stream uses protocol decode data, and popup event to controller
  //调度收到的数据。在数据读取的场景，他链接来了connection和stream两个对象：分配读缓冲区buffer到stream；
  //stream使用相关的协议解码数据，并且发送event到controller。
	Dispatch(buffer IoBuffer)

	// Protocol on the connection
	Protocol() Protocol

	// Active streams count 激活的流数量
	ActiveStreamsNum() int

	// GoAway sends go away to remote for graceful shutdown
  //发送 go way到远端的服务，优雅关闭连接
	GoAway()

	// Reset underlying streams
  //重置底层的流
	Reset(reason StreamResetReason)
}



```

#### 1. ServerStreamConnection 服务端侧流连接

```go

// ServerStreamConnection is a server side stream connection.
type ServerStreamConnection interface {
	StreamConnection
}
```

#### 2. ClientStreamConnection 客户端侧流连接

```go
// ClientStreamConnection is a client side stream connection.
type ClientStreamConnection interface {
	StreamConnection

	// NewStream starts to create a new outgoing request stream and returns a sender to write data
	// responseReceiveListener supplies the response listener on decode event
	// StreamSender supplies the sender to write request data
  //创建了一个外向的请求流 并且返回一个sender来发送数据。
  //responseReceiveListener 提供了应用于解码事件的response监听器
  //StreamSender 提供了应写请求数据的sender
	NewStream(ctx context.Context, responseReceiveListener StreamReceiveListener) StreamSender
}
```



#### 3. 流过滤器相关



```go
// StreamFilterHandler is called by stream filter to interact with underlying stream
//当与底层的流交互的时候，触发的流Filter
type StreamFilterHandler interface {
	// Route returns a route for current stream
  //返回一个路由，用于当前流的路由
	Route() Route

	// RequestInfo returns request info related to the stream
  //返回相关流的请求信息
	RequestInfo() RequestInfo

	// Connection returns the originating connection
	Connection() Connection
}
```



```go
// StreamSenderFilter is a stream sender filter
//StreamSenderFilter ：流发送过滤器
type StreamSenderFilter interface {
	StreamFilterBase

	// Append encodes request/response
  //扩展 编码请求／返回参数
	Append(ctx context.Context, headers HeaderMap, buf IoBuffer, trailers HeaderMap) StreamFilterStatus

	// SetSenderFilterHandler sets the StreamSenderFilterHandler
  //设置流发送处理器Handler
	SetSenderFilterHandler(handler StreamSenderFilterHandler)
}

// StreamSenderFilterHandler is a StreamFilterHandler wrapper
//流发送处理器Handler:流过滤处理器Handler的封装
type StreamSenderFilterHandler interface {
	StreamFilterHandler

	// TODO :remove all of the following when proxy changed to single request @lieyuan
	// StreamFilters will modified headers/data/trailer in different steps
	// for example, maybe modify headers in AppendData
	GetResponseHeaders() HeaderMap
	SetResponseHeaders(headers HeaderMap)

	GetResponseData() IoBuffer
	SetResponseData(buf IoBuffer)

	GetResponseTrailers() HeaderMap
	SetResponseTrailers(trailers HeaderMap)
}

// StreamReceiverFilter is a StreamFilterBase wrapper
type StreamReceiverFilter interface {
	StreamFilterBase

	// OnReceive is called with decoded request/response
	OnReceive(ctx context.Context, headers HeaderMap, buf IoBuffer, trailers HeaderMap) StreamFilterStatus

	// SetReceiveFilterHandler sets decoder filter callbacks
	SetReceiveFilterHandler(handler StreamReceiverFilterHandler)
}

// StreamReceiverFilterHandler add additional callbacks that allow a decoding filter to restart
// decoding if they decide to hold data
type StreamReceiverFilterHandler interface {
	StreamFilterHandler

	// TODO: consider receiver filter needs AppendXXX or not

	// AppendHeaders is called with headers to be encoded, optionally indicating end of stream
	// Filter uses this function to send out request/response headers of the stream
	// endStream supplies whether this is a header only request/response
	AppendHeaders(headers HeaderMap, endStream bool)

	// AppendData is called with data to be encoded, optionally indicating end of stream.
	// Filter uses this function to send out request/response data of the stream
	// endStream supplies whether this is the last data
	AppendData(buf IoBuffer, endStream bool)

	// AppendTrailers is called with trailers to be encoded, implicitly ends the stream.
	// Filter uses this function to send out request/response trailers of the stream
	AppendTrailers(trailers HeaderMap)

	// SendHijackReply is called when the filter will response directly
	SendHijackReply(code int, headers HeaderMap)

	// SendDirectRespoonse is call when the filter will response directly
	SendDirectResponse(headers HeaderMap, buf IoBuffer, trailers HeaderMap)

	// TODO: remove all of the following when proxy changed to single request @lieyuan
	// StreamFilters will modified headers/data/trailer in different steps
	// for example, maybe modify headers in on receive data
	GetRequestHeaders() HeaderMap
	SetRequestHeaders(headers HeaderMap)

	GetRequestData() IoBuffer
	SetRequestData(buf IoBuffer)

	GetRequestTrailers() HeaderMap
	SetRequestTrailers(trailers HeaderMap)

	SetConvert(on bool)
}
```















## 实现方案

采用Iostio+mosn来支持service mesh的部署，其中：

- Isotio使用其中的Pilot来动态发现服务
- mosn定期查询pilot来获取最新的配置，动态更新配置信息。



需要解决的问题：

- 服务的发现如何整合现有的技术栈：mainstay、southgate==>pilot对接
- 控制平面后台的可视化
- sidecar部署的方式，如何整合
- macvlan不支持serviceIp怎么破？===》必须升级到calico





### 第一期

目标：

- 实现node、c++这些需要直连静态ip的微服务
- 通过虚拟的VIP或者域名，可以访问到相关的服务（即，本地访问需要有一个标识符，表明我要访问的是哪个服务）



概念映射

| 术语 | Mainstay     | service mesh（envoy） |
| ---- | ------------ | --------------------- |
| 集群 | group        | 集群                  |
| 实例 | 每个服务实例 | 集群下面的HOST        |
|      |              |                       |
|      |              |                       |
|      |              |                       |
|      |              |                       |
|      |              |                       |



应用起来之后，应用属于那个集群（包含应用的元信息：app、version，用于路由）；































