---
title: Kafka：消费者全解
categories:
  - 消息中间件
tags:
  - 消息中间件
  - Kafka
cover: >-
  https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202307091722145.png
abbrlink: 48794
---

# 前言
目前主流的MQ中间件都是基于**发布/订阅模式**实现，生产者生产消息到某个主题topic，消费者订阅了该topic后，当有消费写入该主题就可以进行消费。本篇主要介绍Kafka消费者，包括消费者群组以及遇到再均衡的情况及处理措施。

# 消费者
消费者通过检查消息的**偏移量**来区分已经读取过的消息。**在给定的分区里，每个消息的偏移量是唯一的**。

消费者把每个分区读取到的消息偏移量保存在Zk或者kafka上，**即便是消费者关闭或者重启**，它的读取状态都不会丢失，因为他知道偏移量之后，就知道该从哪里开始读。

![image-20220505220706711](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9a82da10c5364bf2bf93af27c6750364~tplv-k3u1fbpfcp-zoom-1.image)

# 多个消费者

> 多个消费者可以消费同一个消息流

Kafka支持多个消费者从一个单独的小溪流上读取数据，并且消费者之间互不影响。

与其他队列系统不同的是，**其他队列系统的消息一旦被一个客户端读取，其他客户端旧无法再读取他**。

多个消费者可以组成一个群组，他们共享一个消息流，**并保证整个群组对每个给定的消息只处理一次**。

# 消费者群组

假设我们有一个应用程序需要去Kafka读取消息，首先就是要创建一个消费者对象，订阅主题并开始接收消息。

有一种情况就是：**假设我生产者生产消息的速度远远超过消费者消费的速度**，这时候如果还用单个消费者处理消息，就会产生`消息堆积`现象。**所有有必要对消费者进行横向伸缩，从而提高消息消费速率**。

Kafka消费者从属于消费者群组，**同一个消费者群组的消费者都是订阅同一个主题，每一个消费者接受主题一部分分区的消息**（轮询Round-Robin）
> 注意：不要让消费者的数量超过分区数量，不然的话，就会出现多余的消费者出现闲置的情况，如下图

![image-20220512164218300](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f37470eb639843a6ad06fb55ea8156f0~tplv-k3u1fbpfcp-zoom-1.image)

特别的，如果新增另外的消费者群组2，来订阅主题T1，那么消费者群组2会收到和消费者群组1一样的消息，**两者是不会发生冲突的**。

> 总结来说就是：如果想要加快消费速度，
>
> 0.  **那么就增加同一个消费者群组下的消费者的数量**。最好不要超过分区数量。
> 0.  增加**分区数**

**RabbitMq**是支持多个消费者消费同一个队列消息。

RocketMq、KafKa，**每个队列或者分区只能被同一个消费者消费。**

# 消费者群组和分区再均衡
> 再均衡，均衡的是**消费者对于分区的所属权**



![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ef237c2cd77d45f08d8e744e556112b4~tplv-k3u1fbpfcp-watermark.image?)
消费者负责某一个分区的消息，意为着这个消费者拥有**该分区的所有权**，当我们发生新增或删除消费者、主题分区发生变化等情况时候，分区的所有权就会发生变化，这种变化行为称为**再均衡**。

**再均衡**的出现其实是提高了消费者群组的高可用性和伸缩性，（允许我们随意的新增或者删除消费者）。不过我们也不希望频繁出现再均衡行为。

原因在于再均衡的时候，**消费者群组是无法读取消息，会出现群组一小段时间不可用类似STW。** 并且分区重新分配给另外一个消费者的时候，消息的读取状态（偏移量）也会丢失，有可能还需要去刷新缓存等，带来性能上的消耗。


## 群组协调器

> 群组协调器是一个broker，不同的群组可以有不同的协调器。

消费者通过向被指派为**群组协调器**的broker发送心跳来维持他们和**群组的从属关系**以及它们对**分区的所有权关系**。

## 心跳机制

只要消费者在正常的时间间隔发送心跳，就被认为是活跃的。如果没有的话，就会被认为是死亡的消费者。

消费者会在**轮询消息**或者**提交偏移量时**发送心跳。

