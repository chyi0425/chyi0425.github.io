---
title: dubbo-spi
date: 2018-12-28 15:30:25
tags: [Java,Dubbo]
toc: true
---


```Java
public class TestExtension {
    public static void main(String[] args) {
        ExtensionLoader<Protocol> loader = ExtensionLoader.getExtensionLoader(Protocol.class);
        final Protocol dubboProtocol = loader.getExtension("dubbo");
        final Protocol adaptiveExtension = loader.getAdaptiveExtension();
    }
}
```

讲解这三行代码的源码。

## Protocol接口的定义

```Java
/**
 * Protocol. (API/SPI, Singleton, ThreadSafe)
 */
@SPI("dubbo")
public interface Protocol {

    /**
     * Get default port when user doesn't config the port.
     *
     * @return default port
     */
    int getDefaultPort();

    /**
     * Export service for remote invocation: <br>
     * 1. Protocol should record request source address after receive a request:
     * RpcContext.getContext().setRemoteAddress();<br>
     * 2. export() must be idempotent, that is, there's no difference between invoking once and invoking twice when
     * export the same URL<br>
     * 3. Invoker instance is passed in by the framework, protocol needs not to care <br>
     *
     * @param <T>     Service type
     * @param invoker Service invoker
     * @return exporter reference for exported service, useful for unexport the service later
     * @throws RpcException thrown when error occurs during export the service, for example: port is occupied
     */
    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    /**
     * Refer a remote service: <br>
     * 1. When user calls `invoke()` method of `Invoker` object which's returned from `refer()` call, the protocol
     * needs to correspondingly execute `invoke()` method of `Invoker` object <br>
     * 2. It's protocol's responsibility to implement `Invoker` which's returned from `refer()`. Generally speaking,
     * protocol sends remote request in the `Invoker` implementation. <br>
     * 3. When there's check=false set in URL, the implementation must not throw exception but try to recover when
     * connection fails.
     *
     * @param <T>  Service type
     * @param type Service class
     * @param url  URL address for the remote service
     * @return invoker service's local proxy
     * @throws RpcException when there's any error while connecting to the service provider
     */
    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;

    /**
     * Destroy protocol: <br>
     * 1. Cancel all services this protocol exports and refers <br>
     * 2. Release all occupied resources, for example: connection, port, etc. <br>
     * 3. Protocol can continue to export and refer new service even after it's destroyed.
     */
    void destroy();

}
```

注意：这里有两个核心注解

@SPI：指定一个接口为SPI接口（可扩展接口）
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
public @interface SPI {

    /**
     * default extension name
     */
    String value() default "";

}
```

@Adaptive：该注解可以注解在两个地方：

接口上：例如AdaptiveExtensionFactory（该类不是工厂类，有特殊的逻辑）  AdaptiveCompiler（实际上也是工厂类，但是不能靠动态生成，否则会形成死循环）
接口的方法上：会动态生成相应的动态类（实际上是一个工厂类，工厂设计模式），例如Protocol$Adapter
这个接口极其重要，后续的整个服务暴露和服务调用会用到该接口的两个方法。

## 获取ExtensionLoader

```Java
ExtensionLoader<Protocol> loader = ExtensionLoader.getExtensionLoader(Protocol.class);
```
ExtensionLoader可以类比为JDK-SPI中的**ServiceLoader**。

首先来看一下ExtensionLoader的类属性：

```Java
    /** 存放SPI文件的三个目录,其中META-INF/services/也是jdk的SPI文件的存放目录 */
    private static final String SERVICES_DIRECTORY = "META-INF/services/";

    private static final String DUBBO_DIRECTORY = "META-INF/dubbo/";

    private static final String DUBBO_INTERNAL_DIRECTORY = DUBBO_DIRECTORY + "internal/";

    private static final Pattern NAME_SEPARATOR = Pattern.compile("\\s*[,]+\\s*");
    /** key: SPI接口Class value: 该接口的ExtensionLoader */
    private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<Class<?>, ExtensionLoader<?>>();
    /** key: SPI接口Class value: SPI实现类的对象实例 */
    private static final ConcurrentMap<Class<?>, Object> EXTENSION_INSTANCES = new ConcurrentHashMap<Class<?>, Object>();
