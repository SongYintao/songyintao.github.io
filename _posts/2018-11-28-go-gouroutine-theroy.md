---
title: Go高级：goroutine原理
subtitle: goroutine原理
tags: [go]
layout: post
---

​	从根本上说，goroutine就是一种Go语言版本的协程。因此，要理解goroutine的运行原理，关键就是理解传统意义上协程的工作原理。

​	协程这个术语应该是随着Lua语言的流行而流行起来，第一次出现时用于汇编编程。

### 1. 协程

​	协程，也可以称为轻量级线程。具备以下几个特点：

- 能够在单一的系统线程中模拟多个任务的并发执行。
- 在一个特定的时间，只有一个任务在运行，即并非真正地执行。
- 被动的任务调度方式，即任务没有主动抢占时间片。当一个任务正在执行时，外部没有办法终止它。要进行任务切换，只能通过由该任务自身调用yield()来主动出让CPU使用权。
- 每个协程都有自己的堆栈和局部变量

​      每个协程都包含3种运行状态：**挂起**、**运行**和**停止**。**停止**通常表示该协程已经执行完成（包括遇到问题后明确退出执行的情况），挂起表示该协程尚未执行完成，但出让了时间片，以后有机会时会由调度器继续执行。

### 2.协程的c语言实现

​	为了更好的剖析协程的运行原理，从轻量级双人协程库`libtask`入手。goroutine和用于goroutine之间通信的channel都是参照libtask库实现的。

### 3. 协程库概述

​	这个libtask库实现了以下几个关键模块：

- [ ] 任务及任务管理

- [ ] 任务调度器

- [ ] 异步IO

- [ ] channel

  这个静态库直接提供了一个main()入口函数作为协程的驱动，因此库的使用者只需要按该库约定的规则实现任务函数taskmain()，启动后这些任务自然会被以协程的方式创建和调度执行。taskmain()函数的声明如下：

  ```c
  void taskmain(int argc,char *argv[]);
  ```

看一个例子`primes.c`，该程序从命令行得到一个整型数作为质数的查找范围，比如用户输入了100，则该程序会列出0到100之间的所有质数，具体代码如下：

```c
 #include <stdio.h>
 #include <stdlib.h>
 #include <unistd.h>
 #include <task.h>

int quiet;
int goal;
int buffer;
void primetask(void *arg){
    Channel *c,*nc;
    int p,i;
    c=arg;
    
    p=chanrecvul(c);
    if(p >goal)
        taskexitall(0);
    if(!quiet)
        printf("%d\n",p);
    nc=chancreate(sizeof(unsigned long),buffer);
    taskcreate(primetask,nc,32768);
    for(;;){
        i=chanrecvul(c);
        if(i%p)
            chansendul(nc,i);
    }
}

void taskmain(int argc,char **argv){
    int i;
    Channel *c;
    if(argc>1)
        goal=atio(argv[1])
    else
        goal=100;
    printf("goal=%d\n",goal);
    
    c=chancreate(sizeof(unsigned long),buffer);
    taskcreate(primetask,nc,32768);
    for(i=2;;i++)
        chansendul(c,i);
        
}
```



go的实现如下：

```go
var goal int
func primetask(c chan int){
    p:=<-c
    if p>goal{
        os.Exit(0)
    }
    fmt.Println(p)
    
    nc:=make(chan int)
    
    go primetask(nc)
    for{
        i:=<-c
        if i%p!=0{
            nc<-i
        }
    }
}

func main(){
    flag.Parse()
    args:=flag.Args()
    if args!=nil&&len(args)>0{
        var err error
        goal,err=strconv.Atoi(args[0])
        if err!=nil{
            goal=100
        }
    }else{
        goal=100
    }
    fmt.Println("goal=",goal)
    c:=make(chan int)
    go primetask(c)
    for i:=2;;i++{
        c<-i
    }
}
```

​	

### 4. 任务

​	从上面的例子可以看出，在实现了一个任务函数后，真要让这个函数加入到调度队列中，我们还需要显式调用taskcreate()函数。下面我们大致介绍一下任务的概念，以及taskcreate() 到底做了哪些事情。 