当消费者停止发送心跳时间过长的话，会话（session）就会过期，群组协调器就会认为它已经死亡，**会触发再均衡**。

在旧版本Kafka里面，Kafka社区是引入一个**独立的心跳线程**，可以在轮询消息的空档发送心跳。这样是使得发送心跳和消息轮询之间是相互独立的。

但是这样会有一个缺点，就是说消费者处理消息也是需要一定的时间的，**如果是一个长时间作业的话，那么有时候就会产生误判消费者死掉了**。

在新版Kafka里面，是可以**指定消费者在离开群组并触发再均衡之前可以有多长时间不再进行消息轮询**。这样就可以避免出现**活锁**（livelock）。（优点类似ZK里面的session心跳机制）

## 分配分区

> 重分配的一个过程

当消费者加入群组的时候，会发送一个`JoinGroup`请求。第一个消费者默认是**群主**。**群主负责给每一个消费者分配分区**。

**每个消费者只知道自己的分配信息，只有群主知道群组内所有消费者的分配信息。**

# 创建Kafka消费者

> 在读取消息之前，第一步就是要创建消费者对象，也就是KafkaConsumer

跟生产者大同小异，也需要定义3个必须的配置：**bootstrap.servers**、**key.deserializer**、**value.deserializer**。注意的是这里是**反序列化器**。**是将字节数组转换为Java对象**。

特别的，还有一个属性叫`group.id`，他不是必须的，它指定了**KafkaConsumer属于哪一个消费者群组**。

## 订阅主题

> 主题topic，对于RabbitMq来说其实就是队列que

```
 consumer.subscribe(Collections.singletonList("customerCountries"));
```

我们除了这种指定的名字的订阅主题方式，还有可以传入一个**正则表达式**，当我们**创建了新的主题**，只要主题的名字和这个匹配得上，那么就会触发一次**再均衡**。**这是非常常见得做法**。

## 轮询

> 消费者通过轮询Round-robin的方式去消费消息，可以获得主题、分区、偏移量等数据。
>
> 并且还有许多机制处理，也是通过轮询方式，比如：群组协调、分区再均衡、发送心跳、**提交偏移量**等。
>
> **它不只是做数据获取的工作，还负责处理事件。**




需要注意的是：我们没办法用一个线程去运行多个消费者，因为这样会出现线程安全问题。

 一般我们都是一个消费者，一个线程。所以我们需要自己用多线程的方式去轮询。比如用**ExcutorService**等。
 
 
# 消费者的配置

> 除了前面提到的几个必须的，还有一些可选的配置，这些配置虽然是可选，但是也是起到非常重要的作用。

## fetch.min.bytes

指定了**消费者从服务器获取记录的最小字节数**。broker在响应消费者的获取消息请求的时候，会先去看可用的数据量，如果小于这个值，那么就不会给消费者，而是要存起来，等满足条件之后再返回给消费者。

这样做可以降低消费者和broker的工作量。

## fetch.max.wait.ms

前面那个参数是限制单次最小字节数，这个参数是**限制最长间隔时间**。因为有可能在不活跃的时间内，要过很久才能存到最小字节数。所以提供了一个broker最长等待时间。当满足哪个条件都可以返回给消费者消息。

## auto.offset.reset

这个属性指定了**消费者在读取一个没有偏移量或者偏移量失效的情况下**（比如消费者长时间失效，包含偏移量的记录已经过时并被删除等情况）该如何处理。

-   `latest`：这种模式下，消费者会从**最新的记录**开始读取数据。
-   `earliest`：该模式下，消费者会从**最起始位置**开始读取数据。

默认情况下是用**latest**。

## enable.auto.commit

这个属性指定了**消费者是否自动提交偏移量**。默认情况是true（相当于rabbitMq里面的ack确认）。

**消费者提交了偏移量，就代表了消费者认为成功地消费了这条消息**。为了避免出现重复数据和数据丢失，可以设置为falsle，由我们手动控制什么时候提交偏移量。

# 提交和偏移量

> 我们把更新分区当前位置的操作叫做**提交**。

调用`poll()`方法，**得到的消息都是写入了Kafka，但是还没有被消费者读取过的记录**。因此我们可以追钟到那些记录是被群组里的哪个消费者读取到的。

