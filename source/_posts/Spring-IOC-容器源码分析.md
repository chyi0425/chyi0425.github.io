---
title: Spring IOC 容器源码分析
date: 2018-08-06 14:22:13
tags: [Java,Spring] #文章标签，多于一项时用这种格式
toc: true
---

先来看看大体的继承结构

![示意图](/img/spring-ioc-1.png)

我们可以看到，ClassPathXmlApplicationContext 兜兜转转了好久才到 ApplicationContext 接口，同样的，我们也可以使用绿颜色的 FileSystemXmlApplicationContext 和 AnnotationConfigApplicationContext 这两个类。

FileSystemXmlApplicationContext 的构造函数需要一个 xml 配置文件在系统中的路径，其他和 ClassPathXmlApplicationContext 基本上一样。

AnnotationConfigApplicationContext 是基于注解来使用的，它不需要配置文件，采用 java 配置类和各种注解来配置，是比较简单的方式，也是大势所趋吧。

本文使用 ClassPathXmlApplicationContext 进行分析。

## BeanFactory 简介

BeanFactory，从名字上也很好理解，生产 bean 的工厂，它负责生产和管理各个 bean 实例。

初学者可别以为我之前说那么多和 BeanFactory 无关，前面说的 ApplicationContext 其实就是一个 BeanFactory。我们来看下和 BeanFactory 接口相关的主要的继承结构：

![示意图](/img/spring-ioc-2.png)

1. ApplicationContext 继承了 ListableBeanFactory，这个 Listable 的意思就是，通过这个接口，我们可以获取多个 Bean，大家看源码会发现，最顶层 BeanFactory 接口的方法都是获取单个 Bean 的。

2. ApplicationContext 继承了 HierarchicalBeanFactory，Hierarchical 单词本身已经能说明问题了，也就是说我们可以在应用中起多个 BeanFactory，然后可以将各个 BeanFactory 设置为父子关系。

3. AutowireCapableBeanFactory 这个名字中的 Autowire 大家都非常熟悉，它就是用来自动装配 Bean 用的，但是仔细看上图，ApplicationContext 并没有继承它，不过不用担心，不使用继承，不代表不可以使用组合，如果你看到 ApplicationContext 接口定义中的最后一个方法 getAutowireCapableBeanFactory() 就知道了。

4. ConfigurableListableBeanFactory 也是一个特殊的接口，看图，特殊之处在于它继承了第二层所有的三个接口，而 ApplicationContext 没有。这点之后会用到。

5. 请先不用花时间在其他的接口和类上，先理解我说的这几点就可以了。

### 启动过程分析

第一步，我们肯定要从 ClassPathXmlApplicationContext 的构造方法说起。

```Java
public class ClassPathXmlApplicationContext extends AbstractXmlApplicationContext {
  private Resource[] configResources;

  // 如果已经有 ApplicationContext 并需要配置成父子关系，那么调用这个构造方法
  public ClassPathXmlApplicationContext(ApplicationContext parent) {
    super(parent);
  }
  ...
  public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
      throws BeansException {

    super(parent);
    // 根据提供的路径，处理成配置文件数组(以分号、逗号、空格、tab、换行符分割)
    setConfigLocations(configLocations);
    if (refresh) {
      refresh(); // 核心方法
    }
  }
    ...
}
```
接下来，就是 refresh()，这里简单说下为什么是 refresh()，而不是 init() 这种名字的方法。因为 ApplicationContext 建立起来以后，其实我们是可以通过调用 refresh() 这个方法重建的，refresh() 会将原来的 ApplicationContext 销毁，然后再重新执行一次初始化操作。

往下看，refresh() 方法里面调用了那么多方法，就知道肯定不简单了，请读者先看个大概，细节之后会详细说。

