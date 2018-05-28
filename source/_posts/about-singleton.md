---
title: 谈谈单例模式
date: 2018-05-16 16:35:17
categories: tools #文章文类
toc: true
tags: [Java,Design Patterns] #文章标签，多于一项时用这种格式
---
关于单例，一般来说Java有如下实现模式
### 第一种
``` Java
public class Singleton {

    private static Singleton instance = new Singleton();

    private Singleton(){
    }

    public Singleton getInstance(){
        return instance;
    }
}
```

### 第二种
用volatile以及双重检查机制保证能够懒加载
``` Java
public class Singleton {
    // private static Singleton instance = new Singleton();

    private volatile static Singleton instance = new Singleton();

    private Singleton(){
    }

    public Singleton getInstance(){
        if(instance == null){                       // first check
            synchronized (Singleton.class){         // lock
                if(instance==null){                 // second check
                    instance = new Singleton();     // the problem
                }
            }
        }
        return instance;
    }
}
```
如果没有volatile修饰 instance,看起来也很完美。
但是在执行instance = new Singleton() 时，这一行代码可分解为如下三行伪代码
``` C++
memory = allocate();    //1: 分配对象的内存空间
ctorInstance(memory);   //2: 初始化对象
instance = memory;      //3: 设置instance指向刚分配的内存地址
```
上面的三行伪代码2，3可能会被重排序。
``` C++
memory = allocate();    //1: 分配对象的内存空间
instance = memory;      //3: 设置instance指向刚分配的内存地址
                        //注意，此时对象还没有被初始化！
ctorInstance(memory);   //2: 初始化对象
```
#### intra-thread semantics
intra-thread semantics允许那些在单线程内，不会改变单线程程序执行结果的重排序。
![示意图](/img/1008100.png)
如上图所示，2和3重排也不会违反intra-thread semantics

下面，再让我们看看如果发生重排序后多线程并发执行的可能出现情况
![示意图](/img/1008101.png)

|时间|线程A|线程B|
| - | - |
|t1|A1：分配对象的内存空间||
|t2|A3：设置instance的指向内存空间||
|t3||B1：判断instance是否为空|
|t4||B2：由于instance不为null，线程B将访问instance引用的对象|
|t5|A2：初始化对象| |
|t6|A4：访问instance引用的对象|   |

A2和A3的重排序，将导致现场B在B1处判断出instance不为空，线程B接下来将访问instance引用的对象。此时，线程B将会访问到一个还未初始化的对象。

既然知道了原因，那么有2个方案来实现线程安全的延迟初始化：
1. 不允许2和3重排序；
2. 允许2和3重排序，但不允许其他线程"看到"这个重排序。
我们只需要做一点小的修改（把instance声明为volatile型），就可以实现线程安全的延迟初始化。
注意，这个解决方案需要JDK5或更高版本（因为从JDK5开始使用新的JSR-133内存模型规范，这个规范增强了volatile的语义）。
![禁止重排序](/img/1008102.png)


### 第三种 基于类初始化的解决方案
JVM在类初始化阶段(Class被加载后，且被线程使用之前)，会执行类的初始化。在执行类的初始化期间，JVM会去获取一个锁。这个锁可以同步多个线程对同一个类的初始化。  
使用Java的静态内部类以及多线程缺省同步锁来实现
``` Java
public class Singleton {
    private Singleton(){}

    private static class HolderClass{
        private final static Singleton instance = new Singleton();
    }

    public static Singleton getInstance(){
        return HolderClass.instance;
    }
}
```
假设2个线程并发执行getInstance()
![jvm锁](/img/1008103.png)
这个方案的实质是：允许"问题的根源"的三行伪代码中的2和3重排序，但是不允许非构造线程"看到"这个重排序。

初始化一个类，包括执行这个类的镜头初始化和初始化在这个类中声明的静态字段。根据Java语言规范，在首次发生下列任一种情况时，一个类或接口类型T将立即初始化：
* T是一个类，而且一个T类型的实例被创建；
* T是一个类，而且T中声明的一个静态方法被调用；
* T中声明的一个静态字段被赋值；
* T中声明的一个静态字段被使用，而且这个字段不是一个常量字段；
* T是一个顶级类(top level class)，而且一个断言语句嵌套在T内部被执行。

在第三种方法中，首次执行getInstance()的线程将导致HolderClass类被初始化。
由于Java语言是多线程的，多个线程可能在同一时间尝试去初始化一个类或者接口。因此在Java中初始化一个类或者接口时，需要做细致的同步处理。

