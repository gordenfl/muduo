# MUDUO 源代码分析

首先先讲讲我对这个项目的看法：

* C++项目似乎不是那么古老了，有点看头，不会像 C 一样让人。。
* 他[README](README)里面说了是一个“multithreaded C++ network library based on the reactor pattern.”这里的 reactor pattern 刚开始不太理解，后面查出来了结果。
* Reactor Pattern（反应堆模式） 是一种用于处理并发 I/O 事件的设计模式，它适用于高性能服务器、网络应用和异步 I/O 处理。核心思想是通过一个单一的事件分发器（Event Demultiplexer）监听多个事件，并且在事件就绪的时候分派给多个对应的处理器（Handler）去处理。
* 除了 Reactor Pattern 的同时还有其他的几种模式，他们是 Proactor，Thread-per-connection，Worker Thread Pool 他们之间是有不同的接下来会有专门一节介绍这些的不同之处。


## Reactor 模式 vs. 其他模式
* Reactor：主要特点是时间驱动，单线程或少量线程，I/O 事件发生后才执行，适用场景是低延迟，高并发的网络服务器 如 Nginx， Kafka， 游戏服务器， 数据库
* Proactor：主要特点是时间驱动，操作系统负责 I/O完成，应用层只处理数据， 适用场景在需要完成 I/O 操作后处理数据的场景
* Thread-Pre-connection： 每个连接创建一个线程，线程独立处理 I/O, 适用于低并发，线程切换开销大
* Worker Thread Pool：事件到来后交给线程池处理，适用于计算密集型任务

# 源代码结构
- 源代码分为 base 和 net 两个目录，里面的存储的文件完成相应的功能
- 编译出来是一套库，有静态库，*.a 记住只有静态库没有动态库
- 静态库有四个文件分别是 libmuduo_base.a, libmuduo_http.a libmuduo_inspect.a libmuduo_net.a libmuduo_pubsub.a 这五个文件的具体功能稍后整理 
- libmuduo_pubsub.a：是这堆代码生成的一个库做 server broadcasting 和 publishing content
    ```
    hub - a server for broadcasting
    pubsub - a client library of hub
    pub - a command line tool for publishing content on a topic
    sub - a demo tool for subscribing a topic
    ```
- libmuduo_net.a 是 source code 中的 muduo/net 下面的代码生成
- libmuduo_base.a 是 source code 中的 muduo/base 下面的代码生成
- libmuduo_http.a 是 source code 中的 muduo/net/http 下面代码生成
- libmuduo_inspect.a 是 source code 中的 muduo/net/inspect 下面代码生成

他没有文档解释代码模块关系，只能凭自己的直觉来看。CMakefile 我现在不是很懂，所以也没有办法解释。
我们就分别按照这5个库所对应的源代码就进行阅读。首先他不是一个可执行的工具，他是一个库代码，所以没有 main 函数，只会有函数暴露出来给人调用。

## libmuduo_base.a
线程的包装，线程安全的包装，日期，异常，文件访问， Filelog StreamLog， Mutex，进程的包装，时区的包装，weakCallback 和 Types是在 type_cast 的时候的逻辑。

## libmuduo_net.a
网络部分，包括TCP,很多很详细的封装。包括 创建了 acceptor channel，eventloop socket, tcpconnection, tcpclient, tcpserver, timer ,poller,  timerqueue, zlibstream 这些对象, EventLoop 是最重要的。整合了 protocol buff，包装了 Linux 下面才有的 poll 和 epoll

## libmuduo_http.a
基于 libmuduo_net.a 中实现的基本函数做了一个上层协议的封装，HTTP 协议的实现，包括 request， response， 每个请求和响应的 header 和 body，不同的content-type 对应的逻辑，包括 stream 的操作


## libmuduo_inspect.a
提供的一个 轻量级 HTTP 服务器，用于运行时调试和状态监控。
* Inspector（核心类）：处理 HTTP 请求，返回服务器状态信息。
*	ProcessInspector：查看进程状态，如 CPU、内存、进程 ID 等。
*	PerformanceInspector：查看性能数据，如定时器、日志等。
*	SystemInspector：获取系统相关信息，如负载、线程等。

提供一个模块，可以启动一个 HTTP 服务器去观看这些信息。

## libmuduo_pubsub.a
是 Muduo 提供的一个轻量级的发布-订阅（Pub/Sub）库，用于构建基于 TCP 的发布-订阅消息系统。