```Java
@Override
public void refresh() throws BeansException, IllegalStateException {
   // 来个锁，不然 refresh() 还没结束，你又来个启动或销毁容器的操作，那不就乱套了嘛
   synchronized (this.startupShutdownMonitor) {

      // 准备工作，记录下容器的启动时间、标记“已启动”状态、处理配置文件中的占位符
      prepareRefresh();

      // 这步比较关键，这步完成后，配置文件就会解析成一个个 Bean 定义，注册到 BeanFactory 中，
      // 当然，这里说的 Bean 还没有初始化，只是配置信息都提取出来了，
      // 注册也只是将这些信息都保存到了注册中心(说到底核心是一个 beanName-> beanDefinition 的 map)
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      // 设置 BeanFactory 的类加载器，添加几个 BeanPostProcessor，手动注册几个特殊的 bean
      // 这块待会会展开说
      prepareBeanFactory(beanFactory);

      try {
         // 【这里需要知道 BeanFactoryPostProcessor 这个知识点，Bean 如果实现了此接口，
         // 那么在容器初始化以后，Spring 会负责调用里面的 postProcessBeanFactory 方法。】

         // 这里是提供给子类的扩展点，到这里的时候，所有的 Bean 都加载、注册完成了，但是都还没有初始化
         // 具体的子类可以在这步的时候添加一些特殊的 BeanFactoryPostProcessor 的实现类或做点什么事
         postProcessBeanFactory(beanFactory);
         // 调用 BeanFactoryPostProcessor 各个实现类的 postProcessBeanFactory(factory) 方法
         invokeBeanFactoryPostProcessors(beanFactory);

         // 注册 BeanPostProcessor 的实现类，注意看和 BeanFactoryPostProcessor 的区别
         // 此接口两个方法: postProcessBeforeInitialization 和 postProcessAfterInitialization
         // 两个方法分别在 Bean 初始化之前和初始化之后得到执行。注意，到这里 Bean 还没初始化
         registerBeanPostProcessors(beanFactory);

         // 初始化当前 ApplicationContext 的 MessageSource，国际化这里就不展开说了，不然没完没了了
         initMessageSource();

         // 初始化当前 ApplicationContext 的事件广播器，这里也不展开了
         initApplicationEventMulticaster();

         // 从方法名就可以知道，典型的模板方法(钩子方法)，
         // 具体的子类可以在这里初始化一些特殊的 Bean（在初始化 singleton beans 之前）
         onRefresh();

         // 注册事件监听器，监听器需要实现 ApplicationListener 接口。这也不是我们的重点，过
         registerListeners();

         // 重点，重点，重点
         // 初始化所有的 singleton beans
         //（lazy-init 的除外）
         finishBeanFactoryInitialization(beanFactory);

         // 最后，广播事件，ApplicationContext 初始化完成
         finishRefresh();
      }

      catch (BeansException ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
                  "cancelling refresh attempt: " + ex);
         }

         // Destroy already created singletons to avoid dangling resources.
         // 销毁已经初始化的 singleton 的 Beans，以免有些 bean 会一直占用资源
         destroyBeans();

         // Reset 'active' flag.
         cancelRefresh(ex);

         // 把异常往外抛
         throw ex;
      }

      finally {
         // Reset common introspection caches in Spring's core, since we
         // might not ever need metadata for singleton beans anymore...
         resetCommonCaches();
      }
   }
}

```
下面，我们开始一步步来肢解这个 refresh() 方法。


#### 创建 Bean 容器前的准备工作

这个比较简单，直接看代码中的几个注释即可。

```Java
protected void prepareRefresh() {
   // 记录启动时间，
   // 将 active 属性设置为 true，closed 属性设置为 false，它们都是 AtomicBoolean 类型
   this.startupDate = System.currentTimeMillis();
   this.closed.set(false);
   this.active.set(true);

   if (logger.isInfoEnabled()) {
      logger.info("Refreshing " + this);
   }

   // Initialize any placeholder property sources in the context environment
   initPropertySources();

   // 校验 xml 配置文件
   getEnvironment().validateRequiredProperties();

   this.earlyApplicationEvents = new LinkedHashSet<ApplicationEvent>();
}
```

#### 创建 Bean 容器，加载并注册 Bean

我们回到 refresh() 方法中的下一行 obtainFreshBeanFactory()。

注意，这个方法是全文最重要的部分之一，这里将会初始化 BeanFactory、加载 Bean、注册 Bean 等等。

当然，这步结束后，Bean 并没有完成初始化。这里指的是 Bean 实例并未在这一步生成。

