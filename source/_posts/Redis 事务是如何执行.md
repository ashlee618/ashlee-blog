---
title: Redis 事务是如何执行
categories:
  - 缓存中间件
tags:
  - Redis
  - 缓存中间件
cover: >-
  https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/202307091752364.png
abbrlink: 47020
---



# 前言 
学习过Mysql的hxd一定不会对事务感到陌生，一提起事务就会想到ACID，和事务隔离性从而扯出脏读、幻读、不可重复读以及mvcc等一系列问题，可谓是面试重灾区。
本片介绍Reids中事务是如何执行的。

# Redis事务
redis的事务执行过程：
![image-20211009165826013](https://hmf-typora-images.oss-cn-guangzhou.aliyuncs.com/images/image-20211009165826013.png)

1. 开始事务
2. 指令入队
3. 执行事务

开启事务之后，事务中涉及的指令会存入一个队列中，不会立刻执行，等redis接收到执行事务的命令（EXEC）之后看，才会队列中的全部命令

## 事务相关指令
-   **MULTI ：** 开启事务，redis会将后续的命令逐个放入队列中，然后使用**EXEC命令来原子化**执行这个命令系列。
-   **EXEC：** 执行事务中的所有操作命令。
-   **DISCARD：** 取消事务，放弃执行事务块中的所有命令。
-   **WATCH：** 监视一个或多个key,如果事务在执行前，这个key(或多个key)被其他命令修改，则事务被中断，不会执行事务中的任何命令。
-   **UNWATCH：** 取消WATCH对所有key的监视。


## Redis没有回滚操作

Redis没有像Mysql中的通过`undo log`进行回滚的操作，因此在一个事务中如果某一个指令出现**操作失败**了该如何进行下一步操作呢？

### 语法性操作失败
Redis提供语法拼写的检查机制，在事务中，A指令执行成功，B指令的语法出现错误，比如对一个字符串的Key进行了`incr`自增操作，那么立刻终止事务。

- 此时A的执行结果依然保存（提交）
- B指令的执行结果不保存

## Reids事务中Watch机制
**WATCH：** 监视一个或多个key,如果事务在执行前，这个key(或多个key)被其他命令修改，则事务被中断，不会执行事务中的任何命令。

翻译成人话：你在开启事务的时候对某个值进行Watch（监听），如果在执行过程中，这个指发生了变化，那么事务就会强制性地中断

