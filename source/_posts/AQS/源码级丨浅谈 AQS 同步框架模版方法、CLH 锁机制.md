---
categories:
  - '源码解读'
cover: >-
  https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202311172154410.webp
title: 源码级丨浅谈 AQS 同步框架模版方法、CLH 锁机制
abbrlink: 50623
date: 2023-11-17 21:52:18
updated: 2023-11-17 21:52:18
tags:
---


# 引言

`AQS` (**AbstractQueuedSynchronizer**) 是一个提供了锁和同步器基础功能的框架。

本文尝试通过以 `ReentrantLock` 为例子，去介绍 `AQS` 这个框架是如何从框架的角度上，去辅助实现同步机制的。

从源码的角度上，侧重介绍 `AQS` 中的 `acquire` 模版方法里面涉及到的整个流程。

这里并不会去过分去介绍 `ReentrantLock` 的特性，只是一个辅助讲解 `AQS` 的引子。

>为什么我会选择 `acquire` 这个方法捏？

```java
    static final class NonfairSync extends Sync {
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
    }

    // 公平锁
    static final class FairSync extends Sync {
        final void lock() {
            acquire(1);
        }
    }
```

我们可以看到 `ReentrantLock`，无论是 公平锁还是非公平锁，里面都会出现这个 `acquire` 方法的调用。

由此可见，里面很有可能就包含着我们想要知道的一切。

>**注意**：本文是基于 `JDK8` 分析的，在高版本中源码的写法可能不一样。

# acquire

```java
public final void acquire(long arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

在 `acquire` 方法中，我们可以看到 主要的核心方法有：

- **tryAcquire**
- **addWaiter**
- **acquireQueued**

下面就带着大家一起分析这三个方法到底在整个流程是干什么的。

## tryAcquire


```java
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }
```

在 `AQS` 中，`tryAcquire` 方法是直接抛异常。

相信读者也能从许多框架源码中看到很多类似的，一个父类方法写的是直接一个抛异常。

这样的写法框架的作者通常是考虑：

- 不需要所有的子类都实现。
- 如果有的子类没有自己实现就调用了，那么我应该直接给他抛异常，实现 `fail-fast` 机制。

>快去复习一下 `fail-fast` 和 `fail-save` 两种机制！


## addWaiter(Node.EXCLUSIVE)

```java
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```

尝试添加 `Node` 节点到 `tail` **尾部**。

如果队列为空，才会去走 `enq` 方法。

源码注释中提到了 **fast path** 这个词，`addwaiter` 这里考虑的情况是 尾节点 `tail` 不为空的情况。

- 如果 `tail` 不为空，那么核心逻辑就只存在 `addWaiter` 这里。
- 如果 `tail` 为空，那么核心逻辑会去走 `enq`，`enq` 里面讨论了 空 和 非空 两种情况。

这里只是一个 **“偷鸡”** ，**fast path** 的尝试。

## enq

```java
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                // 队尾为 null 则通过 CAS 把自己添加到队尾
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                // 发现不为空表明这里已经出现并发竞争，有一个线程成功把他加入到队尾了
                // 那么我这个线程就只能在他后面加入了
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

# acquireQueue