```
注意：上述的都是类属性，即所有该类的实例都共享。而后边的实例属性就属于每一个类的实例私有。

再来看一下ExtensionLoader的实例属性：

```Java
    /** SPI接口Class */
    private final Class<?> type;
    /** SPI实现类对象实例的创建工厂 */
    private final ExtensionFactory objectFactory;
    /** key: ExtensionClass的Class value: SPI实现类的key */
    private final ConcurrentMap<Class<?>, String> cachedNames = new ConcurrentHashMap<Class<?>, String>();
    /** 存放所有的extensionClass */
    private final Holder<Map<String, Class<?>>> cachedClasses = new Holder<Map<String, Class<?>>>();

    private final Map<String, Activate> cachedActivates = new ConcurrentHashMap<String, Activate>();
    /** 缓存创建好的extensionClass实例 */
    private final ConcurrentMap<String, Holder<Object>> cachedInstances = new ConcurrentHashMap<String, Holder<Object>>();
    /** 缓存创建好的适配类实例 */
    private final Holder<Object> cachedAdaptiveInstance = new Holder<Object>();
    /** 存储类上带有@Adaptive注解的Class */
    private volatile Class<?> cachedAdaptiveClass = null;
    /** 默认的SPI文件中的key */
    private String cachedDefaultName;
    /** 存储在创建适配类实例这个过程中发生的错误 */
    private volatile Throwable createAdaptiveInstanceError;
    /** 存放具有一个type入参的构造器的实现类的Class对象 */
    private Set<Class<?>> cachedWrapperClasses;
    /** key :实现类的全类名  value: exception, 防止真正的异常被吞掉 */
    private Map<String, IllegalStateException> exceptions = new ConcurrentHashMap<String, IllegalStateException>();
```

来看一下getExtensionLoader(Class<T> type)的源码：

```Java
    @SuppressWarnings("unchecked")
    public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        // 1 校验入参type：非空 + 接口 + 含有@SPI注解
        if (type == null)
            throw new IllegalArgumentException("Extension type == null");
        if (!type.isInterface()) {
            throw new IllegalArgumentException("Extension type(" + type + ") is not interface!");
        }
        if (!withExtensionAnnotation(type)) {
            throw new IllegalArgumentException("Extension type(" + type +
                    ") is not extension, because WITHOUT @" + SPI.class.getSimpleName() + " Annotation!");
        }
        // 2 根据type接口从全局缓存EXTENSION_LOADERS中获取ExtensionLoader,如果有直接返回;如果没有,则先创建,之后放入缓存,最后返回
        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }
```

创建ExtensionLoader：

```Java
    private ExtensionLoader(Class<?> type) {
        this.type = type;
        objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
    }
```

当前创建的ExtensionLoader对象（我们取名为ExtensionLoader对象1）的type是com.alibaba.dubbo.rpc.Protocol，所以此时会执行：ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension()。

首先是创建ExtensionFactory，通过上边核心类部分ExtensionFactory接口的源码可以看出，此类也是一个SPI接口类，且没有指定默认的实现类的key。

```Java
ExtensionLoader.getExtensionLoader(ExtensionFactory.class)
```

下面的代码与上述的过程相似，只是此时创建的另外一个ExtensionLoader对象（我们取名为ExtensionLoader对象2）的type是com.alibaba.dubbo.common.extension.ExtensionFactory，而objectFactory是null。之后，这个ExtensionLoader对象2被放入EXTENSION_LOADERS缓存。这里给出ExtensionFactory的定义，该类也极其重要。


```Java
@SPI
public interface ExtensionFactory {

