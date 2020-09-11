---
title: Unix Domain Socket
tags:
  - socket
  - network
categories:
  - network
abbrlink: e850c11
date: 2020-09-05 22:06:00
---

平常工作中，大家或多或少都听过或者使用过 Unix Domain Socket, 但可能没有系统的总结梳理过，比如与 TCP/UDP Socket 差别在哪里，除了普通的数据传递还有哪些玩法，能否利用 tcpdump 抓取流量呢? 该文章主要从 Unix Domain Socket 是什么、怎么用、性能如何以及如何抓包排障这些进度进行介绍。

<!--more-->

## 是什么

Unix Domain Socket 也叫做 IPC Socket，与其他 IPC 的机制 Signal、Pipe、FIFO、Message Queue、Semaphore 和 Shared Memory 类似，都可以用于同一台机器上不同进程间的数据传输。

接口上与我们平常接触到的 Internet Socket 基本相同，用户如果想从 TCP/UDP Socket 切换到 Unix Domian Socket，代码几乎不用做变更。不过，不同的是其底层实现不依赖任何 Network Protocol，发送数据时发送方直接将数据写到接收方的 Socket Buffer, 而不会涉及到任何 TCP/IP 头部添加、校验和计算、Packet 分段、Packet 确认、窗口变化以及路由选择等操作，理论上开销更小，有着更好的性能。

另外，Unix Domain Socket 不使用 Internet Socket 的 ip:port 作为地址，而是用文件系统作为地址命名空间，比如 /var/run/docker.sock 是 redis-server 监听的地址，redis-cli 可以指定该地址与 server 建立连接。

## 如何用

### 普通收发

使用方式上与平常接触的 TCP/UDP Socket 基本一致，下面是一些具体的例子:

#### HTTP

{%gist kirk91/7569d02256566b0179963a35a63f92eb %}

在一个终端启动上面的程序:

```shell
$ go run http_unix.go
```

在另外一个终端上用 curl 进行测试:

```shell
$ # query over the unix socket
$ curl --unix-socket /tmp/http-unix.sock http://unix
Hello, world!

$ curl --unix-socket /tmp/http-unix.sock http://unix/path1
Hello, world!

$ curl --unix-socket /tmp/http-unix.sock http://unix/path2
Hello, world!
```

#### TCP

{%gist kirk91/a6c8f6d07c754980b69ffc42127812a2 %}

在一个终端上启动上面的程序:

```shell
$ go run tcp_unix.go
```

在另外一个终端上使用 netcat 进行测试:
> netcat 有两个版本，分别是 netcat-traditional 和 netcat-openbsd，traditional 版本不支持 Unix Domain Socket(-U)，测试时要首先确认自己安装的是 openbsd 重写的。

```shell
$ # connect to the socket
$ nc -U /tmp/tcp-unix.sock
hello
hello
hi
hi
bla bla bla
bla bla bla
^C
```

认真查看上面的两段代码，会发现每次在执行 `net.Listen` 前都会先执行 `os.Remove(udsPath)` 把 socket 文件删除掉，初次看到的话应该会感到非常奇怪，会好奇为啥要这么玩，是不是写错了? 

其实不是写错了，这么写是有意为之，原因是 Unix Socket 不像 Internet Socket, 进程退出(包括 Crash)时操作系统不会自动清理掉创建的 Socket 文件，为了保证本进程能够成功监听，必须确保先前创建的 Socket 文件被删除掉，否则会一直报 `bind: address already in use` 的错误。

除此之外还可能会有权限的问题，比如在 Linux 当前的实现中，要求 Socket 的创建者需要拥有该文件所在目录的 write 和 search(execute) 权限，连接到 Socket 的一方需要有该文件的 write 权限，否则会抛出 Permission denied 的错误。

因此，如果使用文件系统作为地址命名空间的话，都需要处理这两种情况，不然程序会抛出错误无法运行。但是，在 Linux 上有第二种选择，可以使用 Abstract Socket Namspace。具体来说，Linux 额外开发了一个叫做 Abstract Namespace 的特性，它允许我们创建一个 Socket 而不用绑定文件，甚至在进程退出(无引用)时会被自动清理掉，下面是一个具体的例子:

{%gist kirk91/688cf5a2b8fb8dcc879f420552ff827f %}

开启一个终端执行上面程序:

```shell
$ go run tcp_abstract_unix.go
```

另开一个终端用 socat 进行测试:
> - netcat 不支持 abstract socket
> - socat 介绍 https://medium.com/@copyconstruct/socat-29453e9fc8a6

```shell
$ # show tcp-unix.sock
$ ss -xlp | grep tcp-unix.sock
u_str LISTEN 0 16384  @tcp-unix.sock 136766257 * 0 users:(("tcp_abstract_un",pid=1878602,fd=3))
$ # connect to the abstract socket
$ socat - ABSTRACT-CONNECT:tcp-unix.sock
hello
hello
world
world
😊
😊
^C
```