​	任务用以下的结构表达: 

```c
struct Task{
    char name[256];
    char state[256]; //状态
    Task *next;
    Task *prev;
    Task *allnext;
    Task *allprev;
    Context context;//任务上下文
    uvlong alarmtime;
    unit id;
    uchar *stk;//栈
    uint stksize;
    int exiting;
    int alltaskslot;
    int system;
    int ready;
    void (*startfn)(void*);//该任务所对应的业务函数
    void *startarg; //任务的调用参数
    void *udata;    
};
```



可以看到，每个任务需要保存这几个关键数据：

- [ ] 任务上下文，用于任务切换时保存当前任务的运行环境
- [ ] 栈
- [ ] 状态
- [ ] 该任务所对应的业务函数
- [ ] 任务的调用参数
- [ ] 之前和之后的任务

分析一下任务的创建过程：

```c
static int taskidgen;
/**
void (*fn)(void*)://任务函数
stack//栈大小
*/
static Task* taskalloc(void (*fn)(void*),void *arg,uint stack){
    Task *t;
    sigset_t zero;
    uint x,y;
    ulong z;
    /*一起分配任务和栈需要的内存*/
    t=malloc(sizeof *t+stack);
    if(t==nil){
        fprint(2,"taskalloc malloc:%r\n");
        abort();
    }
    memset(t,0,sizeof *t);
    t->stk=(uchar*)(t+1);
    t->stksize=stack;
    t->id=++taskidgen;
    t->startfn=fn;
    t->startarg=arg;
    
    /*初始化*/
    memset(&t->context.uc,0,sizeof t->context.uc);
    sigemptyset(&zero);
    sigprocmask(SIG_BLOCK,&zero,&t->context.uc.uc_sigmask);
    
    /*必须使用当前的上下文初始化*/
    if(getcontext(&t->context.uc)<0){
        fprint(2,"getcontext"%r\n);
        abort();
    }
    /* 调用makecontext来完成真正的工作 */
	/*头尾都留点空间*/
	t->context.uc.uc_stack.ss_sp = t->stk+8; 
    t->context.uc.uc_stack.ss_size = t->stksize-64;
	#if defined(__sun__) && !defined(__MAKECONTEXT_V2_SOURCE)/* sigh */ 
    #warning "doing sun thing" t->context.uc.uc_stack.ss_sp =(char*)t->context.uc.uc_stack.ss_sp+t->context.uc.uc_stack.ss_size; 
    #endif
    //print("make %p\n", t);
    z = (ulong)t;
    y = z;
    z >>= 16;
	x = z>>16;
	makecontext(&t->context.uc, (void(*)())taskstart, 2, y, x);
	return t;
}

int taskcreate(void (*fn)(void*),void *arg,uint stack){
    int id;
    Task *t;
    t=taskalloc(fn,arg,stack);
    taskcount++;
    id=t->id;
    if(nalltask%64==0){
        alltask=realloc(alltask,(nalltask+64)*sizeof(alltask[0]));
        if(alltask==nil){
            fprint(2,"out of memory\n");
            abort();
        }
    }
    t->alltaskslot=nalltask;
    alltask[nalltask++] = t; 
    taskready(t);
	return id;
}
```

​	可以看到，这个过程其实就是创建并设置了一个Task对象，然后将这个对象 加到alltask 列表中，接着将该Task对象的状态设置为就绪，表示该任务可以接受调度器的调度。 

### 5. 任务调度

​	上面提到了任务列表alltask，那么到底就绪的这些任务是如何被调度的呢?我们可以看一下调度器的实现，整个代码量也不是很多:

```c
static void taskscheduler(void) {
	int i; 
    Task *t;
	taskdebug("scheduler enter"); 
    for(;;){
		if(taskcount == 0) 
            exit(taskexitval); 
        t = taskrunqueue.head;
		if(t == nil){
			fprint(2, "no runnable tasks! %d tasks stalled\n", taskcount); 
            exit(1);
        }
        deltask(&taskrunqueue, t);
        t->ready = 0;
        taskrunning = t;
        tasknswitch++;
        taskdebug("run %d (%s)", t->id, t->name);
        contextswitch(&taskschedcontext, &t->context);
        //print("back in scheduler\n");
        taskrunning = nil;
        if(t->exiting){ 
            if(!t->system)
    			taskcount--;
			i = t->alltaskslot;
			alltask[i] = alltask[--nalltask];
			alltask[i]->alltaskslot = i;
			free(t);
        }
    }
}
```