// AbstractApplicationContext.java

```Java
    /**
     * Tell the subclass to refresh the internal bean factory.
     * @return the fresh BeanFactory instance
     * @see #refreshBeanFactory()
     * @see #getBeanFactory()
     */
    protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
        // 关闭旧的 BeanFactory (如果有)，创建新的 BeanFactory，加载 Bean 定义、注册 Bean 等等
        refreshBeanFactory();
        // 返回刚刚创建的 BeanFactory
        ConfigurableListableBeanFactory beanFactory = getBeanFactory();
        if (logger.isDebugEnabled()) {
            logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
        }
        return beanFactory;
    }


        /**
     * This implementation performs an actual refresh of this context's underlying
     * bean factory, shutting down the previous bean factory (if any) and
     * initializing a fresh bean factory for the next phase of the context's lifecycle.
     */
    @Override
    protected final void refreshBeanFactory() throws BeansException {
        // 如果 ApplicationContext 中已经加载过 BeanFactory 了，销毁所有 Bean，关闭 BeanFactory
        // 注意，应用中 BeanFactory 本来就是可以多个的，这里可不是说应用全局是否有 BeanFactory，而是当前ApplicationContext 是否有 BeanFactory
        if (hasBeanFactory()) {
            destroyBeans();
            closeBeanFactory();
        }
        try {
            // 初始化一个 DefaultListableBeanFactory，为什么用这个，我们马上说。
            DefaultListableBeanFactory beanFactory = createBeanFactory();
            // 用于 BeanFactory 的序列化，我想不部分人应该都用不到
            beanFactory.setSerializationId(getId());

            // 下面这两个方法很重要，别跟丢了，具体细节之后说
            // 设置 BeanFactory 的两个配置属性：是否允许 Bean 覆盖、是否允许循环引用
            customizeBeanFactory(beanFactory);

            // 加载 Bean 到 BeanFactory 中
            loadBeanDefinitions(beanFactory);
            synchronized (this.beanFactoryMonitor) {
                this.beanFactory = beanFactory;
            }
        }
        catch (IOException ex) {
            throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
        }
    }
```

> 看到这里的时候，我觉得读者就应该站在高处看 ApplicationContext 了，ApplicationContext 继承自 BeanFactory，但是它不应该被理解为 BeanFactory 的实现类，而是说其内部持有一个实例化的 BeanFactory（DefaultListableBeanFactory）。以后所有的 BeanFactory 相关的操作其实是委托给这个实例来处理的。

我们说说为什么选择实例化**DefaultListableBeanFactory** ？前面我们说了有个很重要的接口 ConfigurableListableBeanFactory，它实现了 BeanFactory 下面一层的所有三个接口，我把之前的继承图再拿过来大家再仔细看一下：

![示意图](/img/spring-ioc-2.png)

我们可以看到 ConfigurableListableBeanFactory 只有一个实现类 DefaultListableBeanFactory，而且实现类 DefaultListableBeanFactory 还通过实现右边的 AbstractAutowireCapableBeanFactory 通吃了右路。所以结论就是，最底下这个家伙 DefaultListableBeanFactory 基本上是最牛的 BeanFactory 了，这也是为什么这边会使用这个类来实例化的原因。

> 如果你想要在程序运行的时候动态往 Spring IOC 容器注册新的 bean，就会使用到这个类。那我们怎么在运行时获得这个实例呢？
>之前我们说过 ApplicationContext 接口能获取到 AutowireCapableBeanFactory，就是最右上角那个，然后它向下转型就能得到 DefaultListableBeanFactory 了。

在继续往下之前，我们需要先了解 BeanDefinition。**我们说 BeanFactory 是 Bean 容器，那么 Bean 又是什么呢？**

这里的 BeanDefinition 就是我们所说的 Spring 的 Bean，我们自己定义的各个 Bean 其实会转换成一个个 BeanDefinition 存在于 Spring 的 BeanFactory 中。

所以，如果有人问你 Bean 是什么的时候，你要知道 Bean 在代码层面上可以认为是 BeanDefinition 的实例。

> BeanDefinition 中保存了我们的 Bean 信息，比如这个 Bean 指向的是哪个类、是否是单例的、是否懒加载、这个 Bean 依赖了哪些 Bean 等等。

