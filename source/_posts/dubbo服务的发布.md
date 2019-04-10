---
title: dubbo服务的发布
date: 2019-03-18 15:12:46
tags: [JAVA,Concurrency]
toc: true
---

## dubbo与spring接入

dubbo的官方文档也说明了，dubbo可以不依赖任何Spring。这一块日后再详细说明，目前先介绍dubbo与Spring的集成。与spring的集成是基于Spring的Schema扩展进行加载

### spring对外留出的扩展

用过Spring就知道可以在xml文件中进行如下配置：

```xml
<context:component-scan base-package="com.demo.dubbo.server.serviceimpl"/>

<context:property-placeholder location="classpath:config.properties"/>

<tx:annotation-driven transaction-manager="transactionManager"/>
```

Spring是如何来解析这些配置呢？如果我们想自己定义配置该如何做呢？

对于上述的xml配置，分成三个部分

- 命名空间namespace，如tx、context
- 元素element，如component-scan、property-placeholder、annotation-driven
- 属性attribute，如base-package、location、transaction-manager

Spring定义了两个接口，来分别解析上述内容：

- NamespaceHandler：注册了一堆BeanDefinitionParser,利用他们来解析
- BeanDefinitionParser:用于解析每个element的内容

来看下具体的一个案例，就以spring的context命名空间为例，对应的NamespaceHandler实现是ContextNamespaceHandler:

```Java
public class ContextNamespaceHandler extends NamespaceHandlerSupport {

    @Override
    public void init() {
        registerBeanDefinitionParser("property-placeholder", new PropertyPlaceholderBeanDefinitionParser());
        registerBeanDefinitionParser("property-override", new PropertyOverrideBeanDefinitionParser());
        registerBeanDefinitionParser("annotation-config", new AnnotationConfigBeanDefinitionParser());
        registerBeanDefinitionParser("component-scan", new ComponentScanBeanDefinitionParser());
        registerBeanDefinitionParser("load-time-weaver", new LoadTimeWeaverBeanDefinitionParser());
        registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
        registerBeanDefinitionParser("mbean-export", new MBeanExportBeanDefinitionParser());
        registerBeanDefinitionParser("mbean-server", new MBeanServerBeanDefinitionParser());
    }

}
```

注册了一堆BeanDefinitionParser,如果我们想看“component-scan”是如何实现的，就可以去看对应的ComponentScanBeanDefinitionParser的源码

如果自定义了NamespaceHandler，如何加入到Spring中呢

Spring默认会加载jar包下的META-INF/spring.handlers文件下寻找NamespaceHandler,默认的Spring文件如下：

![示意图](/img/dubbo-service-provider-publish.png)

文件内容如下：

```properties
http\://www.springframework.org/schema/context=org.springframework.context.config.ContextNamespaceHandler
http\://www.springframework.org/schema/jee=org.springframework.ejb.config.JeeNamespaceHandler
http\://www.springframework.org/schema/lang=org.springframework.scripting.config.LangNamespaceHandler
http\://www.springframework.org/schema/task=org.springframework.scheduling.config.TaskNamespaceHandler
http\://www.springframework.org/schema/cache=org.springframework.cache.config.CacheNamespaceHandler
```

相应的命名空间使用相应的NamespaceHandler

## dubbo的接入实现

dubbo就是自定义类型的，所以也要给出NamespaceHandler、BeanDefinitionParser。NamespaceHandler是DubboNamespaceHandler:

```Java
public class DubboNamespaceHandler extends NamespaceHandlerSupport {

    static {
        Version.checkDuplicate(DubboNamespaceHandler.class);
    }

    public void init() {
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
    }

}
```

给出的BeanDefinitionParser全部是DubboBeanDefinitionParser，如果我们想看看dubbo:registry是怎么解析的，就可以去看看DubboBeanDefinitionParser的源代码。

而dubbo的jar包下，存在着META-INF/spring.handlers文件，内容如下：

```properties
http\://code.alibabatech.com/schema/dubbo=com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler
```

具体解析过程就不再说明了。结果就是不同的配置分别转换成Spring容器中的一个bean对象。

- application对应ApplicationConfig
- registry对应RegistryConfig
- monitor对应MonitorConfig
- provider对应ProviderConfig
- consumer对应ConsumerConfig
- protocol对应ProtocolConfig
- service对应ServiceConfig
- reference对应ReferenceConfig

上面的对象不依赖Spring，也就是说你可以手动去创建上述对象。

为了在Spring启动的时候，也相应的启动provider发布服务注册服务的过程：又加入了一个和Spring相关联的ServiceBean，继承了ServiceConfig

为了在Spring启动的时候，也相应的启动consumer发现服务的过程：又加入了一个和Spring相关联的ReferenceBean，继承了ReferenceConfig

利用Spring就做了上述过程，得到相应的配置数据，然后启动相应的服务。如果想剥离Spring，我们就可以手动来创建上述配置对象，通过ServiceConfig和ReferenceConfig的API来启动相应的服务

## 服务的发布过程

### 案例介绍

从上面知道，利用Spring的解析收集到很多一些配置，然后将这些配置都存至ServiceConfig中，然后调用ServiceConfig的export()方法来进行服务的发布与注册

先看一个简单的服务端例子，dubbo配置如下：

```xml
<dubbo:application name="helloService-app" />

<dubbo:registry  protocol="zookeeper"  address="127.0.0.1:2181"  />

<dubbo:service interface="com.demo.dubbo.service.HelloService" ref="helloService" />

<bean id="helloService" class="com.demo.dubbo.server.serviceimpl.HelloServiceImpl"/>

```

- 有一个服务接口，HelloService，以及它对应的实现类HelloServiceImpl
- 将HelloService标记为dubbo服务，使用HelloServiceImpl对象来提供具体的服务
- 使用zooKeeper作为注册中心

### 服务发布的过程

一个服务可以有多个注册中心、多个服务协议

多注册中心信息：

首选根据注册中心配置，即上述的ZooKeeper配置信息，将注册信息聚合在一个URL对象中，registryURLs内容如下：

```
[registry://192.168.1.104:2181/com.alibaba.dubbo.registry.RegistryService?application=helloService-app&localhost=true&registry=zookeeper]
```

多协议信息：

由于上述我们没有配置任何协议信息，就会使用默认的dubbo协议，开放在20880端口，也就是在该端口，对外提供上述的HelloService服务，注册的协议信息也转化成一个URL对象，如下:

```
dubbo://192.168.1.104:20880/com.demo.dubbo.service.HelloService?anyhost=true&application=helloService-app&dubbo=2.0.13&interface=com.demo.dubbo.service.HelloService&methods=hello&prompt=dubbo&revision=
```

依据注册中心信息和协议信息的组合起来，依次来进行服务的发布。整个过程伪代码如下：