​	逻辑其实很简单，就是循环执行正在等待中的任务，直到执行完所有的任务后退出。这个函数里根本没有调用任务所对应的业务函数的代码，那么那些代码到底是怎么执行的呢?最关键的是下面这一句调用: 

```c
    contextswitch(&taskschedcontext, &t->context);
```

### 6. 上下文切换

要理解函数执行过程中的上下文切换，需要理解linux底层的系统函数：makecontext()和swapcontext()。

```c
#include<stdio.h>
#include<ucontext.h>

static ucontext_t ctx[3];

static void f1(void){
    puts("start f1");
    swapcontext(&ctx[1],&ctx[2]);
    puts("finish f1");
}

static void f2(void){
    puts("start f2");
    swapcontext(&ctx[2],&ctx[1]);
    puts("finish f2");
}
int main(void){
    char st1[8192];
    char st2[8192];
    getcontext(&ctx[1]);
    ctx[1].uc_stack.ss_sp=st1;
    ctx[1].uc_stack.ss_size=sizeof st1;
    ctx[1].uc_link=&ctx[0];
    makecontext(&ctx[1],f1,0);
    
    getcontext(&ctx[2]);
    ctx[2].uc_stack.ss_sp=st2;
    ctx[2].uc_stack.ss_size=sizeof st2;
    ctx[2].uc_link=&ctx[1];
    makecontext(&ctx[2],f2,0);
    
    swapcontext(&ctx[0],&ctx[2]);
    return 0;
}

```

执行结果为：

```c
start f2
start f1
finish f2
finish f1
```

​	主函数里的swapcontext()调用将导致f2()函数被调用，因为ctx[2]中填充的内容为f2()函数的执行信息。在执行f2()的过程中又遇到一次swapcontext()调用，这次切换到了f1()函数。以此类推。

​	因为在taskalloc()中的最后一行，我们可以看到每一个任务的上下文被设置为taskstart()函数相关，所以一旦调用swapcontext()切换到任务所记录的上下文，则将会导 taskstart()函数被调用，从而在taskstart()函数中进一步调用真正的业务函数，比如上例中的primetask()函数就是这么被调用到的(被设置为任务的startfn成员)。下面是taskstart()函数的具体实现代码:

```c
static void taskstart(uint y,uint x){
    Task *t;
    ulong z;
    z=x<<16;
    z<<16;
    z|=y;
    t=(Task*)z;
    
    t->startfn(t->startarg);
     taskexit(0);
}
```

​	上下文切换的原理我们基本上已经解释完 ，那么到底什么时候应该发生上下文切换呢?我们知道，在任务的执行过程中发生任务切换只会因为以下原因之一:

- [ ] 该任务的业务代码主动要求切换，即主动让出执行权；
- [ ] 发生IO，导致执行阻塞；

第一种，主动出让执行权。主压实通过主动调用taskyield()来完成。在下面的代码中，taskswitch()切换上下文以具体做到任务切换，taskready()函数将一个具体的任务设置为等待执行状态，taskyield()则借助其他的函数完成执行权出让：

```c
void taskswitch(void){
    needstack(0);
    contextswitch(&taskrunning->context,&taskschedcontext);
}

void taskready(Task *t){
    t->ready=1;
    addtask(&taskrunqueue,t);
}

int taskyield(void){
    int n;
    n=taskswitch;
    taskready(taskrunning);
    taskstate("yield");
    taskswitch();
    return taskswitch-n-1;
}
```

上面的带面执行了如下事情：

- [ ] 将正在执行的任务放回到等待队列中，避免后面永远也再切换回来；
- [ ] 将该任务的状态设置为yield；
- [ ] 进行任务的切换。



