---
date: 2023-12-14 22:06:15
updated: 2023-12-14 22:06:17
tags: 
categories:
  - 源码解读
cover: https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202311172154410.webp
title: 源码级丨从 jdk 的提交记录告诉你 AQS 读写锁 ReentrantReadWriteLock 为什么需要特意维护 firstReader 和 cacheHoldCounter
---


>**源码篇** 预防针💉：
>**源码篇** 必然会有很多代码，希望大家不要有畏难情绪，虽然我也是看到一大串代码就头疼。
>我贴代码只是为了方便我的文字解答，源码只是辅助我文字讲解，所以大家尽量关注我的文字就好啦。
# 前言

在上一篇文章中，我们得知了 `ReentrantReadWriteLock` 给每个线程都维护了一个 `ThreadLocal` 类型的 **HoldCount** 对象，用来维护每个线程的读锁重入次数。

然后我留下了一些疑问：

- **为什么每个线程都维护了 HoldCounter，还需要额外维护一个 firstReader ？**

- **为什么需要特意维护 CacheHoldCounter？**

# 为什么需要特意维护 firstReader 和 cacheHoldCount

为什么 `ReentrantReadWriteLock` 特意维护了 第一个线程 和 最后一个 获取锁成功的线程？

只维护头和尾能带来性能上的提升吗？具体体现在哪里？

## ThreadLocal 的查询

>我们先来看 为什么需要 `cacheHoldCount`

在源码 `fullTryAcquireShared` 方法中其实我们可以看到有这么一个注释：

![image.png](https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202312142123659.png)

显然 `cacheHoldCounter` 是为了 **释放锁** 而特意维护的。

那 `cacheHoldCounter` 究竟能帮助我们怎么去优化 **释放锁** 呢？

![image.png](https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202312142127748.png)

看到这里基本上就是真相大白了。

`cachedHoldCounter` 是用来优化 **锁释放时** 的 `ThreadLocal` 查找的。

而且这里提到了一个普遍的例子就是：**通常来说需要释放的线程，就是最后一个获取锁的线程**。

在这种场景下，我们就不需要去通过  `ThreadLocal` 去查，而是直接通过我们这个 `cacheHoldCounter`。

>那为什么需要维护 `firstReader` 捏？

![image.png](https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202312142134557.png)


通过注释其实是不太好找到线索的。

但是我们可以先确定的一件事就是，`firstReader` 肯定能起到跟 `cachHoldCounter` 的作用，也就是在优化了在 `ThreadLocal` 的查找。

>`tryReleaseShared` 读锁释放逻辑：

![image.png](https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202312142137505.png)


## ThreadLocal 的内存溢出风险

>我们通过 `jdk` 的 `git` 历史提交记录可以看到，这个 `firstReader` 其实是后来才加进来的。


![image.png](https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202312142149820.png)


![](https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202312142147094.png)

**Excessive ThreadLocal storage used by ReentrantReadWriteLock**。

大概意思就是：`ReentrantReadWriteLock` 用 `ThreadLocal` 太多了。

`ThreadLocal` 虽然是一个好东西，但是它有一个非常大的问题就是它存在 **内存溢出** 的风险。


![image.png](https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202312142149078.png)

现在多加了一个 `firstReader`，相当于是一把 `ReentrantReadWriteLock` 就优化一个 `ThreadLocal` 变量。

积少成多，也是对性能比较极致的追求。

而且我们还可以看到这个提交记录中还包括了对锁中的 `ThreadLocal` 对象手动清除： `readHolds.remove()`。

这属于是一个 `bug`，也是在这次提交中修复了。

感兴趣的大兄弟可以去具体 `oracle` 官网查看：[Bug ID: JDK-6625723 Excessive ThreadLocal storage used by ReentrantReadWriteLock (java.com)](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=6625723)


# 总结

>为什么需要 `cacheHoldCounter` ?

`cacheHoldCounter` 是维护最后一个成功获取到锁的线程。

维护它是为了在 **锁释放** 中减少对于 `ThreadLocal` 的查询。

>为什么需要 `firstReader` ？

目的是为了优化 **读写锁** 中对 `ThreadLocal` 的过度使用导致内存溢出问题。

在早期设计 **读写锁** 中是没有 `firstReader`，后来人们提出他对于 `ThreadLocal` 的使用已经达到了 **滥用** 这个级别。

但是又不得不用 `ThreadLocal`，所以 **Doug Lea** 采用了维护一个 `firstReader` 的方法，尽可能地去减少这个内存的占用。

同时 `fistReader` 在 **锁释放** 的时候也可以优化 `ThreadLocal` 的查询。

# 参考

- [一文带你搞懂什么是AQS及其组件的核心原理](https://zhuanlan.zhihu.com/p/267679376)

- [关于ReentrantReadWriteLock，首个获取读锁的线程单独记录问题讨论（firstReader和firstReaderHoldCount） - 掘金 (juejin.cn)](https://juejin.cn/post/6951566514017271821) 

![85b6a1a2d29c98f28f5fc4aa584f54ba.jpg](https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202312112241047.jpg)

> 来都来了，点个赞，留个言再走吧彦祖👍，这对我来说非常重要！