消费者通过往一个叫做`_consumer_offset`的特殊**主题**发送消息。**消息里面包含每个分区的偏移量**。代表着我这个分区处理情况如何。

当我们发生消费者崩溃或者有新的消费者进入消费者群组的时候，就会触发**再均衡**。此时消费者就不再是处理原先的分区。有可能会被分配到其他分区。**此时消费者只需要去读取每个分区最后一次提交的偏移量**，就可以在原先的地方进行处理。

现在会出现一个问题：

**提交的偏移量和客户端处理的最后一个消息的偏移量之间前后关系**

如果提交的偏移量**小于**客户端处理的最后一个消息的偏移量。那么当发生**再均衡**的情况时，偏移量的指针就会往**前滑**，导致两者之间的**消息会被重复消费**。

![image-20220512205224559](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d4ef319fd01542a29c57186d7e12e967~tplv-k3u1fbpfcp-zoom-1.image)

如果提交的偏移量**大于**客户端处理的最后一个消息的偏移量。那么当发生**再均衡**的情况时，偏移量的指针就会往**后滑**，导致两者之间的**消息会被丢失**。

![image-20220512205403234](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f0f06a326bb7494aacc3ff6b7fb01626~tplv-k3u1fbpfcp-zoom-1.image)

**所以Kafka提供了多种方式来提交偏移量，来确保上面的情况不发生**。

## __consume_offset：位移主题

> 所有消费者消费消息的时候，都会往这个分区写入偏移量信息，他会保存所有分区的偏移量信息

老版本的**消费者偏移量**是依靠`Zookeeper`来管理的，消费者把当前分区的偏移量写入到**Zk**里，当发生再均衡的时候，只需要去读取对应分区的偏移位置即可。

但是**ZK并不适用于这种高频写操作**，后来社区推出了Kafka自己维护分区偏移的信息。

**消费者消费的时候**，把所有的数据写入`__consumer_offset`这个主题。Kafka可以保证他的吞吐量。所以解决了**ZK**的吞吐量问题。

**缺点是**：

因为是写在Kafka里面的，所以当Kafka变得不可用的时候，`__consumer_offset`就会变得不可用。

有一种解决办法就是借助**外部数据源**。比如Mysql、甚至Redis、等其他地方，只能能存东西就可以。

其实也引入了另外一个问题，**数据放在不同的地方，总是会有数据一致性的问题。要取舍**。

## 自动提交（批次提交偏移量）

> 这个是最简单的提交方式，**交给消费者自己主动提交偏移量**。

如果`enable.auto.commit`被设置为`Ture`，那么消费者会自动把`poll()`方法接收到的最大偏移量提交上去，间隔是由`auto.commit.interval.ms`控制，默认是**5s**。这个动作也是再**轮询**里进行的。

消费者在进行轮询的时候，会先检查是否该要提交偏移量了，如果要，则会提交从上一次轮询返回的偏移量。

那其实，他的问题也是非常明显的。因为他有一个5s的间隔。那么在这个间隔当中，发生**再均衡**的话，就可能会发生**重复消费**的问题。**因为他的指针肯定是在前面的，他没有来得及去更新最后的偏移量。**

就是说，**它太依赖于时间了**，跟redis里面的`redlock`一个道理。

## 手动同步提交（提交当前偏移量）

> 大部分开发者通过控制**偏移量提交时间**来消除丢失消息的可能性。

这个方式需要把`enable.auto.commit`被设置为`Flase`。让应用程序选择何时提交偏移量。

使用`commitSync()`提交偏移量是**最简单也是是最可靠的**。

commitSync是提交poll方法返回的最新偏移量，所以我们必须在调用poll方法之后，确保调用commitSync，不然还是会出现再均衡后消息重复消费的情况。

只要没有**发生不可恢复的错误**，commitSync方法都会一只尝试直到提交成功。 **（最大努力交付）**

```
 while (true) {
     ConsumerRecords<String, String> records = consumer.poll(100);
     for (ConsumerRecord<String, String> record : records)
     {
         System.out.printf("topic = %s, partition = %s, offset =
                           %d, customer = %s, country = %s\n",
                           record.topic(), record.partition(),
                           record.offset(), record.key(), record.value()); 
     }
     try { 
         consumer.commitSync(); 
     } catch (CommitFailedException e) {
         log.error("commit failed", e) 
     }
 }
```

