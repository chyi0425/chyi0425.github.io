---
title: dubbo-ioc
date: 2019-01-09 21:02:37
tags: [Java,dubbo]
toc: true
---

## dubbo-ioc

dubbo的IOC具体实现在： T injectExtension(T instance) 方法中。该方法只在三个地方被使用：

```Java
    @SuppressWarnings("unchecked")
    private T createAdaptiveExtension() {
        try {
            return injectExtension((T) getAdaptiveExtensionClass().newInstance());  // 为创建好的AdaptiveExtensionClass实例进行属性注入
        } catch (Exception e) {
            throw new IllegalStateException("Can not create adaptive extension " + type + ", cause: " + e.getMessage(), e);
        }
    }

    private T createExtension(String name) {
        ...
        injectExtension(instance);  // 为创建好的Extension实例进行属性注入
        ...
        instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance)) // 为创建好的wrapperClass实例进行属性注入
    }
```

源码：

```Java
    /**
     * dubbo-IOC的核心
    **/
    private T injectExtension(T instance) {
        try {
            if (objectFactory != null) {
                for (Method method : instance.getClass().getMethods()) {
                    // 一个参数的public的setXXX(T param)方法.例如,setName(String name)
                    if (method.getName().startsWith("set")
                            && method.getParameterTypes().length == 1
                            && Modifier.isPublic(method.getModifiers())) {
                        /**
                         * Check {@link DisableInject} to see if we need auto injection for this property
                         */
                        // 如果是DisableInject，跳过
                        if (method.getAnnotation(DisableInject.class) != null) {
                            continue;
                        }
                        // 获取参数类型
                        Class<?> pt = method.getParameterTypes()[0];
                        try {
                            // 获取属性名称
                            String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
                            // 实例化参数
                            Object object = objectFactory.getExtension(pt, property);
                            if (object != null) {
                                // 执行instance.method(object)，在此处就是执行instance的setter方法
                                method.invoke(instance, object);
                            }
                        } catch (Exception e) {
                            logger.error("fail to inject via method " + method.getName()
                                    + " of interface " + type.getName() + ": " + e.getMessage(), e);
                        }
                    }
                }
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return instance;
    }
```

整个方法就是通过instance对象实例的setter方法为instance属性赋值，完成setter注入，即IOC的最经典的注入方式。

详细步骤：
* 获取instance的setter方法，通过setter方法获取属性名称property和属性类型pt
* 使用objectFactory创建一个property名称(类型为pt)的对象实例
* 执行instance的setter方法，注入property实例

其中比较重要的就是

```Java
// 这里 objectFactory 是AdaptiveExtensionFactory，AdaptiveExtensionFactory的属性factories=[SpringExtensionFactory实例, SpiExtensionFactory实例]
Object object = objectFactory.getExtension(pt, property);
// ----------------------

    @Adaptive
    public class AdaptiveExtensionFactory implements ExtensionFactory {

    private final List<ExtensionFactory> factories;

    public AdaptiveExtensionFactory() {
        ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);
        List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();
        for (String name : loader.getSupportedExtensions()) {
            list.add(loader.getExtension(name));
        }
        factories = Collections.unmodifiableList(list);
    }

    @Override
    public <T> T getExtension(Class<T> type, String name) {
        // 先调用SpiExtensionFactory实例化，如果不行，再调用SpringExtensionFactory来实例化
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
SpiExtensionFactory

```Java
public class SpiExtensionFactory implements ExtensionFactory {

    @Override
    public <T> T getExtension(Class<T> type, String name) {
        // type是接口且必须有@SPI注解
        if (type.isInterface() && type.isAnnotationPresent(SPI.class)) {
            ExtensionLoader<T> loader = ExtensionLoader.getExtensionLoader(type);
            if (!loader.getSupportedExtensions().isEmpty()) {
                // 获取type的装饰类，如果有@Adaptive注解的类，则返回该类的实例，否则返回一个动态代理类的实例
                return loader.getAdaptiveExtension();
            }
        }
        return null;
    }
}

```

从这里我们可以看出dubbo-SPI的另外一个好处：可以为SPI实现类注入SPI的装饰类或动态代理类。

SpringExtensionFactory

```Java
public class SpringExtensionFactory implements ExtensionFactory {
    private static final Logger logger = LoggerFactory.getLogger(SpringExtensionFactory.class);

    private static final Set<ApplicationContext> contexts = new ConcurrentHashSet<ApplicationContext>();
    private static final ApplicationListener shutdownHookListener = new ShutdownHookListener();

    public static void addApplicationContext(ApplicationContext context) {
        contexts.add(context);
        BeanFactoryUtils.addApplicationListener(context, shutdownHookListener);
    }

    public static void removeApplicationContext(ApplicationContext context) {
        contexts.remove(context);
    }

    public static Set<ApplicationContext> getContexts() {
        return contexts;
    }

    // currently for test purpose
    public static void clearContexts() {
        contexts.clear();
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T> T getExtension(Class<T> type, String name) {

        //SPI should be get from SpiExtensionFactory
        if (type.isInterface() && type.isAnnotationPresent(SPI.class)) {
            return null;
        }

        for (ApplicationContext context : contexts) {
            // 该context是否包含name的bean
            if (context.containsBean(name)) {
                // 获取name的bean，如果是懒加载或多例的bean，此时会实例化name的bean
                Object bean = context.getBean(name);
                if (type.isInstance(bean)) {
                    return (T) bean;
                }
            }
        }

        logger.warn("No spring extension (bean) named:" + name + ", try to find an extension (bean) of type " + type.getName());

        if (Object.class == type) {
            return null;
        }

        for (ApplicationContext context : contexts) {
            try {
                return context.getBean(type);
            } catch (NoUniqueBeanDefinitionException multiBeanExe) {
                logger.warn("Find more than 1 spring extensions (beans) of type " + type.getName() + ", will stop auto injection. Please make sure you have specified the concrete parameter type and there's only one extension of that type.");
            } catch (NoSuchBeanDefinitionException noBeanExe) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Error when get spring extension(bean) for type:" + type.getName(), noBeanExe);
                }
            }
        }

        logger.warn("No spring extension (bean) named:" + name + ", type:" + type.getName() + " found, stop get bean.");

        return null;
    }

    private static class ShutdownHookListener implements ApplicationListener {
        @Override
        public void onApplicationEvent(ApplicationEvent event) {
            if (event instanceof ContextClosedEvent) {
                // we call it anyway since dubbo shutdown hook make sure its destroyAll() is re-entrant.
                // pls. note we should not remove dubbo shutdown hook when spring framework is present, this is because
                // its shutdown hook may not be installed.
                DubboShutdownHook shutdownHook = DubboShutdownHook.getDubboShutdownHook();
                shutdownHook.destroyAll();
            }
        }
    }
}
```

至此，IOC就干完了。但是有一个遗留问题，ApplicationContext是什么时候加入到contexts中呢？当讲解ServiceBean的时候来说。

