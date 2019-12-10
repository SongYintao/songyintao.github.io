---
title: 深入理解GO-Memory2
subtitle: go内存管理
tags: [go,memory]
layout: post
---

关于go，我前还是停留在使用的阶段，关于内存管理部分，没有深入的去研究。在网上找了几篇不错的博文，学习一下。

[博文地址](https://povilasv.me/go-memory-management-part-2/#)

## Recap

Previously on [povilasv.me](https://povilasv.me/), we explored [Go Memory Management](https://povilasv.me/go-memory-management/) and we left with 2 small Go programs having different virtual memory size.

Firstly, let’s take a look at **ex1** program which consumes a lot of virtual memory. The code for **ex1** looks like this:

```go
func main() {
  http.HandleFunc("/bar", 
      func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello, %q", 
           html.EscapeString(r.URL.Path))
    })

  http.ListenAndServe(":8080", nil)
}
```

In order to view virtual memory size, I executed **ps**. Take a look at the output showing large virtual memory size below. Note that **ps** output is in kibibytes. 388496 KiB is around 379.390625 MiB.

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT 
povilasv 16609  0.0  0.0 388496  5236 pts/9    Sl+
```

Secondly, let’s take a look at **ex2** program, which doesn’t consume a lot of virtual memory:

```go
func main() {
    go func() {
        for {
            var m runtime.MemStats
            runtime.ReadMemStats(&m)

            log.Println(float64(m.Sys) / 1024 / 1024)
            log.Println(float64(m.HeapAlloc) / 1024 / 1024)
            time.Sleep(10 * time.Second)
        }
    }()

    fmt.Println("hello")
    time.Sleep(1 * time.Hour)
}
```

Finally, let’s take a look at **ps** output for this program. As you can see this program runs with small virtual memory size. Note that 4900 KiB is around 4.79 MiB.

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT 
povilasv  3642  0.0  0.0   4900   948 pts/10   Sl+ 
```

It’s important to note that these programs are compiled with a bit dated **Go 1.10**version. There are differences on newer versions of Go.
For example, compiling **ex1** program with Go 1.11 produces **virtual memory size** of 466MiB and **resident set size** of 3.22MiB. Additionally, doing the same with **Go 1.12** provides us with 100.37MiB of virtual memory and 1.44MiB of resident set size.

**As we can see, the difference between HTTP server and a simple command line application made all the difference to virtual memory size.**

## *Aha* moment

Since then I had an aha moment. Maybe I can debug this interesting behavior with **strace**. Take a look at **strace** description:

> [strace](https://strace.io/) is a diagnostic, debugging and instructional userspace utility for Linux. It is used to monitor and tamper篡改 with interactions between processes and the Linux kernel, which include system calls, signal deliveries, and changes of process state.

So the plan is to run both programs with **strace**, in order to compare what is happening on the operating system level. Running and using **strace** is really simple, you just add **strace** in front of compiled program. For instance, in order to trace **ex1** program I executed:

```
strace ./ex1
```

Which produces the following output**:**

- brk和sbrk主要的工作是实现虚拟内存到内存的映射  
- execve 程序的调用执行

```shell
execve("./ex1", ["./ex1"], 0x7fffe12acd60 /* 97 vars */) = 0
brk(NULL)                             = 0x573000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/local/lib/tls/haswell/x86_64/libpthread.so.0", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/usr/local/lib/tls/haswell/x86_64", 0x7ffdaa923fa0) = -1 ENOENT (No such file or directory)
...
stat("/lib/x86_64", 0x7ffdaa923fa0)     = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/lib/libpthread.so.0", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\340b\0\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=146152, ...}) = 0
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fc8a8d11000
mmap(NULL, 2225248, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7fc8a88cd000
mprotect(0x7fc8a88e8000, 2093056, PROT_NONE) = 0
mmap(0x7fc8a8ae7000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1a000) = 0x7fc8a8ae7000
mmap(0x7fc8a8ae9000, 13408, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7fc8a8ae9000
close(3)                                = 0
openat(AT_FDCWD, "/usr/local/lib/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/lib/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\34\2\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=1857312, ...}) = 0
mmap(NULL, 3963464, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7fc8a8505000
mprotect(0x7fc8a86c3000, 2097152, PROT_NONE) = 0
mmap(0x7fc8a88c3000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1be000) = 0x7fc8a88c3000
mmap(0x7fc8a88c9000, 14920, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7fc8a88c9000
close(3)                                = 0
mmap(NULL, 12288, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fc8a8d0e000
arch_prctl(ARCH_SET_FS, 0x7fc8a8d0e740) = 0
mprotect(0x7fc8a88c3000, 16384, PROT_READ) = 0
mprotect(0x7fc8a8ae7000, 4096, PROT_READ) = 0
mprotect(0x7fc8a8d13000, 4096, PROT_READ) = 0
set_tid_address(0x7fc8a8d0ea10)         = 2109
set_robust_list(0x7fc8a8d0ea20, 24)     = 0
rt_sigaction(SIGRTMIN, {sa_handler=0x7fc8a88d2ca0, sa_mask=[], sa_flags=SA_RESTORER|SA_SIGINFO, sa_restorer=0x7fc8a88e1140}, NULL, 8) = 0
rt_sigaction(SIGRT_1, {sa_handler=0x7fc8a88d2d50, sa_mask=[], sa_flags=SA_RESTORER|SA_RESTART|SA_SIGINFO, sa_restorer=0x7fc8a88e1140}, NULL, 8) = 0
rt_sigprocmask(SIG_UNBLOCK, [RTMIN RT_1], NULL, 8) = 0
prlimit64(0, RLIMIT_STACK, NULL, {rlim_cur=8192*1024, rlim_max=RLIM64_INFINITY}) = 0
brk(NULL)                               = 0x573000
brk(0x594000)                           = 0x594000
sched_getaffinity(0, 8192, [0, 1, 2, 3]) = 8
mmap(0xc000000000, 65536, PROT_NONE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0xc000000000
munmap(0xc000000000, 65536)             = 0
mmap(NULL, 262144, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fc8a8cce000
mmap(0xc420000000, 1048576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0xc420000000
mmap(0xc41fff8000, 32768, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0xc41fff8000
mmap(0xc000000000, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0xc000000000
mmap(NULL, 65536, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fc8a8cbe000
mmap(NULL, 65536, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fc8a8cae000
rt_sigprocmask(SIG_SETMASK, NULL, [], 8) = 0
sigaltstack(NULL, {ss_sp=NULL, ss_flags=SS_DISABLE, ss_size=0}) = 0
sigaltstack({ss_sp=0xc420002000, ss_flags=0, ss_size=32768}, NULL) = 0
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
gettid()                                = 2109
...
```

Similarly, let’s trace the **ex2** program and take a look at it’s output:

```shell
strace ./ex2
```

**Output:**

```
execve("./ex2", ["./ex2"], 0x7ffc2965ca40 /* 97 vars */) = 0
arch_prctl(ARCH_SET_FS, 0x5397b0)       = 0
sched_getaffinity(0, 8192, [0, 1, 2, 3]) = 8
mmap(0xc000000000, 65536, PROT_NONE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0xc000000000
munmap(0xc000000000, 65536)             = 0
mmap(NULL, 262144, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7ff1c637b000
mmap(0xc420000000, 1048576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0xc420000000
mmap(0xc41fff8000, 32768, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0xc41fff8000
mmap(0xc000000000, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0xc000000000
mmap(NULL, 65536, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7ff1c636b000
mmap(NULL, 65536, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7ff1c635b000
rt_sigprocmask(SIG_SETMASK, NULL, [], 8) = 0
sigaltstack(NULL, {ss_sp=NULL, ss_flags=SS_DISABLE, ss_size=0}) = 0
sigaltstack({ss_sp=0xc420002000, ss_flags=0, ss_size=32768}, NULL) = 0
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
gettid()                                = 22982
```

Note that actual output is longer. For readability sake, I just copied output for **strace** up to the point, where both programs call **gettid()**. This point was selected for the reason that it appeared only once in both **strace** dumps.

Let’s compare the outputs. 

First of all, **ex1** trace output is way longer than **ex2**. **ex1** l**ooks for some .so libraries and then loads them into memory**. 

For instance, this is how trace output for loading **libpthread.so.0** looks:

```shell
...
openat(AT_FDCWD, "/lib/libpthread.so.0", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\340b\0\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=146152, ...}) = 0
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fc8a8d11000
mmap(NULL, 2225248, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7fc8a88cd000
mprotect(0x7fc8a88e8000, 2093056, PROT_NONE) = 0
mmap(0x7fc8a8ae7000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1a000) = 0x7fc8a8ae7000
mmap(0x7fc8a8ae9000, 13408, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7fc8a8ae9000
close(3)                                = 0
```

In this example, we can see that file is being opened, read into memory and finally closed. Some of those memory regions are mapped with **PROTO_EXEC**flag, which allows our program to execute code located in that memory region. We can see the same thing happening for **libc.so.6**:

```shell
...
openat(AT_FDCWD, "/lib/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\34\2\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=1857312, ...}) = 0
mmap(NULL, 3963464, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7fc8a8505000
mprotect(0x7fc8a86c3000, 2097152, PROT_NONE) = 0
mmap(0x7fc8a88c3000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1be000) = 0x7fc8a88c3000
mmap(0x7fc8a88c9000, 14920, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7fc8a88c9000
close(3)                                = 0
```

After loading these libraries, both programs start to behave similarly. We can see that **strace** outputs after loading, show mapping of the same memory regions and contain traces of running similar commands up to the point, where programs execute **gettid()**.

This is interesting. So **ex1** has loaded **libpthread** & **libc**, but **ex2** didn’t.

**cgo must be involved.**

## CGO

Let’s explore **cgo** and how it works. As [godoc](https://golang.org/cmd/cgo/) explains:

> Cgo enables the creation of Go packages that call C code.

Turns out in order to call C code, you need to add a special comment and use special **C** package. Let’s take a look at this mini example:

```go
package main

// #include 
import "C"
import "fmt"

func main() {
    char := C.getchar()
    fmt.Printf("%T %#v", char, char)
}
```

This program includes **stdio.h** from C standard library and then calls **getchar()**and prints resulting variable. **getchar()** gets a character (an unsigned char) from **stdin**. Let’s try it:

```
go build
./ex3
```

When executing this program, it will ask you to enter a character and simply print it. Here is an example of this interaction:

```
a
main._Ctype_int 97
```

As we can see, it behaves just like Go. Interestingly it compiles just like native Go code, as you just **go build** and that’s it. I bet If you didn’t look at the code, you wouldn’t notice a difference.

Obviously it has a lot more interesting features. For instance, if you place your **.c**and **.h** files in the same directory as native go code, **go build** will compile and link it together with your native code.

I suggest you checkout [godoc](https://golang.org/cmd/cgo/) and [C? Go? Cgo!](https://blog.golang.org/c-go-cgo) blog post if you want to learn more. Now let’s, get back to our interesting problem. Why did **ex1** program use cgo and **ex2** didn’t?

### Exploring difference

The difference between **ex1** and **ex2** programs is import path. **ex1** imports **net/http**, while **ex2** doesn’t. Interestingly, Grepping thru **net/http** package doesn’t show any signs of **C** statements. But it’s enough to look one level up and volia, it’s the **net** package.

Take a look at these **net** package files:

- [net/cgo_unix.go](https://golang.org/src/net/cgo_unix.go)
- [net/cgo_linux.go](https://golang.org/src/net/cgo_linux.go)
- [net/cgo_stub.go](https://golang.org/src/net/cgo_stub.go)

For example, **net/cgo_linux.go** contains:

```go
// +build !android,cgo,!netgo

package net

/*
#include <netdb.h>
*/
import "C"

// NOTE(rsc): In theory there are approximately balanced
// arguments for and against including AI_ADDRCONFIG
// in the flags (it includes IPv4 results only on IPv4 systems,
// and similarly for IPv6), but in practice setting it causes
// getaddrinfo to return the wrong canonical name on Linux.
// So definitely leave it out.
const cgoAddrInfoFlags = C.AI_CANONNAME | C.AI_V4MAPPED | C.AI_ALL
```

As we can see net package includes **C** header file **netdb.h** and uses a couple of variables from there. But why do we need this? Let’s explore it.

### What is netdb.h ?

If you look up **netdb.h** documentation, you will see that **netdb.h** is part of libc. It’s documentation states:

> **netdb.h** it provides definitions for network database operations.

Additionally, documentation explains these constants. Let’s have a look:

- **AI_CANONNAME** – Request for canonical name.
- **AI_V4MAPPED** – If no IPv6 addresses are found, query for IPv4 addresses and return them to the caller as IPv4-mapped IPv6 addresses
- **AI_ALL** – Query for both IPv4 and IPv6 addresses.

Looking at how these flags are actually used, it turns out that they are passed to **getaddrinfo()**, which is a call to resolve DNS name using libc. So simply those flags, control how DNS name resolution is going to take place.

Likewise if you open [net/cgo_bsd.go](https://golang.org/src/net/cgo_bsd.go), you will see a bit diffferent version of **cgoAddrInfoFlags** constant. Let’s take a look:

```go
// +build cgo,!netgo
// +build darwin dragonfly freebsd

package net

/*
#include <netdb.h>
*/
import "C"

const cgoAddrInfoFlags = (C.AI_CANONNAME | C.AI_V4MAPPED |
 C.AI_ALL) & C.AI_MASK
```

So this hints us, that there is a mechanism to set OS specific flags for DNS resolution amd we are using **cgo** to do DNS queries correctly. That’s cool. Let’s explore **net** package a little more.

Have a read thru [**net**](https://golang.org/pkg/net/#hdr-Name_Resolution) package documentation:

> Name Resolution
>
> The method for resolving domain names, whether indirectly with functions like Dial or directly with functions like LookupHost and LookupAddr, varies by operating system.
>
> On Unix systems, the resolver has two options for resolving names. It can use a pure Go resolver that sends DNS requests directly to the servers listed in /etc/resolv.conf, or it can use a cgo-based resolver that calls C library routines such as getaddrinfo and getnameinfo.
>
> By default the pure Go resolver is used, because a blocked DNS request consumes only a goroutine, while a blocked C call consumes an operating system thread. When cgo is available, the cgo-based resolver is used instead under a variety of conditions: on systems that do not let programs make direct DNS requests (OS X), when the LOCALDOMAIN environment variable is present (even if empty), when the RES_OPTIONS or HOSTALIASES environment variable is non-empty, when the ASR_CONFIG environment variable is non-empty (OpenBSD only), when /etc/resolv.conf or /etc/nsswitch.conf specify the use of features that the Go resolver does not implement, and when the name being looked up ends in .local or is an mDNS name.
>
> The resolver decision can be overridden by setting the netdns value of the GODEBUG environment variable (see package runtime) to go or cgo, as in:

```shell
export GODEBUG=netdns=go    # force pure Go resolver
export GODEBUG=netdns=cgo   # force cgo resolver
```

> The decision can also be forced while building the Go source tree by setting the netgo or netcgo build tag.
>
> A numeric netdns setting, as in GODEBUG=netdns=1, causes the resolver to print debugging information about its decisions.
>
> https://golang.org/pkg/net/#hdr-Name_Resolution

Enough reading, let’s play with it and try to use different DNS client implementations.

### Build tags

So as per documentation, we can use environment variables to use one or the other DNS client implementation. This is quite neat as you don’t have to recompile your go code, in order to switch implementations.

Additionally, looking thru the code I found that we can use Go build tags to compile Go binary with different DNS client. In addition we can check, which implementation is actually used, by setting `GODEBUG=netdns=1` environment variable and doing an actual DNS call.

By looking thru **net** package source files, I figured that there is 3 build modes. The build modes are enabled using different build tags. Here is the list:

1. **!cgo –** no cgo means that we are forced to use Go’s resolver.
2. **netcgo** or **cgo** – means that we are using libc DNS resolution.
3. **netgo + cgo** – means that we are using go native DNS implementation, but we can still include C code.

Let’s try all of these combinations to see how this actually works out. 
We need to use different program as none of our programs execute DNS queries. Therefore, we will use this code:

```go
func main() {
    addr, err := net.LookupHost("povilasv.me")
    fmt.Println(addr, err)
}
```

Let’s build it

```
export CGO_ENABLED=0
export GODEBUG=netdns=1
go build -tags netgo
```

And run it:

```
./testnetgo
```

**Output:**

```
go package net: built with netgo build tag; using Go's DNS resolver
104.28.1.75 104.28.0.75 2606:4700:30::681c:4b 2606:4700:30::681c:14b <nil>
```

Now let’s build it with **libc** resolver:

```
export GODEBUG=netdns=1
go build -tags netcgo
Click to copy
```

And run it:

```
./testnetgo
```

**Output:**

```
go package net: using cgo DNS resolver
104.28.0.75 104.28.1.75 2606:4700:30::681c:14b 2606:4700:30::681c:4b <nil>
```

And finally let’s use **netgo cgo**:

```
export GODEBUG=netdns=1
go build -tags 'netgo cgo' .
```

And lastly run it:

```
./testnetgo
```

**Output:**

```
go package net: built with netgo build tag; using Go's DNS resolver
104.28.0.75 104.28.1.75 2606:4700:30::681c:14b 2606:4700:30::681c:4b <nil>
```

We can see that build tags actually work. Now let’s get back to our virtual memory problem.

## Back to Virtual Memory

Now I would like to see how virtual memory behaves with those 3 combinations of build tags with our **ex1** program, which is a simple HTTP web server.

Let’s build it in **netgo** mode:

```
export CGO_ENABLED=0
go build -tags netgo
```

**Process stats:**

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT 
povilasv  3524  0.0  0.0   7216  4076 pts/17   Sl+
```

As we can see virtual memory is low in this case.

Now let’s do the same with **netcgo** on:

```
go build -tags netcgo
```

**Process stats:**

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT 
povilasv  6361  0.0  0.0 382296  4988 pts/17   Sl+
```

As we can see this suffers form large virtual memory size (382296 KiB).

Lastly, let’s try **netgo cgo** mode:

```
go build -tags 'netgo cgo' .
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT 
povilasv  8175  0.0  0.0   7216  3968 pts/17   Sl+
```

And we can see that virtual memory is low in this case (7216 KiB).

So We can definitely roll out the **netgo**, as it doesn’t use a lot of virtual memory. On the other hand, we can’t yet blame **cgo** for bloated virtual memory. As our **ex1**example, doesn’t include any additional C code, so **netgo** would actually compile the same thing as **netgo** **cgo**, skipping the whole **cgo** process of building and linking C files.

Therefore, we need to try **netcgo** and **netgo cgo** with additional C code included. This would check how program behaves **in cgo mode** with **disabled libc DNS clien**t and **cgo mode with enabled libc DNS client**.

Let’s try this example:

```
package main

// #include 
import "C"
import "fmt"

func main() {
  char := C.getchar()
  fmt.Printf("%T %#v", char, char)

  http.HandleFunc("/bar", 
    func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello, %q", 
          html.EscapeString(r.URL.Path))
  })

   http.ListenAndServe(":8080", nil)
}
```

As we can see this example should work well for us. Because, it uses both **cgo** and would use **netgo** or **libc** DNS client implementations based on build tags.

Let’s try it out:

```
go build -tags netcgo .
```

**Process stats:**

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT 
povilasv 12594  0.0  0.0 382208  4824 pts/17   Sl+
```

We can see the same virtual memory behavior. Now let’s try **netgo cgo**:

```
go build -tags 'netgo cgo' .
```

**Process stats:**

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT 
povilasv  1026  0.0  0.0 382208  4824 pts/17   Sl+ 
```

Finally, we can rule out libc DNS client implementation, as disabling it didn’t help. We can clearly see it is related to **cgo**.

To explore this further, let’s simplify our program. **ex1** program starts an HTTP server, which would be way harder to debug than a simple command line application. So take a look this code:

```go
package main

// #include 
// #include 
import "C"
import (
    "time"
    "unsafe"
)

func main() {
    cs := C.CString("Hello from stdio")
    C.puts(cs)

    time.Sleep(1 * time.Second)

    C.free(unsafe.Pointer(cs))
}
```

Let’s run it and check it’s memory:

```
go build .
./ex6
```

**Process stats:**

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT 
povilasv 15972  0.0  0.0 378228  2476 pts/17   Sl+
```

Cool virtual memory is big, this means that we need to investigate研究调查 **cgo**.

Checkout [Go Memory Management Part 3](https://povilasv.me/go-memory-management-part-3/) for deeper investigation.

That’s it for the day. If you are interested to get my blog posts first, join the [newsletter](https://povilasv.me/newsletter). If you want to support my writing, I have a public [wish list](https://www.amazon.com/hz/wishlist/ls/2NLKE1Z1SND3W?ref_=wl_share), you can buy me a book or a whatever

Thanks for reading & see you next time!



# GO MEMORY MANAGEMENT PART 3

Previously on [povilasv.me](https://povilasv.me/), we explored [Go Memory Management](https://povilasv.me/go-memory-management/) and [Go Memory Management part 2](https://povilasv.me/go-memory-management-2/). 

In last blog post, we found that when using **cgo**, virtual memory grows. Let’s deep dive into **cgo**.



## CGO internals

So as we can see, once we start using **cgo** virtual memory grows. Moreover for most users the switch to **cgo** happens automatically, once they import **net** package or any package **net**‘s child package (like **http**).

I found a lot of documentation on how **cgo** calls work inside actualstandard library code. For example looking into [cgocall.go](https://golang.org/src/runtime/cgocall.go), you will find, really helpful comments:

> To call into the C function f from Go, the cgo-generated code calls **runtime.cgocall(_cgo_Cfunc_f, frame)**, where **_cgo_Cfunc_f** is a **gcc-compiled function written by cgo**.
>
> **runtime.cgocall** (below) calls **entersyscall** so as not to block other goroutines or the garbage collector, and then calls **runtime.asmcgocall(_cgo_Cfunc_f, frame)**.
>
> **runtime.asmcgocall** (in asm_$GOARCH.s) switches to the m->g0 stack (assumed to be an operating system-allocated stack, so safe to run gcc-compiled code on) and calls **cgo_Cfunc_f(frame)**.
>
> **_cgo_Cfunc_f** invokes the actual C function f with **arguments taken from the frame structure**, **records the results in the fram**e, and returns to **runtime.asmcgocall**.
>
> After it regains control, **runtime.asmcgocall** switches back to the original g (m->curg)’s stack and returns to **runtime.cgocall**.
>
> After it regains control, **runtime.cgocall** calls **exitsyscall**, which blocks until this m can run Go code without violating the $GOMAXPROCS limit, and then unlocks g from m.
>
> The above description skipped over the possibility of the gcc-compiled function f calling back into Go. If that happens, we continue down the rabbit hole during the execution of f.
>
> cgocall.go source (https://golang.org/src/runtime/cgocall.go)

The comments go even deeper about how go implements a call from **cgo** to **go**. **I encourage you to explore the code and the comments.** I learnt a lot by looking under the covers. From these comments, we can see that behavior is completely different when Go calls out to C vs when it doesn’t.

## Runtime Tracing

One cool way of exploring Go behaviour, is to use Go runtime tracing. Checkout [Go Execution Tracer](https://blog.gopheracademy.com/advent-2017/go-execution-tracer/) blog post for more detailed information around Go tracing. For now, let’s change our code to add tracing:

```go
func main() {
    trace.Start(os.Stderr)
    cs := C.CString("Hello from stdio")

    time.Sleep(10 * time.Second)

    C.puts(cs)
    C.free(unsafe.Pointer(cs))

    trace.Stop()
}
```

Let’s build it and forward standard error output to a file:

```
/ex7 2> trace.out 
```

Lastly, we can view the trace:

```
go tool trace trace.out
```

That’s it. Next time when I have weirdly behaving command line application I know how to trace it ![🙂](https://s.w.org/images/core/emoji/12.0.0-1/svg/1f642.svg) By the way if you want to trace web server, GO has **httptrace** packge, which is even simpler to use. Checkout [HTTP Tracing](https://blog.golang.org/http-tracing) blog post for more information.

So I’ve compiled this program and a similar program without any C statements and compared the traces using **go tool trace**. This is how the go native code looks:

```go
func main() {
   trace.Start(os.Stderr)
   str := "Hello from stdio"
   time.Sleep(10 * time.Second)
   fmt.Println(str)

   trace.Stop()
}
```

There isn’t much difference between traces in **cgo** program & native **go** program. I’ve noticed that some stats are a bit different. For example, **cgo** program didn’t include **Heap** profile in trace section.

![img](https://povilasv.me/wp-content/uploads/2019/05/cgo.png)

​														cgo program’s trace statistics

![img](https://povilasv.me/wp-content/uploads/2019/05/noncgo.png)

​													Go native program’s trace statistics

I explored a bunch of different views, but didn’t see any more significant differences. **My guess is that Go doesn’t add traces for compiled C code.**

So I decided to explore the differences using **strace**.

## Exploring cgo with strace

Just to clarify we will be exploring 2 programs, both of them are sort of doing the same thing. The same exact programs, just go tracing removed.

**cgo program:**

```go
func main() {
    cs := C.CString("Hello from stdio")

    time.Sleep(10 * time.Second)

    C.puts(cs)
    C.free(unsafe.Pointer(cs))
}
```

**Go native program**:

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    str := "Hello from stdio"
    time.Sleep(10 * time.Second)

    fmt.Println(str)
}
```

To **strace** those programs, build them and run:

```
sudo strace -f ./program_name
```

I’ve added **-f** flag, which will make strace to also follow threads.

> **-f** Trace child processes as they are created by currently traced processes as a result of the fork(2), vfork(2) and clone(2) system calls.

### cgo results

As we saw previously **cgo** programs load **libc** & **pthreads** C libraries in order to perform their work. Also, as it turns out, **cgo** programs create threads differently. When a new thread is created, you would see a call to allocate **8mb** of memory for Thread stack:

```
mmap(NULL, 8392704, PROT_NONE, MAP_PRIVATE|MAP_ANONYMOUS|MAP_STACK, -1, 0) = 0x7f1990629000
// We allocate 8mb for stack
mprotect(0x7f199062a000, 8388608, PROT_READ|PROT_WRITE) = 0
// Allow to read and write, but no code execution in this memory region.
```

After stack is setup, you would see a call to **clone** system call, which would have different arguments than a typical go native program:

```shell
clone( child_stack=0x7f1990e28fb0, flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_SYSVSEM|CLONE_SETTLS|CLONE_PARENT_SETTID|CLONE_CHILD_CLEARTID, parent_tidptr=0x7f1990e299d0, tls=0x7f1990e29700, child_tidptr=0x7f1990e299d0) = 3600
```

IF you are intresetd what those arguments mean, check out their descriptions below (taken from **clone** man pages):

> **CLONE_VM** – the calling process and the child process run in the same memory space.The arguments here:
> **CLONE_FS** – the caller and the child process share the same filesystem information. 
> **CLONE_FILES** – the calling process and the child process share the same file descriptor table. 
> **CLONE_SIGHAND –** the calling process and the child process share the same file descriptor table. **CLONE_THREAD** – the child is placed in the same thread group as the calling process.**CLONE__SYSVSEM** – then the child and the calling process share a single list of System V semaphore adjustment values.**CLONE_SETTLS** – The TLS (Thread Local Storage) descriptor is set to newtls.**CLONE_PARENT_SETTID** – Store the child thread ID at the location ptid in the parent’s memory. **CLONE_CHILD_CLEARTID** – Store the child thread ID at the location ctid in the child’s memory.
>
> man pages for clone system call

**After a clone call, a thread would reserve 保留 128mb of ram,** then unreserve 释放  57.8mb and 8mb. Take a look at the **strace** section below:

```shell
mmap(NULL, 134217728, PROT_NONE, MAP_PRIVATE|MAP_ANONYMOUS|MAP_NORESERVE, -1, 0) = 0x7f1988629000
//134217728 / 1024 / 1024 = 128 MiB
munmap(0x7f1988629000, 60649472 )
// Remove memory mapping from 0x7f1988629000 to + 57.8 MiB
munmap(0x7f1990000000, 6459392)
// Remove memory mapping from 0x7f1990000000 to + 8 MiB
mprotect(0x7f198c000000, 135168, PROT_READ|PROT_WRITE 
```

Now this totally makes sense. In **cgo** programs we saw around 373.25 MiB virtual memory and this 100% explains it. Even more it actually explains why I haven’t seen the memory mappings in **/proc/PID/maps** in the [first part of the article](https://povilasv.me/go-memory-management/). It’s threads reserving the memory and they have their own PIDs. In addition, as threads do a call to **mmap**, but never actually use that memory region, this won’t be accounted in resident set size, but would be in virtual memory size.

**Let’s do some napkin calculations:**

There was 5 calls to **clone** system call in strace output. This reserves 8mib for stack + 128 MiB, then unreserves 57.8 MiB and 8 Mib. This ends up in ~70 MiB per thread. Also one thread actually reserved 128 MiB but didn’t **unmap** anything and another one didn’t **unmap** 8 MiB. So the calculation looks as follows:

4 * 70 + 8 + 1 * 128 = ~ 416 MiB.

Additionally let us not forget that there is some additional memory reservations on program initialization. So + some constant.

Obviously it’s super hard to figure out at which point we sampled the memory (executed **ps**), i.e. we could have executed **ps** only when 2 or 3 threads were running, memory could have been **mmaped** but not released, etc. So, in my opinion this is the explanation I was looking for when I originally started the [Go Memory Management](https://povilasv.me/go-memory-management/) blog post.

If you are interested what those arguments to **mmap** mean, here are the definitions:

> **MAP_ANONYMOUS** – The mapping is not backed by any file; its contents are initialized to zero.
> **MAP_NORESERVE** – Do not reserve swap space for this mapping.
> **MAP_PRIVATE** – Create a private **copy-on-write mapping**. Updates to the mapping are not visible to other processes mapping the same file, and are not carried through to the underlying file.
>
> man pages for mmap system call

Lastly, let’s take a look how native go programs create threads.

### Go native results

In go native code there were only **4 calls** to **clone** system calls. None of the newly created threads did memory allocation (there were no **mmap** calls). Additionally there were no 8MiB reservation for stack. This is roughly how go native creates a thread:

```shell
clone( child_stack=0xc420042000, flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_SYSVSEM) = 3935
```

Note the difference between go and cgo arguments for **clone** calls.

Additonally. in go native code you can clearly see the **Println** statement in **strace**:

```shell
[pid  3934] write(1, "Hello from stdio\n", 17Hello from stdio
) = 17
```

Somehow I couldn’t find a similar system call for **fputs()**, statement in cgo version.

What I love about native Go, is that **strace** output is way smaller and just easier to understand. There are generally less things happening. For instance **strace** for go native produced 281 lines of output, compared to 342 for cgo.

## Conclusion

If there is something you can take away from my exploration is that:

- Go might **auto-magically switch to cgo**, if package that uses C is involved. For instance, **net**, **net/http** packages.
- Go **has two DNS resolver implementations**: **netgo** and **netcgo**.
- You can learn which DNS client you are using via **export GODEBUG=netdns=1**environment variable.
- You can change them during runtime via **export GODEBUG=netdns=go** and **export GODEBUG=netdns=go** environment variables.
- You can compile with one DNS implementation via go build tags. **go build -tags netcgo** and **go build -tags netgo**.
- **/proc** filesystem is useful, but don’t forget about the threads!
- **/prod/PID/status** and **/proc/PID/maps** can be helpful to **quickly dump, whats going on**.
- **Go Execution Tracer** can help to debug your software.
- **strace -f**, when you don’t know what to do.

Finally:

- cgo is different from Go.
- Big virtual memory **isn’t bad**.
- Newer versions of Go behave differently, so what’s true for Go 1.10 isn’t for Go 1.12.

If you take anything from this blogpost, please just don’t take that you need to build go programs with `CGO_ENABLED=0`. There are reasons that Go authors decided to do the way it’s done. And this behavior might change in future as it changed for Go 1.12.

That’s it for the day. If you are interested to get my blog posts first, join the [newsletter](https://povilasv.me/newsletter). If you want to support my writing, I have a public [wish list](https://www.amazon.com/hz/wishlist/ls/2NLKE1Z1SND3W?ref_=wl_share), you can buy me a book or a whatever ![😉](https://s.w.org/images/core/emoji/12.0.0-1/svg/1f609.svg).

Thanks for reading & see you next time!