```Java
List<URL> registryURLs = loadRegistries();
for (ProtocolConfig protocolConfig : protocols) {
    //根据每一个协议配置构建一个URL
    URL url = new URL(name, host, port, (contextPath == null || contextPath.length() == 0 ? "" : contextPath + "/") + path, map);       
    for (URL registryURL : registryURLs) {
        String providerURL = url.toFullString();
        Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(RpcConstants.EXPORT_KEY, providerURL));
        Exporter<?> exporter = protocol.export(invoker);
    }
}
```

所以服务发布过程大致分成两步：

- 第一步：通过ProxyFactory将HelloServiceImpl封装成一个Invoker
- 第二步：使用Protocol将invoker导出成一个Exporter

这里面就涉及到几个大的概念。ProxyFactory、Invoker、Protocol、Exporter。下面来一一介绍

### 概念介绍

分别介绍下Invoker、ProxyFactory、Protocol、Exporter的概念

#### Invoker概念

Invoker：一个可以被执行的对象，能够根据方法名称、参数得到相应的执行结果。接口内容简略如下：

```Java
public interface Invoker<T> {

    Class<T> getInterface();

    URL getUrl();

    boolean isAvailable();
    
    Result invoke(Invocation invocation) throws RpcException;

    void destroy();

}

```

而Invocation则包含了需要执行的方法、参数等信息，接口定义简略如下：

```Java
public interface Invocation {

    URL getUrl();
    
    String getMethodName();

    Class<?>[] getParameterTypes();

    Object[] getArguments();

}
```

目前其实现类只有一个RpcInvocation。内容大致如下：

```Java
public class RpcInvocation implements Invocation, Serializable {

    private String              methodName;

    private Class<?>[]          parameterTypes;

    private Object[]            arguments;

    private transient URL       url;
}
```

仅仅提供了Invocation所需要的参数而已，继续回到Invoker

这个可执行对象的执行过程分成三种类型：

- 类型1：本地执行类的Invoker
- 类型2：远程通信执行类的Invoker
- 类型3：多个类型的Invoker聚合成的集群版的Invoker

以HelloService接口方法为例：

- 本地执行类的Invoker： server端，含有对应的HelloServiceImpl实现，要执行该接口方法，仅仅只需要通过反射执行HelloServiceImpl对应的方法即可
- 远程通信执行类的Invoker： client端，要想执行该接口方法，需要需要进行远程通信，发送要执行的参数信息给server端，server端利用上述本地执行的Invoker执行相应的方法，然后将返回的结果发送给client端。这整个过程算是该类Invoker的典型的执行过程
- 集群版的Invoker：client端，拥有某个服务的多个Invoker，此时client端需要做的就是将这个多个Invoker聚合成一个集群版的Invoker，client端使用的时候，仅仅通过集群版的Invoker来进行操作。集群版的Invoker会从众多的远程通信类型的Invoker中选择一个来执行（从中加入负载均衡策略），还可以采用一些失败转移策略等

![示意图](/img/dubbo-service-provider-publish-2.png)

#### ProxyFactory概念

对于server端，主要负责将服务如HelloServiceImple统一包装成一个Invoker，这些Invoker通过反射来执行具体的HelloServiceImpl对象的方法。

接口定义如下

```Java
@SPI("javassist")
public interface ProxyFactory {
    // 针对client端，创建出代理对象
    @Adaptive({"proxy"})
    <T> T getProxy(Invoker<T> var1) throws RpcException;

    // 针对server端，将服务对象如HelloServiceImpl包装成一个Invoker对象
    @Adaptive({"proxy"})
    <T> Invoker<T> getInvoker(T var1, Class<T> var2, URL var3) throws RpcException;
}
```

ProxyFactory的接口实现有JdkProxyFactory、JavassistProxyuFactory,默认是JavassistProxyFactory,JdkProxyFactory内容如下：

```Java
public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
    return new AbstractProxyInvoker<T>(proxy, type, url) {
        @Override
        protected Object doInvoke(T proxy, String methodName, 
                                  Class<?>[] parameterTypes, 
                                  Object[] arguments) throws Throwable {
            Method method = proxy.getClass().getMethod(methodName, parameterTypes);
            return method.invoke(proxy, arguments);
        }
    };
}
```

可以看到的是创建了一个AbstractProxyFactory(这个类就是本地执行的Invoker)，它对Invoker的Result invoke(Invocation invocation)实现如下：

```Java
    public Result invoke(Invocation invocation) throws RpcException {
        try {
            return new RpcResult(doInvoke(proxy, invocation.getMethodName(), invocation.getParameterTypes(), invocation.getArguments()));
        } catch (InvocationTargetException e) {
            return new RpcResult(e.getTargetException());
        } catch (Throwable e) {
            throw new RpcException("Failed to invoke remote proxy method " + invocation.getMethodName() + " to " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```

综上所述，服务发布的第一个过程就是：

使用ProxyFactory将HelloServiceImpl封装成一个本地执行的Invoker。

#### Protocol概念

从上面得知服务发布的第一个过程就是：

使用ProxyFactory将HelloServiceImpl封装成一个本地执行的Invoker。

执行这个服务，即执行这个本地Invoker，即调用这个本地Invoker的invoke(Invocation invocation)方法，方法的执行过程就是通过反射执行了HelloServiceImpl的内容。现在的问题是：这个方法的参数Invocation invocation的来源问题。

针对server端来说，Protocol要解决的问题就是：根据指定协议对外公布HelloService服务，当客户端根据协议调用这个服务时，将客户端传递过来的Invocation参数交给上述的Invoker来执行。所以Protocol加入了远程通信协议的这一块，根据客户端的请求来获取参数Invocation invocation。

先来看下Protocol的接口定义：

```Java
@Extension("dubbo")
public interface Protocol {
    
    int getDefaultPort();

    //针对server端来说，将本地执行类的Invoker通过协议暴漏给外部。这样外部就可以通过协议发送执行参数Invocation，然后交给本地Invoker来执行
    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    //这个是针对客户端的，客户端从注册中心获取服务器端发布的服务信息
    //通过服务信息得知服务器端使用的协议，然后客户端仍然使用该协议构造一个Invoker。这个Invoker是远程通信类的Invoker。
    //执行时，需要将执行信息通过指定协议发送给服务器端，服务器端接收到参数Invocation，然后交给服务器端的本地Invoker来执行
    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;

    void destroy();

}
```

我们再来详细看看服务发布的第二步：

```Java
Exporter<?> exporter = protocol.export(invoker);
```

protocol的来历是：

```Java
Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```

我们从第一篇文章dubbo源码分析系列（1）扩展机制的实现,可以知道上述获取Protocol protocol的原理，这里就不再多说了，直接贴出最终的Protocol的实现代码：

