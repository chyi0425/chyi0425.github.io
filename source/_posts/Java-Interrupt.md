---
title: Java-Interrupt
date: 2018-07-04 21:25:12
tags: [Java,concurrency] #文章标签，多于一项时用这种格式
toc: true
---
### 线程中断
首先，我们要明白，中断不是类似 linux 里面的命令 kill -9 pid，不是说我们中断某个线程，这个线程就停止运行了。中断代表线程状态，每个线程都关联了一个中断状态，是一个 true 或 false 的 boolean 值，初始值为 false。

关于中断状态，我们需要重点关注以下几个方法：
```Java
// Thread 类中的实例方法，持有线程实例引用即可检测线程中断状态
public boolean isInterrupted() {}

// Thread 中的静态方法，检测调用这个方法的线程是否已经中断
// 注意：这个方法返回中断状态的同时，会将此线程的中断状态重置为 false
// 所以，如果我们连续调用两次这个方法的话，第二次的返回值肯定就是 false 了
public static boolean interrupted() {}

// Thread 类中的实例方法，用于设置一个线程的中断状态为 true
public void interrupt() {}
```
我们说中断一个线程，其实就是设置了线程的 interrupted status 为 true，至于说被中断的线程怎么处理这个状态，那是那个线程自己的事。如以下代码：

```Java
while (!Thread.interrupted()) {
   doWork();
   System.out.println("我做完一件事了，准备做下一件，如果没有其他线程中断我的话");
}
```
当然，中断除了是线程状态外，还有其他含义，否则也不需要专门搞一个这个概念出来了。
如果线程处于以下三种情况，那么当线程被中断的时候，能自动感知到：
1. 来自Object类的wait()、wait(long),wait(long,int),来自Thread类的join()、join(long)、join(long,int)、sleep(long)、sleep(long,int)

> 这几个方法的共同之处是，方法上都有：throws InterruptedException
> 如果线程阻塞在这些方法上，这个时候如果其他线程对这个线程进行了中断，那么这个线程会从这些方法中立即返回，抛出InterruptedException，同时重置中断状态为false。

2. 实现了InttruptibleChannel接口的类中的一些I/O操作，如DatagramChannel中的Channel方法和receive方法等

> 如果线程阻塞在这里，中断线程会导致这些方法抛出ClosedByInterruptException并重置中断状态

3. Selector中的select方法

> 一旦中断，方法立即返回，并重置中断状态

对于以上 3 种情况是最特殊的，因为他们能自动感知到中断（这里说自动，当然也是基于底层实现），**并且在做出相应的操作后都会重置中断状态为 false**。