### FD 传递

Unix Socket 除了能够传输普通的数据外，还能够在完全不相关(非父子)的进程间传递 FD, 很多开源的项目 HAProxy、Nginx 和 Envoy 都有用到该特性。下面是一个将 HTTP Listener 通过 Unix Socket 从一个进程传递到另外一个进程的例子:

- fd_send.go: 发送方
- fd_receive.go: 接受方

{%gist kirk91/ec25703848172e8f56f671e0e1c73751 %}

开启两个终端分别启动 fd_send.go 和 fd_receive.go 程序:

```shell
$ go run fd_send.go
2020/08/10 19:02:19 Server is listening on 127.0.0.1:8080 ...
```

```shell
$ go run fd_receive.go
2020/08/10 19:17:58 Wait receiving listener ...
```

另外开启一个终端进行测试:

```shell
$ # query, receive the response from server1
$ curl http://localhost:8080
[server1] Hello, world!
$ # trigger passing file descriptor
$ curl http://localhost:8080/passfd
Success
$ # query again, receive the response from server2
$ curl http://localhost:8080
[server2] Hello, world!
```

![](https://i.imgur.com/wqCSh9n.png)

> 1. Unix Socket 不支持 SO_REUSEPORT, 多个进程没法直接监听同一 Socket
> 2. FD 传递，并不是将 FD 的值传递给另一个进程，而是传递的内核同一 File 结构的引用，两者都指向内核 Open File Table 的同一个 File, 两个 FD 的值不要求一样，并且在现实中大概率是不同的

从测试结果看，HTTP Listener 的 FD 成功的传递到了接受方，并且接受方能够正常处理用户请求。之所以能够通过 Unix Socket 能够传递 FD，是因为数据发送调用了 sendmg 接口，其参数 msg 支持携带辅助数据，下面是他们的签名:

```cpp
ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags);

struct msghdr {
    void         *msg_name;       /* optional address */
    socklen_t     msg_namelen;    /* size of address */
    struct iovec *msg_iov;        /* scatter/gather array */
    size_t        msg_iovlen;     /* # elements in msg_iov */
    void         *msg_control;    /* ancillary data, see below */
    size_t        msg_controllen; /* ancillary data buffer len */
    int           msg_flags;      /* flags on received message */
};
```

其中 msg_control 和 msg_controllen 是用来传递辅助数据的，msg_control 对应的数据结构定义如下:

```cpp
struct cmsghdr {
    socklen_t cmsg_len;    /* data byte count, including header */
    int       cmsg_level;  /* originating protocol */
    int       cmsg_type;   /* protocol-specific type */
    /* followed by */
    unsigned char cmsg_data[];
};
```
其各参数的含义和用法可以参考 https://man7.org/linux/man-pages/man3/cmsg.3.html

最后，除了传递 FD 外，在 Linux 当前实现中还支持传递 credentials 和 selinux context, 服务端可以用来验证客户端的身份，由于涉及到的东西比较多，这里不再详细描述，具体用法参见 https://www.man7.org/linux/man-pages/man7/unix.7.html

## 性能

前面提到 Unix Domain Socket 发送数据的时候直接将数据写到接收方的 Socket Buffer，不经过网络协议栈，理论上开销更小，不过缺少实际的数据指标，下面以 redis 为例子，测试下在同样环境下 Unix Domain Socket 和 TCP/IP Loopback 的表现。

### 测试环境

Redis Version: 4.0.9
Linux Kernel: 4.15.0-112-generic
CPU: 1 x Intel(R) Core(TM) i7-8557U CPU @ 1.70GHz

### 测试结果

**TCP/IP Loopback**
```shell
$ redis-benchmark -t ping,get,set -q -d 256 -n 100000
PING_INLINE: 60060.06 requests per second
PING_BULK: 56211.35 requests per second
SET: 54884.74 requests per second
GET: 54141.85 requests per second
```

**Unix Domain Socket**
```shell
$ redis-benchmark -t ping,get,set -q -s /var/run/redis/redis-server.sock -d 256 -n 100000
PING_INLINE: 82169.27 requests per second
PING_BULK: 82712.98 requests per second
SET: 85984.52 requests per second
GET: 82101.80 requests per second
```

从测试结果上看，Unix Domain Socket 的吞吐量是 TCP/IP Loopback 的 1.5 倍，提升了约 50%。

## 排障

网络相关的程序在运行过程中，不可避免的会出现预期之外的情况，为了搞清楚原因大多时候我们会用 tcpdump 抓包并进行分析。然而 Unix Socket 底层实现不依赖任何 Network Protocol，其数据不经过任何网络协议栈，tcpdump 和 tshark 完全没法发挥其用处。

并且，虽然 Unix Socket 被很多基础组件 MySQL、Redis、Docker 等支持，但社区却没有一个类似于 tcpdump 的标准抓包工具，对开发者不是很友好，出错排障成本也比较高。不过，没有标准的工具不代表不能分析流量，一定要做的话还是有一些折衷的办法的，下面以 docker 为例子介绍下社区常见的方案:

> docker 是 client-server 架构，两者通信默认采用 unix domain socket

### Man in the middle

核心思想是创建一个中间的 TCP Socket, 然后在 TCP Socket 上借助 tcpdump 抓包，具体玩法如下:

在一个终端上执行下面的命令
```shell
$ # 1. 获取 docker 监听的 socket 文件
$ lsof -p $(pgrep dockerd) | grep docker.sock
dockerd 16087 root 6u unix 0xffff9cec3aee4000 0t0 45084 /var/run/docker.sock type=STREAM
$ # 2. 重命名原来的 socket 文件
$ sudo mv /var/run/docker.sock{,.orig}
$ # 3. 创建中间的 tcp socket 并拷贝流量
$ sudo socat TCP-LISTEN:8080,reuseaddr,fork UNIX-CONNECT:/var/run/docker.sock.orig &
$ # 4. 创建原来的 socket 文件并拷贝流量
$ sudo socat UNIX-LISTEN:/var/run/docker.sock,fork TCP-CONNECT:127.0.0.1:8080 &
$ # 5. 使用 tcpdump 在中间的 tcp socket 上抓包
$ sudo tcpdump -i lo tcp port 8080 -XX
```

另外开启一个终端执行 docker images 进行测试，下面是 tcpdump 抓包的部分结果，可以看到成功抓取到了通信过程中的 HTTP 请求。

![](https://i.imgur.com/QHhQzmI.png)

该方案能够利用现成的 tcpdump 工具，比较友好，但缺点是需要重启 client 重建连接，对于没法重启的就玩不转了。

### Strace

核心思想 trace 数据收发的 read 和 write 系统调用，获取 read/write 的参数, 具体玩法如下:

```shell
$ # 展示 docker images 涉及到的 write 调用
$ strace -e trace=write -e write=3 -v -s 1024 docker images 1>/dev/null
```

![](https://i.imgur.com/QukkDvw.png)

该方案无需重建连接，能够在已有的连接上抓取流量，不过 strace 对程序的性能有比较大的影响，在生产环境使用的时候需要额外慎重。

### eBPF

> eBPF 是 Linux 内核在 3.18 版本以后引入了的一种扩展的 BPF 虚拟机，允许用户动态的获取、修改内核中的关键数据和执行逻辑，并且有着非常优秀的性能。

核心思想与 strace 类似，都是通过 trace 来分析读写流量，不过 eBPF 有着更小的开销，对应用程序性能影响较小。[unixdump] 就是这样一个基于 eBPF 和 [bcc] 开发的工具，能够非常方便地 dump 系统所有的 unix domain sockets 流量，也支持过滤某一个 socket，下面是具体的玩法:

首先开启一个终端启动 unixdump 程序

```shell
$ sudo unixdump -s /var/run/docker.sock
```

另外开启一个终端执行 docker iamges 执行测试，下面是 dump 的结果:

![](https://i.imgur.com/THiUMKa.png)

该方案无需重建连接，对应用程序性能影响较小，操作方便，有比较强的普适性；不过由于依赖了 eBPF，对内核版本有一定的要求。

注: 上面省略了 [bcc] 和 [unixdump] 的安装步骤，在实际测试的时候需要首先进行安装。


## 总结

Unix Domain Socket 提供了一种单机不同进间程通信的方案，接口上与 Internet Socket 类似，功能上除了支持发送普通数据外，还能在进程间传递 FD、Credentials 和 SELinux Security Context信息，性能上相对于 TCP/IP 本地回环网络有 50% 左右的性能提升，排障上社区没有类似于 tcpdump 的成熟工具，流量分析和排障的成本会比较高。

## 参考


- [unix domain sockets vs. internet sockets](https://lists.freebsd.org/pipermail/freebsd-performance/2005-February/001143.html)
- [udtrace - Unix domain socket tracing](http://laforge.gnumonks.org/blog/20180330-udtrace/)
-  [strace fight for performance](https://archive.fosdem.org/2020/schedule/event/debugging_strace_perfotmance/attachments/slides/4046/export/events/attachments/debugging_strace_perfotmance/slides/4046/fosdem_2020_slides_strace_fight_for_performance.pdf)
- [ebpf adventures fiddling with the linux kernel and unix domain sockets](https://www.nccgroup.com/us/about-us/newsroom-and-events/blog/2019/march/ebpf-adventures-fiddling-with-the-linux-kernel-and-unix-domain-sockets/)



[bcc]: https://www.iovisor.org/technology/bcc
[unixdump]: https://www.nccgroup.com/us/about-us/newsroom-and-events/blog/2019/march/ebpf-adventures-fiddling-with-the-linux-kernel-and-unix-domain-sockets/