    /**
     * Get extension.
     *
     * @param type object type.
     * @param name object name.
     * @return object instance.
     */
    <T> T getExtension(Class<T> type, String name);

}
```

之后执行ExtensionLoader对象2的getAdaptiveExtension()方法。

```Java
    @SuppressWarnings("unchecked")
    public T getAdaptiveExtension() {
        // 首先从cachedAdaptiveInstance缓存中获取AdaptiveExtension实例,如果不为null, 直接返回;
        Object instance = cachedAdaptiveInstance.get();
        if (instance == null) {
            if (createAdaptiveInstanceError == null) {
                synchronized (cachedAdaptiveInstance) {
                    instance = cachedAdaptiveInstance.get();
                    if (instance == null) {
                        try {
                            // 如果为null, 先创建AdaptiveExtension实例, 之后放入cachedAdaptiveInstance缓存中,最后返回
                            instance = createAdaptiveExtension();
                            cachedAdaptiveInstance.set(instance);
                        } catch (Throwable t) {
                            createAdaptiveInstanceError = t;
                            throw new IllegalStateException("fail to create adaptive instance: " + t.toString(), t);
                        }
                    }
                }
            } else {
                throw new IllegalStateException("fail to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
            }
        }

        return (T) instance;
    }
```

来看createAdaptiveExtension()创建AdaptiveExtension的源码：

```Java
    /**
     *createAdaptiveExtension()
     *--getAdaptiveExtensionClass()
     *从dubbo-spi配置文件中获取AdaptiveExtensionClass
     *----getExtensionClasses()
     *------loadExtensionClasses()
     *--------loadFile(Map<String, Class<?>> extensionClasses, String dir)
     * 创建动态代理类
     *----createAdaptiveExtensionClass()
     *injectExtension()
     */
    @SuppressWarnings("unchecked")
    private T createAdaptiveExtension() {
        try {
            return injectExtension((T) getAdaptiveExtensionClass().newInstance());
        } catch (Exception e) {
            throw new IllegalStateException("Can not create adaptive extension " + type + ", cause: " + e.getMessage(), e);
        }
    }
```

调用层级看注释。injectExtension(T instance)方法只对objectFactory有用，如果objectFactory==null，则直接返回T instance。所以这里返回的是getAdaptiveExtensionClass().newInstance()

来看getAdaptiveExtensionClass()的源码：

```Java
<<<<<<< HEAD
=======
    /**
     * 获取ExtensionClasses和适配类
     * 如果实现类上带有@Adaptive注解，直接创建修饰类
     * 如果方法上带有@Adaptive注解，动态生成代理类
     */
>>>>>>> 291712022f23203bfaa774b21cc7d9602f88c4e6
    private Class<?> getAdaptiveExtensionClass() {
        getExtensionClasses();
        if (cachedAdaptiveClass != null) {
            return cachedAdaptiveClass;
        }
        return cachedAdaptiveClass = createAdaptiveExtensionClass();
    }
```
<<<<<<< HEAD

现在来看getExtensionClasses()：

```Java
    
    private Map<String, Class<?>> getExtensionClasses() {
        // 先从cachedClasses缓存中获取所有的ExtensionClass,如果有,直接返回;
=======
现在来看getExtensionClasses()
```Java
    /**
     * 先从cachedClasses缓存中获取所有的ExtensionClass,如果有,直接返回;
     * 如果没有,通过loadExtensionClasses()从SPI文件中去读取,之后写入缓存
     */
    private Map<String, Class<?>> getExtensionClasses() {
>>>>>>> 291712022f23203bfaa774b21cc7d9602f88c4e6
        Map<String, Class<?>> classes = cachedClasses.get();
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                if (classes == null) {
<<<<<<< HEAD
                    // 如果没有,通过loadExtensionClasses()从SPI文件中去读取,之后写入缓存
=======
>>>>>>> 291712022f23203bfaa774b21cc7d9602f88c4e6
                    classes = loadExtensionClasses();
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
    }
```

<<<<<<< HEAD
现在来看loadExtensionClasses()


```Java
    // synchronized in getExtensionClasses
=======
现在来看loadExtensionClasses

```Java
    /**
     * 1 从@SPI注解中将默认值解析出来,并缓存到cachedDefaultName中
     * 2 从SPI文件中获取extensionClass并存储到extensionClasses中,最后返回extensionClasses
     * 注意：此方法已经getExtensionClasses方法同步过。
     */
>>>>>>> 291712022f23203bfaa774b21cc7d9602f88c4e6
    private Map<String, Class<?>> loadExtensionClasses() {
        final SPI defaultAnnotation = type.getAnnotation(SPI.class);
        if (defaultAnnotation != null) {
            String value = defaultAnnotation.value();
            if (value != null && (value = value.trim()).length() > 0) {
                String[] names = NAME_SEPARATOR.split(value);
                if (names.length > 1) {
                    throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
                            + ": " + Arrays.toString(names));
                }
<<<<<<< HEAD
                // 从@SPI注解中将默认值解析出来,并缓存到cachedDefaultName中
                if (names.length == 1) cachedDefaultName = names[0];
            }
        }
        //从SPI文件中获取extensionClass并存储到extensionClasses中,最后返回extensionClasses
=======
                if (names.length == 1) cachedDefaultName = names[0];
            }
        }