## Examples 代码的分析
这里举一两个有代表性的例子，看一下具体每一个 lib 是如何被调用他们的关系如何。

主要功能是：
*	允许多个客户端通过 TCP 连接到服务器，进行消息的发布和订阅。
*	基于 Muduo 网络库，采用事件驱动的方式处理 Pub/Sub 逻辑。
*	提供了简单的主题（Topic）管理机制，可以让多个客户端订阅相同的主题，并接收推送的消息。

源代码位置在 muduo/examples/pubsub/  我就是说在 example 外面找不到这个玩意儿

muduo_pubsub 的应用场景
* 实时消息推送（如聊天室、股票行情、日志收集）
* 轻量级的分布式事件总线
* 多人协作应用（如在线文档编辑）

---
## 项目的基本结构
1. 基础结构
*	muduo/base/ 目录包含基础工具类，例如日志 (Logging)、时间戳 (Timestamp)、线程 (Thread)、原子操作 (Atomic) 等。
*	muduo/net/ 目录是核心，封装了网络通信相关的组件。

2. 事件循环（Event Loop）
*	muduo/net/EventLoop.h / EventLoop.cc
*	事件循环的核心类，封装了 epoll_wait，并负责执行定时任务、I/O 事件处理等。
*	muduo/net/Poller.h / EPollPoller.h
*	Poller 是一个抽象类，EPollPoller 是具体实现，封装了 epoll 相关的操作，如 epoll_ctl。

3. Channel（事件通道）
*	muduo/net/Channel.h / Channel.cc
*	Channel 绑定一个 file descriptor（通常是 socket），并在事件发生时调用回调函数。

4. Acceptor（监听连接）
*	muduo/net/Acceptor.h / Acceptor.cc
*	负责监听新连接，内部使用 Socket 和 Channel 处理 accept 事件。

5. TcpConnection（TCP 连接管理）
*	muduo/net/TcpConnection.h / TcpConnection.cc
*	维护单个 TCP 连接，封装 read/write 操作，管理读写缓冲区。

6. TcpServer（服务器）
*	muduo/net/TcpServer.h / TcpServer.cc
*	负责管理多个 TcpConnection，支持多线程 EventLoop 运行。

7. TcpClient（客户端）
*	muduo/net/TcpClient.h / TcpClient.cc
*	提供一个简单的 TCP 客户端封装。

## EventLoop 和 Channel 基础
在 muduo/net/EventLoop.h/cc 
单独看这个模块有点迷茫，因为不知道从何入手，应该找寻一个 Example 里面的例子，我就找了 example/simple/echo 这个例子：
```C++
  LOG_INFO << "pid = " << getpid();
  muduo::net::EventLoop loop;
  muduo::net::InetAddress listenAddr(2007);
  EchoServer server(&loop, listenAddr);
  loop.loop();
```
这里很容易懂，就是创建一个 loop 对象一个 listenAddress 自己就可以开始使用这两个对象了，让我们进去 server 的构造和loop
这个了EchoServer 类中用到了一个 TcpServer 对象

EventLoop 创建的时候里面包含了 TimerQueue，Eventfd[他的作用是用于通知事件，技术信号，通过 write 增加技术，通过 read 减少信号]

EventLoop 最主要的就是 loop 成员函数，

关于 Channel 基本的逻辑描述如下：
```CPP
//这两个函数很重要 外部靠这两个函数来获取读，写能力
setWriteCallback
setReadCallback
```
我现在能理解的就是每当Epoll 或者 Poll 拿到网络信息（Event）的时候，就会将每个这些 Event，push 到他对应的 Channel 里面，通过 set_revents. 可以通过events_[i].data.ptr 这个指针获取到channel 的 id，拿到 channel 的对象以后push event进去，然后把这个 Channel 对象 push 到activeChannel 里面，调用 PollPoller 中 poll 函数的人就能拿到这个 activeChannel和一个 timeStamp, 什么地方调用 poll 呢就是 EventLoop 里面的 loop 函数的 while 里面，针对 activeChannel 里面的每个Channel去处理网络发过来的消息s。 也就是说一次批量处理多个 Channel 的 Event，

这个时候 Channel 里面的 handleEvent 就会调用handleEventWithGuard 来处理每一个 event，来分析每个 event，分别调用外面传进来的 readCallback 和 writeCallback。
这样整个逻辑就圆满了。