Java语言规范规定，对于每一个类或接口C，都有一个唯一的初始化锁LC与之对应。从C到LC的映射，由JVM的具体实现去自由实现。JVM在类初始化期间会获取这个初始化锁，并且每个线程至少获取一次锁来确保这个类以已经被初始化了。
对于类和接口的初始化，Java语言规范制定了精巧而复杂的类初始化处理过程。Java初始化一个类或接口的处理过程如下
1. 通过在Class对象上同步(即获取Class对象的初始化锁)，来控制类或接口的初始化。这个获取锁会一直等待，直到当前线程能够获取这个初始化锁。
假设Class对象当前还没有被初始化(初始化状态state此时被标记为state=noInitialization)。且有两个线程A和B试图同时初始化这个Class对象。
![noInitialization](/img/1008104.png)
下面是这个示意图的说明： 

|时间|线程A|线程B|
| - | - | -|
|t1|A1：尝试获取Class对象的初始化锁。这里假设线程A获取到了初始化锁|B1：尝试获取CLass对象的初始化锁，由于线程A获取到了锁，线程B将一直等待获取初始化锁|
|t2|A2：线程A看到线程还未被初始化(因为读取到state==noInitialization)，线程设置state=initializing||
|t3|A3：线程A释放初始化锁| |

2. 线程A执行类的初始化，同时线程B在初始化锁对应的condition上等待
![condition](/img/1008105.png)

下面是这个示意图的说明：
|时间|线程A|线程B|
| - | - | -|
|t1|A1：执行类的静态字段初始化和初始化类中的声明的静态字段|B1：获取到初始化锁|
|t2| |B2：读取到state== initializing|
|t3| |B3：释放初始化锁|
|t4| |B4：在初始化锁的condition中等待|

3. 线程A设置state=initialized，然后唤醒condition中等待的所有线程：
![initialized](/img/1008106.png)
下面是这个示意图的说明：

|时间|线程A|
| - | - |
|t1|A1：获取初始化锁|
|t2|A2：设置state=initialized|
|t3|A3：唤醒在condition中等待的所有线程|
|t4|A4：释放初始化锁|
|t5|A5：线程A的初始化处理过程完成|

4. 线程B结束类的初始化过程：
![initialized1](/img/1008107.png)

|时间|线程B|
| - | - |
|t1|B1：获取初始化锁|
|t2|B2：读取到state==initialized|
|t3|B3：释放初始化锁|
|t4|B4：线程B的类初始化过程完成|
线程A在第二阶段的A1执行类的初始化，并在第三阶段的A4释放初始化锁；线程B在第四阶段的B1获取同一个初始化锁，并在第四阶段的B4之后才开始访问这个类。根据java内存模型规范的锁规则，这里将存在如下的happens-before关系：
![initialized happens-before](/img/1008108.png)
这个happens-before关系将保证：线程A执行类的初始化时的写入操作（执行类的静态初始化和初始化类中声明的静态字段），线程B一定能看到。

5. 线程C结束类的初始化过程：
![initialized2](/img/1008109.png)

|时间|线程C|
| - | - |
|t1|C1：获取初始化锁|
|t2|C2：读取到state==initialized|
|t3|C3：释放初始化锁|
|t4|C4：线程C的类初始化过程完成|

在第三阶段之后，类已经完成了初始化。因此线程C在第五阶段的类初始化处理过程相对简单一些（前面的线程A和B的类初始化处理过程都经历了两次锁获取-锁释放，而线程C的类初始化处理只需要经历一次锁获取-锁释放）。

线程A在第二阶段的A1执行类的初始化，并在第三阶段的A4释放锁；线程C在第五阶段的C1获取同一个锁，并在在第五阶段的C4之后才开始访问这个类。根据java内存模型规范的锁规则，这里将存在如下的happens-before关系：
![initialized2 happens-before](/img/1008110.png)
这个happens-before关系将保证：线程A执行类的初始化时的写入操作，线程C一定能看到。

第三种也是懒加载，利用了classloader的机制来保证初始化instance时只有一个线程，所以也是线程安全的，同时没有性能损耗。
因为静态内部类是要在有了引用以后才会加载到内存，所以在调用getInstance()之前，HolderClass并没有被加载到内存，只有调用了getInstance()之后，产生了对HolderClass的引用，静态内部类的实例才会真正加载。


#### 静态内部类
1. 静态内部类相当于其外部类的static成分，它的对象与外部类对象间不存在依赖关系，因此 
可以直接创建。而对象级内部类的实例，是绑定在外部对象实例中的。 
2. 静态内部类中，可以定义静态的方法。在静态方法中只能引用外部类中的静态成员方法或变量。 
3. 静态内部类相当于其外部类的成员，只有在第一次被使用的时候才会被加载。

#### 多线程缺省同步锁来
在多线程开发中，为了解决并发问题，主要使用synchronized或者其他方式来加锁进行同步控制，但是在某些情况下，JVM以及隐式的执行了同步，这些情况下就不用自己再来同步控制了。
1. 由静态初始化器()初始化数据时
2. 访问final字段时
3. 在创建线程之前创建对象时
4. 线程可以看见它将要处理的对象时

参考文献
1. http://www.infoq.com/cn/articles/double-checked-locking-with-delay-initialization#anch102163