```Java
public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.RpcException{
    if (arg0 == null)  { 
        throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null"); 
    }
    if (arg0.getUrl() == null) { 
        throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null"); 
    }
    com.alibaba.dubbo.common.URL url = arg0.getUrl();
    String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
    if(extName == null) {
        throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])"); 
    }
    com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)com.alibaba.dubbo.common.ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
    return extension.export(arg0);
}

public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0,com.alibaba.dubbo.common.URL arg1) throws com.alibaba.dubbo.rpc.RpcException{
    if (arg1 == null)  { 
        throw new IllegalArgumentException("url == null"); 
    }
    com.alibaba.dubbo.common.URL url = arg1;
    String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
    if(extName == null) {
        throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])"); 
    }
    com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)com.alibaba.dubbo.common.ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
    return extension.refer(arg0, arg1);
}
```

export(Invoker invoker)的过程即根据Invoker中url的配置信息来最终选择的Protocol来实现，默认实现是"dubbo"的扩展实现即DubboProtocol，然后在对DubboProtocol进行依赖植入，进行wrap包装。先来看看Protocol的实现情况：

![示意图](/img/dubbo-service-provider-publish-3.png)

可以看到在返回DubboProtocol之前，经过了ProtocolFilterWrapper、ProtocolListenerWrapper、RegistryProtocol的包装。

所谓的包装就是如下类似的内容：

```Java
package com.alibaba.xxx;

import com.alibaba.dubbo.rpc.Protocol;
 
public class XxxProtocolWrapper implemenets Protocol {
    Protocol impl;
 
    public XxxProtocol(Protocol protocol) { impl = protocol; }
 
    // 接口方法做一个操作后，再调用extension的方法
    public Exporter<T> export(final Invoker<T> invoker) {
        //... 一些操作
        impl .export(invoker);
        // ... 一些操作
    }
 
    // ...
}
```

使用装饰器模式，类似AOP的功能。

下面主要讲解RegistryProtocol和DubboProtocol，先暂时忽略ProtocolFilterWrapper、ProtocolListenerWrapper

所以上述服务发布的过程

```Java
Exporter<?> exporter = protocol.export(invoker)；
```

会先经过RegistryProtocol,它干了哪些事情呢？

- 利用内部的Protocol即DubboProtocol，将服务进行导出，如下
```Java
exporter = protocol.export(new InvokerWrapper<T>(invoker, url));
```

- 根据注册中心的registryUrl获取注册服务的Registry,然后将serviceUrl注册到注册中心上，供客户的订阅

```Java
Registry registry = registryFactory.getRegistry(registryUrl);
registry.register(serviceUrl)
```

来详细看看上述DubboProtocol的服务导出功能：

- 首先根据Invoker的url获取ExchangeServer通信对象（负责与客户端的通信模块），以url中的host和port作为key存至Map<String, ExchangeServer> serverMap中。即可以采用全部服务的通信交给这一个ExchangeServer通信对象，也可以某些服务单独使用新的ExchangeServer通信对象。

```Java
String key = url.getAddress();
//client 也可以暴露一个只有server可以调用的服务。
boolean isServer = url.getParameter(RpcConstants.IS_SERVER_KEY,true);
if (isServer && ! serverMap.containsKey(key)) {
    serverMap.put(key, getServer(url));
}
```
- 创建一个DubboExporter，封装invoker。然后根据url的port、path（接口的名称）、版本号、分组号作为key，将DubboExporter存至Map<String, Exporter<?>> exporterMap中

```Java
key = serviceKey(url);
DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
exporterMap.put(key, exporter);
```

现在我们要搞清楚我们的目的：通过通信对象获取客户端传来的Invocation invocation参数，然后找到对应的DubboExporter(即能够获取到本地Invoker)就可以执行服务了。

上述每一个ExchangeServer通信对象都绑定了一个ExchangeHandler requestHandler对象，内容简略如下：

```Java
private ExchangeHandler requestHandler = new ExchangeHandlerAdapter() {
    
    public Object reply(ExchangeChannel channel, Object message) throws RemotingException {
        if (message instanceof Invocation) {
            Invocation inv = (Invocation) message;
            Invoker<?> invoker = getInvoker(channel, inv);
            RpcContext.getContext().setRemoteAddress(channel.getRemoteAddress());
            return invoker.invoke(inv);
        }
        throw new RemotingException(channel, "Unsupported request: " + message == null ? null : (message.getClass().getName() + ": " + message) + ", channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress());
    }
};
```

可以看到在获取到Invocation参数后，调用getInvoker(channel,inv)来获取本地Invoker。获取的过程就是根据channel获取port，根据Invocation inv信息获取要调用的服务接口，版本号、分组号等，组装成key，从上述Map<String, Exporter<?>> exporterMap中获取Exporter,然后就可以找到对应的Invoker了，就可以顺利的调用服务了。

#### Exporter概念

负责维护invoker的生命周期，接口定义如下
```
public interface Exporter<T> {

    Invoker<T> getInvoker();

    void unexport();

}
```

包含了一个Invoker对象，一旦想撤销该服务，就会调用Invoker的destroy()方法，同时清理上述exporterMap中的数据。对于RegistryProtocol来说就需要向注册中心撤销该服务。

## 服务的引用

先看一个简单的客户端引用服务的例子，dubbo配置如下：

```xml
<dubbo:application name="consumer-of-helloService" />

<dubbo:registry  protocol="zookeeper"  address="127.0.0.1:2181" />

<dubbo:reference id="helloService" interface="com.demo.dubbo.service.HelloService" />

```

- 使用zooKeeper作为注册中心
- 引用远程的HelloService接口服务

HelloService接口内容如下：
```Java
public interface HelloService {
    public String hello(String msg);
}
```

从前面部分就可以看到dubbo与Spring的接入过程的实质：

利用Spring的xml配置创建出一系列的配置对象，存至Spring容器中

- application对应ApplicationConfig
- registry对应RegistryConfig
- monitor对应MonitorConfig
- provider对应ProviderConfig
- consumer对应ConsumerConfig
- protocol对应ProtocolConfig
- service对应ServiceConfig
- reference对应ReferenceConfig

上面的对象不依赖Spring，也就是说你可以手动去创建上述对象。

为了在Spring启动的时候，也相应的启动provider发布服务注册服务的过程：又加入了一个和Spring相关联的ServiceBean，继承了ServiceConfig

为了在Spring启动的时候，也相应的启动consumer发现服务的过程：又加入了一个和Spring相关联的ReferenceBean，继承了ReferenceConfig

利用Spring就做了上述过程，得到相应的配置数据，然后启动相应的服务。如果想剥离Spring，我们就可以手动来创建上述配置对象，通过ServiceConfig和ReferenceConfig的API来启动相应的服务

具体针对上述案例，则是 根据dubbo:reference配置创建了一个ReferenceBean，该bean又实现了Spring的org.springframework.beans.factory.FactoryBean接口，所以我们如下方式使用时：

