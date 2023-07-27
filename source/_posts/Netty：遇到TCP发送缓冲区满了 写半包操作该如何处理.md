---
title: Netty：遇到TCP发送缓冲区满了 写半包操作该如何处理
categories: Netty
tags:
  - IO
  - Netty
cover: >-
  https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202307091602399.png
abbrlink: 49726
---




# 什么是写半包

> **写半包**：一份数据，一次发送没有把他全部发送，**需要循环发送**，那么第一次的操作称为写半包

什么情况下会出现写半包：

发送方发送200byte，但是接收方只能接受100byte，因此发送方只会发送小于100byte的数据。

说到这里，机智的小伙伴已经想到了这跟**TCP滑动窗口**和消息中间件中常见的**消息堆积**是一个道理。

总的来说：**接收方顶不住来自发送方的数据压力**。


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9a12eef23e8a42f197353c582e9f8a3b~tplv-k3u1fbpfcp-watermark.image?)
对于Netty来说就是，这个时刻**TCP发送缓冲区满了**，无法再接收**整包数据**，剩下的数据则会通过Channel去监听写操作，当触发写操作的时候，再把这部分数据给带上，那么这部分数据才完整地传输。


# Netty中的写半包处理

前提知识：Netty中的网络数据读写，都先经过`ByteBuf`
![image-20230227222618518](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/084a4472e5b8497a8b8b2ad60105b30f~tplv-k3u1fbpfcp-zoom-1.image)
- **读操作**：从`ByteBuf`中读取数据
- **写操作**：将数据写入到`ByteBuf`，然后再通过其他方式把ByteBuf的数据写入（`#doWrite`）

Netty中的网路操作都是通过`Channel`和里面聚合的对象`Unsafe`对象进行操作，简单介绍一下。


# Channel
> Channel的作用：给Netty用来进行网络网络

JDK 也有自己原生的Channel，但是为了方便框架扩展使用，Netty采用的是封装了一层`Facade`（门面模式）。

最重要的是能够支持Netty的**自定义Channel**来应对不同的业务场景。

Channel会被注册到`EventLoop`上，在注册的时候定义好感兴趣的事件，他采用的是**基于事件触发**的方式，当Channel上触发相对应的事件时，就会主动回调通知，然后交给对应的`ChannelHandler`进行处理。

由于本篇讲的是写半包，因此不再过多解释。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bee9a52f4bb043c09ccb94244762efb3~tplv-k3u1fbpfcp-watermark.image?)

总的来说：
**Channel就是Netty用来处理网络数据流的**

回到本篇的主题：**写半包**

# AbstractNioByteChannel
> 主要负责处理写半包

总的流程如图：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/15837f5b11494f618f04085b5958f133~tplv-k3u1fbpfcp-watermark.image?)

> **ChannelOutboundBuffer**：环形发送数组

1. 不停地从ChannelOutboundBuffer读取数据，看是否有可以发送的数据
2. 如果有，并且是ByteBuf类型的，可以选择发送数据
   - 如果一次发送没有发送完，则**采取一定次数的循环发送**（写半包）

3. 数据最后还是没有发送完，则会开一条新线程专门进行剩余数据的发送
4. 在最后会去同步数据写入进度


## 源码解析 #doWrite

![image-20230311150751449](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e9864635b4884e2fb3d63d9304e05bcd~tplv-k3u1fbpfcp-zoom-1.image)

不停地去环形发送数组里面取数据出来

-   如果是空了，代表发送完了，把写标志位置空（`clearOpWrite`）


![image-20230311150858669](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c7eb703b2a114c758d5bfbe124787af9~tplv-k3u1fbpfcp-zoom-1.image)


如果不是空数据，则判断是不是`ByteBuf`数据

-   对其进行强转，**若可读字节数是0，代表消息不可读（reidIndex >= writeIndex)，则把他在环形发送数组中移除**。



第一次读的时候，会先去获取循环发送次数`writeSpinCount`。循环发送次数就是指：第一次发送没有完成时（**写半包**）进行循环发送的次数。

给他设置一个阈值，为的就是当循环发送的时候，**IO线程会一直尝试写操作，此时IO线程无法处理其他操作，相当于局部阻塞、死锁、假死的情况**。

像这种处理手法非常常见，比如一般我们会给分布式锁设置一个锁的超时时间，**除此之外还需要设置一个客户端的超时时间**，避免客户端在拿到锁的时候，这把锁已经过期了。客户端的超时时间会比锁的超时时间**要短**。

![image-20230311151404816](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/460ff154b0b34ba5b803007d3f8cc88c~tplv-k3u1fbpfcp-zoom-1.image)

然后就是进行循环发送了

![image-20230311151426008](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d4e861e1c1624ea292debcde58c4349a~tplv-k3u1fbpfcp-zoom-1.image)

消息发送操作完成时候，会调用ChannelOutboundBuffer更新发送进度的消息，**并且还会判断是否需要写半包处理**

![image-20230311151525404](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/39dfd72f8e134632811355f38a15f9d2~tplv-k3u1fbpfcp-zoom-1.image)

如果没有发完，则**设置写半包标识位，启动专门的写半包线程继续发后续的消息**

# 总结

写半包问题本质上是：**对于接收方来说，来自发送方的数据压力太大了，因此不得不采取的一种降保护措施**

可以在发送方进行解决、也可以在接收方进行解决

Netty并没有采取说，遇到TCP缓冲区满了之后，这个数据包就等下一次再等发，**而是能发多少就发多少，不够的
下次再发**，是一种追求性能的选择。

像消息中间件遇到消息堆积问题，在消接收方（消费者）增大消费的速度，比如：加消费队列或扩充消费者群组等。

又或者限制发送方（生产者）的发送速度，比如TCP的滑动窗口。

所以互联想的技术都是有相关联的，能看到互相的影子。

> 彦祖，看都看到这里了，点个赞再走吧，这对我来说非常重要👍

***本文正在参加[「金石计划」](https://juejin.cn/post/7207698564641996856/ "https://juejin.cn/post/7207698564641996856/")***