>>>>>>> 291712022f23203bfaa774b21cc7d9602f88c4e6
        Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
        loadFile(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
        loadFile(extensionClasses, DUBBO_DIRECTORY);
        loadFile(extensionClasses, SERVICES_DIRECTORY);
        return extensionClasses;
    }
```

<<<<<<< HEAD
之后来看一下非常重要的一个方法loadFile(Map<String, Class<?>> extensionClasses, String dir)。

```Java
    /**
    * 1 加载dir目录下的指定type名称的文件(例如:dubbo-2.5.5.jar中的/META-INF/dubbo/internal/com.alibaba.dubbo.common.extension.ExtensionFactory)
    * 2 遍历该文件中的每一行
    * (1)获取实现类key和value, 例如 name=spi, line=com.alibaba.dubbo.common.extension.factory.SpiExtensionFactory
    * (2)根据line创建Class对象
    * (3)将具有@Adaptive注解的实现类的Class对象放在cachedAdaptiveClass缓存中, 注意该缓存只能存放一个具有@Adaptive注解的实现类的Class对象,如果有两个满足条件,则抛异常
    * 下面的都是对不含@Adaptive注解的实现类的Class对象:
    * (4)查看是否具有含有一个type入参的构造器, 如果有（就是wrapper类）, 将当前的Class对象放置到cachedWrapperClasses缓存中
    * (5)如果没有含有一个type入参的构造器, 获取无参构造器. 如果Class对象具有@Active注解, 将该对象以<实现类的key, active>存储起来
    * (6)最后,将<Class对象, 实现类的key>存入cachedNames缓存,并将这些Class存入extensionClasses中.
    * @param extensionClasses
    * @param dir
    */
=======
之后来看下非常重要的一个方法loadFile(Map<String,Class<?>> extensionClasses,String dir)

