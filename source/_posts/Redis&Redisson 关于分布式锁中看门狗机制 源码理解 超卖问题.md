---
title: Redis&Redisson 关于分布式锁中看门狗机制 源码理解 超卖问题
categories:
  - 缓存中间件
tags:
  - Redis
  - 缓存中间件
cover: >-
  https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202307091752364.png
abbrlink: 3997
---




超卖问题不管是业务中，还是面试上都是比较热门和头疼的问题，本篇文章记录一下笔者学习redis个人笔记。


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af42f1fb2e5d408ca616f9d1f002a121~tplv-k3u1fbpfcp-watermark.image?)

# 场景重现
我们都知道jvm级别的锁（synchronized）是无法在分布式微服务下解决超卖问题。这种级别的锁只能锁住当前进程，对于其他微服务是无法奏效。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/70a098fecf064c1f9291ee088e07c2f7~tplv-k3u1fbpfcp-watermark.image?)

# 使用redis实现分布式锁
``` java
    //标识当前客户端上的锁
    String clienId = UUID.randomUUID().toString();
    String lockKey = "lockKey";
    //上锁 并设置标识 并且设置超时时间
    Boolean result = stringRedisTemplate.opsForValue().setIfAbsent(lockKey, clienId, 10, TimeUnit.SECONDS);
    if (!result) {
        return "error_code";
    }
    try {
        //业务代码
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        //判断是不是当前客户端上的锁 是否存在
        if (clienId.equals(StringRedisTemplate.opsForValue().get(lockKey))) {
            //存在 则释放锁
            stringRedisTemplate.delete("lockKey");
        }
    }
```

这是一个简单的分布式锁的实现，但是里面存在严重的问题

如果业务逻辑处理的时间 > 我们自己设置的超时时间；redis按超时处理的情况释放了，而被另外一个线程趁虚而入，抢到了这把锁。但是我当前的线程是在finally语句里面要执行释放锁
```
    if (clienId.equals(StringRedisTemplate.opsForValue().get(lockKey))) { 
        //存在 则释放锁 
        stringRedisTemplate.delete("lockKey");
    }
```
这样就会出现问题

我们的需求是
- 我们的业务逻辑执行时间可能会大于超时时间，但是我们不希望按超时处理，想要一个好比网吧加时的一个操作
- 我们希望锁的释放具有原子性

因此引出我们今天的主角 Redisson
# Redisson
[github](https://github.com/redisson/redisson)
 redisson提出了一种看门狗的机制，可以对锁进行续命

 源码基于当前最新版本的 Redisson v3.16.3

 我们直接定位的核心代码： `scheduleExpirationRenewal`方法

 ```java
protected void scheduleExpirationRenewal(long threadId) {
    ExpirationEntry entry = new ExpirationEntry();
    //如果原先没有值 则返回null 有则返回对应的value
    ExpirationEntry oldEntry = EXPIRATION_RENEWAL_MAP.putIfAbsent(getEntryName(), entry);
    if (oldEntry != null) {
        //已经放好了
        oldEntry.addThreadId(threadId);
    } else {
        //第一次放
        entry.addThreadId(threadId);
        try {
            //锁续命
            renewExpiration();
        } finally {
            if (Thread.currentThread().isInterrupted()) {
                //释放锁
                cancelExpirationRenewal(threadId);
            }
        }
    }
}
 ```
 这里出现了`ExpirationEntry`对象，是redisson进行锁续命的一个操作对象，类似Spring源码中的beanDefinition对象
## ExpirationEntry 
``` java
    public static class ExpirationEntry {

        //用来存放需要续命threadId集合
        private final Map<Long, Integer> threadIds = new LinkedHashMap<>();
        //设置的超时时间
        private volatile Timeout timeout;

        //省略
    }
```
从源码上面可以非常清晰地知道 这个数据类型就是用来续命的一个个对象

然后我们看源码主要的续命核心代码 `renewExpiration`

## renewExpiration
``` java
    private void renewExpiration() {
    ExpirationEntry ee = EXPIRATION_RENEWAL_MAP.get(getEntryName());
    if (ee == null) {
        return;
    }
    
    Timeout task = commandExecutor.getConnectionManager().newTimeout(new TimerTask() {
        @Override
        public void run(Timeout timeout) throws Exception {
            ExpirationEntry ent = EXPIRATION_RENEWAL_MAP.get(getEntryName());
            if (ent == null) {
                return;
            }
            Long threadId = ent.getFirstThreadId();
            if (threadId == null) {
                return;
            }
            
            //续命操作
            RFuture<Boolean> future = renewExpirationAsync(threadId);
            future.onComplete((res, e) -> {
                if (e != null) {
                    log.error("Can't update lock " + getRawName() + " expiration", e);
                    EXPIRATION_RENEWAL_MAP.remove(getEntryName());
                    return;
                }
                
                if (res) {
                    // reschedule itself 重新调用自己
                    renewExpiration();
                } else {
                    cancelExpirationRenewal(null);
                }
            });
        }
    }, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);
    
    ee.setTimeout(task);
}
```
他是新创建了一个子线程去反复调用；根据EntryName来去存放Entry的Map里面查，这个Entry在不在，如果不在说明被删除了，不需要再续命了，就不再调用；否则则会间隔执行 
`internalLockLeaseTime` / 3个时间
续命的操作，比较核心的就是执行`renewExpirationAsync`这个方法

## renewExpirationAsync

``` java
protected RFuture<Boolean> renewExpirationAsync(long threadId) {
    return evalWriteAsync(getRawName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
            "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                    "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                    "return 1; " +
                    "end; " +
                    "return 0;",
            Collections.singletonList(getRawName()),
            internalLockLeaseTime, getLockName(threadId));
}
```
redisson是通过lua脚本来实现语句锁续命，也就是给这个值的超时时间再延长一点，并且因为是lua脚本，所以他带有**原子性**
## cancelExpirationRenewal



当我们不需要续命的时候，就会调用`cancelExpirationRenewal`

``` java
protected void cancelExpirationRenewal(Long threadId) {
    ExpirationEntry task = EXPIRATION_RENEWAL_MAP.get(getEntryName());
    if (task == null) {
        return;
    }
    
    if (threadId != null) {
        task.removeThreadId(threadId);
    }

    if (threadId == null || task.hasNoThreads()) {
        Timeout timeout = task.getTimeout();
        if (timeout != null) {
            timeout.cancel();
        }
        //将Entry移除需要续命的map
        EXPIRATION_RENEWAL_MAP.remove(getEntryName());
    }
}
```
看最后一句就知道，就是简单地把他移除即可，之前我们在最开始地`scheduleExpirationRenewal`的finally语句块中也看到这个


 ## 看门狗名字的由来
 我们去找找这个间隔时间 `internalLockLeasetime`的赋值

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/28a8a05445ca4f89b27459ccedd12bf5~tplv-k3u1fbpfcp-watermark.image?)
``` java
private long lockWatchdogTimeout = 30 * 1000;
```
可以看到他默认是30秒的，也就是间隔10秒，就判断是否需要续命

