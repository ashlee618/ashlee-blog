---
categories:
  - 源码解读
cover: >-
  https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202311172154410.webp
title: 源码级丨AQS 共享锁 读锁是如何实现多线程共享捏？
abbrlink: 18775
date: 2023-12-11 22:22:37
updated: 2023-12-11 22:22:44
tags:
---


>**源码篇** 预防针💉：
>**源码篇** 必然会有很多代码，希望大家不要有畏难情绪，虽然我也是看到一大串代码就头疼。
>我贴代码只是为了方便我的文字解答，源码只是辅助我文字讲解，所以大家尽量关注我的文字就好啦。

# ReadLock：读锁

## acquireShared：读锁的模版方法

```java
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```

我们可以看到读锁用到了新的模版方法：`acquireShared`

其实我们可以看看 `AQS` 提供了哪些有意思的模版方法：

![image.png](https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202312081605520.png)

按种类其实可以分为：

- 支持共享/排他模式（acquireShared）
- 支持可中断的（acquireInterruptibly）
- 支持带超时时间的（acquireNanos）

>后面会剖析一下他们是怎么支持的，本篇还是继续聊 `ReentrantReadWrite`，继续看 `tryAcquireShared`。

父类方法，同样的也是采用一个抛异常的。

至于为什么这样处理上第一盘文章也介绍了原因，这里就不多啰嗦了：

```java
    protected int tryAcquireShared(int arg) {
        throw new UnsupportedOperationException();
    }
```

这里我们直接看他的实现类 `ReentrantReadWriteLock`，这里其实也给了我们一个信息：

`Semaphore`、`CountDownLatch`、`ReentrantReadWriteLock`。**这类锁都具有共享排他性质**。

所以当我们在进行 **知识整理、知识联想** 的时候，可以从他们几个出发，阐释他们的区别和特性，这也是不错的线索！

![image.png](https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202311202014984.png)

好，我们来到了 `ReentrantReadWriteLock` 里面：


## tryAcquireShared：读锁的核心加锁逻辑


```java
    protected final int tryAcquireShared(int unused) {
        Thread current = Thread.currentThread();
        int c = getState();
        if (exclusiveCount(c) != 0 &&
            getExclusiveOwnerThread() != current)
            return -1;
        int r = sharedCount(c);
        if (!readerShouldBlock() &&
            r < MAX_COUNT &&
            compareAndSetState(c, c + SHARED_UNIT)) {
            if (r == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    cachedHoldCounter = rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
            }
            return 1;
        }
        return fullTryAcquireShared(current);
    }
```

这里我用图片来给你划分好区域方便给你解释:

![image.png](https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202312111955205.png)

我们可以看到：

- **第一部分**：还是比较正常的判断可重入

- **第二部分**：有个比较 **现眼包** 的 `readerShouldBlock`，这是关于 **公平性** 的处理。

- **第三部分**：这里有很多新的东西，这些都是跟 **读锁多线程共享** 相关的。

我们可以大致地这样划分，后面我们就比较好去探究他们到底是干嘛的。

## 给锁降级留的 Hook

```java
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
            return -1;
```

如果你是顺着我的思路下来阅读的话，那么你可能忽略到一个关键点，也就是 **第一部分** 里面的 **可重入判断条件**。

这里正是 **读写锁** 的 `锁降级` 的核心扩展点。

本篇先不详细展开聊 `锁降级`，我会通过新开一篇文章，通过 `oracle` 官网给出的 `锁降级` 例子来讲清楚。卖个关子。

## readerShouldBlock：读锁的公平性体现

```java
    /** 公平锁 */
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -2274990926593161451L;
        final boolean writerShouldBlock() {
            return hasQueuedPredecessors();
        }
        final boolean readerShouldBlock() {
            return hasQueuedPredecessors();
        }
    }
    /** 非公平锁 */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -8159625535654395037L;
        final boolean writerShouldBlock() {
            return false; // writers can always barge
        }
        final boolean readerShouldBlock() {
            /* As a heuristic to avoid indefinite writer starvation,
             * block if the thread that momentarily appears to be head
             * of queue, if one exists, is a waiting writer.  This is
             * only a probabilistic effect since a new reader will not
             * block if there is a waiting writer behind other enabled
             * readers that have not yet drained from the queue.
             */
            return apparentlyFirstQueuedIsExclusive();
        }
    }
```

我们可以看到在 `ReentrantReadWrite` 眼里看来：