具体针对上述案例，则是 根据dubbo:reference配置创建了一个ReferenceBean，该bean又实现了Spring的org.springframework.beans.factory.FactoryBean接口，所以我们如下方式使用时：

```Java
@Autowired
private HelloService helloService;
```

使用的不是ReferenceBean对象，而是ReferenceBean的getObject()方法返回的对象。该对象通过代理实现了HelloService接口。所以要看服务引用的整个过程就需要从ReferenceBean的getObject()方法开始入手。

## 服务引用过程

第一步：收集配置的参数，参数如下：

```
methods=hello,
timestamp=1443695417847,
dubbo=2.5.3
application=consumer-of-helloService
side=consumer
pid=7748
interface=com.demo.dubbo.service.HelloService
```

第二步：从注册中心引用服务，创建出 Invoker对象

如果是单个注册中心，代码如下：

```Java
Protocol refprotocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();

invoker = refprotocol.refer(interfaceClass, url);
```

上述url内容如下：

```properties
registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?
application=consumer-of-helloService&
dubbo=2.5.3&
pid=8292&
registry=zookeeper&
timestamp=1443707173909&
refer=
    application=consumer-of-helloService&
    dubbo=2.5.3&
    interface=com.demo.dubbo.service.HelloService&
    methods=hello&
    pid=8292&
    side=consumer&
    timestamp=1443707173884&

```

前面的信息是注册中心的配置信息，如使用zookeeper来作为注册中心

后面refer的内容是要引用的服务信息，如引用HelloService服务

使用协议Protocol根据上述的url和服务接口来引用服务，创建出一个Invoker对象

第三步：使用ProxyFactory创建出一个接口的代理对象，该代理对象的方法的执行都交给上述Invoker来执行，代码如下
```Java
ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();

proxyFactory.getProxy(invoker);
```

下面就来详细的说明下上述第二步和第三步的过程中涉及到的几个概念

## 概念介绍

### Invoker概念

Invoker介绍参考上文

对于客户端来说，Invoker则应该是远程通信执行类的Invoker、多个远程通信类型的Invoker聚合成的集群版的Invoker这两种类型。先来说说非集群版的Invoker，即远程通信类型的Invoker。来看下DubboInvoker的具体实现

```Java
    @Override
    protected Result doInvoke(final Invocation invocation) throws Throwable {
        RpcInvocation inv = (RpcInvocation) invocation;
        final String methodName = RpcUtils.getMethodName(invocation);
        inv.setAttachment(Constants.PATH_KEY, getUrl().getPath());
        inv.setAttachment(Constants.VERSION_KEY, version);

        ExchangeClient currentClient;
        if (clients.length == 1) {
            currentClient = clients[0];
        } else {
            currentClient = clients[index.getAndIncrement() % clients.length];
        }
        try {
            boolean isAsync = RpcUtils.isAsync(getUrl(), invocation);
            boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
            int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
            // 不需要返回值
            if (isOneway) {
                boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
                currentClient.send(inv, isSent);
                RpcContext.getContext().setFuture(null);
                return new RpcResult();
            } else if (isAsync) {
                // 异步
                ResponseFuture future = currentClient.request(inv, timeout);
                RpcContext.getContext().setFuture(new FutureAdapter<Object>(future));
                return new RpcResult();
            } else {
                // 同步
                RpcContext.getContext().setFuture(null);
                return (Result) currentClient.request(inv, timeout).get();
            }
        } catch (TimeoutException e) {
            throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        } catch (RemotingException e) {
            throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```

大概内容就是：

将通过远程通信将Invocation信息传递给服务器端，服务器端接收到该Invocation信息后，找到对应的本地Invoker，然后通过反射执行相应的方法，将方法的返回值再通过远程通信将结果传递给客户端。

这里分成3种情况：

- 执行的方法不需要返回值：直接使用ExchangeClient的send方法
- 执行的方法的结果需要异步返回：使用ExchangeClient的request方法，返回一个ResponseFuture,通过ThreadLocal方式与当前线程绑定，未等服务端相应结果就直接返回
- 执行的方法的结果需要同步返回：使用ExchangeClient的request方法，返回一个ResponseFuture,一直阻塞到服务端返回响应结果。

#### Protocol概念

从上面得知服务引用的第二个过程就是：

```Java
invoker = refprotocol.refer(interfaceClass, url);
```

使用协议Protocol根据上述的url和服务接口来引用服务，创建出一个Invoker对象

针对server端来说，会如下使用Protocol

```Java
Exporter<?> exporter = protocol.export(invoker);
```
Protocol要解决的问题就是：根据url中指定的协议(默认dubbo)对外公布这个HelloService服务，当客户端根据协议调用这个服务时，将客户端传递过来的Invocation参数交给服务端的Invoker来执行。所以Protocol加入了远程通信协议的这一块，根据客户端的请求来获取Invocation invocation。

而针对客户端，则需要根据服务器开放的协议(服务端在注册中心注册的url地址中含有该信息)来创建协议的Invoker对象，如
- DubboInvoker
- InJvmInvoker
- ThriftInvoker
等等

如服务器端在注册中心注册的url地址为：
```properties
dubbo://192.168.1.104:20880/com.demo.dubbo.service.HelloService?
anyhost=true&
application=helloService-app&dubbo=2.5.3&
interface=com.demo.dubbo.service.HelloService&
methods=hello&
pid=3904&
side=provider&
timestamp=1444003718316
```

会看到上述服务是以dubbo协议注册的，所以这里产生的Invoker就是DubboInvoker。我们来具体的看下这个过程

先来看下Protocol接口的定义
```Java
@SPI("dubbo")
public interface Protocol {
    
    int getDefaultPort();

    //针对server端来说，将本地执行类的Invoker通过协议暴漏给外部。这样外部就可以通过协议发送执行参数Invocation，然后交给本地Invoker来执行
    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    //这个是针对客户端的，客户端从注册中心获取服务器端发布的服务信息
    //通过服务信息得知服务器端使用的协议，然后客户端仍然使用该协议构造一个Invoker。这个Invoker是远程通信类的Invoker。
    //执行时，需要将执行信息通过指定协议发送给服务器端，服务器端接收到参数Invocation，然后交给服务器端的本地Invoker来执行
    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;

    void destroy();

}

```

我们再来详细看看服务引用的第二步：

```Java
invoker = refprotocol.refer(interfaceClass, url);
```
protocol的来历是：

```Java
Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```