```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            // 自旋
            for (;;) {
                final Node p = node.predecessor();
                // 只有前驱节点是 head 才能再尝试 tryAcquire 去获取锁
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                // 上面尝试获取锁失败了走到这里，决定当前线程是否该阻塞
                // p 和 前驱节点，也就是根据 前驱节点 判断 当前线程是否阻塞
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

## CLH 锁机制

这里我们可以看到只有 前驱节点 是 `head`，才有资格进行 `tryAcquire` 去获取锁，并不是所有在同步队列的线程，都有机会去获取锁。`AQS` 所采用的这种机制，其实是 `CLH` 锁的一种变种。

>`CLH` 锁是一种简单高效的 **自旋锁**，得名于其发明者 **Craig**, **Landin** 和 **Hagersten**。

它基于 **链表**，其每个节点代表一个试图获取锁的线程。

在 `CLH` 锁中，线程不会直接对锁进行自旋等待，而是在其前驱节点的一个副本上自旋，这样减少了对共享变量的访问，从而降低了系统的总线和缓存压力。

要说 `CLH` 好，那么它好在哪里？它又解决了什么问题？

- 在没有 `CLH` 的时候，10个线程过来，通过自旋去获取锁。

- 有了 `CLH`，10个线程过来，只有1个线程会自旋去获取锁。

自旋的成本，就可以大大的降下来了。所以我们会说 `CLH` 是一种 **高效的自旋锁**。

>那么就会有人会问：维护竞争的线程，`addWaiter` 也是通过 `CAS` 将线程添加到队尾维护的呀，不也是自旋嘛？

好好好，当你注意到这点的时候，我只能说你很细，但是还不够！

需要注意的是在 `addWaiter` 自旋，它的竞争对象是 **前驱线程对象**，因为我要把自己加到前驱对象的后面。

但是另外一个是具体的一个 **共享资源对象**，这两者的 **竞争压力是完全不一样的**。

而且当我 `CLH` 线程排好队了之后，以后的竞争压力就大大减少，性能就更加高效，所以我们说他是一个好东西。


### CLH 是天然的公平锁，非公平锁怎么办？

在前面我们已经知道了 `CLH` 是一种 **公平锁** 的机制。`AQS` 作为一种通用的同步框架，那是不是意味着所有的子类都是公平锁呢？那么我们为什么说 `ReentrantLock` 有公平和非公平两种模式。

直接上源码！

ReentrantLock的 公平模式 是通过 里面的 `sync` 类型来实现的：

- **公平锁**：`FairSync`
- **非公平锁**：`NonfairSync`

```java
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
    
    public void lock() {
        sync.lock();
    }
```

我们再去看两者有啥区别：

```java
    // 非公平锁
    static final class NonfairSync extends Sync {
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
    }

    // 公平锁
    static final class FairSync extends Sync {
        final void lock() {
            acquire(1);
        }
    }
```

我们可以很清楚地看到，非公平锁很 “**鸡贼**” 地在提前去通过 `CAS` 就进行获取锁了。

它不走 `acquire` 那套模版方法，所以这里就体现了 **非公平锁** 的非公平之处。

但是如果这次这个 **非公平锁** 没有成功获取到锁，那么它也是会被扔到同步队列里面去，让他 **乖乖地排队去排队自旋**。

### 分布式锁羊群效应

`CLH` 其实跟分布式锁中解决 **羊群效应** 非常相似。

>**羊群效应**：在分布式锁系统中，当多个请求同时到达时，它们可能会形成一种“群集”，或在某个资源（例如数据库或服务）上排队。

以 `ZooKeeper` 举例子，当一个持有分布式锁的节点释放锁时，它会在 `ZooKeeper` 中更新对应的节点状态。这个状态更新会被其他正在等待锁的节点观察到，从而触发它们几乎同时去尝试获取锁。

`Curator` 是一个优秀的 `ZooKeeper` 客户端框架，它是通过 **顺序节点** 和 **最小节点观察** 的办法去解决这种羊群问题。

原理上跟我们这里的 `CLH` 思想是一致的：**我只关心前驱节点，而不是具体的共享资源对象**。

由此我们还可以联想到：`push` 和 `pull` 两种负载均衡模型他们各自的特点。

由于该内容并不是本文主要介绍对象，所以这里就点到为止。

## shouldParkAfterFailedAcquire

```java

    /** 
     ** Node 节点状态
     **/
    static final class Node {
        
        /** waitStatus value to indicate thread has cancelled */
        static final int CANCELLED =  1;
        /** waitStatus value to indicate successor's thread needs unparking */
        static final int SIGNAL    = -1;
        /** waitStatus value to indicate thread is waiting on condition */
        static final int CONDITION = -2;
        /**
         * waitStatus value to indicate the next acquireShared should
         * unconditionally propagate
         */
        static final int PROPAGATE = -3;
        
    }

    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        // 前驱节点状态是 SIGNAL 的话，当前线程可以直接挂起
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        // 前驱节点状态 > 0，也就是 CANCELLED，会一直往前去找非 CANCELLED 的节点
        // 把自己的前驱指向它
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
             // 通过 CAS 方式将前驱节点修改为 SIGNAL，让他来通知我当前线程
             // 我当前线程睡着就可以了
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

根据前驱节点的状态，来判断我当前线程是否应该 `park`，也就是阻塞，前驱节点的取值就是要看 `waitStatus`。

`waitStatus` 的取值有 4 种，我们这边只针对以 `ReentrantLock` 为例子，因为其他的涉及到一些 **读写锁** 的，样例不够通用，不好进行阐释。因此我们这里只针对 `ReentrantLock` 举例子，那么 `waitStatus` 就有：

