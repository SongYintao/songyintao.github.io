---
title: Go基础：网络编程
subtitle: go网络编程简介
layout: post
tags: [go]
---

​	我们将学习一下Go语言的网络编程。Go语言标准库里提供的net包，支持基于IP层、TCP／UDP层及更高层面（如HTTP、FTP、SMTP）的网络操作，其中用于IP层的称为`Raw Socket`。



### 1.Socket 编程

​	在Go语言中编写网络程序时，不会看到传统的编码方式。之前使用Socket编程时，会按照如下的步骤展开。

1. 建立Socket：使用`socket()`函数。
2. 绑定Socket：使用`bind() `函数。
3. 监听：使用`listen()`函数。或者连接：使用`connect()`函数。
4. 接受连接：使用`accept()`函数。
5. 接收：使用`receive()`函数。或者发送：使用`send()`函数。

​	`GO`语言标准库对此过程进行了抽象和封装。无论我们期望使用什么协议建立什么形式的连接，都只要调用`net.Dial()`即可。

#### 1. Dial函数

​	`Dial()`函数原型如下：

```go
func Dial(net,addr string)(Conn,error)
```

​	其中，`net`参数为网络协议的名字，`addr`参数是IP地址或域名，而端口号以`“:”`的形式跟随在地址或域名的后面，端口号可选。如果连接成功，返回连接对象，否则`error`。

​	常用的调用方式如下。

TCP链接：

```go
conn,err:=net.Dial("tcp","192.168.1.1:2100")
```

UDP链接：

```go
conn,err:=net.Dial("udp","192.168.1.1:975")
```

ICMP链接(使用协议名称)：

```go
conn,err:=net.Dial("ip4:icmp","www.baidu.com")
```

ICMP链接(使用协议编号)：

```go
conn,err:=net.Dial("ip4:1","www.baidu.com")
```

​	目前，`Dial()`函数支持如下几种网络协议：tcp、tcp4、tcp6、udp、udp4、udp6、ip、ip4、ip6 。

​	成功建立连接后，我们就可以进行数据的发送和接收。发送时，使用conn的`Write()`成员方法，接收数据时使用`Read()`方法。

#### 2. HTTP编程

​	Go 语言标准库内建提供了net/http包，涵盖了HTTP客户端和服务端到的具体实现。使用net/http包，我们可以很方便的编写Http客户端或服务端的程序。

##### 2.1 HTTP客户端

​	Go内置的net/http包提供了最简洁的Http客户端实现，可以直接使用HTTP中使用最多的GET和POSY方式请求数据。

###### 1.基本方法

```go
func (c *Client) Get(url string) (r *Response,err error)
func (c *Client) Post(url string,bodyType string,body io.Reader) (r *Response,err error)
func (c *Client) PostForm(url string,data url.Values)(r *Response,err error)
func (c *Client) Head(url string)(r *Response,err error)
func (c *Client) Do(req *Request) (resp *Response,err,error)
```

###### 2. 高级封装

​	除了前面介绍的基本HTTP操作，Go语言标准库也暴露了比较底层的HTTP相关库，开发可以基于这些库灵活定制Http服务器和使用Http服务。

- 自定义http.Client

前面的基础方法都是基于`http.DefaultClient`进行调用的。比如，`http.Get()`等价于`http.DefaultClient.Get()`。

​	`http.Client`结构体如下：

```go
type Client struct {
    //Transport 用于确定HTTP请求的创建机制
    //如果为空，将会使用DefaultTransport
    Transport RoundTripper
    //CheckRedirect定义重定向策略
    //如果CheckRedirect不为空，客户端将在跟踪Http重定向前调用函数。两个参数req和via分别为即将发起的请求和已经发起的所有请求，最早的已发起请求在最前面。
    //如果CheckRedirect返回错误，客户度啊将直接返回错误，不会再发去请求。
    //如果CheckRedirect为空，Client将采用一种确认策略，将在10个连续请求后终止
    CheckRedirect func(req *Request,via []*Request)error
    //如果Jar为空，Cookie将不会在请求中发送，并会在响应中被忽略
    Jar CookieJar
}
```

​	其中`Transport`类型必须实现`http.RoundTripper`接口。`Transport `指定了执行一个HTTP 请求的运行机制，如果不指定具体的`Transport`，默认会使用`http.DefaultTransport`，这意味着`http.Transport`也是可以自定义的。`net/http`包中的`http.Transport`类型实现了`http.RoundTripper`接口。

​	`CheckRedirect`函数 定处理重定向的  。当使用 HTTP Client 的`Get()`或者是`Head()`方法发送 HTTP 请求时， 响应返回的状态码为 30x (比如 301 / 302 / 303 / 307)，HTTP Client 会在遵循跳转规则之前先调用这个`CheckRedirect`函数。

​	`Jar`可用于在 `HTTP Client `中设定` Cookie`，`Jar`的类型必须实现了 `http.CookieJar` 接口， 该接口 定义了 `SetCookies()`和`Cookies()`两个方法。如果 `HTTP Client `中没有设定` Jar`， `Cookie`将被忽略而不会发送到用户端。实际上，我们一般都用 `http.SetCookie() `方法来设定 `Cookie`。 

