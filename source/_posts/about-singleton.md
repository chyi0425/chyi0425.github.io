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

    private volatile static Singleton instance = new Singleton();

    private Singleton(){
    }

    public Singleton getInstance(){
        if(instance == null){
            synchronized (LoadBalancer.class){
                if(instance==null){
                    instance = new LoadBalancer();
                }
            }
        }
        return instance;
    }
}
```


### 第三种  
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

