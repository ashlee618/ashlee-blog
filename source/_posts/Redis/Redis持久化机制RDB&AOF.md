---
title: Redis持久化机制RDB&AOF
categories:
  - 缓存中间件
tags:
  - Redis
  - 缓存中间件
cover: >-
  https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202307091752364.png
abbrlink: 17818
updated: 2023-07-19 10:54:43
---



# 为什么需要持久化
我们都知道redis之所以快，原因之一是因为他操作是在内存之上的。但是我们也知道内存快是快，但是无法保证数据的安全性，一旦发生故障，带来的可能是毁灭性的损失

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a4a78b16ea994d1491f5c880731fad29~tplv-k3u1fbpfcp-watermark.image?)

Redis提供了两种持久化的机制，一种是RDB，另外一种是AOF

# RDB持久化机制

RBD持久化机制是基于快照（Snapshot），redis每隔一段时间（用户可以设置）产生一个快照文件（*.dump）进行持久化存储。redis可以通过`Save`命令或`Bgsave`来触发RDB持久化存储。

## Save

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8c299f60788f41c896395774db32a1bd~tplv-k3u1fbpfcp-watermark.image?)

在生成快照的期间，会阻塞服务器，在此期间，redis不能进行读写操作


## Bgsave

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c16a14259fe54c019f3987c665e080a7~tplv-k3u1fbpfcp-watermark.image?)

这种方式会通过调用Linux中的fork命令，`fork`中实现了`COW`（copy on write）读时拷贝，所以bgsave可以概括为 fork cow 

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/73d4ebccf3b6429893be6ec3f72733ab~tplv-k3u1fbpfcp-watermark.image?)

区别于sava，他是fork出一个子线程来完成dump文件的生成，并且fork而引起的阻塞相比save方式是非常小的。
Linux中的`fork`命令实现了`cow`机制，只有当父进程发生写操作，数据发生修改的时候，才会真正分配内存空间，并复制内存数据。
> 注意： 
> - 数据是精确到某一时刻的数据，并不是一个时间段
> - 复制到内存也中的数据是被修改的数据，并不是全部的内存数据




👍 **RBD方式持久化的优点：**

1.  只有一个dump文件，方便维护，轻量
1.  方便数据备份，容灾性好
1.  性能最大化，采用fork的方法可以不影响redis服务
1.  面对数据集大的时候，比AOF的启动效率高

👎 **RBD方式持久化的缺点：**

1.  安全性比AOF低，因为他是间隔触发的，因此不适合对数据要求特别严苛的场景，比如金融之类的
1.  fork出的子线程在生成dump的时候也会影响cpu的效率

# AOF持久化机制

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/14f9a2655f8d4bad9c58173e614c7362~tplv-k3u1fbpfcp-watermark.image?)

AOF基于日志的形式进行持久化数据，有点像mysql的binlog。不过binlog存放的是二进制数据，而AOF的日志存放的是一行一行的命令。

**流程：**

1.  所有的写命令都会追加到AOF缓冲中
1.  AOP缓冲区根据**对应的策略**向硬盘进行同步操作
1.  随着AOF文件越来越大，需要定期对AOF文件进行**重写**
1.  当Redis重启时，可以加载AOF文件进行数据回复


**同步策略：**

-   每秒同步：异步完成，效率非常高
-   每修改同步：同步完成，每发生一次数据变化都会被立即记录到磁盘中
-   不同步：交给操作系统控制

👍**AOF方式持久化的优点：**

1.  数据安全
1.  通过append模式写文件，即使中途宕机也不会破坏已经存在的内容

👎**AOF方式持久化的缺点：**

1.  AOF文件比RDB文件大，且回复速度慢
1.  数据集打的时候，比RDB启动效率慢
1.  运行效率也没有比RDB高

## 重写机制

随着指令的增加，AOF文件大小也会越来越大。为了压缩文件大小，redis提供了一个文件重写的机制

## 混合模式

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/97aa06b4e0fb47b5b1565584642ddfc9~tplv-k3u1fbpfcp-watermark.image?)
redis4.x开始支持混合模式，开启方式： `aof-use-rdb-preamble true`

redis在重启时会采用AOF模式，因为RDB数据并不完整；在触发重写机制的时候，会把内存中的数据以RDB的方式写入AOF中，再把AOF缓冲区的数据以AOF写入到文件、
