---
title: AQS-condition
date: 2018-06-30 12:27:15
tags: [Java,concurrency] #文章标签，多于一项时用这种格式
---

## Condition
每个ReentrantLock实例可以通过调用多次newCondition产生多个ConditionObject的实例：
``` Java
final ConditionObject newCondition() {
    return new ConditionObject();
}

public class ConditionObject implements Condition, java.io.Serializable {
        private static final long serialVersionUID = 1173984872572414699L;
        // 条件队列的第一个节点
          // 不要管这里的关键字 transient，是不参与序列化的意思
        private transient Node firstWaiter;
        // 条件队列的最后一个节点
        private transient Node lastWaiter;
        ......
```
在上一篇介绍AQS时，我们有一个阻塞队列，用于保存等待获取锁的线程的队列。这里我们引入另一个概念，叫做条件队列(Condition queue)。
![示意图](/img/aqs2-2.png)
回顾下Node的属性：
``` Java
// prev 和 next 用于实现阻塞队列的双向链表，nextWaiter 用于实现条件队列的单向链表

volatile int waitStatus; // 可取值 0、CANCELLED(1)、SIGNAL(-1)、CONDITION(-2)、PROPAGATE(-3)
volatile Node prev;
volatile Node next;
volatile Thread thread;
Node nextWaiter;
```
