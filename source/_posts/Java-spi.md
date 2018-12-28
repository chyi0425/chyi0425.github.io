---
title: Java spi
date: 2018-11-29 20:57:32
tags: [Java,Dubbo,JDK]
toc: true
---

## JAVA-SPI 实现原理

### 作用

* 为接口自动寻找实现类

### 实现方式

* 标准制定者制定接口
* 不同厂商编写针对该接口的实现类，并在jar的"classpath:META-INF/services/全接口名称" 文件中指定相应的实现类全类名
* 开发者直接引入相应的jar，就可以实现为接口自动寻找实现类的功能

### 使用方法

META-INF/services/com.chiyi.spi.DemoService
```python
#English implementation
com.chiyi.spi.impl.EnglishDemoServiceImpl

#Chinese implementation
com.chiyi.spi.impl.ChineseDemoServiceImpl
```

两个实现类
```Java
public class ChineseDemoServiceImpl implements DemoService {
    @Override
    public String sayHi(String msg) {
        return "你好, " + msg;
    }
}

public class EnglishDemoServiceImpl implements DemoService {
    @Override
    public String sayHi(String msg) {
        return "Hello, " + msg;
    }
}
```

使用

```Java
    public static void main(String[] args) {
        ServiceLoader<DemoService> serviceLoader = ServiceLoader.load(DemoService.class);
        Iterator<DemoService> it = serviceLoader.iterator();
        while (it.hasNext()) {
            DemoService demoService = it.next();
            System.out.println("class:" + demoService.getClass().getName() + "***" + demoService.sayHi("World"));
        }
    }
```

注意：

* <font color=red>ServiceLoader不是实例化以后，就去读取配置文件中的具体实现，并进行实例化。而是等到使用迭代器去遍历的时候，才会加载对应的配置文件去解析，调用hasNext方法的时候会去加载配置文件进行解析，调用next方法的时候进行实例化并缓存。</font>

### 源码解析

获取ServiceLoad
```Java
ServiceLoader<DemoService> serviceLoader = ServiceLoader.load(DemoService.class);
```

首先看一下ServiceLoad的6个属性

```Java
    private static final String PREFIX = "META-INF/services/";  // 定义实现类的接口文件所在目录

    // The class or interface representing the service being loaded
    private final Class<S> service;     // 接口

    // The class loader used to locate, load, and instantiate providers
    private final ClassLoader loader;   // 定位、加载、实例化实现类

    // The access control context taken when the ServiceLoader is created
    private final AccessControlContext acc; // 权限控制上下文

    // Cached providers, in instantiation order
    private LinkedHashMap<String,S> providers = new LinkedHashMap<>();  // 以初始化的顺序缓存<接口全名称，实例类名称>

    // The current lazy-lookup iterator
    private LazyIterator lookupIterator;    // 真正进行迭代的迭代器
```

其中 LazyIterator是ServiceLoader的一个内部类

```Java
    public static <S> ServiceLoader<S> load(Class<S> service) {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return ServiceLoader.load(service, cl);
    }

    public static <S> ServiceLoader<S> load(Class<S> service,
                                            ClassLoader loader)
    {
        return new ServiceLoader<>(service, loader);
    }

    private ServiceLoader(Class<S> svc, ClassLoader cl) {
        service = Objects.requireNonNull(svc, "Service interface cannot be null");
        loader = (cl == null) ? ClassLoader.getSystemClassLoader() : cl;
        acc = (System.getSecurityManager() != null) ? AccessController.getContext() : null;
        reload();
    }

    public void reload() {
        providers.clear(); // 清空缓存
        lookupIterator = new LazyIterator(service, loader);
    }
```

这样一个ServiceLoader实例就创建成功了。在创建的过程中，我们看到还实例化了一个LazyIterator，该类下边会说。

2、获取迭代器并迭代

```Java
    Iterator<DemoService> it = serviceLoader.iterator();
    while (it.hasNext()) {
        DemoService demoService = it.next();
        System.out.println("class:" + demoService.getClass().getName() + "***" +demoService.sayHi("World"));
    }
```