Protocol的实现代码
```Java
public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.RpcException{
    if (arg0 == null)  { 
        throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null"); 
    }
    if (arg0.getUrl() == null) { 
        throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null"); 
    }
    com.alibaba.dubbo.common.URL url = arg0.getUrl();
    String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
    if(extName == null) {
        throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])"); 
    }
    com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)com.alibaba.dubbo.common.ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
    return extension.export(arg0);
}

public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0,com.alibaba.dubbo.common.URL arg1) throws com.alibaba.dubbo.rpc.RpcException{
    if (arg1 == null)  { 
        throw new IllegalArgumentException("url == null"); 
    }
    com.alibaba.dubbo.common.URL url = arg1;
    String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
    if(extName == null) {
        throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])"); 
    }
    com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)com.alibaba.dubbo.common.ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
    return extension.refer(arg0, arg1);
}

```

refer(interfaceClass,url)的过程即根据url的配置信息来最终选择Protocol实现，默认实现是"dubbo"的扩展实现即DubboProtocol，然后再对DubboProtocol进行依赖注入，进行wrap包装。先来看看Protocol的实现情况：

![示意图](/img/05083015_UsSq.png)

可以看到在返回DubboProtocol之前，经过了ProtocolFilterWrapper、ProtocolListenerWrapper、RegistryProtocol的包装。

所谓的包装就是如下类似的内容：

```Java
package com.alibaba.xxx;

import com.alibaba.dubbo.rpc.Protocol;
 
public class XxxProtocolWrapper implemenets Protocol {
    Protocol impl;
 
    public XxxProtocol(Protocol protocol) { impl = protocol; }
 
    // 接口方法做一个操作后，再调用extension的方法
    public Exporter<T> export(final Invoker<T> invoker) {
        //... 一些操作
        impl .export(invoker);
        // ... 一些操作
    }
 
    // ...
}
```

使用装饰器模式，类似AOP的功能。

所以上述服务引用的过程

```Java
invoker = refprotocol.refer(interfaceClass, urls.get(0));
```

中的refprotocol会先经过RegistryProtocol(先暂时忽略ProtocolFilterWrapper、ProtocolListenerWrapper)，它干了哪些事呢？

- 根据注册中心的registryUrl获取注册服务Registry,将自身的consumer信息注册到注册中心上

```Java
//先根据客户端的注册中心配置找到对应注册服务
Registry registry = registryFactory.getRegistry(url);

//使用注册服务将客户端的信息注册到注册中心上
registry.register(subscribeUrl.addParameters(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY,
            Constants.CHECK_KEY, String.valueOf(false)));
```

上述subscribeUrl地址如下：

```
consumer://192.168.1.104/com.demo.dubbo.service.HelloService?
    application=consumer-of-helloService&
    dubbo=2.5.3&
    interface=com.demo.dubbo.service.HelloService&
    methods=hello&
    pid=6444&
    side=consumer&
    timestamp=1444606047076
```
- 创建一个RegistryDirectory，从注册中心订阅自己引用的服务，将订阅到url在RegistryDirectory内部转换成Invoker

```Java
RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
directory.setRegistry(registry);
directory.setProtocol(protocol);
directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY, 
        Constants.PROVIDERS_CATEGORY 
        + "," + Constants.CONFIGURATORS_CATEGORY 
        + "," + Constants.ROUTERS_CATEGORY));

```
上述RegistryDirectory是Directory的实现，Directory代表多个Invoker，可以把它看成List类型的Invoker，但与List不同的是，它的值可能是动态变化的，比如注册中心推送变更。

RegistryDirectory内部含有两者重要属性：

- 注册中心服务Registry registry
- Protocol protocol。

它会利用注册中心服务Registry registry来获取最新的服务器端注册的url地址，然后再利用协议Protocol protocol将这些url地址转换成一个具有远程通信功能的Invoker对象，如DubboInvoker

- 然后使用Cluster cluster对象将上述多个Invoker对象（此时还没有真正创建出来，异步订阅，订阅成功之后，回调时才会创建出Invoker）聚合成一个集群版的Invoker对象。

```Java
Cluster cluster = ExtensionLoader.getExtensionLoader(Cluster.class).getAdaptiveExtension();

cluster.join(directory)
```

这里再详细看看Cluster接口：

```Java
@SPI(FailoverCluster.NAME)
public interface Cluster {

    /**
     * Merge the directory invokers to a virtual invoker.
     * 
     * @param <T>
     * @param directory
     * @return cluster invoker
     * @throws RpcException
     */
    @Adaptive
    <T> Invoker<T> join(Directory<T> directory) throws RpcException;

}
```

只有一个功能就是把上述Directory（相当于一个List类型的Invoker）聚合成一个Invoker，同时也可以对List进行过滤处理（这些过滤操作也是配置在注册中心的）等实现路由的功能，主要是对用户进行透明。看看接口实现情况：

![示意图](/img/12080157_wsju.png)

默认采用的是FailoverCluster，看下FailoverCluster：

```Java
/**
 * 失败转移，当出现失败，重试其它服务器，通常用于读操作，但重试会带来更长延迟。 
 * 
 * <a href="http://en.wikipedia.org/wiki/Failover">Failover</a>
 * 
 * @author william.liangf
 */
public class FailoverCluster implements Cluster {

    public final static String NAME = "failover";

    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
        return new FailoverClusterInvoker<T>(directory);
    }

}
```

仅仅是创建了一个FailoverClusterInvoker，具体的逻辑留在调用的时候即调用该Invoker的invoke(final Invocation invocation)方法时来进行处理。其中又会涉及到另一个接口LoadBalance（从众多的Invoker中挑选出一个Invoker来执行此次调用任务），接口如下：

```Java
@SPI(RandomLoadBalance.NAME)
public interface LoadBalance {

    /**
     * select one invoker in list.
     * 
     * @param invokers invokers.
     * @param url refer url
     * @param invocation invocation.
     * @return selected invoker.
     */
    @Adaptive("loadbalance")
    <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException;

}
```

实现情况如下：

![示意图](/img/12082143_bWmp.png)

默认采用的是随机策略，具体的内容就请各自详细去研究。

#### ProxyFactory概念

前一篇文章已经讲过了，对于server端，ProxyFactory主要负责将服务如HelloServiceImpl统一进行包装成一个Invoker，这些Invoker通过反射来执行具体的HelloServiceImpl对象的方法。而对于client端，则是将上述创建的集群版Invoker创建出代理对象。

接口定义如下：

```Java
@SPI("javassist")
public interface ProxyFactory {
    // 针对client端，对Invoker对象创建出代理对象
    @Adaptive({"proxy"})
    <T> T getProxy(Invoker<T> var1) throws RpcException;

    // 针对server端，将服务对象如HelloServiceImpl包装成一个Invoker对象
    @Adaptive({"proxy"})
    <T> Invoker<T> getInvoker(T var1, Class<T> var2, URL var3) throws RpcException;
}
```

ProxyFactory的接口实现有JdkProxyFactory、JavassistProxyFactory，默认是JavassistProxyFactory， JdkProxyFactory内容如下：

```Java
public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
    return (T) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), interfaces, new InvokerInvocationHandler(invoker));
}
```

