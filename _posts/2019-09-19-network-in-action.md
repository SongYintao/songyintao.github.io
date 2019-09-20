---
title: 网络编程进阶
subtitle: 网络编程精进之路
layout: post
tags: [网络]
---

话说，计算机网络虽然没有在本科系统的学习过，自学了那么多的概念，有实际工作了两年多。是时候好好梳理、总结一下。



##  背景

网络编程的意义。



To be continue。。。。



##  提高篇



### 1. TIME_WAIT ：隐藏在细节下的魔鬼











### 2. 优雅的关闭还是粗暴的关闭？



- shutdown
- close



####  1. close函数

```c
int close(int sockfd)
```

这个函数会对套接字引用计数减一，一旦发现引用计数到达0，就会对套接字进行彻底的释放，并且会关闭`TCP`的两个方向的数据流。

**套接字引用计数**：因为套接字可以被多个进程共享，简单理解为我们给每个套接字都设置了一个积分。如果我们通过`fork`产生了子进程，套接字加1. 每一次调用`close`就会减一。

具体关闭两个方向的流程：

- 输出方向，内核尝试将发送缓冲期的数据发送给对端，并最后向对端发送一个`FIN`报文，接下来如果再对该套接字进行写操作会返回异常。
- 输入方向，内核会将套接字设置为不可读，任何读操作都会返回异常。



如果对端没有检测到套接字已经关闭，还继续发送报文，就会收到一个`RST`报文，告诉对端："大佬，我已经离职了。。别再给我发送数据了"。

我们会发现，`colse`并不会给我们只关闭连接的一个方向。`shutdown`上场，支持关闭TCP连接的一个方向。



#### 2. shutdown函数

```c
int shutdown(int sockfd, int howto)
```

`howto`参数的设置:

- `SHUT_RD(0)` : 关闭这个链接的读方向，对该套接字进行读操作的时候直接返回EOF。从数据角度来看，套接字接受缓冲区已有的数据将被丢弃，如果再有新的数据流到达，会对数据流进行ACK，然后悄悄的丢掉。也就是说，对端还是会收到ACK，这种情况下根本不知道数据已经被丢弃了。
- `SHUT_WR(1) `: 关闭这个连接的"写"这个方向，这就是常说的*半关闭*的连接。此时，不管套接字的引用计数是多少，都会直接关闭连接的写方向。套接字上发送缓冲区已有的数据将被立即发送出去，并发送一个`FIN`报文给对端。应用程序对该套接字进行写操作会报错。
- `SHUT_RDWR(2) `:上面的各执行一次。



#### 3. 示例——close和shutdown的差别

客户端程序：

```c
# include "lib/common.h"
# define    MAXLINE     4096

int main(int argc, char **argv) {
    if (argc != 2) {
        error(1, 0, "usage: graceclient <IPaddress>");
    }
    
    int socket_fd;
    socket_fd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in server_addr;
    bzero(&server_addr, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERV_PORT);
    inet_pton(AF_INET, argv[1], &server_addr.sin_addr);

    socklen_t server_len = sizeof(server_addr);
    int connect_rt = connect(socket_fd, (struct sockaddr *) &server_addr, server_len);
    if (connect_rt < 0) {
        error(1, errno, "connect failed ");
    }

    char send_line[MAXLINE], recv_line[MAXLINE + 1];
    int n;
		// 初始化 FD集合
    fd_set readmask;
    fd_set allreads;

    FD_ZERO(&allreads);
  //标准输入加入fd集合
    FD_SET(0, &allreads);
  //socket fd 加入集合
    FD_SET(socket_fd, &allreads);
    
    for (;;) {
        readmask = allreads;
      // 使用select多路复用 观测在连接套接字和标准输入上面的两个IO事件
        int rc = select(socket_fd + 1, &readmask, NULL, NULL, NULL);
        if (rc <= 0)
            error(1, errno, "select failed");
      //当连接套接字上有数据可读，将数据读入到程序缓冲区中。
        if (FD_ISSET(socket_fd, &readmask)) {
            
            n = read(socket_fd, recv_line, MAXLINE);
            if (n < 0) {
                error(1, errno, "read error");
            } else if (n == 0) {
              //服务端发送EOF，退出
                error(1, 0, "server terminated \n");
            }
            recv_line[n] = 0;
            fputs(recv_line, stdout);
            fputs("\n", stdout);
        }
        // 标准输入有数据可读
        if (FD_ISSET(0, &readmask)) {
            if (fgets(send_line, MAXLINE, stdin) != NULL) {
                if (strncmp(send_line, "shutdown", 8) == 0) {
                    // delete stdin in fd_Set
                    FD_CLR(0, &allreads);
                   //shutdown只关闭了写的方向===》参数1
                    if (shutdown(socket_fd, 1)) {
                        error(1, errno, "shutdown failed");
                    }
                } else if (strncmp(send_line, "close", 5) == 0) {
                    FD_CLR(0, &allreads);
                    if (close(socket_fd)) {
                        error(1, errno, "close failed");
                    }
                    sleep(6);
                    exit(0);
                } else {
                    int i = strlen(send_line);
                    if (send_line[i - 1] == '\n') {
                        send_line[i - 1] = 0;
                    }

                    printf("now sending %s\n", send_line);
                  	// write data to socket and send to server
                    size_t rt = write(socket_fd, send_line, strlen(send_line));
                    if (rt < 0) {
                        error(1, errno, "write failed ");
                    }
                    printf("send bytes: %zu \n", rt);
                }
            }
        }
    }
}

```





