---
title: happens-before
date: 2018-05-24 19:40:40
tags: [Java,concurrency] #文章标签，多于一项时用这种格式
toc: true
---

## happens-before学习

synchronized、大部分锁，众所周知的一个功能就是多个线程互斥/串行的(共享锁允许多个线程同时访问，如读锁)访问临界区，但是他们的第二个功能————保证变量的可见性，常常被遗忘。Every thread is defined to have a working memory (an abstraction of caches and registers) in which to store values. 每个线程都有自己的工作存储(内存)，并在某个特定的时候回写到内存。单线程时，这没用问题，如果是多线程要同时访问同一个变量呢？内存中的一个变量会存在于多个工作存储中，线程1修改了变量a的值什么时候对线程2可见？此外，编译器或运行时为了效率可以在允许的时候对指令进行重排序，重排序后的执行顺序就与代码不一致了，这样线程2读取某个变量的时候线程1可能还没进行写入操作呢，虽然代码顺序是写操作是在前面的。这就是可见性问题的由来。
我们无法枚举所有的场景来规定某个线程修改的变量何时对另一个线程可见。但可以制定一些通用的规则，这就是happens-before。它是一个偏序关系，Java内存模型定义了很多Action，有些Action之间存在happens-before关系(并不是所有的Action两两之间都存在happens-before关系)。
从Java内存模型中取两条happens-before关系来瞅瞅：

* An unlock on a monitor happens-before every subsequent lock on that monitor.
* A write to a volatile field happens-before every subsequent read of that volatile.

我们看一下在《Java并发编程实战》中的翻译

* 监视器锁规则。在监视器锁上的解锁操作必须在同一个监视器上的加锁操作之前。
* volatile变量规则。对volatile变量的写入操作必须在对该变量的读操作之前执行。
实际上，happens-before不是描述实际操作中的先后顺序，它是用来描述可见性的一种规则，来看另外一种翻译

* 如果线程1解锁了monitor a，接着线程2锁定了monitor a，那么线程1解锁a之前的写操作都对线程2可见(线程1和线程2可以是同一个线程)。
* 如果线程1写入了volatile变量v(这里和后续的"变量"都指的是对象的字段、类字段和数组元素)，接着线程2读取了v，那么线程1写入v以及之前的写操作都对线程2可见(线程1和线程2可以是同一个线程)。

其实就是"ActionA happens-before ActionB"，那么a以及之前的写操作在另一个线程t1进行b操作时都对t1可见。
接下来再看两条happens-before规则：
* All actions in a thread happens-before any other thread successfully returns from a join() on that thread.
* Each action in a thread happens-before every subsequent action in that thread.

通俗翻译如下：
* 线程t1写入的所有变量(所有action都与那个join有hb关系，当然也包括线程t1终止前的最后一个action了，最后一个action及之前的所有写入操作，所以是所有变量)，在任意其他线程t2调用t1.join()成功返回之后，都对t2可见。
* 线程中上一个动作及之前的所有写操作在该线程执行下一个动作时对该线程可见(也就是说，同一个线程中前面的写操作对后面的操作可见)。