```Java
    /**
     * 1 加载dir目录下的指定type名称的文件(例如:dubbo-2.5.5.jar中的/META-INF/dubbo/internal/com.alibaba.dubbo.common.extension.ExtensionFactory)
     * 2 遍历该文件中的每一行
     * (1)获取实现类key和value, 例如 name=spi, line=com.alibaba.dubbo.common.extension.factory.SpiExtensionFactory
     * (2)根据line创建Class对象
     * (3)将具有@Adaptive注解的实现类的Class对象放在cachedAdaptiveClass缓存中, 注意该缓存只能存放一个具有@Adaptive注解的实现类的Class对象,如果有两个满足条件,则抛异常
     * 下面的都是对不含@Adaptive注解的实现类的Class对象:
     * (4)查看是否具有含有一个type入参的构造器, 如果有（就是wrapper类）, 将当前的Class对象放置到cachedWrapperClasses缓存中
     * (5)如果没有含有一个type入参的构造器, 获取无参构造器. 如果Class对象具有@Active注解, 将该对象以<实现类的key, active>存储起来
     * (6)最后,将<Class对象, 实现类的key>存入cachedNames缓存,并将这些Class存入extensionClasses中.
     * @param extensionClasses
     * @param dir
     */
>>>>>>> 291712022f23203bfaa774b21cc7d9602f88c4e6
    private void loadFile(Map<String, Class<?>> extensionClasses, String dir) {
        String fileName = dir + type.getName();
        try {
            Enumeration<java.net.URL> urls;
            ClassLoader classLoader = findClassLoader();
            if (classLoader != null) {
                urls = classLoader.getResources(fileName);
            } else {
                urls = ClassLoader.getSystemResources(fileName);
            }
            if (urls != null) {
                while (urls.hasMoreElements()) {
                    java.net.URL url = urls.nextElement();
                    try {
                        BufferedReader reader = new BufferedReader(new InputStreamReader(url.openStream(), "utf-8"));
                        try {
                            String line = null;
                            while ((line = reader.readLine()) != null) {
                                final int ci = line.indexOf('#');
                                if (ci >= 0) line = line.substring(0, ci);
                                line = line.trim();
                                if (line.length() > 0) {
                                    try {
                                        String name = null;
                                        int i = line.indexOf('=');
                                        if (i > 0) {
                                            name = line.substring(0, i).trim();
                                            line = line.substring(i + 1).trim();
                                        }
                                        if (line.length() > 0) {
                                            Class<?> clazz = Class.forName(line, true, classLoader);
                                            if (!type.isAssignableFrom(clazz)) {
                                                throw new IllegalStateException("Error when load extension class(interface: " +
                                                        type + ", class line: " + clazz.getName() + "), class "
                                                        + clazz.getName() + "is not subtype of interface.");
                                            }
                                            if (clazz.isAnnotationPresent(Adaptive.class)) {
                                                if (cachedAdaptiveClass == null) {
                                                    cachedAdaptiveClass = clazz;
                                                } else if (!cachedAdaptiveClass.equals(clazz)) {
                                                    throw new IllegalStateException("More than 1 adaptive class found: "
                                                            + cachedAdaptiveClass.getClass().getName()
                                                            + ", " + clazz.getClass().getName());
                                                }
                                            } else {
                                                try {
                                                    clazz.getConstructor(type);
                                                    Set<Class<?>> wrappers = cachedWrapperClasses;
                                                    if (wrappers == null) {
                                                        cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
                                                        wrappers = cachedWrapperClasses;
                                                    }
                                                    wrappers.add(clazz);
                                                } catch (NoSuchMethodException e) {
                                                    clazz.getConstructor();
                                                    if (name == null || name.length() == 0) {
                                                        name = findAnnotationName(clazz);
                                                        if (name == null || name.length() == 0) {
                                                            if (clazz.getSimpleName().length() > type.getSimpleName().length()
                                                                    && clazz.getSimpleName().endsWith(type.getSimpleName())) {
                                                                name = clazz.getSimpleName().substring(0, clazz.getSimpleName().length() - type.getSimpleName().length()).toLowerCase();
                                                            } else {
                                                                throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + url);
                                                            }
                                                        }
                                                    }
                                                    String[] names = NAME_SEPARATOR.split(name);
                                                    if (names != null && names.length > 0) {
                                                        Activate activate = clazz.getAnnotation(Activate.class);
                                                        if (activate != null) {
                                                            cachedActivates.put(names[0], activate);
                                                        }
                                                        for (String n : names) {
                                                            if (!cachedNames.containsKey(clazz)) {
                                                                cachedNames.put(clazz, n);
                                                            }
                                                            Class<?> c = extensionClasses.get(n);
                                                            if (c == null) {
                                                                extensionClasses.put(n, clazz);
                                                            } else if (c != clazz) {
                                                                throw new IllegalStateException("Duplicate extension " + type.getName() + " name " + n + " on " + c.getName() + " and " + clazz.getName());
                                                            }
                                                        }
                                                    }
                                                }
                                            }
                                        }
                                    } catch (Throwable t) {
                                        IllegalStateException e = new IllegalStateException("Failed to load extension class(interface: " + type + ", class line: " + line + ") in " + url + ", cause: " + t.getMessage(), t);
                                        exceptions.put(line, e);
                                    }
                                }
                            } // end of while read lines
                        } finally {
                            reader.close();
                        }
                    } catch (Throwable t) {
                        logger.error("Exception when load extension class(interface: " +
                                type + ", class file: " + url + ") in " + url, t);
                    }
                } // end of while urls
            }
        } catch (Throwable t) {
            logger.error("Exception when load extension class(interface: " +
                    type + ", description file: " + fileName + ").", t);
        }
    }
<<<<<<< HEAD
```
=======
```

上述的方法分别从三个目录查找SPI文件并进行加载。在这里只有在META-INF/dubbo/internal/com.alibaba.dubbo.common.extension.ExtensionFactory中有值，内容如下：

```propertites
1 adaptive=com.alibaba.dubbo.common.extension.factory.AdaptiveExtensionFactory
2 spi=com.alibaba.dubbo.common.extension.factory.SpiExtensionFactory
3 spring=com.alibaba.dubbo.config.spring.extension.SpringExtensionFactory
```

其中AdaptiveExtensionFactory在类上具有@Adaptive注解，这个类会在后续去讲，这里先略过。

执行过后，看一下：
```Java
cachedAdaptiveClass=class com.alibaba.dubbo.common.extension.factory.AdaptiveExtensionFactory
extensionClasses=[{"spring","class com.alibaba.dubbo.config.spring.extension.SpringExtensionFactory"}, {"spi", "class com.alibaba.dubbo.common.extension.factory.SpiExtensionFactory"}]，后续会这个集合存储在cachedClasses缓存中。
```
上边一直在讲解getAdaptiveExtensionClass().newInstance()这句代码中的getAdaptiveExtensionClass()，此方法返回一个com.alibaba.dubbo.common.extension.factory.AdaptiveExtensionFactory类，之后来看一下其newInstance()代码，调用这个方法，默认会执行AdaptiveExtensionFactory的无参构造器。这里给出AdaptiveExtensionFactory的完整代码：

```Java
package com.alibaba.dubbo.common.extension.factory;