```CPP
muduo::net::TcpServer server_;
```
初始化只需要一个 loop 指针，listenAddr引用，在给他一个字符串名字，初始化好了，然后 server 就可以setConnectionCallback， setMessageCallBack 两个函数，简单吧。

```CPP
 //这个参数类型const muduo::net::TcpConnectionPtr&
server_.setConnectionCallback(std::bind(&EchoServer::onConnection, this, _1));
/*
这三个参数分别是：
const muduo::net::TcpConnectionPtr&
muduo::net::Buffer*
muduo::Timestamp 
*/
server_.setMessageCallback(std::bind(&EchoServer::onMessage, this, _1, _2, _3));
```
就是说明我们可以通过这个设置钩子函数的情况来分开具体操作逻辑和底层的收发包的逻辑。🐂🍺

具体 server_ 内部是怎样的，我们走进去看看：

## TcpServer
首先初始化：
```CPP
TcpServer::TcpServer(EventLoop* loop,
                     const InetAddress& listenAddr,
                     const string& nameArg,
                     Option option)
  : loop_(CHECK_NOTNULL(loop)),
    ipPort_(listenAddr.toIpPort()),
    name_(nameArg),
    acceptor_(new Acceptor(loop, listenAddr, option == kReusePort)),
    threadPool_(new EventLoopThreadPool(loop, name_)),
    connectionCallback_(defaultConnectionCallback),
    messageCallback_(defaultMessageCallback),
    nextConnId_(1)
{
  acceptor_->setNewConnectionCallback(
      std::bind(&TcpServer::newConnection, this, _1, _2));
}
```
可以看到他的内部存储了几个关键的对象来帮助处理连接发送，接收数据需要用到的东西。
```
loop_ : 我们所说的 EventLoop 对象，接下来分析他内部是怎么运作的
ipPort: 端口，是从传入的 listenAddr 里面提取的
name_: is name of this instance
acceptor_: is an Acceptor instance created by the loop, listenAddr  etc.
threadPool_: Here is an thread Pool with the type of EventLoopThreadPool. We need to analysis it later
connectionCallback_: this is a callback function while accept an new connection will be called
messageCallback_: this is a callback function while received a new message
nextConnId: TODO:  
```

## Acceptor

当服务器调用acceptor setNewConnectionCallback()函数的时候会传入 newConnection，这是对的，因为在 accept 一个连接的时候，会给 TcpServer传回两个参数一个是客户端的连接 fd， 一个是客户端的地址。另外 server 端的 socket 是在 Acceptor 对象构造的时候利用传入的 listenAddr 创建的。
```CPP
Acceptor::Acceptor(EventLoop* loop, const InetAddress& listenAddr, bool reuseport)
  : loop_(loop),
    acceptSocket_(sockets::createNonblockingOrDie(listenAddr.family())), //<--- 在这里
    acceptChannel_(loop, acceptSocket_.fd()), // <----TODO: 分析
    listening_(false),
    idleFd_(::open("/dev/null", O_RDONLY | O_CLOEXEC))
{
  assert(idleFd_ >= 0);
  acceptSocket_.setReuseAddr(true);
  acceptSocket_.setReusePort(reuseport);
  acceptSocket_.bindAddress(listenAddr);
  acceptChannel_.setReadCallback(
      std::bind(&Acceptor::handleRead, this));
}
```
同时在创建 Acceptor 的时候，还会创建 acceptChannel，这是什么接下来会分析。