服务端程序如下：

```c
#include "lib/common.h"

static int count;

static void sig_int(int signo) {
    printf("\nreceived %d datagrams\n", count);
    exit(0);
}

int main(int argc, char **argv) {
    int listenfd;
    listenfd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in server_addr;
    bzero(&server_addr, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    server_addr.sin_port = htons(SERV_PORT);

    int rt1 = bind(listenfd, (struct sockaddr *) &server_addr, sizeof(server_addr));
    if (rt1 < 0) {
        error(1, errno, "bind failed ");
    }

    int rt2 = listen(listenfd, LISTENQ);
    if (rt2 < 0) {
        error(1, errno, "listen failed ");
    }

    signal(SIGINT, sig_int);
    signal(SIGPIPE, SIG_IGN);

    int connfd;
    struct sockaddr_in client_addr;
    socklen_t client_len = sizeof(client_addr);

    if ((connfd = accept(listenfd, (struct sockaddr *) &client_addr, &client_len)) < 0) {
        error(1, errno, "bind failed ");
    }

    char message[MAXLINE];
    count = 0;

    for (;;) {
        int n = read(connfd, message, MAXLINE);
        if (n < 0) {
            error(1, errno, "error read");
        } else if (n == 0) {
            error(1, 0, "client closed \n");
        }
        message[n] = 0;
        printf("received %d bytes: %s\n", n, message);
        count++;

        char send_line[MAXLINE];
        sprintf(send_line, "Hi, %s", message);

        sleep(5);

        int write_nc = send(connfd, send_line, strlen(send_line), 0);
        printf("send bytes: %zu \n", write_nc);
        if (write_nc < 0) {
            error(1, errno, "error write");
        }
    }
}

```



需要特别注意的是SIGPIPE信号的处理。在收到RST的套接字进行写操作的时候，会直接触发SIGPIPE信号。

启动服务端，再启动客户端，依此在标准输入上输入data1、data2、close，观察输出结果。

```
$./graceclient 127.0.0.1
data1
now sending data1
send bytes:5
data2
now sending data2
send bytes:5
close

```



```
$./graceserver
received 5 bytes: data1
send bytes: 9
received 5 bytes: data2
send bytes: 9
client closed

```

我们发现在客户端输入close之后，调用colse函数，服务端收到SIGPIPE信号，直接退出。客户端并没有受到服务端的应答数据。

![](../img/net-action-1.png)



这样，我们注册一个信号处理函数，对SIGPIPE信号进行处理，避免程序莫名的退出：

```c
static void sig_pipe(int signo) {
    printf("\nreceived %d datagrams\n", count);
    exit(0);
}
signal(SIGINT, sig_pipe);

```

再次，启动服务器，再启动客户端，依此在标准输入上面输入data1,data2,shutdown。观察输出



```
$./graceclient 127.0.0.1
data1
now sending data1
send bytes:5
data2
now sending data2
send bytes:5
shutdown
Hi, data1
Hi，data2
server terminated

```



```
$./graceserver
received 5 bytes: data1
send bytes: 9
received 5 bytes: data2
send bytes: 9
client closed

```



和之前的输出不同，服务端和客户端都输出了双方应该得到的数据，正常退出。

为什么会出现这样的状况呢？

主要是因为客户端调用shutdown的时候只是关闭了连接的一个方向，服务端到客户端的这个方向还可以继续进行数据的发送和接收。当服务端读到EOF时，立即向客户端发送了FIN报文，客户端也在read函数中感知了EOF，也进行了正常的退出。



#### 总结

- close函数只是把套接字的应用技术减一，未必会立即关闭连接；
- close 函数如果套接字的引用计数达到0时，丽姬种植读和写两个方向的数据传送。

基于这两个确定，在期望关闭其中一个方向时，应该使用shutdown函数。