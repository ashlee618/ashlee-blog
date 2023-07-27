---
title: 哈希碰撞之拉链法 vs 开放地址法
tags: 数据结构
cover: >-
  https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202307271958315.png
abbrlink: 40248
date: 2023-07-27 20:02:00
updated: 2023-07-27 20:02:00
---

# 哈希碰撞

> `哈希碰撞`也称`哈希冲突`。

经过`哈希计算`之后得到**哈希值**，根据**哈希值**去找对应的**哈希槽**。

不同的对象得到的哈希值有可能是一样的，**那么就会出现大家都去抢同一个哈希槽**，都去抢同一个坑位了。这就是`哈希碰撞`。

为了能**让元素在更加均匀、散列**分布在哈希槽中，我们设计了许多解决哈希碰撞的方法。本文将结合 `HashMap` 和 `ThreadLocal` 介绍一下**拉链法**和**开放地址法**。


# 拉链法

![image.png](https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202307272001398.png)


在 `HashMap` 采取的就是 **拉链法** 来解决哈希冲突。

当发生哈希碰撞的时候，依然会选择在这个槽位，这个槽位会维护一个链表或者其他数据结构（红黑树），会把这个元素添加到里面去。


# 开放地址法

![image.png](https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202307272001423.png)

在 `ThreadLocal` 中采取的是 **开放地址法** 的方法来解决哈希冲突。

当遇到哈希冲突时，会再次进行 **探测**，探测的意思其实就是去寻找下一个空位。

关于这个探测有非常多的实现方式，常见的有：

- **线性探测**：如果当前位置被占用，则依次往后查找，直到找到一个空槽位。 
- **二次探测**：如果当前位置被占用，双向地去找。di = i + 1，i - 1， i + 3, i - 3 
- **双重哈希**：使用两个哈希函数，根据第一个哈希函数的结果找到一个位置，如果该位置已被占用，则使用第二个哈希函数计算下一个位置，以此类推，直到找到一个空槽位。

在 `ThreadLocal` 里面采用的时是最简单的 **线性探测**。

```java
private ThreadLocalMap(ThreadLocalMap parentMap) {
    Entry[] parentTable = parentMap.table;
    int len = parentTable.length;
    setThreshold(len);
    table = new Entry[len];

    for (int j = 0; j < len; j++) {
        Entry e = parentTable[j];
        if (e != null) {
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            if (key != null) {
                Object value = key.childValue(e.value);
                Entry c = new Entry(key, value);
                int h = key.threadLocalHashCode & (len - 1);
                // 这里采用的是线性探测，一直找到空的位置
                while (table[h] != null)
                    h = nextIndex(h, len);
                table[h] = c;
                size++;
            }
        }
    }
}

private static int nextIndex(int i, int len) {
    // 步长为1, 一直往右边去找
    return ((i + 1 < len) ? i + 1 : 0);
}
```

# 拉链法和开放地址法比较

**结构决定功能**。我们从他们的结构其实就能非常明显地分析出他们的优缺点。

这里我们从拉链法的角度来讨论。

>拉链法适合用于解决频繁产生哈希冲突的场景。

![image.png](https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202307272001404.png)

图中采用开放地址法， `23` 经过了 3轮探测才能找到他自己的位置，当堆积的元素足够多的时候，这里就是性能的瓶颈了。

然而在拉链法中，并不会产生堆积现象，发生哈希冲突的元素直接会添加到槽位中的集合。

>拉链法适用于数据量不可预测的场景。

当元素数量达到一定大小的时候，这里会涉及到 **负载因子**（load factor），`tab` 就要进行扩容。

在容器元素比较多的时候，如果进行一次扩容，而这个扩容又是按 **2次幂** 进行扩容的（不知道为啥是2次幂的赶紧去补知识），很容易就会产生多余的空间。

然而开放地址法每一个位置只能存一个元素，在相同元素的条件下，开放地址法的 `tab` 会会比拉链法的 `tab` 容量要大，**所以开放地址法适合可以提前确定数据量的场景，拉链法适合数据量不可预测的场景**。

# 为什么ThreadLocal选择了开放地址法而不是拉链法

看到这里了，其实你也应该隐约知道为什么ThreadLocal采取的是开放地址法而不是拉链法了。

存储  `TheadLocal`  的 `ThreadLocalMap` 采取的是：`<ThreadLoacl, Object>` 这样的存储结构。

>ThreadLocal#setInitalValue

```java
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
```

我们可以看到他的 **key** 是 `ThreadLocal`，相比起 `HashMap` 的 `<Object, Object>` 存储结构，`ThreadLocalMap` **发生哈希碰撞的几率是非常小的**。

所以对于 `ThreadLocalMap` 来说，选择开放地址法会更好。

# 总结

- 拉链法适合解决**哈希碰撞频繁**的场景。
- 拉链法适合解决**数据集不可观测**的场景。
- 拉链法适合解决**数据集大**的场景。

开放地址法则反之。


![\_ZW\$55A8PHP)%8HW6RVRV2E.gif](https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202307272001833.gif)

> 来都来了，点个赞再走吧彦祖👍，这对我来说非常重要！