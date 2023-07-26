---
title: Netty：ChannelPipeline和ChannelHandler为什么会鬼混在一起？
categories: Netty
tags:
  - IO
  - Netty
cover: >-
  https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202307091602399.png
abbrlink: 41945
updated: 2023-07-19 10:54:43
---




# 前言

>**数据流**是编程范式中的其中一种，何为数据流?

数据流具有**无穷数据源**、**数据流向**的特点：
- 无穷数据源：你无法预估你需要处理的数据流，你也不知道什么时候会产生数据
- 数据流向：数据传递具有顺序性

在网络编程中，客户端想要跟服务器进行数据传递，首先就要通过三次握手建立`传输通道`，绑定好端口，此后通过这个socket进行数据传输。

当通道关闭后，如果想要建立连接，还需再次由客户端发起建立连接的请求，所以后面演进了使用`长连接`的方式来优化连接频繁地建立和销毁。

在网络编程中，`Channel`就是干这个事情的。


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/72552a9e01c34768925b28364654ae48~tplv-k3u1fbpfcp-watermark.image?)

类似于NIO的Channel，Netty提供了自己的`Channel`，使得更符合自身框架，叫做`ChannelPiple`。



# Channel
>JDK的NIO类库种Channel分2种：
>-   客户端的：SockerChannel
>-   服务端的：ServerSocketChannel


![image-20230314192757251](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1b7c7630df3c4a3fb16569f826a7ba5f~tplv-k3u1fbpfcp-zoom-1.image)

`Unsafe`是一个内部接口，**聚合在Channel种协助进行网络读写相关的操作**。

Unsafe本身是作为**Channel的一个内部辅助类**来用的，其实并不应该被Netty框架的上层使用者使用，所以是被命名为Unsafe

## Unsafe
> 是Channel内部的一个内部实现类，是一个接口类型

![image-20230313171637248](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/72f7fe911e3149048140d2684319dc83~tplv-k3u1fbpfcp-zoom-1.image)

**他不应该被用户代码直接调用，而是由实际负责IO的读写线程来调用**。这也是他的名字`Unsafe`的含义。

### Unsafe中基础方法


#### registry方法

> 作用：**将Channel注册到EventLoop的多路复用器上**

为了**避免多线程并发操作Channe**l的问题，这里还需要做并发处理

判断当前线程是否是Channel对应的NioEventLoop线程

-   如果是，则直接执行调用`#register0`执行注册逻辑
-   如果不是，则以包成`Runnable`丢给**NioEventLoop**那边线程池的任务队列里去等待执行

![image-20230313174814089](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5b7e7e727f614759b8b425e7115ee7a5~tplv-k3u1fbpfcp-zoom-1.image)

#### bind方法

> **给Channel绑定对应的Socket**
>
> -   对于服务端，用于绑定监听端口
> -   对于客户端，用于指定客户端Channel的本地绑定Socker地址

#### write方法

> 把消息添加到唤醒发送数组：**ChannelOutboundBuffer**里面，并不是真正的写Channel，`flush`才是把消息写进Channel

![image-20230313194858950](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f2e5aecee6174e62b9bf30c0bfb156eb~tplv-k3u1fbpfcp-zoom-1.image)


## ChannelPipeline功能说明

> io.netty.channel.Channel是`Netty`**网络操作抽象类**，里面聚合了JDK的Channel对象

可以Channel理解为操作网络的一个对象，委派角色，他聚合了一组功能，包括但不限于：

-   **网络的读写**
-   **客户端的发起连接、主动关闭连接、链路关闭**
-   **获取双方通信的地址**
-   …

### JDK原生Channel

> JDK 也有NIO原生的Channel，但是Netty是另起炉灶自己抽象出来一个新的Channel，那这里又要讲讲，为什么不用原生JDK的东西了



1.  JDK的**SocketChannel**和**ServerSocketChannel**主要职责是负责**网络IO**操作。由于他们是`SPI`类接口，所以通过继承SPI功能类来扩展其功能难度大

    ***SPI和API的区别！还不懂的赶紧去复习，面试常考内容！***
    
![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5d80ae83866c4def9048d6b0ddb9a39f~tplv-k3u1fbpfcp-watermark.image?)

2.  Netty的Channel能够跟Netty的整体架构融合在一起，**跟架构亲和力更足**。

3.  **自定义Channel**，功能实现更加灵活（感觉这个才是最重要的）

> Netty的Channel设计理念

1.  在`Channel`接口层，采用`Facade`（外观模式）进行统一封装，**相当于一个聚合了一个JDK原生Channel的一个操作对象**
2.  为SocketChannel、ServerSocketChannel提供**统一的视图**，公共功能在父类中实现，子类实现更为具体的功能
3.  具体实现**采用聚合而非包含的方式**，将相关的功能聚合在Channel种，由Channel**统一负责分配和调度**。


### 网络读写操作

> 网络IO操作会触发`ChannelPipeline`中对应的事件方法

Neety是基于**事件驱动**，当`Channel`进行IO操作时，会产生对应的IO事件，然后**驱动事件在ChannelPipeline中传播**，最后**由ChannelHandler对事件进行拦截和处理**，不关心的事件可以直接忽略。

采用**事件驱动**的方式，有很多优点：

-   首先可以避免监听者列表过长，触发一次事件，采用**顺序遍历**性能会线性下降
-   采用事件驱动可以很轻松**通过事件来划分事件拦截切面**，效果等价于`AOP`，甚至比他性能更高

**网络IO操作**直接调用`DefaultChannelPipeline`的相关方法，由前者中对应的`ChannelHandler`来进行逻辑处理。