-  `SIGNAL`：如果前驱节点是 `SIGNAL`，当前驱节点 **获取到锁** 或 **释放锁** 时，会唤醒我当前节点，因此我当前节点直接挂起即可，让出 CPU 资源。
  
  可以想象这么一个场景，小明和你一起去问老师问题（锁获取），但是老师每次只能处理一个学生，那么小明跟你说，我先去问问题，等我问完（锁释放），就来告诉你（唤醒你），你先睡一会吧（当前线程挂起），这样就会更好懂一点。

- `CANCELLED`：如果是前驱节点是 `CANCELLED`，代表前驱节点是一个 由于中断 `Interrupt` 而放弃竞争的线程，那么我这个节点的前驱节点要往前一直修改去 **找到前面是有效的节点**，也就是 非`CANCELLED` 的前驱节点。

- **其他的**：对于其他的状态值，会通过 `CAS`，将前驱节点修改为 `SIGNAL`，让他能够通知我当前这个节点。

## parkAndCheckInterrupt

```java
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```


可以看到 `AQS` 这里是通过 `LockSupport` 来阻塞当前线程的。


### 为什么这里用的是 LockSupport.park 而不用 Object.wait

>`LockSupport.park` 提供了一种更灵活、简单且有效的方式来控制线程的阻塞和唤醒。
>特别适合于构建复杂的同步工具和用于高级并发场景。
>相比之下，`Object.wait` 和 `notify` 虽然也是有效的同步机制，但在使用上更为复杂和受限。

### 不需要持有锁

- **LockSupport**：`LockSupport.park` 不需要线程持有任何特定的锁。这意味着它可以在任何上下文中被使用，而无需担心死锁或者遵守特定的锁协议。
- **Object.wait**：相比之下，`Object.wait` 必须在同步块（即持有对象锁的范围内）中调用，否则会抛出 `IllegalMonitorStateException` 异常。

```java
    Object lock = new Object();
    
    synchronized (lock) {
        // 必须在同步块内调用 wait，确保持有 lock 的锁
        lock.wait();
    }
```

如果你没有拿到锁，然后你调用了 `Object.wait` 方法，那么就会抛出 `IllegalMonitorStateException` 异常。

`LockSupport` 无需担心这个，因此它减少了死锁的风险，简化了一些编码，我并不需要去担心我是否当前线程能否拿到锁。

所以说 `LockSupport` 很适合 同步工具和 一些并发场景。


### permit 许可机制：允许先 unpark 然后再 park

- **LockSupport**：`LockSupport` 提供了一种“许可”机制，其中 `unpark` 操作可以预先发生，为后续的 `park` 操作“积累”一个许可。这种机制使得线程控制更加精细。
- **Object.wait / notify** ： `Object.wait` 依赖于 `notify` 或 `notifyAll` 方法来唤醒等待的线程。如果 `wait` 在 `notify` 之后发生，线程仍然会阻塞，直到另一个 `notify` 或 `notifyAll` 被调用。

```java
    Thread thread = Thread.currentThread();
    
    // 提前 unpark
    LockSupport.unpark(thread);
    
    // 后续的 park 会立即返回，因为已经有一个可用的许可
    LockSupport.park();
    
    System.out.println("Thread continued execution");
```

### 简化的使用

- **LockSupport**：`LockSupport` 的使用相对简单，不需要与特定的对象或锁关联。
- **Object.wait / notify**：`Object.wait` 和 `notify` 必须在特定对象的上下文中使用，并且需要正确处理锁和异常，这使得它们的使用更加复杂。



# 总结

- `AQS` 提供了一系列的模版方法 `aquire`、`tryaquire`、`aquireQueue`，来帮助我们实现同步机制。
- `AQS` 内维护的同步队列中的大量操作，都是通过 `CAS` 去实现的。
- `AQS` 内的 `CLH` 锁机制提供了一种 **高效的自旋锁** 实现。

拥有 `AQS`，我们就可以通过对 `state` 的语义不同，亦或者去重写某个方法，从而实现更为灵活，更加强大的锁：

- `ReentranLock`：可重入锁。
- `ReentrantReadWriteLock`：读写锁。
- `CountDownLatch`：异步转同步，需要等待的。
- ...

下篇文章就讲 `AQS` 的实现类是如何实现自己的特性！