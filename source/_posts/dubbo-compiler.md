---
title: dubbo-compiler
date: 2019-01-10 14:49:00
tags: [Java,dubbo]
toc: true
---

## dubbo-compiler

```Java
ExtensionLoader<Protocol> loader = ExtensionLoader.getExtensionLoader(Protocol.class);
final Protocol dubboProtocol = loader.getExtension("dubbo");
final Protocol adaptiveExtension = loader.getAdaptiveExtension();
```

getAdaptiveExtension()层级结构：

```Java
ExtensionLoader<T>.getAdaptiveExtension()
    createAdaptiveExtension()
        injectExtension((T) getAdaptiveExtensionClass().newInstance())
            getAdaptiveExtensionClass()
                getExtensionClasses()   // 从spi文件中查找实现类上具有@Adaptive注解的类
                    loadExtensionClasses()
                        loadDirectory()
                createAdaptiveExtensionClass()  // 如果从spi文件中没有找到实现类上具有@Adaptive注解的类，则动态创建类
```

createAdaptiveExtensionClass

```Java
    private Class<?> createAdaptiveExtensionClass() {
        String code = createAdaptiveExtensionClassCode();
        ClassLoader classLoader = findClassLoader();
        org.apache.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
        return compiler.compile(code, classLoader);
    }
```

### 构造代码串

createAdaptiveExtensionClassCode()方法中会判断如果一个类中没有@Adaptive注解的方法，则直接抛出IllegalStateException异常；否则，会为有@Adaptive注解的方法构造代码，而没有@Adaptive注解的方法直接抛出UnsupportedOperationException异常。


### 获取Compiler装饰类

```Java
org.apache.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();

```

先看一下org.apache.dubbo.common.compiler.Compiler接口

```Java
@SPI("javassist")
public interface Compiler {

    /**
     * Compile java source code.
     *
     * @param code        Java source code
     * @param classLoader classloader
     * @return Compiled class
     */
    Class<?> compile(String code, ClassLoader classLoader);

}
```

@SPI的默认值为javassist，根据上一节的经验，默认获取的Compiler接口的实现类将是META-INF/dubbo/internal/com.alibaba.dubbo.common.compiler.Compiler文件中的key为javassit的实现类。文件内容如下：

```properties
adaptive=org.apache.dubbo.common.compiler.support.AdaptiveCompiler
jdk=org.apache.dubbo.common.compiler.support.JdkCompiler
javassist=org.apache.dubbo.common.compiler.support.JavassistCompiler
```