### AbstractChannel网络IO操作源码实现
成员变量：
-   `SelectableChannel`：用于设置参数和进行IO操作
-   `readInterestOp`：代表了**JDK SelectionKey**的**OP_READ**
-   `SelectionKey`：用**volatile**修饰，由于Channel会面临**多个业务线程并发写操作**，所以为了能让其他线程对其感知变化，所以使用volatile保证可见性

#### Channel的注册
> doRegister，把Channel注册到EventLoop中

![image-20230311150250967](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ba3ec0c7611d435c82c99a3abb52fdb4~tplv-k3u1fbpfcp-zoom-1.image)

可以看到第二个参数`ops`，代表感兴趣的事件，doRegistry的时候直接传了个0，代表对任何事件都不感兴趣

![image-20230311150311969](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2bd7bb3068f74f678ea312527f9ea14b~tplv-k3u1fbpfcp-zoom-1.image)

Channel对哪些事件感兴趣呢，比如：

-   `OP_READ` = 1 << 0； 读操作位
-   `OP_WRITE` = 1 << 2；写操作位
-   `OP_CONNECT` = 1 << 3；客户端连接服务端，建立连接操作
-   `OP_ACCEPT` = 1 << 4；服务端接受客户端连接，成功建立连接操作

第三个参数是把自己`this`传过去了，这是为了能让触发事件的时候返回`SelectionKey`。

**根据SelectionKey是可以在多路复用器拿到对应感兴趣的Channel对象**，然后去进行处理。


# ChannelPipeline

## ChannelPipeline和ChannelHandler

> `ChannelPipleine`和`ChannelHandler`机制类似于**Servlet**和**Filter**过滤器

本质上是**责任链模式**的实现。

**Servlet Filter**能够以**声明式**的方法插入到HTTP请求处理过程中。用于**拦截请求和响应**，实现对请求的**前置处理**和**后置处理**。

Netty的Channel过滤器实现原理跟Servlet Filter机制一致，它将Channel的数据管道抽象成：`ChannelPipeline`。

![image-20230314192757251](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b1b7e5dd2e6c49109b459b1b4ca6a4ed~tplv-k3u1fbpfcp-zoom-1.image)

数据在`ChannelPipeline`中流动和传递，`ChannelPipeline`中持有IO事件拦截器`ChannelHandler`的**链表**。

意味着：**`ChannelHandler`可以对IO事件进行拦截和处理**

`ChannelHandler`是**可插拔式**的，并且我们可以通过**新增或者删除**`ChannelHandler`来实现不同业务逻辑，

## ChannelPipeline功能说明
> ChannelPipeline是ChannelHandler的容器
> 
> 负责ChannelHandler的管理和**事件拦截与调度**

主要是：**负责事件的分发和预处理**

一个消息被ChannelPipeline的ChannelHandler链拦截和处理的全过程：

![image-20230314095713102](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0a63890e8e054403b7423865be663d9d~tplv-k3u1fbpfcp-zoom-1.image)

1.  底层的SocketChannel `#read`方法读取ByteBuf，触发ChannelRead事件（`OP_READ`），由IO线程`NioEventLoop`调用ChannelPipeline的`#fireChannelRead`方法，将消息（ByteBuf）传输到**ChannelPipeline**中

2.  消息依次被添加好的ChannelHandler链处理，首先是HeadHandler，然后是ChannelHandler1、ChannelHandler2….，**任何Handler都可以中断消息的传递，也就是他们都可以把消息丢掉**。

    注意：这里是先是被`HeadHandler`处理，后面是反过来，**是有顺序的**

3.  ChannelHandlerContext的`#write`方法发送消息，消息从`TailHandler`开始，途径ChannelHandlerN，ChannelHandlerN-1….

    最终被添加到**消息发送缓冲区中**，等待刷新（`flush`）和发送（`write`）

> 注意：**这里为什么是具有顺序性地处理**，因为有些ChanelHandler就有**顺序性**，比如解码那些，需要先将ByteBuf数据解析成对象，然后再将对象二次解码成pojo对象。

Netty中的事件分为：`inbound`事件和`outbound`事件，**图中左半边对应inboud事件，右半边对应outbound事件**。

-   `inbound`：由外部触发的事件，应用程序以外的，并非应用程序自己请求的。比如某个socket有数据读取进来了，有Channel注册到Eventloop上了
-   `outbound`：由应用程序主动请求而触发的事件，比如向某个socket写入数据，或从某个socket读取数据

**ChannelInboundHandler**处理`inboud`事件

**ChannelOutboundHandler**处理`outbound`事件

> 误区：区分inboud、outbound并不是简单的IO流，而是**触发事件的源头**

# 总结

本篇简单介绍了JDK自带的`Channel`以及`Unsafe`其内部的一个实现类。并且从`Unsafe`的职责来更好的理解它这个名字的含义。

Netty提供的`ChannelPipeline`内部聚合了一个`ChannelHandler`链表。`ChannelPipeline`负责网络IO事件中的调度，而`ChannelHandler`则是负责拦截，如果是对应感兴趣的事件则处理，否则抛给下一个`ChannleHandler`。

粗略地讲了一下ChannelHandler为什么要基于**责任链**的设计模式，做成链表的形式。下一篇文章会着重讲一下ChannelHandler在Netty里面干了些什么事情。


![_ZW$55A8PHP)%8HW6RVRV2E.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a33dad9d24be48478cd52e4f15dae0ee~tplv-k3u1fbpfcp-watermark.image?)

> 来都来了，点个赞再走吧彦祖👍，这对我来说非常重要！
