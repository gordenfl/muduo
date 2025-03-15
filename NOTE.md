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
TODO：

## libmuduo_net.a
TODO：

## libmuduo_http.a
TODO：


## libmuduo_inspect.a
TODO：


## libmuduo_pubsub.a
TODO：


## Examples 代码的分析
这里举一两个有代表性的例子，看一下具体每一个 lib 是如何被调用他们的关系如何。