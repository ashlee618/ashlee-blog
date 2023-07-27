---
title: 支持热插拔的ChannelHandler，了解一下
categories: Netty
tags:
  - IO
  - Netty
cover: >-
  https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202307091602399.png
abbrlink: 4805
---




> 🥇🥈🥉🏅🏆往期相关文章：

-   [IO：阻塞和非阻塞、同步和异步 - 掘金 (juejin.cn)](https://juejin.cn/post/7203993298817630268 "https://juejin.cn/post/7203993298817630268")
-   [序列化和反序列化 - 掘金 (juejin.cn)](https://juejin.cn/post/7205478140914843709 "https://juejin.cn/post/7205478140914843709")
-   [Netty框架内的宝藏：ByteBuf - 掘金 (juejin.cn)](https://juejin.cn/post/7206924411256176696 "https://juejin.cn/post/7206924411256176696")
-   [Netty：遇到TCP发送缓冲区满了 写半包操作该如何处理 - 掘金 (juejin.cn)](https://juejin.cn/post/7209319092515356733)
-   [Netty：与AbstractByteBuf渐进式步进截然不同的扩容规则--AdaptiveRecvByteBufAllocator - 掘金 (juejin.cn)](https://juejin.cn/post/7213944791171170359)

# 前言

在上一篇文章中：[Netty：ChannelPipeline和ChannelHandler为什么会鬼混在一起？ - 掘金 (juejin.cn)](https://juejin.cn/post/7213653429416362039)

已经提到了ChannelPipeline里面维护了ChannelHandler的链表，ChannelHandler主要就是负责里面的时间进行拦截和处理。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/06237bb4920e44789d6d483eb655f80e~tplv-k3u1fbpfcp-watermark.image?)
本篇主要介绍：

-   `ChannelHandler`功能说明
-   接口的**透传设计**
-   基于责任链模式的ChannelHandler实现的**热插拔功能**

# ChannelHandler功能说明

> 基于责任链模式实现，负责对**IO事件进行拦截**和处理， 也可以终止事件的传递。

ChannelHandler支持注解，Netty提供了2个：

-   `Sharable`：多个ChannelPipline**公用**同一个ChannelHandler
-   `Skip`：被Skip注解的方法不会被调用

## ChannelHandlerAdapter

> 减少臃肿的代码，只需要实现我感兴趣的事件就好，不感兴趣的不用实现

对于大多数ChannelHandler只会关心**很少一部分事件**，其他事件就会忽略交给下一个ChannelHandler处理。

这就会有一个问题就是，用户自己定义的ChannelHandler**必须实现`ChannelHandler`的所有接口**，包括他不关心的那些事件处理接口，这会让**代码显得很臃肿**。

## 事件透传

为了解决这个问题，Netty提供了`ChannelHandlerAdapter`，他的所有接口实现都是**事件透传**的。

> **透传**：这个概念是通信传输的，指的是不管业务内容怎么变，只负责将内容从来源，传输到目的地

在这里，就是他实现了一个空的方法，留了个钩子Hook，里面没有处理任何事情，如果子类需要有相关需要的逻辑，重写就可以，没有也不需要实现。

![image-20230403162204981](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f1d19d7515524a5cb745cbbe1b7043d3~tplv-k3u1fbpfcp-zoom-1.image)

但是后面Netty高版本之后，ChannelHandler也只有2个需要实现的方法：

-   `handlerAdded`
-   `handlerRemoved`

不过它提供了这个`Adaptor`的思路也是非常不错的。

# ChannelHander热插拔性

> `ChannelPipeline`**支持运行态动态地添加或者删除ChannelHandler**

首先解释一下什么是热插拔，对于玩键盘的人来说应该是再熟悉不过了。

![image-20230403163712991](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/87eb138481c24658a1704c94977f0d2e~tplv-k3u1fbpfcp-zoom-1.image)

感谢百度百科对本篇文章的大力赞助。

对于程序来说其实就是，**支持在程序运行时动态地去修改**。看到这里是不是想到很多相关的技术？

-   ASM字节码
-   反射
-   AOP
-   动态代理
-   ...

支持动态可插拔：意味着，**我们可以随时根据业务场景进行调整Handler的处理逻辑**。

> 流量压力其实也是符合`28原则`，20%的时间可能承受的是一天80%的流量压力，80%的时间其实只是承担一天流量的20%。

因此，我们可以**在业务高峰期的时候，动态地将系统拥塞保护、或者是一些限流的ChannelHandler添加到当前的ChannelPipeline中**。

![image-20230403161624537](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/478358cd264b49ac81c4945220b1789f~tplv-k3u1fbpfcp-zoom-1.image)

然后等流量压力小了之后，再把这些`ChannelHandler`给删掉

![image-20230403161757174](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/80c6cd089e0143aa9668f97e64c7cc47~tplv-k3u1fbpfcp-zoom-1.image)

特别的，一些业务场景中**Handler之间具有顺序性**

比如HandlerB，需要在HandlerA前面执行

`ChannelPipleline`支持在任意地方添加，他有：

-   `addFirts`
-   `addBefore`
-   `addLast`

这些方法，在指定位置添加或者删除**ChannelHandler**

## 线程安全性

`ChannelPipeline`是**线程安全**的。N个业务线程可以并发操作CHannelPipeline而不存在多线程并发问题，这个是框架实现的。（他是通过简单的加锁`synchronized`**悲观锁**来实现的）

![image-20230314172433352](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1f7a9aa24e19473aae67fed6d71b7c36~tplv-k3u1fbpfcp-zoom-1.image)

但是`ChannelHandler`**不是线程安全的**。这个还需要通过`user-code`，程序员来编写代码自己保证。

# 总结

ChannelHandler只在意自己关心的事件，但是在父类里面定义了所有事件的处理方法，为了减少代码的臃肿，子类不需要实现所有父类的抽象方法，Netty把这些方法定义成**事件透传**。

Netty的`ChannelPipeline`基于责任链的设计特性，他是一个链表的形式存在，所以对于`ChannelHandler`的添加和删除都非常方便，他具有**热插拔性**。

![_ZW$55A8PHP)%8HW6RVRV2E.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6f53b6ba7b3b4dfda6919605175d7d43~tplv-k3u1fbpfcp-zoom-1.image)

> 来都来了，点个赞再走吧彦祖👍，这对我来说非常重要！
