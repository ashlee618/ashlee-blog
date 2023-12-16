---
categories:
  - 源码解读
cover: >-
  https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202311172154410.webp
title: 源码级丨AQS 共享锁 写锁核心加锁逻辑
abbrlink: 46557
date: 2023-12-10 16:50:11
updated: 2023-12-10 16:50:16
tags:
---


>**源码篇** 预防针💉：
>**源码篇** 必然会有很多代码，希望大家不要有畏难情绪，虽然我也是看到一大串代码就头疼。
>我贴代码只是为了方便我的文字解答，源码只是辅助我文字讲解，所以大家尽量关注我的文字就好啦。
# 模版方法

好了到这里就要开始进入主题了，按照之前的步骤，我们要开始从 `AQS` 的模版方法开始介绍 `ReentrantReadWriteLock`。
# WriteLock：写锁

```Java
    public static class WriteLock implements Lock, java.io.Serializable {
        private static final long serialVersionUID = -4992448646407690164L;
        private final Sync sync;

        protected WriteLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }
        /** 调用父类的 acquire 方法 */
        public void lock() {
            sync.acquire(1);
        }
    }
```

## acquire：写锁的模版方法

```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
        }
```

这里再贴一下 `AQS` 的模版方法，然后我们需要关注的就是 `ReentrantReadWriteLock` 实现的 `tryAcquire` 方法。
## tryAcquire：写锁的加锁逻辑

`WriteLock` 核心的加锁逻辑。

```java
    protected final boolean tryAcquire(int acquires) {
        /*
         * Walkthrough:
         * 1. If read count nonzero or write count nonzero
         *    and owner is a different thread, fail.
         * 2. If count would saturate, fail. (This can only
         *    happen if count is already nonzero.)
         * 3. Otherwise, this thread is eligible for lock if
         *    it is either a reentrant acquire or
         *    queue policy allows it. If so, update state
         *    and set owner.
         */
        Thread current = Thread.currentThread();
        int c = getState();
        int w = exclusiveCount(c);
        // c ！= 0 代表着我并不是首次来获取锁，之前就已经存在了读写锁了
        // 如果我是首次来获取锁的线程，我可以很开心的直接通过下面的 cas 去进行加锁
        if (c != 0) {
            // 当我发现我不是读写锁
            // (Note: if c != 0 and w == 0 then shared count != 0)
            if (w == 0 || current != getExclusiveOwnerThread())
                return false;
            if (w + exclusiveCount(acquires) > MAX_COUNT)
                throw new Error("Maximum lock count exceeded");
            // Reentrant acquire
            setState(c + acquires);
            return true;
        }
        if (writerShouldBlock() ||
            !compareAndSetState(c, c + acquires))
            return false;
        setExclusiveOwnerThread(current);
        return true;
    }
```

如果你把源码上面的注释放到翻译软件上面看，就会得到这坨东西：

![image.png](https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202312071720436.png)

不知道对你有没有帮助，反正对我是没有一点帮助。

对于 `tryAcquire` 相信你看过我之前对 `ReentrantLock` 的讲解应该基本是没问题，需要特别注意的只有他多了一个 `writerShouldBlock` 方法，而这个方法就是 **写锁公平性** 的体现。
## writerShouldBlock：写锁的公平性体现

我们再来看看 `writerShouldBlock` 这个方法


```java
    /** 公平锁 */
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -2274990926593161451L;
        final boolean writerShouldBlock() {
            return hasQueuedPredecessors();
        }
    }
    /** 非公平锁 */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -8159625535654395037L;
        final boolean writerShouldBlock() {
            return false; // writers can always barge
        }
    }
```

- **公平锁**：就很简单，直接让交给 `CLH` 锁去处理，关于 `CLH` 锁的介绍在上一篇文章有解释：TODO：。

- **非公平锁**：是直接返回 `False`，**始终允许插队**， 这也是 **排他锁** 的特性体现。


这边贴一个我用更直白的话语注释好的，方便读者了解详细的过程

```java
    protected final boolean tryAcquire(int acquires) {
        Thread current = Thread.currentThread();
        int c = getState();
        int w = exclusiveCount(c);
        // c ！= 0 代表着我并不是首次来获取锁，之前就已经存在了读锁或写锁了
        // 如果我是首次来获取锁的线程，我可以很开心的直接通过下面的 cas 去进行加锁
        if (c != 0) {
            // 排他性的判断 + 写锁可重入的判断 + 最大多数判断
            // 当我发现我不是首个线程来获取的时候，然后我要获取写锁，但是当前线程并不是写锁持有线程
            // 这是一种什么情况？其实下面的注释已经告诉我们了，就是在获取写锁之前，全是持有读锁。
            // 写锁和读锁是互斥的，所以也是不能获取锁成功，这里就体现了读写锁互斥性的一个特点
            // (Note: if c != 0 and w == 0 then shared count != 0)
            if (w == 0 || current != getExclusiveOwnerThread())
                return false;
            // 这里判断是否超过最大的锁的次数
            if (w + exclusiveCount(acquires) > MAX_COUNT)
                throw new Error("Maximum lock count exceeded");
            // 都满足了合法性检查，那么就可以直接更新 state 代表获取锁成功了
            // Reentrant acquire
            setState(c + acquires);
            return true;
        }
        // 来到这里还需要判断 公平性
        // 公平锁：如果有前置节点，代表需要排队，就交给 CLH 去排队
        // 非公平锁：总是允许插队，直接通过 CAS 插队
        if (writerShouldBlock() ||
            !compareAndSetState(c, c + acquires))
            return false;
        setExclusiveOwnerThread(current);
        return true;
    }
```

## tryRelease：写锁的释放锁逻辑

```java
    protected final boolean tryRelease(int releases) {
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        int nextc = getState() - releases;
        boolean free = exclusiveCount(nextc) == 0;
        if (free)
            setExclusiveOwnerThread(null);
        setState(nextc);
        return free;
    }
```

先判断是否是 **锁持有线程**，然后再根据 **锁重入次数** 判断 **是否完整释放锁**。

# 写锁获取锁流程


![image.png](https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202312111922170.png)



- 判断是否是 **之前是否有持有锁** 来获取写锁：
    - 如果是：进行 **合法性判断**：
        - 排他性判断（如果已经有共享锁，则会失败，因为排他性质）。
        - 可重入性判断。
        - 最大次数判断。
    - 如果不是：则还需要根据 **公平性** 进行 `CAS` 获取。
        - **公平锁**：就扔去 `CLH` 队列排队获取
        - **非公平锁**：直接插队进行 `CAS` 获取锁，不行再丢去 `CLH` 队列排队。

# 总结


- **写锁的互斥性**：在获取锁的时候，会先根据 `state` 判断是否持有锁，这里包含了 **排他性** 的判断，由于写锁和所有锁都互斥，所以只需要判断 `state ≠ 0`，代表这里产生互斥了并且如果不是锁持有的线程，那么就会获取锁失败。

- **写锁的公平性** ：体现在 `writerShouldBlock` 这个方法中，**公平锁** 直接丢 `CLH` 队列乖乖排队，**非公平锁** 可以先插队进行 `CAS` 尝试获取锁，获取失败再丢去 `CLH`。


![85b6a1a2d29c98f28f5fc4aa584f54ba.jpg](https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202312112241047.jpg)

> 来都来了，点个赞，留个言再走吧彦祖👍，这对我来说非常重要！