- **公平锁**：交给 `CLH` 队列
- **非公平锁**：特殊判断 `apparentlyFirstQueuedIsExclusive`

>`apparentlyFirstQueuedIsExclusive` 究竟是一个什么东西捏？

读者可以尝试理解一些源码的注释，我虽然单词都能看懂，但是串在一起根本看不懂，暴露自己英语水平了....，所以还是直接一点看代码。**show me the code**。
## apparentlyFirstQueuedIsExclusive：读锁的非公平锁插队

```java
    final boolean apparentlyFirstQueuedIsExclusive() {
        Node h, s;
        return (h = head) != null &&
            (s = h.next)  != null &&
            !s.isShared()         &&
            s.thread != null;
    }
```

代码还是比较直观：队列有元素，并且 **队首等待节点** 是 **写锁**，那么就返回 `true`。

那么再用简洁的话语来概括这个方法就是：**如果队首节点是写锁的话**。

```java
        protected final int tryAcquireShared(int unused) {
            Thread current = Thread.currentThread();
            int c = getState();
                // 如果其他线程已经持有排它锁，那么返回失败
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
            int r = sharedCount(c);
            // 非公平锁：如果首个节点不是写锁的话才能进去
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            return fullTryAcquireShared(current);
        }
```

即便是 **非公平锁** 也要遵循 **排他和共享** 的性质，所以这里我们可以给出一个结论就是：

>总的来说：对于 **读写锁** 的公平性来说，必须要先保证 **排他/共享** 性质之后，才能追求 **公平性**。

- **写锁**：能不能插队？**不能**！因为任意给写锁插队带来的后果就是可能对 **排他/共享** 性质的破坏。

- **读锁**：能不能插队？**能**！但是要注意，他不能插 **写锁** 的队，所以要先判断 **队首元素不是写锁**，才能进行插队。

然后就可以进行 **读锁** 的获取了，可以看到这里有很多 **读锁** 新的东西 `firstReader`，`HoldCounter`、`cachedHoldCounter`、`readHolds`。

## readHolds：读锁计数器对象

>由于我们每个线程都可以同时持有 **读锁**，所以我们需要在每个线程中维护一个 **读锁计数器** `HoldCounter`。

![image.png](https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202312112203932.png)


这个 **读锁计数器** 很像一个东西，不知道你读到这里有没有跟我一样的感悟。

就是我们 `JVM` 中的 **程序计数器**，程序计数器是用来记录当前线程执行字节码的信息，他是 **线程私有**的。

同样的我们 **读锁计数器** 也是 **线程私有** 的。他是基于 `ThreadLocal` 实现的。

```java
    /**
     * The number of reentrant read locks held by current thread.
     * Initialized only in constructor and readObject.
     * Removed whenever a thread's read hold count drops to 0.
     */
    private transient ThreadLocalHoldCounter readHolds;
    
    /**
     * ThreadLocal subclass. Easiest to explicitly define for sake
     * of deserialization mechanics.
     */
    static final class ThreadLocalHoldCounter
        extends ThreadLocal<HoldCounter> {
        public HoldCounter initialValue() {
            return new HoldCounter();
        }
    }
```

我们可以看到他是一个 `ThreadLocalHoldCounter` 对象，他是一个 `ThreadLocal`。里面有一个 `HoldCounter`。

### HoldCounter：计数器信息

```java
    /**
     * A counter for per-thread read hold counts.
     * Maintained as a ThreadLocal; cached in cachedHoldCounter
     */
    static final class HoldCounter {
        int count = 0;
        // Use id, not reference, to avoid garbage retention
        final long tid = getThreadId(Thread.currentThread());
    }
```

- **次数**：重入次数

- **持有的线程id**：而且这里贴心地注释了使用线程id而不是引用，是为了避免存留垃圾

### firstReader 和 firstReaderHoldCount：首个读锁持有者和次数

```java

private transient Thread firstReader = null;
private transient int firstReaderHoldCount;

```

这里特意地保留了首个 **读锁** 的持有者，大家可以思考一下为什么？

这里贴一点部分赋值语句方便理解：

`tryAcquireShared`：

```java
    if (r == 0) {
        // 首个读线程
        firstReader = current;
        firstReaderHoldCount = 1;
    } else if (firstReader == current) {
        firstReaderHoldCount++;
    } else {
        HoldCounter rh = cachedHoldCounter;
        if (rh == null || rh.tid != getThreadId(current))
            // 缓存起来
            cachedHoldCounter = rh = readHolds.get();
        else if (rh.count == 0)
            readHolds.set(rh);
        rh.count++;
    }
```
### cachedHoldCounter：最后一个成功获取读锁持有者的计数器信息

