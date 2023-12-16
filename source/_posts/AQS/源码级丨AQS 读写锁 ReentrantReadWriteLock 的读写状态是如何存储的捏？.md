---
categories:
  - 源码解读
cover: >-
  https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202311172154410.webp
title: 源码级丨AQS 读写锁 ReentrantReadWriteLock 的读写状态是如何存储的捏？
abbrlink: 6501
date: 2023-12-10 16:50:11
updated: 2023-12-10 16:50:16
tags:
---
>**源码篇** 预防针💉：
>**源码篇** 必然会有很多代码，希望大家不要有畏难情绪，虽然我也是看到一大串代码就头疼。
>我贴代码只是为了方便我的文字解答，源码只是辅助我文字讲解，所以大家尽量关注我的文字就好啦。
# 前言

`ReentrantReadWriteLock` 也叫 **读写锁**，支持 **排他/共享** 特性是他的特点。

| 互斥性 | 读锁 | 写锁 |
|:------ |:---- | ---- |
| 读锁   |  🟩    |  🟥    |
| 写锁   |   🟥   |  🟥    |

- 读读：共享
- 读写：互斥
- 写写：互斥

对于 **多读少写** 的场景来说，使用 **读写锁** 可以提高整体 **读的吞吐量**。这是一种 **锁粒度细化** 的体现。

本篇是介绍 `ReentrantReadWriteLock` 的第一篇，首先我们应该从哪里开始入手呢？

本篇着重从 **读写锁的状态存储** 进行介绍。中间会介绍在 `AQS` 中 `Sync` 到底扮演了一个什么角色？为什么说 `AQS` 是一个同步框架？
# 找到线索入口

>我们该从源码的哪里入手呢？

具体而言我们应该去看哪个方法捏？

肯定是要看 **核心的加锁** 逻辑！

先把 **读锁** `ReadLock` 和  **写锁** `WriteLock` 部分核心代码贴一下方便讲解。

>后面我就直接用 **读锁** 和 **写锁** 简称了

**读锁**：

```java
    public static class ReadLock implements Lock, java.io.Serializable {
        private static final long serialVersionUID = -5992448646407690164L;
        private final Sync sync;

        protected ReadLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }

        public void lock() {
            sync.acquireShared(1);
        }

        public void lockInterruptibly() throws InterruptedException {
            sync.acquireSharedInterruptibly(1);
        }

        public boolean tryLock() {
            return sync.tryReadLock();
        }

        public boolean tryLock(long timeout, TimeUnit unit)
                throws InterruptedException {
            return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
        }

    }
```

**写锁**：

```java
    public static class WriteLock implements Lock, java.io.Serializable {
        private static final long serialVersionUID = -4992448646407690164L;
        private final Sync sync;

        protected WriteLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }

        public void lock() {
            sync.acquire(1);
        }

        public void lockInterruptibly() throws InterruptedException {
            sync.acquireInterruptibly(1);
        }

        public boolean tryLock() {
            return sync.tryWriteLock();
        }

        public boolean tryLock(long timeout, TimeUnit unit)
                throws InterruptedException {
            return sync.tryAcquireNanos(1, unit.toNanos(timeout));
        }

        }
    }
```

我们可以看到在 `ReadLock#lock` 里面有一个非常现眼包的方法调用：`sync.acquireShared`，按理来说我们直接去看这个方法去顺藤摸瓜就行了，答案非常明显。

但其实我这里还想让你注意到 `WriteLock#lock` 里面的 `sync.acquire(1)`，这个是不是很熟悉？

没错这个跟之前讲的可重入锁 `ReentrantLock` 是一模一样的，结论是不是就是：

- **读锁** 他有他自己一套的共享锁逻辑，**写锁** 就走跟 `ReentrantLock` 的逻辑呢？

当然不是，这两个根本不是同一个东西，`Sync` 是各自的内部类大家都不一样，洒瓜！

有一个非常核心关键的东西我在前面一直都不提，就是为了在对比 `ReentrantReadWriteLock` 和 `ReentrantLock` 的时候想要提的，`Sync`。我认为它就是体现 `AQS` 是一个同步框架这个理念的核心。


# Sync 是体现 AQS 框架的核心

>这么 `Sync` 具体是个什么东西呢？

先从它的用法来看，我们可以看到 `AQS` 的各种实现类都会继承一个 `Sync`，然后再细化提供 **公平锁** 和 **非公平锁** 的实现逻辑。

![image.png](https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202312041717323.png)

每一个 `AQS` 的实现类 **内部** 都会写一个 `Sync` 去继承 `AQS`，然后在里面修改对应的核心方法比如：`tryAcquire`、`tryRelease`。赋予这个 `Sync` 具有这个子类的一些特性。

- `ReentrantReadWrite`：共享和排他的特性。
- `ReentrantLock`：可重入的特性。
- .....

>`AQS` 就是一个大的框架，定义好了同步的一些模版方法。
>然后我们这些实现子类用一个内部类（`Sync`）去继承它，然后赋予它子类的特性。

# State 在 ReentrantReadWriteLock 的定义

![image.png](https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202312071703545.png)

## 读锁写状态的存储


```java
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 6317671515068378041L;

        /*
         * Read vs write count extraction constants and functions.
         * Lock state is logically divided into two unsigned shorts:
         * The lower one representing the exclusive (writer) lock hold count,
         * and the upper the shared (reader) hold count.
         */

        static final int SHARED_SHIFT   = 16;
        static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
        static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
        static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

        /** Returns the number of shared holds represented in count  */
        static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
        /** Returns the number of exclusive holds represented in count  */
        static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }

    }
```

对于 `sharedCount`，`exclusiveCount` 方法的参数 `c` 其实就是 `AQS` 里面的 **state** 变量。


注释里面已经很清晰地指出：

- 锁的状态在逻辑上分为高 `16` 位和低 `16` 位

- **高16位代表**：读锁持有数

- **低16位代表**：写锁持有数

>为什么会分成 `16` 位呢？

```java
    private volatile int state;
```

因为 `state` 变量定义是一个 `int` 类型，也就是 `4` 字节，`32` 位。

读写各占一半，所以是 `16`，这是可以不用死记硬背的。

## 读写锁次数的获取

>那我们如何获取读锁和写锁的个数？

```java
    
    static final int SHARED_SHIFT   = 16;
    static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
    static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
    static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;
    /** 获取读锁状态 */
    static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
    /** 获取写锁状态 */
    static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

- **读锁**：只需要通过 **右移** `16` 位。

- **写锁**：是 `state` 余上一个 `EXCLUSIVE_MASK`。

那 `EXCLUSIVE_MASK` 长什么样子呢，可以通过计算可以看出来他二进制就是：`15个1`。（这里是包含了读锁的哦，因为写锁读锁之间也会互斥）

到这里我们已经弄明白了 **写锁和读锁的状态存储以及如何获取读锁和写锁的次数**。

但是我们还没有弄清 读锁和写锁之间的 **排他性和共享性** 是如何实现的，下篇文章就剖析关于 **写锁** 的核心逻辑！



![85b6a1a2d29c98f28f5fc4aa584f54ba.jpg](https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202312112241047.jpg)

> 来都来了，点个赞，留个言再走吧彦祖👍，这对我来说非常重要！