​	libtask如何在一个任务遇到阻塞的IO动作时自动让出执行 。库中
的fd.c进行了基于轮训的异步IO封装，并在tcpproxy.c中示范了如何使用异步IO来达成自动出让执行权的效果。这里不再解释整个流程，而把注意力放在以下这个底层函数的理解上:

```c
void fdtask(void *v){
    int i,ms;
    Task *t;
    uvlong now;
    tasksystem();
    taskname("fdtask");
    for(;;){
        /*让给其他任务执行*/
        while(taskyield()>0)
            ;
        //我们是唯一在运行的一个，使用poll来等待IO事件
        errno=0;
        taskstate("poll");
        if((t=sleeping.head)==nil)
            ms=-1;
        else{
            //等待最多5s
            now=nsec();
            if(now >=t->alarmtime)
                ms=0;
            elseif(now+5*1000*1000L>=t->alarmtime)
                ms=(t->alarmtime-now)/1000000;
            else
                ms=5000;
        }
        if(poll(pollfd,npollfd,ms)<0){
            if(errno==EINTR)
                continue;
            fprint(2,"poll: %s\n",strerror(errno));
            taskexitall(0);
        }
        //激活对应的任务
        for(i=0;i<npollfd;i++){
            while(i<npollfd&&pollfd[i].revents){
                taskready(polltask[i]);
                --npollfd;
                pollfd[i]=pollfd[npollfd];
                polltask[i]=polltask[npollfd];
            }
        }
        now=nsec();
        while((t=sleeping.head)&&now>=t->alarmtime){
            deltask(&sleeping,t);
            if(!t->system&&--sleepingcounted==0){
                taskcount--;
            }
            taskready(t);
        }
    }
    
    
}
```



​	当发生IO事件时，程序会先让其他处于yield 态的任务先执行,待清理掉这些可以执行的 任务后，开始调用poll来监听所有处于IO阻塞状态的pollfd，一旦有某些pollfd成功读写， 则将对应的任务切换为可调度状态。此时，IO阻塞导致自动切换的过程就完整展现在我们面前了。



### 7. 通信机制

​	channel总是随着goroutine出现，所以我们了解一下channel的原理也有好处。libtask中也提供了channel的参考实现。

​	channel是推荐的goroutine之间的通信方式。“通信”这个词不太适合。从根本上说，channel只是一个数据结构，可以被写入数据，也可以被读取数据。所谓的发送数据到channel或者从channel读取数据，实际上就是对一个数据结构的操作。

​	channel的数据结构如下：

```c
struct Alt{
    Channel *c;
    void *v;
    unsigned int op;
    Task* task;
    Alt* xalt;
};
struce Altarray{
    Alt **a;
    unsigned int n;
    unsigned int m;
};
struct Channel{
  	unsigned int bufsize;
   	unsigned int elemsize;
    unsigned char *buf;
    unsigned int nbuf;
    unsigned int off;
    Altarray asend;
    Altarray yarecv;
    char *name;
};
```

可以看到channel的基本组成如下：

- [ ] 内存缓存，用于存放元素；
- [ ] 发送队列；
- [ ] 接受队列。

​	从以下这个channel的创建函数可以看出，分配的内存缓存就紧跟在这个channel结构之后:

```c
Channel* chancreate(int elemsize,int bufsize){
    Channel *c;
    c=malloc(sizeof *c+bufsize*elemsize);
    if(c==null){
        fprint(2,"chancreate malloc: %r");
        exit(1);
    }
    memset(c,0,sizeof *c);
    c->elemsize=elemsize;
    c->bufsize=bufsize;
    c->nbuf=0;
    c->buf=(uchar*)(c+1);
    return c;
}
```

​	因为协程原则上不会出现多线程编程中经常遇到的资源竞争问题，所以这个channel的数据结构在访问的时候都不用加锁(因为Go语言支持多CPU核心并发执行多个goroutine，会造成资源竞争，所以在必要的位置还是需要加锁的)。 

​	在理解了数据结构后，我们基本上可以知道这个数据结构会如何用于处理发送和接收数据，所以这里就不再 对此主题展开讨论。