```java
    private transient HoldCounter cachedHoldCounter;
```

详细看上图。


目前为止我们只需要知道 `HoldCounter` 是为了 **维护每个线程读锁的可重入次数** 即可，因为里面的内容实在是太精彩，太多了，所以打算在下一篇单独讲讲！。

上面这里解释介绍 **多线程下读锁重入** 的核心逻辑了。

## fullTryAcquireShared：完整版地acquireShared

```java
final int fullTryAcquireShared(Thread current) {
            /*
             * This code is in part redundant with that in
             * tryAcquireShared but is simpler overall by not
             * complicating tryAcquireShared with interactions between
             * retries and lazily reading hold counts.
             */
            HoldCounter rh = null;
            for (;;) {
                // 维护 HoldCounter 的状态
                int c = getState();
                if (exclusiveCount(c) != 0) {
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                    // else we hold the exclusive lock; blocking here
                    // would cause deadlock.
                } else if (readerShouldBlock()) {
                    // Make sure we're not acquiring read lock reentrantly
                    if (firstReader == current) {
                        // assert firstReaderHoldCount > 0;
                    } else {
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != getThreadId(current)) {
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        if (rh.count == 0)
                            return -1;
                    }
                }
                // 正常的加锁逻辑
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (sharedCount(c) == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;
                }
            }
        }
```

可以看到 `fullTryAcquireShared` **后半段代码** 其实大致上跟 `tryAcquireShared` 是没什么区别的。

唯一不同的是 **上半段代码** 他对 `HoldCounter` 进行了状态维护，比如 `remove` 掉 `ThreadLocal`，这里先按下不表。

好了，到这里其实 **读锁** 的核心逻辑我们也已经捋清楚了，最后来一个文字版的总结。

>读锁文字版解释：

```java
    protected final int tryAcquireShared(int unused) {
        Thread current = Thread.currentThread();
        int c = getState();
        if (exclusiveCount(c) != 0 &&
            getExclusiveOwnerThread() != current)
            return -1;
        int r = sharedCount(c);
        if (!readerShouldBlock() &&
            r < MAX_COUNT &&
            compareAndSetState(c, c + SHARED_UNIT)) {
            if (r == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    cachedHoldCounter = rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
            }
            return 1;
        }
        return fullTryAcquireShared(current);
        }
```

- 先看看 **否存在排它锁和可重入性**，（这里其实涉及到锁降级，下篇文章讲解）
- 通过 `readerShouldBlock` 根据 **公平性** 判断是否需要进行判断
    - **公平锁**：丢给 `CLH` 队列排队。
    - **非公平锁**：判断 **队首节点** 是否是 **排它锁**，如果不是排它锁，则可以进行插队。
    - 继续判断不超过 **最大持有数**，就会进行 `CAS` 获取。
        - **获取成功**：
          因为读写锁是共享的，所以需要每个线程各自维护一个 `HoldCounter`。对于 首次获取锁的线程 `firstReader` 还需要特意记录下来（目的是为了性能优化)。
          读写锁还会维护最后一个成功加锁的计数器信息 `cachedHolderCounter`，目的也是为了性能优化，具体体现在释放锁那里。
        - **获取失败**：
          获取失败的时候，则需要进行完整版的 锁获取 `fullTryAcquireShared`
          里面大致一样，唯一不同的是会维护 `HoldeCounter` 状态，如果 **count** 数为 **0**，则直接 **remove** 这个 `ThreadLocal`，避免内存溢出。

![image.png](https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202312112319405.png)


# 留点疑问？

结合本篇文章内容，你已经对 **读锁** 了解了七七八八，但是这些基本上都是基本水平，如果你希望在面试中 **脱颖而出** ，那么就必须会点 **奇奇怪怪** 的亮点！

我这里稍微给你提点问题，后面再来给你解答：

## 为什么每个线程都维护了 HolderCounter，还需要额外维护一个 firstReader ？

## 为什么需要特意维护 CacheHolderCounter？

以上回答会在通过 `jdk` 的 `git` 提交日志来详细告诉你，为什么需要这两个东西！

期待一下8。


![85b6a1a2d29c98f28f5fc4aa584f54ba.jpg](https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202312112241047.jpg)

> 来都来了，点个赞，留个言再走吧彦祖👍，这对我来说非常重要！

