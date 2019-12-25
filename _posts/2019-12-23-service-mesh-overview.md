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





## 实现方案

采用Iostio+mosn来支持service mesh的部署，其中：

- Isotio使用其中的Pilot来动态发现服务
- mosn定期查询pilot来获取最新的配置，动态更新配置信息。



需要解决的问题：

- 服务的发现如何整合现有的技术栈：mainstay、southgate
- 控制平面后台的可视化
- sidecar部署的方式，如何整合
- macvlan不支持serviceIp怎么破？===》必须升级到calico







