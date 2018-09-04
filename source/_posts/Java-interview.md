---
title: Java_interview
date: 2018-09-04 18:59:44
tags: [Java,Interview]
toc: true
---


# 复习计划
## Java基础
### 常用类源码回顾
#### String
`String()`,`toString()`,`subString()`,`split()`,`indexOf()`,`intern()`
#### StringBuffer,StringBuilder
#### Integer,Bigdecimal
#### List
`ArrayList`,

`LinkedList`,

#### Map
`HashMap`
> - hash()
> - 寻址
> - get(),put()
> - resize()

`TreeMap`
红黑树查询，新增，删除

#### Thread,ThreadLocal
`wait()`,`nofify()`,`join()`,`sleep()`,`run()`,`start()`

#### NIO
`Selector`,

`Buffer`,

`Channel`

#### J.U.C
`ConcurrentHashMap`
> - get(),put()
> - transfer()
> - count()

`atomicXXX`

unsafe - CAS - cmpxchg - #lock - cpu consistency - MESI - false sharing

`AQS`
> - `ReentrantLock`
> - `ReentrantReadWriteLock`
> - `CountDownLatch`
> - `Semaphore`

`Thread pool`
> - `ThreadPoolExecutor`
> - `Executors`
> - `ExecutorService`
> - `FutureTask`

`BlockingQueue`
> - `LinkedBlockingQueue`
> - `ArrayBlockingQueue`
> - `PriorityBlockingQueue`
> - `SynchronousQueue`

`COW`

`Throwable`

## JVM
### 内存管理
> - GC收集器
> - GC算法
> - 常用参数
> - 常用命令

### 类加载
> - class文件
> - 类加载过程
> - ClassLoader
> - JIT

### JSR-133

## 组件
### Spring
源码
#### IOC
> - bean加载过程
> - IOC容器初始化过程
#### AOP
> - Advisor
> - Advice
> - pointcut
> - JDKProxy
> - cglib
> - javaassist
#### 事务
事务级别 有效范围

### Mybatis
源码
### DataSource

### zookeeper
> - ZAB
> - node
> - leader election
#### curator

### dubbo
源码
### rocketmq vs kafka
源码
> - 物理部署
> - replica
> - 消息持久化
> - 消息路由
> - failover
> - 选举

### redis
> - 基础编码
> - 数据结构
> - RDB/AOF
> - 主从同步
> - 哨兵
> - 集群
> - raft
> - 分布式锁
> - red lock

### memcache
> - 内存分配
> - 部署结构
> - 一致性hash

## MySQL
### sql
### 索引
> - b+ tree
> - page,row数据结构
> - 主键索引
> - 辅助索引
> - 索引覆盖
### 事务
> - RR
> - RC
### 锁
> - next-key lock
> - gap lock
### 持久化
redo log,undo log
### 分布式
> - 主从
> - 分库分表
> - DB中间件

## Linux
### 文件
> - cpu
> - 内存
> - 磁盘

### IO
> - select/poll/epoll
> - IO队列
> - IO调度

### 网络
`http`, `https`,`tcp`,`udp`

## 项目