可以看到是利用jdk自带的Proxy来动态代理目标对象Invoker。所以我们调用创建出来的代理对象如HelloService helloService的方法时，会执行InvokerInvocationHandler中的逻辑：

```Java
public class InvokerInvocationHandler implements InvocationHandler {

    private final Invoker<?> invoker;
    
    public InvokerInvocationHandler(Invoker<?> handler){
        this.invoker = handler;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String methodName = method.getName();
        Class<?>[] parameterTypes = method.getParameterTypes();
        if (method.getDeclaringClass() == Object.class) {
            return method.invoke(invoker, args);
        }
        if ("toString".equals(methodName) && parameterTypes.length == 0) {
            return invoker.toString();
        }
        if ("hashCode".equals(methodName) && parameterTypes.length == 0) {
            return invoker.hashCode();
        }
        if ("equals".equals(methodName) && parameterTypes.length == 1) {
            return invoker.equals(args[0]);
        }
        return invoker.invoke(new RpcInvocation(method, args)).recreate();
    }

}
```

可以看到还是交给目标对象Invoker来执行。

## dubbo通信设计

### NIO通信层的抽象

目前dubbo已经集成的有netty、mina、grizzly。

### netty和mina的简单案例

我们先来看下io.netty的案例：

```Java
public static void main(String[] args){
    EventLoopGroup bossGroup=new NioEventLoopGroup();
    EventLoopGroup workerGroup = new NioEventLoopGroup();
    try {
        ServerBootstrap serverBootstrap=new ServerBootstrap();
        serverBootstrap.group(bossGroup,workerGroup)
            .channel(NioServerSocketChannel.class)
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) throws Exception {
                    ch.pipeline().addLast(new TcpServerHandler());
                }
            });
        ChannelFuture f=serverBootstrap.bind(8080).sync();
        f.channel().closeFuture().sync();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }finally {  
        workerGroup.shutdownGracefully();  
        bossGroup.shutdownGracefully();  
    }  
}
```

mina
```Java
public static void main(String[] args) throws IOException{
    IoAcceptor acceptor = new NioSocketAcceptor();
    acceptor.getFilterChain().addLast("codec",new ProtocolCodecFilter(
            new TextLineCodecFactory(Charset.forName("UTF-8"),"\r\n", "\r\n")));
    acceptor.setHandler(new TcpServerHandler());  
    acceptor.bind(new InetSocketAddress(8080));  
}
```

两者都是使用Reactor模型结构。而最原始BIO模型如下：

![图片](/img/20083738_I5mX.png)

每来一个Socket连接都为该Socket创建一个线程来处理。由于总线程数有限制，导致Socket连接受阻，所以BIO模型并发量并不大

Rector多线程模型如下

![rector多线程模型](/img/20083315_ObVg.png)

用一个boss线程，创建selector，用于不断监听socket链接，客户端的读写操作等
用一个线程池，即workers，负责处理selector派发的读写操作。
由于boss线程可以接收更多的socket链接，同时可以充分利用线程池中的每个线程，减少了BIO模型下每个线程为单独的socket的等待时间。

### 服务端如何集成netty和mina

先来简单总结下上述netty和mina的相似之处，然后进行抽象概括成接口

- 1.各自有各自的编程启动方式
- 2.都需要各自的ChannelHandler实现，用于处理各自的Channel或者IoSession的连接、读写等事件，对于netty来说：
     需要继承org.jboss.netty.channel.SimpleChannelHandler（或者其他方式），来处理org.jboss.netty.channel.Channel的连接读写事件
     对于mina来说：需要继承org.apache.mina.common.IoHandlerAdapter（或者其他方式），来处理org.apache.mina.common.IoSession的连接读写事件

为了统一上述问题，dubbo需要做如下事情：

- 1 定义dubbo的com.alibaba.dubbo.remoting.Channel接口
    - 1.1 针对netty，上述Channel的实现为NettyChannel，内部含有一个netty自己的org.jboss.netty.channel.Channel channel对象，即该com.alibaba.dubbo.remoting.Channel接口的功能实现全部委托为底层的org.jboss.netty.channel.Channel channel对象来实现
    - 1.2 针对mina，上述Channel实现为MinaChannel，内部包含一个mina自己的org.apache.mina.common.IoSession session对象，即该com.alibaba.dubbo.remoting.Channel接口的功能实现全部委托为底层的org.apache.mina.common.IoSession session对象来实现
- 2 定义自己的com.alibaba.dubbo.remoting.ChannelHandler接口，用于处理com.alibaba.dubbo.remoting.Channel接口的连接读写事件，如下所示

```Java
public interface ChannelHandler {

    void connected(Channel channel) throws RemotingException;

    void disconnected(Channel channel) throws RemotingException;

    void sent(Channel channel, Object message) throws RemotingException;

    void received(Channel channel, Object message) throws RemotingException;

    void caught(Channel channel, Throwable exception) throws RemotingException;

}
```
    - 2.1 先定义用于处理netty的NettyHandler，需要按照netty的方式继承netty的org.jboss.netty.channel.SimpleChannelHandler，此时NettyHandler就可以委托dubbo的com.alibaba.dubbo.remoting.ChannelHandler接口实现来完成具体的功能，在交给com.alibaba.dubbo.remoting.ChannelHandler接口实现之前，需要先将netty自己的org.jboss.netty.channel.Channel channel转化成上述的NettyChannel，见NettyHandler
```Java
public void channelConnected(ChannelHandlerContext ctx, ChannelStateEvent e) throws Exception {
    NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
    try {
        if (channel != null) {
            channels.put(NetUtils.toAddressString((InetSocketAddress) ctx.getChannel().getRemoteAddress()), channel);
        }
        handler.connected(channel);
    } finally {
        NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
    }
}

@Override
public void channelDisconnected(ChannelHandlerContext ctx, ChannelStateEvent e) throws Exception {
    NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
    try {
        channels.remove(NetUtils.toAddressString((InetSocketAddress) ctx.getChannel().getRemoteAddress()));
        handler.disconnected(channel);
    } finally {
        NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
    }
}
```
    - 2.2 先定义用于处理mina的MinaHandler，需要按照mina的方式继承mina的org.apache.mina.common.IoHandlerAdapter，此时MinaHandler就可以委托dubbo的com.alibaba.dubbo.remoting.ChannelHandler接口实现来完成具体的功能，在交给com.alibaba.dubbo.remoting.ChannelHandler接口实现之前，需要先将mina自己的org.apache.mina.common.IoSession转化成上述的MinaChannel，见MinaHandler

