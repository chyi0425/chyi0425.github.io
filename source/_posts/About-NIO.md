---
title: About-NIO
date: 2019-02-22 16:05:51
tags: [JAVA,MQ]
toc: true
---

## 为什么要使用NIO

可以看到使用过NIO重新实现过的传统IO根本不虚，在大文件下效果还比NIO要好(当然了，个人几次的测试，或许不是很准)

* 而NIO要有一定的学习成本，也没有传统IO那么好理解。

那这意味着我们可以不使用/学习NIO了吗？

答案是否定的，IO操作往往在两个场景下会用到：

* 文件IO
* 网络IO

NIO的**魅力：在网络中使用IO就可以体现出来了**！

## NIO快速入门

首先我们来看看**IO和NIO的区别**：

![示意图](/img/nio-1.png)

* 可简单认为：**IO是面向流的处理，NIO是面向块(缓冲区)的处理**
    * 面向流的I/O系统**一次一个字节地处理数据**。
    * 一个面向块(缓冲区)的I/O系统**以块的形式处理数据**。

NIO主要有三个核心部分组成：
    * **buffer缓冲区**
    * **Channel管道**
    * **Selector选择器**

### buffer缓冲区和Channel管道

在NIO中并不是以流的方式来处理数据的，而是以buffer缓冲区和Channel管道**配合使用**来处理数据。

简单理解一下：

* Channel管道比作成铁路，buffer缓冲区比作成火车(运载着货物)

而我们的NIO就是**通过Channel管道运输着存储数据的Buffer缓冲区的来实现数据的处理**！

* 要时刻记住：Channel不与数据打交道，它只负责运输数据。与数据打交道的是Buffer缓冲区
    * Channel --> 运输
    * Buffer --> 数据

相对于传统IO而言，流是单向的。对于NIO而言，有了Channel管道这个概念，我们的读写都是双向的(铁路上的火车能从广州去北京、自然就能从北京返还到广州)！

#### buffer缓冲区核心要点

我们来看看Buffer缓冲区有什么值得我们注意的地方。

Buffer是缓冲区的抽象类：

```Java
public abstract class Buffer {

    /**
     * The characteristics of Spliterators that traverse and split elements
     * maintained in Buffers.
     */
    static final int SPLITERATOR_CHARACTERISTICS =
        Spliterator.SIZED | Spliterator.SUBSIZED | Spliterator.ORDERED;

    // Invariants: mark <= position <= limit <= capacity
    private int mark = -1;
    private int position = 0;
    private int limit;
    private int capacity;
......
```

其中ByteBuffer是用得最多的实现类(在管道中读写字节数据)。

![示意图](/img/nio-2.png)

拿到一个缓冲区我们往往会做什么？很简单，就是**读取缓冲区的数据/写数据到缓冲区中**。所以，缓冲区的核心方法就是:

* put()
* get()

![示意图](/img/nio-3.png)

![示意图](/img/nio-4.png)

Buffer类维护了4个核心变量属性来提供**关于其所包含的数组的信息**。它们是：

* 容量Capacity
    * **缓冲区能够容纳的数据元素的最大数量**。容量在缓冲区创建时被设定，并且永远不能被改变。(不能被改变的原因也很简单，底层是数组嘛)
* 上界Limit
    * **缓冲区里的数据的总数**，代表了当前缓冲区中一共有多少数据。
* 位置Position
    * **下一个要被读或写的元素的位置**。Position会自动由相应的 get( )和 put( )函数更新。
* 标记Mark
    * 一个备忘位置。用于记录上一次读写的位置。

![示意图](/img/about-nio-5.png)

#### buffer代码演示
```Java
public static void main(String[] args) {
        // create buffer
        ByteBuffer buffer = ByteBuffer.allocate(1024);

        // show the value of the variable
        System.out.println("初始时-->limit-->" + buffer.limit());
        System.out.println("初始时-->position-->" + buffer.position());
        System.out.println("初始时-->capacity-->" + buffer.capacity());
        System.out.println("初始时-->mark-->" + buffer.mark());

        System.out.println("----------------------------------------------");

        // put some data into buffer
        String s = "Java3y";
        buffer.put(s.getBytes());

        // show the value of the variable
        System.out.println("初始时-->limit-->" + buffer.limit());
        System.out.println("初始时-->position-->" + buffer.position());
        System.out.println("初始时-->capacity-->" + buffer.capacity());
        System.out.println("初始时-->mark-->" + buffer.mark());

    }
```

运行结果：

![示意图](/img/about-nio-6.png)

现在我想要**从缓存区拿数据**，怎么拿呀？？NIO给了我们一个flip()方法。这个方法可以**改动position和limit的位置**！

还是上面的代码，我们flip()一下后，再看看4个核心属性的值会发生什么变化：

![示意图](/img/about-nio-7.png)

很明显的是：
* limit变成了position的位置了
* 而position变成了0

看到这里的同学可能就会想到了：当调用完filp()时：limit是限制读到哪里，而position是从哪里读

一般我们称filp()为“切换成读模式”
* 每当要从缓存区的时候读取数据时，就调用filp()“切换成读模式”。

![示意图](/img/about-nio-8.png)

切换成读模式之后，我们就可以读取缓冲区的数据了：