import com.alibaba.dubbo.common.extension.Adaptive;
import com.alibaba.dubbo.common.extension.ExtensionFactory;
import com.alibaba.dubbo.common.extension.ExtensionLoader;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/**
 * AdaptiveExtensionFactory
 */
@Adaptive
public class AdaptiveExtensionFactory implements ExtensionFactory {
    private final List<ExtensionFactory> factories;

    /**
     * 遍历cachedClasses中缓存的extensionClasses的key,之后根据key来实例化对应的实现类,最后放置到EXTENSION_INSTANCES缓存中
     */
    public AdaptiveExtensionFactory() {
        ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);
        List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();
        for (String name : loader.getSupportedExtensions()) {
            list.add(loader.getExtension(name));
        }
        factories = Collections.unmodifiableList(list);
    }

    public <T> T getExtension(Class<T> type, String name) {
        for (ExtensionFactory factory : factories) {
            T extension = factory.getExtension(type, name);
            if (extension != null) {
                return extension;
            }
        }
        return null;
    }
}
```

从上可以看出，这个装饰类只是实例化好了各个ExtensionFactory（这里是SpiExtensionFactory和SpringExtensionFactory），后续通过工厂获取实现类实例都是由具体工厂来完成。

来看一下实例化代码的地方，即loader.getExtension(name)：

```Java
    /**
     * 从cachedInstances缓存中获取name对应的实例,如果没有,通过createExtension(name)创建,之后放入缓存
     * getExtension(String name)
     * --createExtension(String name)
     * ----injectExtension(T instance)
     */
    public T getExtension(String name) {
        if (name == null || name.length() == 0)
            throw new IllegalArgumentException("Extension name == null");
        if ("true".equals(name)) {
            return getDefaultExtension();
        }
        Holder<Object> holder = cachedInstances.get(name);
        if (holder == null) {
            cachedInstances.putIfAbsent(name, new Holder<Object>());
            holder = cachedInstances.get(name);
        }
        Object instance = holder.get();
        if (instance == null) {
            synchronized (holder) {
                instance = holder.get();
                if (instance == null) {
                    instance = createExtension(name);
                    holder.set(instance);
                }
            }
        }
        return (T) instance;
    }
