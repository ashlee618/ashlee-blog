---
title: Reactor线程模型的演进和局部无锁化
categories: Netty
tags:
  - IO
  - Netty
cover: >-
  https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202307091602399.png
abbrlink: 42642
---



# 前言

Netty的线程模型是经典的`Reactor`线程模型。

底层的线程模型，才是最大程度上决定系统的性能、吞吐量，决定了整个系统的瓶颈。

前面介绍的从BIO到NIO中间的一个过度阶段：伪NIO中，就没有对底层线程模型进行修改，所以他并不是真正意义上的NIO。

本篇介绍一下Netty的Reactor线程模型的演进，从单线程模型再到多线程模型，再到主从线程模型，各个阶段面临的问题而做出的改变。

# Reactor线程模型演进

> **好的架构都不是一蹴而就的**
>
> “好的架构是进化来的，不是设计来的”----《淘宝技术这十年》

在《程序员修炼之道》书中也有提到过，设计一款**够好即可**的软件。**够好即可**这四个字用得非常好。

够好即可不是代表程序没有追求，得过且过。

够好即可指的是：你并不需要把所有的事情都做好了，所有的考虑到的，考虑不到的问题都解决了，最后再推出版本。

只要做到你 **“内心能平静”** ，用户能满意，就是够好即可。与之相反的就是**过度设计**。

过度设计是非常令人讨厌的东西，就要是把控够好即可和过度设计之间的平衡点。

对于中间件而言，更多的是：**能解决大多数业务场景问题，那么我的这款中间件就是一款优秀的中间件**。

**业务是推动技术发展的主要驱动力**。

Reactor线程模型有3个阶段：

-   Reactor单线程
-   Reactor多线程
-   Reactor主从模型