```Java
        // 创建一个limit()大小的字节数组(因为就只有limit这么多个数据可读)
        byte[] bytes = new byte[byteBuffer.limit()];

        // 将读取的数据装进我们的字节数组中
        byteBuffer.get(bytes);

        // 输出数据
        System.out.println(new String(bytes, 0, bytes.length));
```
随后输出一下核心变量的值看看：

![示意图](/img/about-nio-9.png)

**读完我们还想写数据到缓冲区**，那就使用clear()函数，这个函数会“清空”缓冲区：

* 数据没有真正被清空，只是被遗忘掉了

![示意图](/img/about-nio-10.png)

#### FileChannel通道核心要点

![示意图](/img/about-nio-11.png)

Channel通道**只负责传输数据、不直接操作数据的**。操作数据都是通过Buffer缓冲区来进行操作！

```Java

        // 1. 通过本地IO的方式来获取通道
        FileInputStream fileInputStream = new FileInputStream("F:\\3yBlog\\JavaEE常用框架\\Elasticsearch就是这么简单.md");

        // 得到文件的输入通道
        FileChannel inchannel = fileInputStream.getChannel();

        // 2. jdk1.7后通过静态方法.open()获取通道
        FileChannel.open(Paths.get("F:\\3yBlog\\JavaEE常用框架\\Elasticsearch就是这么简单2.md"), StandardOpenOption.WRITE);

```

使用**FileChannel配合缓冲区**实现**文件复制**的功能：

![示意图](/img/about-nio-12.png)

使用**内存映射文件**的方式实现**文件复制**的功能(直接操作缓冲区)：

![示意图](/img/about-nio-13.png)

通道之间通过transfer()实现数据的传输(直接操作缓冲区)：

![示意图](/img/about-nio-14.png)

#### 直接与非直接缓冲区

- 非直接缓冲区是需要经过一个：copy的阶段的(从内核空间copy到用户空间)
- 直接缓冲区不需要经过copy阶段，也可以理解成--->**内存映射文件**，(上面的图片也有过例子)。

![示意图](/img/about-nio-15.png)

![示意图](/img/about-nio-16.png)

使用直接缓冲区有两种方式：

- 缓冲区创建的时候分配的是直接缓冲区
- 在FileChannel上调用map()方法，将文件直接映射到内存中创建

![示意图](/img/about-nio-17.png)

#### scatter和gather、字符集

这个知识点我感觉用得挺少的，不过很多教程都有说这个知识点，我也拿过来说说吧：
- 分散读取(scatter)：将一个通道中的数据分散读取到多个缓冲区中
- 聚集写入(gather)：将多个缓冲区中的数据集中写入到一个通道中

![示意图](/img/about-nio-18.png)

![示意图](/img/about-nio-19.png)

##### 分散读取

![示意图](/img/about-nio-20.png)

##### 聚集写入

![示意图](/img/about-nio-21.png)

##### 字符集(只要编码格式和解码格式一致，就没问题了)

![示意图](/img/about-nio-22.png)

### IO模型理解

根据UNIX网络编程对I/O模型的分类，在UNIX可以归纳成5种I/O模型：

- 阻塞IO
- 非阻塞IO
- I/O多路复用
- 信号驱动I/O
- 异步I/O

#### 学习I/O模型需要的基础

##### 文件描述符

Linux 的内核将所有外部设备都看做一个**文件**来操作，对一个文件的读写操作会**调用内核提供的系统命令(api)**，返回一个**file descriptor**fd，文件描述符）。而对一个socket的读写也会有相应的描述符，称为**socket fd**（socket文件描述符），描述符就是一个数字，指向**内核中的一个结构体**（文件路径，数据区等一些属性）。
- 所以说：在Linux下对文件的操作是利用文件描述符(file descriptor)来实现的。

##### 用户空间和内核空间

为了保证用户进程不能直接操作内核（kernel），保证内核的安全，操心系统将虚拟空间划分为两部分
- 一部分为内核空间。
- 一部分为用户空间。

##### I/O运行过程

我们来看看IO在系统中的运行是怎么样的(我们以read为例)

![示意图](/img/about-nio-23.png)

可以发现的是：当应用程序调用read方法时，是需要**等待**的--->从内核空间中找数据，再将内核空间的数据拷贝到用户空间的。
- 这个等待是必要的过程！

下面只讲解用得最多的3个I/0模型：

- 阻塞I/O
- 非阻塞I/O
- I/O多路复用

#### 阻塞I/O模型

在进程(用户)空间中调用recvfrom，其系统调用直到数据包到达且**被复制到应用进程的缓冲区中或者发生错误时**才返回，在此期间一直**等待**。

![示意图](/img/about-nio-24.png)

#### 非阻塞I/O模型

recvfrom从应用层到内核的时候，如果没有数据就直接返回一个EWOULDBLOCK错误，一般都对非阻塞I/O模型进行轮询检查这个状态，看内核是不是有数据到来。

![示意图](/img/about-nio-25.png)

#### I/O复用模型

前面也已经说了：在Linux下对文件的操作是利用**文件描述符(file descriptor)**来实现的。

在Linux下它是这样子实现I/O复用模型的：

- 调用select/poll/epoll/pselect其中一个函数，传入多个文件描述符，如果有一个文件描述符就绪，则返回，否则阻塞直到超时。