## 手动异步提交

> 手动提交有一个缺点就是，应用程序必须等待服务器返回提交偏移量成功的这么一个响应，否则commitSync就会一直尝试直到成功。

所以说，手动提交的**吞吐量**会收到限制。

异步提交就是不需要等待服务器的成功响应，**我们就摁提交，不管他是否成功**。

```
 while (true) {
     ConsumerRecords<String, String> records = consumer.poll(100);
     for (ConsumerRecord<String, String> record : records)
     {
         System.out.printf("topic = %s, partition = %s,
                           offset = %d, customer = %s, country = %s\n",
                           record.topic(), record.partition(), record.offset(),
                           record.key(), record.value());
     } 
     consumer.commitAsync();  }
```

我们可以看到一个细节的地方是，异步提交commitAsync是写在for循环外面的，**意味着我们不需要每个poll都需要提交偏移量，我们只需要提交最后一个偏移量即可**。

但就是说，他容易丢消息把。他的一个可靠性无法得到保证。

> 为什么commitAsync不支持重试？

原因在于，设置我当前提交一个偏移量是2000，**但是这个提交请求可能出现一些原因导致无法成功传到服务器**。

此时我们处理了一些数据，最新的偏移量是3000，如果我们尝试重试提交2000的这个偏移量，**那么有可能这个2000的偏移量在3000之后成功被服务器接收到**，那么就会出现消息重复消费的情况。

但是commitSync他会一直等待上一个提交是否成功，所以它后面的偏移量不会影响，不会出现这个问题。

commitAsync支持**回调**，在broker做出响应时执行回调。回调经常用于**记录提交错误**或**生成度量指标**。如果用来尝试重试提交的话，要注意提交的顺序。不然就会出现上面的问题。

```
 while (true) {
     ConsumerRecords<String, String> records = consumer.poll(100);
     for (ConsumerRecord<String, String> record : records) {
         System.out.printf("topic = %s, partition = %s,
                           offset = %d, customer = %s, country = %s\n",
                           record.topic(), record.partition(), record.offset(),
                           record.key(), record.value());
     }
     consumer.commitAsync(new OffsetCommitCallback() {
         public void onComplete(Map<TopicPartition,
                                OffsetAndMetadata> offsets, Exception e) {
             if (e != null)
                 log.error("Commit failed for offsets {}", offsets, e);
         } 
     }); }
```

> 如果真要实现**重试异步提交**，就要我们手动维护一个单调递增的序列号来维护已提交的偏移量顺序。

## 同步和异步组合提交

> 有时候我们想要**相对来说比较安全，但吞吐量也要能上去**，可以使用同步和异步组合提交。但是这个是在**代码层面**进行实现的。

我们知道，**即便是某一次提交失败了，其实对于整体来说是不影响的，因为下一次提交或者后续提交有机会成功**，成功的话就将功补过了。

但是对于某些情况，要确保当前这次提交是成功，比如**消费者关闭之前**或者是**再均衡前一次提交**。这两种情况如果没提交成功的话，会导致**消费重复消费**。

```
 try {
      while (true) {
          ConsumerRecords<String, String> records = consumer.poll(100);
          for (ConsumerRecord<String, String> record : records) {
              System.out.println("topic = %s, partition = %s, offset = %d,
              customer = %s, country = %s\n",
              record.topic(), record.partition(),
              record.offset(), record.key(), record.value()
          );
      }
      consumer.commitAsync(); 
      }
 } catch (Exception e) {
      log.error("Unexpected error", e);
 } finally {
      try {
          consumer.commitSync(); 
      } finally {
          consumer.close();
      }
 }
```

## 提交特定的偏移量

> 同步和异步的提交方式，**都是只能处理poll同一个批次的最后一个偏移量**，如果我们想要在处理过程种提交偏移量呢？

commitSync和commitAsync提交的偏移量，都是批次里面最后一个偏移量，**所以说只有当我们处理完批次里面的所有消息之后，才能进行提交偏移量**。

但是现在我们想要在没处理完之前也提交，该怎么办呢？

Kafka允许在commitSync或comitAsync里面传入**分区和偏移量的map**。