#### BeanDefinition 接口定义

我们来看下 BeanDefinition 的接口定义：

```Java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {

   // 我们可以看到，默认只提供 sington 和 prototype 两种，
   // 很多读者可能知道还有 request, session, globalSession, application, websocket 这几种，
   // 不过，它们属于基于 web 的扩展。
   String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;
   String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;

   // 比较不重要，直接跳过吧
   int ROLE_APPLICATION = 0;
   int ROLE_SUPPORT = 1;
   int ROLE_INFRASTRUCTURE = 2;

   // 设置父 Bean，这里涉及到 bean 继承，不是 java 继承。请参见附录的详细介绍
   // 一句话就是：继承父 Bean 的配置信息而已
   void setParentName(String parentName);

   // 获取父 Bean
   String getParentName();

   // 设置 Bean 的类名称，将来是要通过反射来生成实例的
   void setBeanClassName(String beanClassName);

   // 获取 Bean 的类名称
   String getBeanClassName();


   // 设置 bean 的 scope
   void setScope(String scope);

   String getScope();

   // 设置是否懒加载
   void setLazyInit(boolean lazyInit);

   boolean isLazyInit();

   // 设置该 Bean 依赖的所有的 Bean，注意，这里的依赖不是指属性依赖(如 @Autowire 标记的)，
   // 是 depends-on="" 属性设置的值。
   void setDependsOn(String... dependsOn);

   // 返回该 Bean 的所有依赖
   String[] getDependsOn();

   // 设置该 Bean 是否可以注入到其他 Bean 中，只对根据类型注入有效，
   // 如果根据名称注入，即使这边设置了 false，也是可以的
   void setAutowireCandidate(boolean autowireCandidate);

   // 该 Bean 是否可以注入到其他 Bean 中
   boolean isAutowireCandidate();

   // 主要的。同一接口的多个实现，如果不指定名字的话，Spring 会优先选择设置 primary 为 true 的 bean
   void setPrimary(boolean primary);

   // 是否是 primary 的
   boolean isPrimary();

   // 如果该 Bean 采用工厂方法生成，指定工厂名称。对工厂不熟悉的读者，请参加附录
   // 一句话就是：有些实例不是用反射生成的，而是用工厂模式生成的
   void setFactoryBeanName(String factoryBeanName);
   // 获取工厂名称
   String getFactoryBeanName();
   // 指定工厂类中的 工厂方法名称
   void setFactoryMethodName(String factoryMethodName);
   // 获取工厂类中的 工厂方法名称
   String getFactoryMethodName();

   // 构造器参数
   ConstructorArgumentValues getConstructorArgumentValues();

   // Bean 中的属性值，后面给 bean 注入属性值的时候会说到
   MutablePropertyValues getPropertyValues();

   // 是否 singleton
   boolean isSingleton();

   // 是否 prototype
   boolean isPrototype();

   // 如果这个 Bean 是被设置为 abstract，那么不能实例化，
   // 常用于作为 父bean 用于继承，其实也很少用......
   boolean isAbstract();

   int getRole();
   String getDescription();
   String getResourceDescription();
   BeanDefinition getOriginatingBeanDefinition();
}
```

> 这个 BeanDefinition 其实已经包含很多的信息了，暂时不清楚所有的方法对应什么东西没关系，希望看完本文后读者可以彻底搞清楚里面的所有东西。

> 这里接口虽然那么多，但是没有类似 getInstance() 这种方法来获取我们定义的类的实例，真正的我们定义的类生成的实例到哪里去了呢？别着急，这个要很后面才能讲到。

有了 BeanDefinition 的概念以后，我们再往下看 refreshBeanFactory() 方法中的剩余部分：

```Java
customizeBeanFactory(beanFactory);
loadBeanDefinitions(beanFactory);
```

#### customizeBeanFactory