​	使用自定义的`http.Client`及其`Do()`方法，我们可以非常灵活地控制 HTTP 请求，比如发送自定义 `HTTP Header` 或是改写重定向策略等。创建自定义的 `HTTP Client` 非常简单，具体代码如下:

```go
client:=&http.Client{
    CheckRedirect: redirectPolicyFunc,
}
resp,err:=client.Get("http://example.com")
//...
req,err:=http.NewRequest("GET","http://example.com",nil)
//...
req.Header.Add("User-agent","Our Custom User-agent")
req.Header.Add("If-None-Match",`W/"TheFileEtag"`)
resp,err:=client.Do(req)
//...
```

- 自定义`http.Transport`

该对象指定执行一个HTTP请求时的运行规则。下面看一下具体的结构：

```go
type Transport struct{
    //Porxy指定用于针对特定请求返回代理的函数
    //如果该函数返回一个非空的错误，请求将终止并返回该错误
    //
    Proxy func(*Request)(*url.URL,error)
    //指定用于创建TCP连接的dial()函数。为空，默认使用net.Dial()
    Dial func(net,addr string) (c net.Conn,err error)
    //指定用于tls.Client的TLS配置
    TLSClientConfig *tls.Config
    
    DisableKeepAlives bool
    DisableCompression bool
    
    MaxIdleConnsPerHost int
    //...
}
```

-   灵活的`http.RoundTripper`接口

` http.RoundTripper `接口的具体定义:

```go
type RoundTripper interface{
    //RoundTrip执行一个单一的HTTP事务，返回相应的响应信息
    RoundTrip(*Request) (*Response,error)
}
```

#### 3. HTTP服务端

​	如何处理HTTP请求和HTTPS请求。

##### 1.处理http请求

​	使用net/http包提供的http.ListenAndServe()方法，可以在指定的地址进行监听，开启一个Http，服务端该方法的原型如下：

```go
func ListenAndServe(addr string,handler Handler) error
```

该方法用于在指定的 TCP 网络地  addr 进行监听，然后调用服务端处理程序来处理传入的连接请求。该方法有两个参数:第一个参数 addr即监听地址;第二个参数表示服务端处理程序， 通常为空，这意味着服务端调用 `http.DefaultServeMux `进行处理，而服务端编写的业务逻辑处理程序 `http.Handle() `或 `http.HandleFunc() ` 默认注入`http.DefaultServeMux` 中， 具体代码如下: 

```go
http.Handle("/foo",fooHandler)
http.HandleFunc("/bar",func(w http.ResponseWriter,r *http.Reqquest){
    fmt.Fprintf(w,"Hello")
})
log.Fatal(http.ListenAndServe(":8080",nil))
```

如果想更多地控制服务端的行为，可以自定义`http.Server`，代码如下：

```go
s:=&http.Server{
    Addr:	":8080",
    Handler: 	myHandler,
    ReadTimeout:	10*time.Second,
    WriteTimeout:	10*time.Second,
    MaxHeaderBytes:1<<20,
}
log.Fatal(s.ListenAndServe())
```

##### 2.处理HTTPS请求

```go
func ListenAndServeTLS(addr string, certFile string, keyFile string, handler Handler) error
```

 务器上必须存在包含证书和与之  的  的相关文件，比如certFile对应SSL证书文件存放  ，keyFile对应证书  文件  。如果证书是由证书颁发机构  签署的，certFile参数指定的路径必须是存放在服务器上的经由CA认证过的SSL证书。

#### 3.RPC编程

##### 1.Go语言的RPC支持与处理

​	在go中，标准库提供的net/rpc包实现了RPC协议需要的相关细节，开发者可以方便地使用该包编写RPC的服务端和客户端程序，这使得用Go语言开发的多个进程之间的通信变得简单。

​	net/rpc包允许 RPC 客户端程序通过网络或是其他 I/O 连接调用一个远程对象的 开方法 (必须是大写字母开头、可外部调用的)。在 RPC服务端 ，可将一个对象注册为可访问的服务，之后该对象的公开方法就能够以远程的方式提供访问。一个 RPC服务端可以注册多个不同类型的对象，但不允许注册同一类型的多个对象。

​	一个对象中只有满足如下条件的方法，才能被RPC服务端设置为可供远程访问：

- [ ] 必须是在对象外部可公开调用的方法；
- [ ] 必须有两个参数，且参数的类型都必须是外部可以访问的类型或者是Go内建支持的类型；
- [ ] 第二个参数必须是一个指针；
- [ ] 方法必须返回一个error类型的值。

```go
func (t *T) MethodName(argType T1,replyType *T2) error
```

类型T、T1和T2默认会使用Go内置的encoding／gob包进行编码解码。第一个参数表示客户端传入的参数，第二个参数表示要返回给客户端的结果。