```Java
    public Iterator<S> iterator() {
        return new Iterator<S>() {

            Iterator<Map.Entry<String,S>> knownProviders
                = providers.entrySet().iterator();


            public boolean hasNext() {
                if (knownProviders.hasNext())
                    return true;
                return lookupIterator.hasNext();
            }

            public S next() {
                if (knownProviders.hasNext())
                    return knownProviders.next().getValue();
                return lookupIterator.next();
            }

            public void remove() {
                throw new UnsupportedOperationException();
            }

        };
    }
```

从查找过程的hashNext()和迭代过程next()来看。
* hashNext();<font color=red>先从provider（缓存）中查找，如果有，直接返回true；如果没有，通过LazyIterator来进行查找。</font>
* next();<font color=red>先从provider（缓存）中直接获取，如果有，直接返回实现类对象实例；如果没有，通过LazyIterator来进行获取。</font>

我们来看一下LazyIterator这个类，首先看一下他的属性

其中，service和loader在上述实例化ServiceLoader的时候就已经实例化好了。

```Java
private class LazyIterator
        implements Iterator<S>
    {

        Class<S> service;   // 接口
        ClassLoader loader; // 类加载器
        Enumeration<URL> configs = null;    // 存放配置文件
        Iterator<String> pending = null;    // 存放配置文件的类容，并存储为ArrayList，即存储多个实现类名称
        String nextName = null; // 当前处理的实现类名称

        private LazyIterator(Class<S> service, ClassLoader loader) {
            this.service = service;
            this.loader = loader;
        }

        private boolean hasNextService() {
            if (nextName != null) {
                return true;
            }
            if (configs == null) {
                try {
                    String fullName = PREFIX + service.getName();
                    if (loader == null)
                        configs = ClassLoader.getSystemResources(fullName);
                    else
                        configs = loader.getResources(fullName);
                } catch (IOException x) {
                    fail(service, "Error locating configuration files", x);
                }
            }
            while ((pending == null) || !pending.hasNext()) {
                if (!configs.hasMoreElements()) {
                    return false;
                }
                pending = parse(service, configs.nextElement());
            }
            nextName = pending.next();
            return true;
        }

        private S nextService() {
            if (!hasNextService())
                throw new NoSuchElementException();
            String cn = nextName;
            nextName = null;
            Class<?> c = null;
            try {
                c = Class.forName(cn, false, loader);
            } catch (ClassNotFoundException x) {
                fail(service,
                     "Provider " + cn + " not found");
            }
            if (!service.isAssignableFrom(c)) {
                fail(service,
                     "Provider " + cn  + " not a subtype");
            }
            try {
                S p = service.cast(c.newInstance());
                providers.put(cn, p);
                return p;
            } catch (Throwable x) {
                fail(service,
                     "Provider " + cn + " could not be instantiated",
                     x);
            }
            throw new Error();          // This cannot happen
        }

        public boolean hasNext() {
            if (acc == null) {
                return hasNextService();
            } else {
                PrivilegedAction<Boolean> action = new PrivilegedAction<Boolean>() {
                    public Boolean run() { return hasNextService(); }
                };
                return AccessController.doPrivileged(action, acc);
            }
        }

        public S next() {
            if (acc == null) {
                return nextService();
            } else {
                PrivilegedAction<S> action = new PrivilegedAction<S>() {
                    public S run() { return nextService(); }
                };
                return AccessController.doPrivileged(action, acc);
            }
        }

        public void remove() {
            throw new UnsupportedOperationException();
        }

    }
```

hasNextService()中，核心实现如下：

* 首先使用loader加载配置文件，此时找到了META-INF/services/com.chiyi.spi.DemoService文件；
* 然后解析这个配置文件，并将各个实现类名称存储在pending的ArrayList中;
* 最后指定nextName; --> 此时nextName=com.chiyi.spi.impl.EnglishDemoServiceImpl

nextService()中，核心实现如下：

* 首先加载nextName代表的类Class，这里为com.chiyi.spi.impl.EnglishDemoServiceImpl；
* 之后创建该类的实例，并转型为所需的接口类型
* 最后存储在provider中，供后续查找，最后返回转型后的实现类实例。