customizeBeanFactory(beanFactory) 比较简单，就是配置是否允许 BeanDefinition 覆盖、是否允许循环引用。
```Java
    /**
     * Customize the internal bean factory used by this context.
     * Called for each {@link #refresh()} attempt.
     * <p>The default implementation applies this context's
     * {@linkplain #setAllowBeanDefinitionOverriding "allowBeanDefinitionOverriding"}
     * and {@linkplain #setAllowCircularReferences "allowCircularReferences"} settings,
     * if specified. Can be overridden in subclasses to customize any of
     * {@link DefaultListableBeanFactory}'s settings.
     * @param beanFactory the newly created bean factory for this context
     * @see DefaultListableBeanFactory#setAllowBeanDefinitionOverriding
     * @see DefaultListableBeanFactory#setAllowCircularReferences
     * @see DefaultListableBeanFactory#setAllowRawInjectionDespiteWrapping
     * @see DefaultListableBeanFactory#setAllowEagerClassLoading
     */
    protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
        if (this.allowBeanDefinitionOverriding != null) {
            beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
        }
        if (this.allowCircularReferences != null) {
            beanFactory.setAllowCircularReferences(this.allowCircularReferences);
        }
    }
```

BeanDefinition 的覆盖问题可能会有开发者碰到这个坑，就是在配置文件中定义 bean 时使用了相同的 id 或 name，默认情况下，allowBeanDefinitionOverriding 属性为 null，如果在同一配置文件中重复了，会抛错，但是如果不是同一配置文件中，会发生覆盖。

循环引用也很好理解：A 依赖 B，而 B 依赖 A。或 A 依赖 B，B 依赖 C，而 C 依赖 A。

默认情况下，Spring 允许循环依赖，当然如果你在 A 的构造方法中依赖 B，在 B 的构造方法中依赖 A 是不行的。

至于这两个属性怎么配置？我在附录中进行了介绍，尤其对于覆盖问题，很多人都希望禁止出现 Bean 覆盖，可是 Spring 默认是不同文件的时候可以覆盖的。

之后的源码中还会出现这两个属性，读者有个印象就可以了。

#### 加载 Bean: loadBeanDefinitions

接下来是最重要的 loadBeanDefinitions(beanFactory) 方法了，这个方法将根据配置，加载各个 Bean，然后放到 BeanFactory 中。

读取配置的操作在 XmlBeanDefinitionReader 中，其负责加载配置、解析。

```Java
    /**
     * Loads the bean definitions via an XmlBeanDefinitionReader.
     * @see org.springframework.beans.factory.xml.XmlBeanDefinitionReader
     * @see #initBeanDefinitionReader
     * @see #loadBeanDefinitions
     */
    @Override
    protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
        // 给这个 BeanFactory 实例化一个 XmlBeanDefinitionReader
        // Create a new XmlBeanDefinitionReader for the given BeanFactory.
        XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

        // Configure the bean definition reader with this context's
        // resource loading environment.
        beanDefinitionReader.setEnvironment(this.getEnvironment());
        beanDefinitionReader.setResourceLoader(this);
        beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

        // 初始化 BeanDefinitionReader，其实这个是提供给子类覆写的，
        // Allow a subclass to provide custom initialization of the reader,
        // then proceed with actually loading the bean definitions.
        initBeanDefinitionReader(beanDefinitionReader);
        // 重点来了，继续往下
        loadBeanDefinitions(beanDefinitionReader);
    }

```

现在还在这个类中，接下来用刚刚初始化的 Reader 开始来加载 xml 配置，这块代码读者可以选择性跳过，不是很重要。也就是说，下面这个代码块，读者可以很轻松地略过。