```Java
public void sessionOpened(IoSession session) throws Exception {
    MinaChannel channel = MinaChannel.getOrAddChannel(session, url, handler);
    try {
        handler.connected(channel);
    } finally {
        MinaChannel.removeChannelIfDisconnectd(session);
    }
}

@Override
public void sessionClosed(IoSession session) throws Exception {
    MinaChannel channel = MinaChannel.getOrAddChannel(session, url, handler);
    try {
        handler.disconnected(channel);
    } finally {
        MinaChannel.removeChannelIfDisconnectd(session);
    }
}
```
做了上述事情之后，全部逻辑就统一到dubbo自己的com.alibaba.dubbo.remoting.ChannelHandler接口如何来处理自己的com.alibaba.dubbo.remoting.Channel接口。

这就需要看下com.alibaba.dubbo.remoting.ChannelHandler接口的实现有哪些：

![channelHandler](/img/23082021_ZCfI.png)

- 3 定义Server接口用于统一大家的启动流程

先来看下整体的Server接口实现情况

![server](/img/24081243_Vrnj.png)

如netty的启动流程：按照netty自己的api启动方式，然后依据外界传递进来的com.alibaba.dubbo.remoting.ChannelHandler接口实现，创建出NettyHandler，最终对用户的连接请求的处理全部交给NettyHandler来处理，nettyHandler又交给了外界传递进来的com.alibaba.dubbo.remoting.ChannelHandler接口实现。
而上述Server接口的另一个分支实现HeaderExchangeServer则充当一个装饰器的角色，为所有的Server实现增添了如下功能：
向该Server所有的Channel依次进行心跳检测：
    - 如果当前时间减去最后的读取时间大于heartbeat时间或者当前时间减去最后的写时间大于heartbeat时间，则向该Channel发送一次心跳检测
    - 如果当前时间减去最后的读取时间大于heartbeatTimeout，则服务器端要关闭该Channel，如果是客户端的话则进行重新连接（客户端也会使用这个心跳检测任务）

### 客户端如何集成netty和mina

服务器端了解了之后，客户端就也非常清楚了，整体类图如下：

![client](/img/24084841_7w2i.png)

如NettyClient在使用netty的API开启客户端之后，仍然使用NettyHandler来处理。还是最终转化成com.alibaba.dubbo.remoting.ChannelHandler接口实现上了。

HeaderExchangeClient和上面的HeaderExchangeServer非常类似，就不再提了。

我们可以看到这样集成完成之后，就完全屏蔽了底层通信细节，将逻辑全部交给了com.alibaba.dubbo.remoting.ChannelHandler接口的实现上了。从上面我们也可以看到，该接口实现也会经过层层装饰类的包装，才会最终交给底层通信。

如HeartbeatHandler装饰类：

```Java
public void sent(Channel channel, Object message) throws RemotingException {
    setWriteTimestamp(channel);
    handler.sent(channel, message);
}

public void received(Channel channel, Object message) throws RemotingException {
    setReadTimestamp(channel);
    if (isHeartbeatRequest(message)) {
        Request req = (Request) message;
        if (req.isTwoWay()) {
            Response res = new Response(req.getId(), req.getVersion());
            res.setEvent(Response.HEARTBEAT_EVENT);
            channel.send(res);
            if (logger.isInfoEnabled()) {
                int heartbeat = channel.getUrl().getParameter(Constants.HEARTBEAT_KEY, 0);
                if(logger.isDebugEnabled()) {
                    logger.debug("Received heartbeat from remote channel " + channel.getRemoteAddress()
                                    + ", cause: The channel has no data-transmission exceeds a heartbeat period"
                                    + (heartbeat > 0 ? ": " + heartbeat + "ms" : ""));
                }
            }
        }
        return;
    }
    if (isHeartbeatResponse(message)) {
        if (logger.isDebugEnabled()) {
            logger.debug(
                new StringBuilder(32)
                    .append("Receive heartbeat response in thread ")
                    .append(Thread.currentThread().getName())
                    .toString());
        }
        return;
    }
    handler.received(channel, message);
}
```

就会拦截那些上述提到的心跳检测请求。更新该Channel的最后读写时间。

### 同步调用和异步调用的实现

首先设想一下我们目前的通信方式，使用netty mina等异步事件驱动的通信框架，将Channel中信息都分发到Handler中去处理了，Handler中的send方法只负责不断的发送消息，receive方法只负责不断接收消息，这时候就产生一个问题：

客户端如何对应同一个Channel的接收的消息和发送的消息之间的匹配呢？

这也很简单，就需要在发送消息的时候，必须要产生一个请求id，将调用的信息连同id一起发给服务器端，服务器端处理完毕后，再将响应信息和上述请求id一起发给客户端，这样的话客户端在接收到响应之后就可以根据id来判断是针对哪次请求的响应结果了。

来看下DubboInvoker中的具体实现

```Java
boolean isAsync = RpcUtils.isAsync(getUrl(), invocation);
boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY,Constants.DEFAULT_TIMEOUT);
if (isOneway) {
    boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
    currentClient.send(inv, isSent);
    RpcContext.getContext().setFuture(null);
    return new RpcResult();
} else if (isAsync) {
    ResponseFuture future = currentClient.request(inv, timeout) ;
    RpcContext.getContext().setFuture(new FutureAdapter<Object>(future));
    return new RpcResult();
} else {
    RpcContext.getContext().setFuture(null);
    return (Result) currentClient.request(inv, timeout).get();
}
```

- 如果不需要返回值，直接使用send方法，发送出去，设置当期和线程绑定RpcContext的future为null
- 如果需要异步通信，使用request方法构建一个ResponseFuture，然后设置到和线程绑定RpcContext中
- 如果需要同步通信，使用request方法构建一个ResponseFuture，阻塞等待请求完成

可以看到的是它把ResponseFuture设置到与当前线程绑定的RpcContext中了，如果我们要获取异步结果，则需要通过RpcContext来获取当前线程绑定的RpcContext，然后就可以获取Future对象。如下所示：

```Java
String result1 = helloService.hello("World");
System.out.println("result :"+result1);
System.out.println("result : "+RpcContext.getContext().getFuture().get());
```

当设置成异步请求的时候，result1则为null,然后通过RpcContext来获取相应的值。

然后我们来看下异步请求的整个实现过程，即上述currentClient.request方法的具体内容：

```Java
public ResponseFuture request(Object request, int timeout) throws RemotingException {
    // create request.
    Request req = new Request();
    req.setVersion("2.0.0");
    req.setTwoWay(true);
    req.setData(request);
    DefaultFuture future = new DefaultFuture(channel, req, timeout);
    try{
        channel.send(req);
    }catch (RemotingException e) {
        future.cancel();
        throw e;
    }
    return future;
}
```

- 第一步：创建出一个request对象，创建过程中就自动产生了requestId,如下

```Java
public class Request {
    private final long    mId;
    private static final AtomicLong INVOKE_ID = new AtomicLong(0);

    public Request() {
        mId = newId();
    }

    private static long newId() {
        // getAndIncrement()增长到MAX_VALUE时，再增长会变为MIN_VALUE，负数也可以做为ID
        return INVOKE_ID.getAndIncrement();
    }
}
```