**这个也是在代码层实现的，非常复杂**。

**后、消费者读取消息前后**。

如何使用，只需要在订阅主题的时候，我们会调用`subscribe`方法，此时我们传入我们的监听器即可。

监听器需要实现`ConsumerRebalanceListener`。需要重写2个方法：

-   **onPartitionsRevoked（Revoked的意思是撤销）** ：该方法会在**再均衡之前**和**消费者停止读取消息之后**调用。一般我们会在这里提交偏移量。
-   **onPartitionsAssigned（Assigned的意思是分配）** ：该方法会在**再均衡之后**和**消费者开始读取消息之前**调用。

原生消费者设置里面，当再均衡发生后，只能采取`earliest` 或者是 `latest`。

我们可以用再均衡监听器，**当发生再均衡之前，把该分区的偏移量，写入数据库**，然后再均衡完了之后，**分区对应的消费者去数据库查询**。

这个分区的上一个偏移量是哪里，然后从哪里开始读即可。 用Kafka提供的`seek`方法来从特定的偏移量开始读。

# 从特定偏移量处开始处理记录

> poll方法每次拉回来的数据，都是最新的，**有时候我们想要指定特定的偏移量处开始读取消息**。

-   Kafka提供了2个API`seekToBeginning`和`seekToEnd`，可以让我们**跳到分区开始**和**分区末尾**处理记录。

但是这样的**横跨**都太高了，我们希望的是我们去主动告诉**Kafka到哪个特定的偏移量开始处理数据**。

Kafka提供了`seek`方法来解决这个问题。

> 注意：这里的偏移量对应的是分区，每个分区有他自己的偏移量，所以seek方法要传入片区和偏移量。

有时候我们想要严格地保证消息不被丢失。**首先要解决的问题就是触发再均衡后，偏移量的位置问题**。我们可以把偏移量和消息绑定好，存放到数据库中，Mysql或者Hadoop这样。当触发再均衡之后，**我们就去数据库，将每个分区最新的偏移量读取回来，就知道了**。

具体操作是要搭配再**均衡监听器来实现**的。

0.  第一次读取时需要初始化，先把每个分区的偏移量从DB中读取过来。
0.  当再均衡之前，将所有分区的偏移量写入DB。
0.  当再均衡之后，从DB读取所有分区的偏移量。

# 消费者退出轮询（消费者优雅退出）

> 轮询我们都是写在死循环中，所以消费者退出轮询需要借助到外部的力量。

如果确定要退出循环，需要借助外部另外一个线程调用`consumer.wakeup`方法（**这是唯一一个可以从其他线程安全调用的方法**）。

调用之后，会抛异常WakupException，**所以说本质上是通过抛异常来跳出循环**。

光是调用wakeup方法还不行，一般还需要调用`consumer.close`，去完成对群驱协调器发送消息，告诉群组我要退出了，然后就会触发再均衡等操作，而不是需要等待会话超时。

# 独立消费者：指定分区流向消费者

> 放弃消费者群组，单独使用一个消费者去读取主题的所有分区或者是指定部分分区。

所有的请求消息都包含一个**标准消息头**：

-   `Reqiest type`：（也就是API key）
-   `Requiest version`：（broker**可以处理不同版本的客户端请求**，并根据客户0端版本做出不同的响应）
-   `Correlation ID`：具有**唯一性**的数据，**用于标识请求消息**，同时也会出现在响应消息和错误日志里面（用于诊断问题）
-   `Client ID`：客户端Id

![image-20220822220346895](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bbee87126c0744b6a383ec9138030685~tplv-k3u1fbpfcp-zoom-1.image)

我们需要明白的是，**再均衡**处理的是分区的所有权，现在的场景是，我们只有一个消费者，所以不需要，我们更简单。

前提：一个消费者

消费者可以订阅主题，这样就可以获得该主题下所有分区的消息，**也可以为自己分配分区，从而让分配的分区消息定向地流向自己**。

> Kafka权威指南，书本上的代码