## Socket
让我们再来看看 Socket 创建的步骤是怎样的：
```CPP
int sockets::createNonblockingOrDie(sa_family_t family)
{
#if VALGRIND //这里是在做内存检测的一个宏定义
  int sockfd = ::socket(family, SOCK_STREAM, IPPROTO_TCP);
  if (sockfd < 0)
  {
    LOG_SYSFATAL << "sockets::createNonblockingOrDie";
  }

  setNonBlockAndCloseOnExec(sockfd);
#else
  /*
  这里解释一下 参数：
    SOCK_STREAM 字节流，
    SOCK_NONBLOCK 非阻塞， 
    SOCK_CLOEXEC 子进程无法使用， 

    IPPROTO_TCP 值是 6 指明了用 TCP
    IPPROTO_UDP  值是 17，
    IPPROTO_ICMP  值是 1
    IPPROTO_RAW 允许我们去控制 IP 层

    如果用TCP 也可以用 0 但是为什么 0 就是 TCP，是因为前面的 SOCK_STREAM 字段只有 TCP 支持， UDP 就是 SOCK_DGRAM
    也就是说：
    ::socket(family, SOCK_STREAM | SOCK_NONBLOCK | SOCK_CLOEXEC, 0) //自动选择，因为前面是 STREAM，所以就是 TCP
    ::socket(family, SOCK_DGRAM ｜ SOCK_CLOEXEC, 0) //他会自动选择 因为前面是DGRAM，就会是 UDP
  */
  int sockfd = ::socket(family, SOCK_STREAM | SOCK_NONBLOCK | SOCK_CLOEXEC, IPPROTO_TCP);
  if (sockfd < 0)
  {
    LOG_SYSFATAL << "sockets::createNonblockingOrDie";
  }
#endif
  return sockfd;
}
```
在这里创建的socket 是一个 NONBLOCK 的，并且SOCK_CLOEXEC，这个的含义是，如果这个进程创建子进程的时候，这个 Socket fd 在子进程中是无法使用的。这代码非常工整啊。

另外他在代码中存在一个模块：SocketsOps.h/cc 这里面定义的全部都是跟 socket 相关的独立函数，不是类，全部放在 muduo::net::sockets 这个命名空间中。这是最基础的函数，包括 connect bind listen accept read readv write
close，这里才是最核心的部分，从现在来看，也就是他对所有的 socket 相关的东西全部做了一次封装。

<strong style="color: red">另外他好像没有对 UDP 协议的发送回传做封装，不知道是我理解错误还是不太懂。TODO:</strong>


## Thread
```CPP
pid_t gettid()
{
  return static_cast<pid_t>(::syscall(SYS_gettid));
}
```
首先这个函数可以通过系统调用 SYS_gettid，来获取一个线程的 id，在 Mac 上是 man不到的在 Linux 系统上是可以 man 2 关于线程相关的信息，包括如何获得线程id，线程的状态等信息

在这个之前提供了个结构体 ThreadData，里面包含线程的 func， name， id，countDownLatch (这个函类似主要管理线程的加锁信息)，这个 ThreadData 包含写函数，比如 runInThread，主要是去调用绑定在这个 Thread 中的函数，以及处理这个函数的结果的处理等等

线程的操作通过一个namespace叫做 CurrentThread 包含所有的 Thread 需要用到的函数，这些函数都是包装了系统调用中的 Thread call。有一个独立的函数startThread 用来创建这个线程，并且利用 ThreadData 空间中的函数去做一些事情，最重要的是调用外部传入的 func_和数据。整个过程都被封装在 Thread 对象中的 start 函数中。

Thread 类里面还提供了join 函数，锁信息，析构函数。这里讲一下 thread 的 join行为:
join 是为了在主线程中 等待子线程的退出，这个时候主线程会停留在 .join() 函数这一行，知道 join 返回以后才继续执行下面的操作。一个线程在做完他所有的事情的时候，一定要调用_detach 在 pthread 中就是 pthread_detach回收资源，否则会导致一些资源的泄露。
线程库中一般提供的线程的函数有：
```CPP
//线程中的具体操作
join()
detach()
joinable()
get_tid()
hardware_concurrency() //获取CPU 核心数目
native_handle() //获取底层线程句柄
yield() //主动让出CPU资源
sleep_for() //让线程休眠
sleep_until() //休眠到具体的某特时间点
```

## ThreadPool  牛逼
管理多个线程的池子，在这里面我看到了很多信息，这里主要分析一下 Thread，在 ThreadPool 中有容量，maxQueueSize_，可以 set 用这个函数 setMaxQueueSize，
就是将所有线程预先创建好，然放在 vector 里面， 这个类里面主要的函数是：
```CPP
void start(int numThreads); //开始创建整个 ThreadPool
void stop(); //停止这个 ThreadPoll，并且回收资源等等
void run(Task f); //这个是核心，就是用一个线程去跑 f 说指向的函数，函数在外面传进来的时候有个的是
thread_poll_.run(::bind(&func, arg1, arg2)); //这就会调用 run，去启动个线程，在线程里面执行 func(arg1, arg2)
```

## protobuf
TODO:

## protorpc
TODO:












