---
title: IO：阻塞和非阻塞、同步和异步
abbrlink: 53559
categories: "Netty"
tags: 
- "IO"
- "Netty"
cover: "https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202307091602399.png"
---

#文章 


# 阻塞和非阻塞

>阻塞的时候线程会被挂起



**阻塞**：

当数据还没准备好时，调用了阻塞的方法，**则线程会被挂起**，会让出CPU时间片，此时是无法处理过来的请求，需要等待其他线程来进行唤醒，该线程才能进行后续操作或者处理其他请求。

**非阻塞**：

意味着，当数据还没准备好的时候，即便我调用了阻塞方法，该线程也不会被挂起，后续的请求也能够被处理。

## 同步

> 同步和异步跟串行和并行非常形似。

假设在一个场景下：完成一个大任务需要4个小任务。

同步的做法：需要依次4个步骤，注意这里是依次，也就是说完成这个步骤，需要先完成前置步骤，也就是说下一个步骤是要看上一个步骤的执行结果。

![image-20230225152049261](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b20a15998c6043c6ad0eede34bb4f53d~tplv-k3u1fbpfcp-zoom-1.image)







异步的做法：可以同时进行4个步骤，无需等待其他步骤的执行结果。

![image-20230225152813351](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/63ee2d9301024852835b4477bdbb485f~tplv-k3u1fbpfcp-zoom-1.image)

阻塞和同步的最本质差别在于：

> **即便是同步，在等待的过程中，线程是不会被挂起，也不需要让出CPU时间片的，**



# 在IO中的体现