```
          List<PartitionInfo> partitionInfos = null;
         partitionInfos = consumer.partitionsFor("topic"); 
         if (partitionInfos != null) {
             for (PartitionInfo partition : partitionInfos)
                 partitions.add(new TopicPartition(partition.topic(),
                         partition.partition()));
             consumer.assign(partitions); 
             while (true) {
                 ConsumerRecords<String, String> records =
                         consumer.poll(1000);
                 for (ConsumerRecord<String, String> record: records) {
                     System.out.println("topic = %s, partition = %s, offset = %d,
                             customer = %s, country = %s\n",
                     record.topic(), record.partition(), record.offset(),
                             record.key(), record.value());
                 }
                 consumer.commitSync();
             }
         }
```

> 网上的代码

分区器的代码，可以**根据消息的内容**，然后将消息**定向投递到指定分区**。

```
 public class MyPartioner implements Partitioner {
  
     @Override
     public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
         String message = value.toString();
         int partition ;
         if(message.contains("congge")){
             partition = 1;
         }else {
             partition = 0;
         }
         return partition;
     }
  
     @Override
     public void close() {
  
     }
  
     @Override
     public void configure(Map<String, ?> map) {
  
     }
 }
```

生产端代码：

```
 /**
  * 发送到指定分区
  * 1、没有传 分区和key的情况下，按照粘性分区进行处理
  * 2、指定了 分区，发送到指定的分区
  * 3、没有指定分区，但是制定了 key 的值，按照 key的hash取值，发送到相应的分区
  */
 public class ProducerMyPartion {
  
     public static void main(String[] args) throws Exception {
  
         // 1. 创建 kafka 生产者的配置对象
         Properties properties = new Properties();
         // 2. 给 kafka 配置对象添加配置信息：bootstrap.servers
         properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "IP:9092");
         properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
         properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
  
         //关联自定义分区器
         properties.put(ProducerConfig.PARTITIONER_CLASS_CONFIG,"com.congge.kafka01.selfs.MyPartioner");
  
         // 3. 创建 kafka 生产者对象
         KafkaProducer<String, String> kafkaProducer = new KafkaProducer<String, String>(properties);
         System.out.println("开始发送数据");
         // 4. 调用 send 方法,发送消息
         for (int i = 0; i < 15; i++) {
             kafkaProducer.send(new ProducerRecord<>("zcy234","congge" + i), new Callback() {
                 @Override
                 public void onCompletion(RecordMetadata metadata, Exception e) {
                     if(e == null){
                         System.out.println("发送成功");
                         System.out.println("主题:" + metadata.topic());
                         System.out.println("分区:" + metadata.partition());
                     }
                 }
             });
         }
         // 5. 关闭资源
         kafkaProducer.close();
     }
  
 }
```

消费端的代码：

```
 /**
  * 消费特定分区的数据
  */
 public class SpecialPartionConsumer {
  
     public static void main(String[] args) {
  
         // 1.创建消费者的配置对象
         Properties properties = new Properties();
         // 2.给消费者配置对象添加参数
         properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "IP:9092");
  
         // 配置序列化 必须
         properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
         properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
  
         // 配置消费者组（组名任意起名） 必须
         properties.put(ConsumerConfig.GROUP_ID_CONFIG, "test-group");
  
         // 创建消费者对象
         KafkaConsumer<String, String> kafkaConsumer = new KafkaConsumer<>(properties);
  
         // 消费某个主题的某个分区数据
         ArrayList<TopicPartition> topicPartitions = new ArrayList<>();
         topicPartitions.add(new TopicPartition("zcy234", 1));
         kafkaConsumer.assign(topicPartitions);
  
         System.out.println("准备接收数据......");
         // 拉取数据打印
         while (true) {
             // 设置 1s 中消费一批数据
             ConsumerRecords<String, String> consumerRecords = kafkaConsumer.poll(Duration.ofSeconds(1));
             // 打印消费到的数据
             for (ConsumerRecord<String, String> consumerRecord : consumerRecords) {
                 System.out.println(consumerRecord);
             }
         }
  
     }
  
 }
```

> 注意：这两个动作不能同时

缺点就是：如果新增了分区，消费者是收不到消息，那么这部分新增分区的消息无法读取到，所以我们要么周期性地去分配分区：

-   调用`consumer.partitionsFor`方法去检查是否有新分区加入。
-   或者在新加分区之后，重启应用程序。