```Java
    /**
     * Load the bean definitions with the given XmlBeanDefinitionReader.
     * <p>The lifecycle of the bean factory is handled by the {@link #refreshBeanFactory}
     * method; hence this method is just supposed to load and/or register bean definitions.
     * @param reader the XmlBeanDefinitionReader to use
     * @throws BeansException in case of bean registration errors
     * @throws IOException if the required XML document isn't found
     * @see #refreshBeanFactory
     * @see #getConfigLocations
     * @see #getResources
     * @see #getResourcePatternResolver
     */
    protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
        Resource[] configResources = getConfigResources();
        if (configResources != null) {
            reader.loadBeanDefinitions(configResources);
        }
        String[] configLocations = getConfigLocations();
        if (configLocations != null) {
            reader.loadBeanDefinitions(configLocations);
        }
    }


    @Override
    public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
        Assert.notNull(resources, "Resource array must not be null");
        int counter = 0;
        // 注意这里是个 for 循环，也就是每个文件是一个 resource
        for (Resource resource : resources) {
            counter += loadBeanDefinitions(resource);
        }
        // 最后返回 counter，表示总共加载了多少的 BeanDefinition
        return counter;
    }

        /**
     * Load bean definitions from the specified XML file.
     * @param resource the resource descriptor for the XML file
     * @return the number of bean definitions found
     * @throws BeanDefinitionStoreException in case of loading or parsing errors
     */
    @Override
    public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
        return loadBeanDefinitions(new EncodedResource(resource));
    }


    /**
     * Load bean definitions from the specified XML file.
     * @param encodedResource the resource descriptor for the XML file,
     * allowing to specify an encoding to use for parsing the file
     * @return the number of bean definitions found
     * @throws BeanDefinitionStoreException in case of loading or parsing errors
     */
    public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
        Assert.notNull(encodedResource, "EncodedResource must not be null");
        if (logger.isInfoEnabled()) {
            logger.info("Loading XML bean definitions from " + encodedResource.getResource());
        }
        // 用一个 ThreadLocal 来存放配置文件资源
        Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
        if (currentResources == null) {
            currentResources = new HashSet<EncodedResource>(4);
            this.resourcesCurrentlyBeingLoaded.set(currentResources);
        }
        if (!currentResources.add(encodedResource)) {
            throw new BeanDefinitionStoreException(
                    "Detected cyclic loading of " + encodedResource + " - check your import definitions!");
        }
        try {
            InputStream inputStream = encodedResource.getResource().getInputStream();
            try {
                InputSource inputSource = new InputSource(inputStream);
                if (encodedResource.getEncoding() != null) {
                    inputSource.setEncoding(encodedResource.getEncoding());
                }
                // 核心部分是这里，往下面看
                return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
            }
            finally {
                inputStream.close();
            }
        }
        catch (IOException ex) {
            throw new BeanDefinitionStoreException(
                    "IOException parsing XML document from " + encodedResource.getResource(), ex);
        }
        finally {
            currentResources.remove(encodedResource);
            if (currentResources.isEmpty()) {
                this.resourcesCurrentlyBeingLoaded.remove();
            }
        }
    }

    /**
     * Actually load bean definitions from the specified XML file.
     * @param inputSource the SAX InputSource to read from
     * @param resource the resource descriptor for the XML file
     * @return the number of bean definitions found
     * @throws BeanDefinitionStoreException in case of loading or parsing errors
     * @see #doLoadDocument
     * @see #registerBeanDefinitions
     */
    protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
            throws BeanDefinitionStoreException {
        try {
            // 这里就不看了，将 xml 文件转换为 Document 对象
            Document doc = doLoadDocument(inputSource, resource);
            // 继续
            return registerBeanDefinitions(doc, resource);
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
        catch (SAXParseException ex) {
            throw new XmlBeanDefinitionStoreException(resource.getDescription(),
                    "Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
        }
        catch (SAXException ex) {
            throw new XmlBeanDefinitionStoreException(resource.getDescription(),
                    "XML document from " + resource + " is invalid", ex);
        }
        catch (ParserConfigurationException ex) {
            throw new BeanDefinitionStoreException(resource.getDescription(),
                    "Parser configuration exception parsing XML from " + resource, ex);
        }
        catch (IOException ex) {
            throw new BeanDefinitionStoreException(resource.getDescription(),
                    "IOException parsing XML document from " + resource, ex);
        }
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(resource.getDescription(),
                    "Unexpected exception parsing XML document from " + resource, ex);
        }
    }

    /**
     * Register the bean definitions contained in the given DOM document.
     * Called by {@code loadBeanDefinitions}.
     * <p>Creates a new instance of the parser class and invokes
     * {@code registerBeanDefinitions} on it.
     * @param doc the DOM document
     * @param resource the resource descriptor (for context information)
     * @return the number of bean definitions found
     * @throws BeanDefinitionStoreException in case of parsing errors
     * @see #loadBeanDefinitions
     * @see #setDocumentReaderClass
     * @see BeanDefinitionDocumentReader#registerBeanDefinitions
     */
    // 返回值：返回从当前配置文件加载了多少数量的 Bean
    public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
        BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
        int countBefore = getRegistry().getBeanDefinitionCount();
        // 这里
        documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
        return getRegistry().getBeanDefinitionCount() - countBefore;
    }

    /**
     * This implementation parses bean definitions according to the "spring-beans" XSD
     * (or DTD, historically).
     * <p>Opens a DOM Document; then initializes the default settings
     * specified at the {@code <beans/>} level; then parses the contained bean definitions.
     */
    @Override
    public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
        this.readerContext = readerContext;
        logger.debug("Loading bean definitions");
        Element root = doc.getDocumentElement();
        // 从 xml 根节点开始解析文件
        doRegisterBeanDefinitions(root);
    }
```
经过漫长的链路，一个配置文件终于转换为一颗 DOM 树了，注意，这里指的是其中一个配置文件，不是所有的，读者可以看到上面有个 for 循环的。下面开始从根节点开始解析：