>网络编程的基本模型是：**Client**/**Server**模型

两个进程之间要相互通信，其中服务端需要提供位置信息，让客户端找到自己。服务端提供IP地址和监听的端口。

客户端拿着这些信息去**向服务端发起建立连接请求**，通过三次握手成功建立连接后，客户端就可以通过`socket`向服务器发送和接受消息。



## BIO

>BIO通信模型采用的是典型的：**一请求一应答通信模型**

采用BIO通信模型的服务端，通常会由一个独立的`Acceptor`线程负责监听客户端的连接。

他不负责处理请求，他只是起到一个**委派工作**的作用，当他接收到请求之后，会**为每个客户端创建一个新的线程进行链路处理**。

处理完之后，**通过输出流，返回应答给客户端，然后线程被销毁，资源被回收**。

![image-20230225154644328](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ef1fb352b34241b99720f2e853c453c3~tplv-k3u1fbpfcp-zoom-1.image)

该模型的最大问题就是**缺乏弹性伸缩能力**，**服务端的线程个数和客户端的并发访问数**是**`1：1`**的关系。

由于线程是Java虚拟机非常宝贵的资源，当线程书膨胀之后，**系统的性能会随着并发量增加**呈正比的趋势下降。

而且会有`OOM`的风险，当没有内存空间创建线程时，就无法处理客户端请求，最终导致进程宕机或卡死，无法对外提供服务。



最大的问题就是：**每当有一个客户端请求接入时，就会创建一个线程来处理请求**。

为了改进这个**一线程一连接模型**，后面又演进出通过：

- **线程池**
- **消息队列**

来实现**1个或者多个线程处理N个客户端的模型**。

>在这里，无论是线程池和消息队列，都是解决**内存空间**，**线程**的问题，并没有实质性地改变**同步阻塞通信**本质问题

所以这种优化版本的BIO也被称为是**伪异步**。





## 伪异步IO

>采用**线程池**和**任务队列**可以实现一种：**伪异步的IO通信**

![image-20230212155813127](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5529494b070b485d9f944d8eeb4bbfab~tplv-k3u1fbpfcp-zoom-1.image)

将客户端的请求封装成一个`Task`（该任务实现java.lang.Runnable接口），投递到消息队列中。

如果通过**线程池维护一堆处理线程**，去消费队列中的消息。

处理完毕之后，再去通过客户端就可以了，**他的资源是可控的，无论客户端的请求量是多少，也不会发生变化**，同样这也是他的缺点之一。



建立连接的`accpet`方法、读取数据的`read`方法都是**阻塞**。

![image-20230225154908390](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b000a134598b45c0aa78b30bd3c82fd0~tplv-k3u1fbpfcp-zoom-1.image)



![image-20230225155120695](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/66da2098eca346799391bf023ad2b559~tplv-k3u1fbpfcp-zoom-1.image)



这就意味着，如果有一方**处理请求或者发出请求的比较慢**，或者是网络传输比较慢，那么都会影响对方。

当调用**OutputStream**的`write`方法写输出流的时候，它将会被阻塞，直到所有要发送的字节全部写入完毕，或者发生异常。

在TCP/IP中，当消息的接收方处理缓慢的时候，由于**消息滑动窗口**的存在，那么它的接收窗口就会变小，就是那个`TCP window size`。

如果这里采用同步阻塞IO，并且`write`操作被阻塞很久，直到`TCP window size` 大于0或者发生IO异常了。



那么**通信对方返回应答时间过长会引起的级联故障**：

- **线程问题**：假如所有的可用线程都被故障服务器阻塞，那么**后续所有的IO消息都将被队列中排队**。
- **队列问题**：如果队列采用的是`有界队列`，**队列满了之后那么就会无法后续处理请求**；如果采用的是`无界队列`，那么会有OOM风险。



## NIO

>NIO，官方叫法是`new IO`，因为它相对于之前出的java.io包是新增的
>
>但是之前老的IO库都是阻塞的，New IO类库目标就是为了让Java支持非阻塞IO，所有更多的人称为`Non-Block IO`





### 缓冲区Buffer

>**Buffer**是一个对象，通常是**ByteBuffer**类型
>
>**任何时候操作NIO中的数据，都需要经过缓冲区**。

在`NIO`库里，**所有数据操作是用缓冲区处理的**。

![image-20230214224938361](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3718edc0cd30424c9857e19dd6102541~tplv-k3u1fbpfcp-zoom-1.image)

- **读取数据**时，是直接读到缓冲区中（这里并没有**直接读到某个地方**，而是都放到缓冲区中）
- **写入数据**时，写入到缓冲区





缓冲区实质上是一个数组，通常是一个字节数组`ByteBuffer`，自身还需要维护读写位置，可以用**指针**或者**偏移量**来实现。

除了ByteBuffer还有其他**基本类型缓冲区**：

- `CharBuffer`：字符缓冲区
- `ShortBuffer`：短整型缓冲区
- `IntBuffer`：整形缓冲区
- `LongBuffer`：长整型缓冲区
- `DoubleBuffer`：双精度缓冲区

>通常是用**ByteBuffer**



### 通道Channel

>**网络数据通过Channel读取和写入**

Channel通道和Stream流最大的区别在于：

- `Channel`的数据流向是**双向**的
- `Stream`的数据流向是**单向**的

![image-20230214223233287](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bebdef382d41451c8fe5423f1278b7ea~tplv-k3u1fbpfcp-zoom-1.image)

这就意味着：使用Channel，**可以同时进行读和写**，他是**全双工**模型。（可以联想到`HTTP1.1` `HTTP2.0` `HTTP3.0 ``websocket`）



### 多路复用器Selector

>**Selector是NIO编程的基础**

`Selector`会**不断轮询注册**在其上的`Channel`。

![image-20230219140034538](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/454a4bdff95243aba1aa3f5b2e7a5d41~tplv-k3u1fbpfcp-zoom-1.image)

如果某个Channel**发生读写事件**，就代表这个Channel是**就绪状态**，会被Selector轮询出来。

然后根据`SelectionKey`可以获取就绪Channel的集合，进行后续IO操作。

一个Selector可以轮询多个Channel，JDK是基于**epoll**代替传统的select，**所以不受句柄fd的限制**。

意味着，**一个线程负责Selector的轮询千万个客户端**，





## AIO

>`NIO2.0`引入了新的**异步通道**的概念，并提供了**异步文件通道**和**异步套接字通道**的实现

- 通过java.util.concurrent.`Future`类来表示**异步操作的结果**。
- 在执行异步操作的时候传入一个java.nio.channels

**CompletionHandler**接口的实现类作为操作完成的**回调**



`NIO2.0`的**异步socket通道是真正的异步非阻塞IO**。

- **同步**socket channel：`SocketServerChannel`
- **异步**socket channel：`AsynchronousServerSocketChannel`

它不需要通过多路复用器（`selector`）对注册到里面的通过进行**轮询**操作，就可以实现**异步读写**。

>AIO和NIO最大的区别在于：**异步Socket Channel是被动执行对象**

- NIO需要我们把channel注册到selector上进行顺序扫描、轮询
- AIO则是通过`Future`类，实现回调方法：**completed**、**failed**





## 4种IO对比

IO模型主要是探讨2个维度：

- **同步/异步**
- **阻塞/非阻塞**

同步/异步的判断标准主要是：`Channel`的问题

阻塞/非阻塞的判断标准主要是：`selector`的问题

阻塞的关键点在于：**建立连接**和**数据传输**

BIO（阻塞）意味着在完成建立连接（`accpet`）动作之后，才能进行后续操作

NIO（非阻塞）在处理客户端的连接时，可以将对应的channel注册到Selector上，此时我不管他好了没有，我有`Selecotr`来帮我去扫就绪态的**channel**，所以他是非阻塞的





### 异步非阻塞IO

>异步非阻塞IO：`AIO`

有的人也叫`JDK1.4`推出的NIO为异步非阻塞IO

但是严格来说，它只能被称为是**非阻塞IO**，并不是真正意义上的**异步**

前期`selector`的底层是通过**select/poll**来实现的，虽然是用**epoll**替代了**select/poll**，上层的API没有变化，**只是一次NIO的性能优化，仍旧没有改变IO的模型**

在`JDK1.7`提供的`NIO2.0`新增了：**异步套接字通道**，他才是真正的异步IO。



### 多路复用器Selector

>Selector的核心功能：**就是用来轮询注册在它上面的Channel**

当发现某个就绪态的Channel，就会找出他的`SelectionKey`，然后进行后续的IO操作。

前期的时候JDK1.4，selector底层是基于select/poll技术实现

后面优化，使用epoll来代替





### 伪异步IO

>只是在**线程层面**上进行了一次优化，IO模型并没有改变

通过**处理任务Task队列+线程池处理请求**的方式来优化资源

解决了BIO的线程和请求：1对1的关系