- 第二步：根据request请求封装成一个DefaultFuture对象，通过该对象的get方法就可以获取到请求结果。该方法会阻塞一直到请求结果产生。同时DefaultFuture对象会被存至DefaultFuture类如下结构中：

```Java
private static final Map<Long, DefaultFuture> FUTURES   = new ConcurrentHashMap<Long, DefaultFuture>();
```

key就是请求id

- 第三步：将上述请求对象发送给服务器端，同时将DefaultFuture对象返给上一层函数，即DubboInvoker中，然后设置到当前线程中

- 第四步：用户通过RpcContext来获取上述DefaultFuture对象来获取请求结果，会阻塞至服务器端返产生结果给客户端

- 第五步：服务器端产生结果，返回给客户端会在客户端的handler的receive方法中接收到，接收到之后判别接收的信息是Response后，

```Java
static void handleResponse(Channel channel, Response response) throws RemotingException {
    if (response != null && !response.isHeartbeat()) {
        DefaultFuture.received(channel, response);
    }
}
```

就会根据response的id从上述FUTURES结构中查出对应的DefaultFuture对象，并把结果设置进去。此时DefaultFuture的get方法则不再阻塞，返回刚刚设置好的结果。

至此异步通信大致就了解了，但是我们会发现一个问题：

当某个线程多次发送异步请求时，都会将返回的DefaultFuture对象设置到当前线程绑定的RpcContext中，就会造成了覆盖问题，如下调用方式：

```Java
String result1 = helloService.hello("World");
String result2 = helloService.hello("java");
System.out.println("result :"+result1);
System.out.println("result :"+result2);
System.out.println("result : "+RpcContext.getContext().getFuture().get());
System.out.println("result : "+RpcContext.getContext().getFuture().get());
```

即异步调用了hello方法，再次异步调用，则前一次的结果就被冲掉了，则就无法获取前一次的结果了。必须要调用一次就立马将DefaultFuture对象获取走，以免被冲掉。即这样写：

```Java
String result1 = helloService.hello("World");
Future<String> result1Future=RpcContext.getContext().getFuture();
String result2 = helloService.hello("java");
Future<String> result2Future=RpcContext.getContext().getFuture();
System.out.println("result :"+result1);
System.out.println("result :"+result2);
System.out.println("result : "+result1Future.get());
System.out.println("result : "+result2Future.get());
```

最后来张dubbo的解释图片：

![dubbo sync](/img/27083612_pxE1.png)

## 通信层与dubbo的结合

从上面可以了解到如何对不同的通信框架进行抽象，屏蔽底层细节，统一将逻辑交给ChannelHandler接口实现来处理。然后我们就来了解下如何与dubbo的业务进行对接，也就是在什么时机来使用上述通信功能：

### 服务的发布过程使用通信功能

如DubboProtocol在发布服务的过程中：

- 1 DubboProtocol中有一个如下结构
```Java
Map<String, ExchangeServer> server
```

在发布一个服务的时候会先根据服务的url获取要发布的服务所在的host和port，以此作为key来从上述结构中寻找是否已经有对应的ExchangeServer 

- 2 如果没有的话，则会创建一个，创建的过程如下：
```Java
ExchangeServer server = Exchangers.bind(url, requestHandler);
```
其中requestHandler就是DubboProtocol自身实现的ChannelHandler。
获取一个ExchangeServer，它的实现主要是Server的装饰类，依托外部传递的Server来实现Server功能，而自己加入一些额外的功能，如ExchangeServer的实现HeaderExchangeServer，就是加入了心跳检测的功能。

所以此时我们可以自定义扩展功能来实现Exchanger。接口定义如下：
```Java
@SPI(HeaderExchanger.NAME)
public interface Exchanger {

    @Adaptive({Constants.EXCHANGER_KEY})
    ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException;

    @Adaptive({Constants.EXCHANGER_KEY})
    ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException;

}
```
默认使用的就是HeaderExchanger，它创建的ExchangeServer是HeaderExchangeServer如下所示：

```Java
public class HeaderExchanger implements Exchanger {

    public static final String NAME = "header";

    public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }

    public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }

}
```
HeaderExchangeServer仅仅是一个Server接口的装饰类，需要依托外部传递Server实现来完成具体的功能。此Server实现可以是netty也可以是mina等。所以我们可以自定义Transporter实现来选择不同底层通信框架，接口定义如下：


```Java
@SPI("netty")
public interface Transporter {

    @Adaptive({Constants.SERVER_KEY, Constants.TRANSPORTER_KEY})
    Server bind(URL url, ChannelHandler handler) throws RemotingException;

    @Adaptive({Constants.CLIENT_KEY, Constants.TRANSPORTER_KEY})
    Client connect(URL url, ChannelHandler handler) throws RemotingException;

}
```
默认采用netty实现，如下：

```Java
public class NettyTransporter implements Transporter {

    public static final String NAME = "netty";

    public Server bind(URL url, ChannelHandler listener) throws RemotingException {
        return new NettyServer(url, listener);
    }

    public Client connect(URL url, ChannelHandler listener) throws RemotingException {
        return new NettyClient(url, listener);
    }

}
```
至此就到了我们上文介绍的内容了。同时DubboProtocol的ChannelHandler实现经过层层装饰器包装，最终传给底层通信Server。

客户端发送请求给服务器端时，底层通信Server会将请求经过层层处理最终传递给DubboProtocol的ChannelHandler实现，在该实现中，会根据请求参数找到对应的服务器端本地Invoker，然后执行，再将返回结果通过底层通信Server发送给客户端。

### 客户端的引用服务使用通信功能

在DubboProtocol引用服务的过程中：

- 1 使用如下方式创建client

```Java
ExchangeClient client=Exchangers.connect(url ,requestHandler)；
```
requestHandler还是DubboProtocol中ChannelHandler实现。

和Server类似，我们可以通过自定义Exchanger实现来创建出不同功能的ExchangeClient。默认的Exchanger实现是HeaderExchanger

```Java
public class HeaderExchanger implements Exchanger {

    public static final String NAME = "header";

    public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }

    public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }

}
```

创建出来的ExchangeClient是HeaderExchangeClient，它也是Client的包装类，仅仅在Client外层加上心跳检测的功能，向它所连接的服务器端发送心跳检测。

HeaderExchangeClient需要外界给它传一个Client实现，这是由Transporter接口实现来定的，默认是NettyTransporter

```Java
public class NettyTransporter implements Transporter {

    public static final String NAME = "netty";

    public Server bind(URL url, ChannelHandler listener) throws RemotingException {
        return new NettyServer(url, listener);
    }

    public Client connect(URL url, ChannelHandler listener) throws RemotingException {
        return new NettyClient(url, listener);
    }

}
```