为了更好地凸显出Reactor线程模型的优势，这里就要跟之前传统的BIO线程模型进行对比啦，详细的*todo* 可以看之前的文章：[IO：阻塞和非阻塞、同步和异步 - 掘金 (juejin.cn)](https://juejin.cn/post/7203993298817630268)

这里稍微做一点前情回顾

## 传统BIO线程模型

![image-20230225154644328](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/77b5f07884584b7bbb0c2db50d6a24c7~tplv-k3u1fbpfcp-zoom-1.image)

-   `Acceptor`线程负责监听客户端的连接。
-   当接收到来自客户端的请求后，会为他分配一个线程负责整个链路处理（客户端请求和线程是**`1：1`**的关系）

> 因为这里线程是吃内存的所以会出现，当内存不够用的时候，无法处理来自客户端的请求，还会有**OOM**的风险。

方便阅读，这里我们需要引入一点前提知识：

### Acceptor线程

> 主要是负责处理客户端的连接。

-   接受客户端**TCP连接**（握手、安全认证），初始化`Channel`参数
-   将链路状态变更事件通知给`ChannelPipeline`

### Reactor线程 / IO线程

> Reactor线程，干活的线程，主要是因为他是负责IO事件的处理，所以也叫IO线程。

-   **异步读取**通信对端的数据包，发送**读事件**到`ChannelPipeline`
-   **异步发送**消息到通信对端，调用ChannelPipeline的消息发送接口
-   **系统Task**
-   **定时任务Task**

## Reactor单线程模型

> **所有的IO操作都在同一个NIO线程上完成**

![image-20230315193238153](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e213e30da86048389ca285ea98bee55f~tplv-k3u1fbpfcp-zoom-1.image)

由于`Reactor`模式使用的**异步非阻塞IO**，所以即便是单线程处理所有操作，也不会出现阻塞的情况。

0.  通过`Acceptor`类接受客户端的TCP连接请求消息
0.  通过三次握手成功建立链路后，`Dispatch`将对应的ByteBuffer转发到对应的**Handler**上，再进行**消息编解码**。

**Reactor单线程的缺点**：

-   **性能问题**：只有一个NIO线程处理所有的连接，这是他的性能上的瓶颈。
-   **消息堆积**：当NIO线程**负载过重**时，处理速度变慢，会导致大量的客户端请求超时，超时要么丢弃，要么重试。**重试也会消耗性能，这会更加导致消息请求堆积现象**。
-   **单点故障问题**：只有一个NIO线程，万一某个任务死循环，那么其他的任务将会出现**饿死现象**。

> 为了解决这些问题，演进出了`Reactor`**多线程模型**

## Reactor多线程模型

> 最大的区别就是：**有一组NIO线程来处理IO操作**

之前单线程模型下，**建立连接这种脏活累活**都是归NIO线程干，现在专门用一个`Acceptor`线程来处理这些，NIO线程就专门去处理IO操作。

并且还给他找了些小弟，是以**NIO线程池**来处理IO操作。

单线程下：指责不分明、人手不够

现在就是：专业事情给专业人干，还多加人手

![image-20230315195136717](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9f3e763d7c9146cea88406088be14253~tplv-k3u1fbpfcp-zoom-1.image)

一个NIO线程可以同时处理N条链路，但是一个链路只对应一个NIO线程，**为的是防止发生并发操作问题**

大多数场景下，Reactor多线程模型已经是满足很多的性能需求，但是在**极个别苛刻的场景下**，**一个NIO线程负责建立TCP连接还是会存在性能压力**，比如：

-   并发百万**客户端连接**
-   服务端需要对客户端握手进行**安全认证**（`SSL`），很耗性能

现在`Acceptor`线程成了整个模型的短板了。

> 为了解决这个问题，又演进了第三种模型——**主从Reactor多线程模型**。

## 主从Reactor多线程模型

> 增强`Acceptor`线程的性能，给他配了个Acceptor线程池
>
> 耗时的处理：安全验证那些丢给Acceptor线程池来做，Acceptor线程还是处理建立连接。

![image-20230404164039115](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/be1f1a9296b14730ad21d7bb4afe83a4~tplv-k3u1fbpfcp-zoom-1.image)

服务端用于接受客户端连接**不再是一个独立的NIO线程，而是一个独立的NIO线程池**，之前Acceptor为这个家杠下了所有。

Acceptor接受到客户端TCP连接请求并处理完了之后，这里是包含安全认证那些，**将新创建的`Socketchannel`注册到IO线程池（sub reactor线程池）的某个IO线程上**，由它负责**SocketChannel的读写和编解码工作**。

> Acceptor线程池只是用来建立连接，三次握手，安全认证，这些都做完了之后，就将链路注册到subreactor线程池上，由IO线程负责后续操作

实际生产环境中，可以通过**配置启动参数，来支持同时支持Reactor单线程、多线程、主从模型多层模型**。

# 局部无锁化设计

> Netty采用了**多个任务并行化**来解决锁资源竞争的问题

解决**并发竞争锁**问题的手段主要有：

-   **乐观锁**（`cas`）
-   **串行化**

Netty在很多地方进行**无锁化的设计**。在**IO线程内部进行串行操作**，避免多线程竞争导致的性能下降问题。

第一印象，**串行化**（同步）很浪费CPU时间片，CPU的利用率不高。

但是通过调整NIO线程池的线程参数，**可以同时启动多个串行化的线程并行运行**，这种**局部无锁化**的串行线程相比一个队列性能更优。

![image-20230315231905429](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/47d03b858e8d4ef0adf3044746336eb8~tplv-k3u1fbpfcp-zoom-1.image)

Netty的`NioEventLoop`读到消息之后，会去调用`ChannelPipeline`的`#fireChannelRead`，去把广播这个事件。

只要用户**不主动切换线程**（包括创建线程），那么就**一直是由NioEventLoop来调用用户的Handler**。串行化处理方法避免了多线程操作导致的锁的竞争。

## AbstractUnsafe

> 我们的老朋友：**AbstractUnsafe**，里面就有无锁化的实现

### AbstractUnsafe#register方法

![image-20230404161717936](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/69ad2a96f8b042bfa9f4168e3d40d4c1~tplv-k3u1fbpfcp-zoom-1.image)

我们可以看到，处理注册事件的时候，会先判断是否是IO线程：

-   如果是就不需要切换，直接**串行化**处理。（大部分情况）
-   如果不是，则需要丢到**用户线程**去执行。

### SingleThreadEventExecutor#execute

![image-20230404162551210](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/23d6719234dc4fd4b44d57011bd30920~tplv-k3u1fbpfcp-zoom-1.image)

具体在用户线程里面的判断，也是非常清晰的逻辑。

先把任务添加到消息队列里`#addTask`，然后启动线程去执行任务`#startThread`。

我们看看这个startThread

![image-20230404162633898](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cf8c7691c3a4440a8ffa1aa080acff1f~tplv-k3u1fbpfcp-zoom-1.image)

这里使用的是`CAS`对状态进行修改，并没有用锁去直接修改，所以说Netty追求性能，在很多地方都精心设计了。

# 总结

`Reactor`线程模型的演进，从单线程NIO线程处理，再到NIO线程池处理，单线程Acceptor处理连接请求。再到Acceptor线程池处理连接后的验证信息。

从单线程Reactor模型到多线程Reactor模型再到主从Reactor模型。可以看到架构在一步一步地进化而来。

此外Netty的**局部无锁化设计**，采用串行化执行任务、CAS的方式来避免资源的竞争，提高Netty的性能。

![_ZW$55A8PHP)%8HW6RVRV2E.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4f39c25f7d144fadaf89fb04a8a5c7f7~tplv-k3u1fbpfcp-zoom-1.image)

> 来都来了，点个赞再走吧彦祖👍，这对我来说非常重要！

