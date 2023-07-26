---
title: Netty：与AbstractByteBuf渐进式步进截然不同的扩容规则--AdaptiveRecvByteBufAllocator
categories: Netty
tags:
  - IO
  - Netty
cover: >-
  https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202307091602399.png
abbrlink: 52287
---




# 前言

在[Netty框架内的宝藏：ByteBuf - 掘金 (juejin.cn)](https://juejin.cn/post/7206924411256176696)已经详细地介绍了`AbstractByteBuf`写入数据时，会先判断是否需要扩容并且扩容的算法采用的是**渐进式步进**的方式进行。还没了解过的hxd可以先去补一下上一节课的知识。

为了方便阅读本文，这里简单的总结一下`AbstractByteBuf`的扩容规则：

-   如果扩容量是超过阈值`threshold`，则以阈值为步长进行扩容。
-   如果没有超过阈值，则以`64`倍增。

> **容量小的时候，扩得幅度大，容量大的时候扩得幅度小**。这就是他的特点。

今天讲的`AdaptiveRecvByteBufAllocator`扩容规则跟前者完全相反：

> **容量小的时候，扩得幅度小，容量大的时候扩得幅度大**。

同属于`ByteBuf`，缺走上了不同的道路，属实是有点叛逆心理。

背后到底是人性..还是道德..呃呃呃...进入正片！

* * *

# AdaptiveRecvByteBufAllocator

> 在`NioBytUnsafe`的`#read`方法会调用`#revBufAllocHandler`创建`handler`

本文不再多阐述`Unsafe`类在Netty中扮演的角色，感兴趣的可以看之前写的文章：[Netty：ChannelPipeline和ChannelHandler为什么会鬼混在一起？ - 掘金 (juejin.cn)](https://juejin.cn/post/7213653429416362039)

## read方法

![image-20230323125416320](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c492a479150b4eb48634ee603fe989e6~tplv-k3u1fbpfcp-zoom-1.image)

重点是看`allocHandle`**的初始化**。他是一个**RecvByteBufAllocator的一个内部类Handler**类型的，主要介绍2个比较特别的子类实现：

-   **FixedRecvByteBufAllocator**
-   **AdaptiveRecvByteBufAllocator**

![image-20230323125210004](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1038939d2f9d4d9a8cfabcb48ab97d81~tplv-k3u1fbpfcp-zoom-1.image)

**FixedRecvByteBufAllocator**非常好理解，人如其名，他是固定大小的一个类型。

主要是看**AdaptiveRecvByteBufAllocator**，他的缓冲区大小是**可以动态调整的ByteBuf分配器**。

## 成员变量

> 缓冲区大小可以动态调整的ByteBuf分配器

他有3个系统默认值：

-   `DEFAULT_MINIMUM`（最低）：**64**
-   `DEFAULT_INITIAL`（初始容量）：**1024**
-   `DEFAULT_MAXIMUM` (最大容量）：**65536**

![image-20230313204340422](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/10fc8477ff7c47898cc7e68f9bd632a6~tplv-k3u1fbpfcp-zoom-1.image)

还定义了2个动态调整容量时用到的步进参数：

-   `INDEX_INCREMENT` （扩张）：**4**
-   `INDEX_DECREMENT` （收缩）：**1**

![image-20230313204320304](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d3b1608141254b3396c6c2b29f0e1366~tplv-k3u1fbpfcp-zoom-1.image)

最后还有一张定义了长度的向量表`SIZE_TABLE`

![image-20230313204220800](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2bb2ecc303b141b2ab9435a533ea0111~tplv-k3u1fbpfcp-zoom-1.image)

## record方法

> 重点的扩缩容方法：`#record`

当NioSocketChannel执行完读操作后，会计算获得本次轮询读取的总字节数，他就是参数：**actualReadBytes**，然后执行`#record`方法，根据**实际读取的字节数对ByteBuf进行动态伸缩和扩展**。

意思就是：**它可以根据上一次实际读取到的大小，来影响下一次接受Buffer缓冲区的大小分配情况**

![image-20230313205518883](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/89827b2089d542e08ead55cf9cdc0e64~tplv-k3u1fbpfcp-zoom-1.image)

优点在于：

-   **通用性**：Netty作为一个**通用的NIO框架**，不对用户的应用场景进行假设，不管你是用来做**流媒体传输这种大流量**的，还是用来做聊天工具。传输的码率千变万化，所以Netty设计得**可以根据上一次实际读取得码流大小来对下一次接受Buffer缓冲区进行预测和调整**。
-   **性能高**：容量大，占内存，容量小，频繁扩容会带来性能下降，这套算法会更加平滑
-   **节约内存**：平时聊天可能就消耗1mb左右，突然用户发送了一个10mb的文件，缓冲区扩展成10mb，如果不能支持收缩，那么每次缓冲区都会被分配10mb，会浪费内存，**这就是收缩的好处**。



## AdaptiveRecvByteBufAllocator平滑扩缩容

> 扩缩容，意味着意味着**当流量变大时，这个byteBuf会扩容，当流量变小时，也懂得缩容来减少空间占用**

刚刚提到了一个他维护了一个**步进长度向量表**，他的样子长成这样：

```
 0-->16 1-->32 2-->48 3-->64 4-->80 5-->96 6-->112 7-->128 8-->144
 9-->160 10-->176 11-->192 12-->208 13-->224 14-->240 15-->256 16-->272
 17-->288 18-->304 19-->320 20-->336 21-->352 22-->368 23-->384 24-->400 25-->416
 26-->432 27-->448 28-->464 29-->480 30-->496 
 ================================================================================
 31-->512 32-->1024 
 33-->2048 34-->4096 
 35-->8192 36-->16384 
 37-->3276838-->65536
 39-->13107240-->262144
 41-->52428842-->1048576 
 43-->2097152 44-->4194304 
 45-->838860846-->16777216
 47-->3355443248-->67108864
 49-->134217728 50-->268435456 
 51-->536870912 52-->1073741824
```

我们可以根据这张表，来总结一些规律：

-   当容量小于**512**的时候，由于缓冲区已经比较小，需要降低步进值，**每次扩容`16`**
-   当容量大于**512**时，**说明需要解码的消息码流比较大，应该调大步进幅度来减少动态扩展的频率**，所以采用`512`的倍数进行扩展

> 这里的扩容规则，跟我们前面提到的`AbstratByteBuf`扩容是完全不同的两个风格
>
> -   AbstractByteBuf：容量小的时候，猛猛扩容。容量大的时候，每次扩容的大小是阈值，这个定值来扩容的，就是怕浪费扩容后的冗余空间。
> -   AdaptiveRecvByteBufAllocator：流量小的时候，小小的扩容。流量大的时候，猛猛扩，害怕扩了之后短时间由继续扩。
>
> 主要是因为他们参考的依据不同：**容量**、**流量**。并且后者支持**缩容**，就算猛猛扩了，后面流量变小了也会自己缩回去。

还有一点就是**扩容错误的代价**两者不同：

- `AbstractByteBuf`扩容大了，顶多就是空间，资源的浪费，并不会严重地影响到应用程序对外提供服务
- `AdaptiveRecvByteBufAllocator`在流量大的时候，应用程序承受的压力绝对是比普通时候要大，出故障率也更多。此时你还多搞这么一出，自己在那里频繁地慢吞吞扩容，占用资源。风险还是很大大。
* * *

# 总结
>- `AbstractByteBuf` 采取**保守节约空间的策略**，因为扩容扩到越后面，如果出现浪费空间的情况，那就是浪费一大块空间。
>- `AdaptiveRecvByteBufAllocator` 采取比较**抵御流量洪水的策略**，在流量大的时候，下一次扩大一点，避免频繁扩容造成资源的消耗。并且支持**缩容**，不必太担心空间上的问题。




|                  | AbstractByteBuf        | AdaptiveRecvByteBufAllocator       |
| ---------------- | ---------------------- | ---------------------------------- |
| 扩容             | 支持是✅ | 支持✅             |
| 缩容             | 不支持❌              | 支持✅             |
| 参考指标         | 容量                   | 流量                               |
| 不正确扩容的代价 | 浪费空间               | 在大流量压力时，给应用程序雪上加霜 |

![_ZW$55A8PHP)%8HW6RVRV2E.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/33417b01140647e5a5d039912b5dba3c~tplv-k3u1fbpfcp-zoom-1.image)

> 来都来了，点个赞再走吧彦祖👍，这对我来说非常重要！