```

来看一下创建createExtension(name)：

```Java
private T createExtension(String name) {
        /** 从cachedClasses缓存中获取所有的实现类map,之后通过name获取到对应的实现类的Class对象 */
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null) {
            throw findException(name);
        }
        try {
            /** 从EXTENSION_INSTANCES缓存中获取对应的实现类的Class对象,如果没有,直接创建,之后放入缓存 */
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            if (instance == null) {
                EXTENSION_INSTANCES.putIfAbsent(clazz, (T) clazz.newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);
            }
            injectExtension(instance);//ioc
            Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            if (wrapperClasses != null && wrapperClasses.size() > 0) {
                for (Class<?> wrapperClass : wrapperClasses) {
                    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                }
            }
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
                    type + ")  could not be instantiated: " + t.getMessage(), t);
        }
    }
```

这里，就体现出来了dubbo-SPI比JDK-SPI的好处：dubbo-SPI不需要遍历所有的实现类来获取想要的实现类，可以直接通过name来获取。

injectExtension(instance)和wrapper包装功能后续再说。

 

到此为止，ExtensionLoader<Protocol> loader = ExtensionLoader.getExtensionLoader(Protocol.class);这行代码的整个源码就讲述完成了。最后来看一下整个代码的执行结果。

类变量

ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS
    "interface com.alibaba.dubbo.rpc.Protocol" -> "com.alibaba.dubbo.common.extension.ExtensionLoader[com.alibaba.dubbo.rpc.Protocol]"
    "interface com.alibaba.dubbo.common.extension.ExtensionFactory" -> "com.alibaba.dubbo.common.extension.ExtensionLoader[com.alibaba.dubbo.common.extension.ExtensionFactory]"
ConcurrentMap<Class<?>, Object> EXTENSION_INSTANCES
    "class com.alibaba.dubbo.config.spring.extension.SpringExtensionFactory" -> SpringExtensionFactory实例
    "class com.alibaba.dubbo.common.extension.factory.SpiExtensionFactory" -> SpiExtensionFactory实例
ExtensionLoader<Protocol> loader的实例变量：

Class<?> type = interface com.alibaba.dubbo.rpc.Protocol
ExtensionFactory objectFactory = AdaptiveExtensionFactory（适配类）
    factories = [SpringExtensionFactory实例, SpiExtensionFactory实例]

第一点：ExtensionLoader<T> loader = ExtensionLoader.getExtensionLoader(Class<T> type)最终得到的实例变量是：

Class<?> type = interface T
ExtensionFactory objectFactory = AdaptiveExtensionFactory（适配类）
    factories = [SpringExtensionFactory实例, SpiExtensionFactory实例]


第二点：ExtensionLoader<T>.getAdaptiveExtension()的调用层级

ExtensionLoader<T>.getAdaptiveExtension()
--createAdaptiveExtension()
----injectExtension(getAdaptiveExtensionClass())
------getAdaptiveExtensionClass()
--------getExtensionClasses()//从spi文件中查找实现类上具有@Adaptive注解的类
----------loadExtensionClasses()
------------loadFile(Map<String, Class<?>> extensionClasses, String dir)
--------createAdaptiveExtensionClass()//如果从spi文件中没有找到实现类上具有@Adaptive注解的类，则动态创建类

最终返回的是创建好的Adaptive类，例如AdaptiveExtensionFactory实例。

第三点：ExtensionLoader<T>.getExtension()的调用层级

ExtensionLoader<T>.getExtension()
--createExtension(String name)
----getExtensionClasses().get(name)//获取扩展类
----injectExtension(instance);//ioc
----wrapper包装;//aop

最终返回的是创建好的具体实现类，例如SpringExtensionFactory实例。 

>>>>>>> 291712022f23203bfaa774b21cc7d9602f88c4e6