##### doRegisterBeanDefinitions：

```Java
    /**
     * Register each bean definition within the given root {@code <beans/>} element.
     */
    protected void doRegisterBeanDefinitions(Element root) {
        // Any nested <beans> elements will cause recursion in this method. In
        // order to propagate and preserve <beans> default-* attributes correctly,
        // keep track of the current (parent) delegate, which may be null. Create
        // the new (child) delegate with a reference to the parent for fallback purposes,
        // then ultimately reset this.delegate back to its original (parent) reference.
        // this behavior emulates a stack of delegates without actually necessitating one.
        // 我们看名字就知道，BeanDefinitionParserDelegate 必定是一个重要的类，它负责解析 Bean 定义，
        // 这里为什么要定义一个 parent? 看到后面就知道了，是递归问题，
        // 因为 <beans /> 内部是可以定义 <beans /> 的，所以这个方法的 root 其实不一定就是 xml 的根节点，也可以是嵌套在里面的 <beans /> 节点，从源码分析的角度，我们当做根节点就好了
        BeanDefinitionParserDelegate parent = this.delegate;
        this.delegate = createDelegate(getReaderContext(), root, parent);

        if (this.delegate.isDefaultNamespace(root)) {
            // 这块说的是根节点 <beans ... profile="dev" /> 中的 profile 是否是当前环境需要的，
            // 如果当前环境配置的 profile 不包含此 profile，那就直接 return 了，不对此 <beans /> 解析
            // 不熟悉 profile 为何物，不熟悉怎么配置 profile 读者的请移步附录区
            String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
            if (StringUtils.hasText(profileSpec)) {
                String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
                        profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
                if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                    if (logger.isInfoEnabled()) {
                        logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
                                "] not matching: " + getReaderContext().getResource());
                    }
                    return;
                }
            }
        }

        preProcessXml(root);// 钩子
        // 往下看
        parseBeanDefinitions(root, this.delegate);
        postProcessXml(root);// 钩子

        this.delegate = parent;
    }
```
preProcessXml(root) 和 postProcessXml(root) 是给子类用的钩子方法，鉴于没有被使用到，也不是我们的重点，我们直接跳过。

接下来，看核心解析方法 parseBeanDefinitions(root, this.delegate) :

```Java
    /**
     * Parse the elements at the root level in the document:
     * "import", "alias", "bean".
     * @param root the DOM root element of the document
     */
    // default namespace 涉及到的就四个标签 <import />、<alias />、<bean /> 和 <beans />，其他的属于 custom 的
    protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
        if (delegate.isDefaultNamespace(root)) {
            NodeList nl = root.getChildNodes();
            for (int i = 0; i < nl.getLength(); i++) {
                Node node = nl.item(i);
                if (node instanceof Element) {
                    Element ele = (Element) node;
                    if (delegate.isDefaultNamespace(ele)) {
                        // 解析 default namespace 下面的几个元素
                        parseDefaultElement(ele, delegate);
                    }
                    else {
                        // 解析其他 namespace 的元素
                        delegate.parseCustomElement(ele);
                    }
                }
            }
        }
        else {
            delegate.parseCustomElement(root);
        }
    }
```

