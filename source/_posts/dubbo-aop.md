---
title: dubbo-aop
date: 2019-01-10 13:40:44
tags: [Java,dubbo]
toc: true
---

## dubbo-aop

```Java
ExtensionLoader<Protocol> loader = ExtensionLoader.getExtensionLoader(Protocol.class);
final Protocol dubboProtocol = loader.getExtension("dubbo");
final Protocol adaptiveExtension = loader.getAdaptiveExtension();
```

### 获取一个ExtensionLoader

第一行代码后获得的loader：
> Class<?> type = interface com.alibaba.dubbo.rpc.Protocol
> ExtensionFactory objectFactory = AdaptiveExtensionFactory
> * factories = [SpringExtensionFactory实例, SpiExtensionFactory实例]

### getExtension("dubbo")

调用层级

```Java
ExtensionLoader<T>.getExtension()
    createExtension(name)
        getExtensionClasses().get(name) //获取扩展类
            loadExtensionClasses()
                loadDirectory()
        injectExtension(instance)   // ioc
        wrapper()   // aop
```

createExtension

```Java
    @SuppressWarnings("unchecked")
    private T createExtension(String name) {
        // 从cachedClasses缓存中获取所有实现类map，之后通过name获取到对应实现类的Class对象
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null) {
            throw findException(name);
        }
        try {
            // 从EXTENSION_INSTANCES缓存中获取对应的实现类的Class对象，如果没有，直接创建并放入缓存
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            if (instance == null) {
                EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);
            }
            injectExtension(instance);
            // wrapper
            Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
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

这里，先给出META-INF/dubbo/internal/com.alibaba.dubbo.rpc.Protocol内容：

```properties
...
filter=com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper
listener=com.alibaba.dubbo.rpc.protocol.ProtocolListenerWrapper
...
```

com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper和com.alibaba.dubbo.rpc.protocol.ProtocolListenerWrapper这2个类不含有@Adaptive注解且具有Protocol的单参构造器，符合这一条件的会被列入AOP增强类。放置在loader的私有属性cachedWrapperClasses中。

此时的loader

```Java
Class<?> type = interface com.alibaba.dubbo.rpc.Protocol
ExtensionFactory objectFactory = AdaptiveExtensionFactory（适配类）
    factories = [SpringExtensionFactory实例, SpiExtensionFactory实例]
cachedWrapperClasses = [class ProtocolListenerWrapper, class ProtocolFilterWrapper]
```

```Java
    Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
                for (Class<?> wrapperClass : wrapperClasses) {
            instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
        }
    }
```
在cachedWrapperClasses中缓存了两个AOP增强类：class ProtocolListenerWrapper和class ProtocolFilterWrapper。

首先是获取ProtocolListenerWrapper的单构造器，然后创建ProtocolListenerWrapper实例，最后完成对ProtocolListenerWrapper实例进行属性注入，注意此时的instance=ProtocolListener实例，而不再是之前的DubboProtocol实例了。之后使用ProtocolFilterWrapper以同样的方式进行包装，只是此时ProtocolFilterWrapper包装的是ProtocolListenerWrapper实例，也就是类似于这样的关系：

```Java
instance = ProtocolFilterWrapper实例{
    protocol = ProtocolListenerWrapper实例{
        protocol = DubboProtocol实例
    }
}
```

最后返回的instance是ProtocolFilterWrapper对象，也就是说final Protocol dubboProtocol = loader.getExtension("dubbo");这句代码最后的dubboProtocol是ProtocolFilterWrapper实例。