从上面的代码，我们可以看到，对于每个配置来说，分别进入到 parseDefaultElement(ele, delegate); 和 delegate.parseCustomElement(ele); 这两个分支了。

parseDefaultElement(ele, delegate) 代表解析的节点是 <import />、<alias />、<bean />、<beans /> 这几个。

> 这里的四个标签之所以是 default 的，是因为它们是处于这个 namespace 下定义的：

> ```xml
 http://www.springframework.org/schema/beans
 ```
> 又到初学者科普时间，不熟悉 namespace 的读者请看下面贴出来的 xml，这里的第二行 xmlns 就是咯。

> ```xml
 <beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="
            http://www.springframework.org/schema/beans
          http://www.springframework.org/schema/beans/spring-beans.xsd"
       default-autowire="byName">
 ```

> 而对于其他的标签，将进入到 delegate.parseCustomElement(element) 这个分支。如我们经常会使用到的 <mvc />、<task />、<context />、<aop />等。

> 这些属于扩展，如果需要使用上面这些 ”非 default“ 标签，那么上面的 xml 头部的地方也要引入相应的 namespace 和 .xsd 文件的路径，如下所示。同时代码中需要提供相应的 parser 来解析，如 MvcNamespaceHandler、TaskNamespaceHandler、ContextNamespaceHandler、AopNamespaceHandler 等。

> 假如读者想分析 <context:property-placeholder location="classpath:xx.properties" /> 的实现原理，就应该到 ContextNamespaceHandler 中找答案。

> ```xml
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns="http://www.springframework.org/schema/beans"
      xmlns:context="http://www.springframework.org/schema/context"
      xmlns:mvc="http://www.springframework.org/schema/mvc"
      xsi:schemaLocation="
           http://www.springframework.org/schema/beans 
           http://www.springframework.org/schema/beans/spring-beans.xsd
           http://www.springframework.org/schema/context
           http://www.springframework.org/schema/context/spring-context.xsd
           http://www.springframework.org/schema/mvc   
           http://www.springframework.org/schema/mvc/spring-mvc.xsd  
       "
      default-autowire="byName">
```

回过神来，看看处理 default 标签的方法：

```Java
    private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
        if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
            //// 处理 <import /> 标签
            importBeanDefinitionResource(ele);
        }
        else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
            // 处理 <alias /> 标签定义
            // <alias name="fromName" alias="toName"/>
            processAliasRegistration(ele);
        }
        else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
            // 处理 <bean /> 标签定义，这也算是我们的重点吧
            processBeanDefinition(ele, delegate);
        }
        else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
            // 如果碰到的是嵌套的 <beans /> 标签，需要递归
            // recurse
            doRegisterBeanDefinitions(ele);
        }
    }
```

下面是 processBeanDefinition 解析 <bean /> 标签：


```Java
    /**
     * Process the given bean element, parsing the bean definition
     * and registering it with the registry.
     */
    protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
        // 将 <bean /> 节点中的信息提取出来，然后封装到一个 BeanDefinitionHolder 中，细节往下看
        BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
        if (bdHolder != null) {
            bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
            try {
                // Register the final decorated instance.
                BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
            }
            catch (BeanDefinitionStoreException ex) {
                getReaderContext().error("Failed to register bean definition with name '" +
                        bdHolder.getBeanName() + "'", ele, ex);
            }
            // Send registration event.
            getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
        }
    }

```

继续往下看怎么解析之前，我们先看下 <bean /> 标签中可以定义哪些属性：

|隔离级别   | |
| :--: | :--: | 
|class|  类的全限定名|
|name|    可指定 id、name(用逗号、分号、空格分隔)|
|scope   |作用域|
|constructor arguments|   指定构造参数|
|properties  |设置属性的值|
|autowiring mode |no(默认值)、byName、byType、 constructor|
|lazy-initialization mode |   是否懒加载(如果被非懒加载的bean依赖了那么其实也就不能懒加载了)|
|initialization method |  bean 属性设置完成后，会调用这个方法|
|destruction method|  bean 销毁后的回调方法|
