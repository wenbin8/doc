# Spring-IOC

## IOC与DI

**IOC（Inversion of Control）控制反转：所谓控制反转，就是把原先我们代码里面需要实现的对象创建、依赖的代码，反转给容器来帮忙实现。那么必然我们需要创建一个容器，同时需要一种描述来让容器知道需要创建的对象与对象的关系。**这个描述最具体的表现就是我们看到的配置文件。

**DI（Dependency Injection）依赖注入：就是指对象被动接受依赖类而不是自己主动去找，换句话说就是指对象不是从容器中查找它依赖的类了，而是在容器实例化对象的时候主动将它依赖的类注入给它。**

先从我们自己设计这样一个视角来考虑：

1，对象和对象的关系怎么样表示？

可以用xml、properties文件等语义话配置文件。

2，描述对象关系的文件存放在哪里？

可能是classpath、fileSystem，或者是URL网络资源，servletContext等。有了配置文件，还需要对配置文件解析。

3，不同的配置文件对对象的描述不一样，如：标准的、自定义的声明方式，如何统一？

在内部需要有一个统一的关于对象的定义，所有外部的描述都必须转化成统一的描述定义。

4，如何对不同的配置文件进行解析？

需要对不同的配置文件语法，采用不同的解析器。

## Spring 核心容器类图

### BeanFactory

Spring Bean的创建是典型的工厂模式，这一系列的Bean工厂，也即IOC容器为开发者管理对象间的依赖关系提供了很多便利和基础服务，在Spring中又许多的IOC容器的实现供用户选择和使用，其相互关系如下：

![image-20190923115938948](/Users/dongwenbin/github/doc/Spring/assets/image-20190923115938948.png)

其中BeanFacotry作为最顶层的一个接口类，它定义了IOC容器的基本功能规范，BeanFactory有三个重要的子类：ListAbleBeanFactory、HierarchicalBeanFactory和AutowireCapableBeanFactory。但是从类图中我们可以发现最终的默认实现是DefaultListableBeanFactory，它实现了所有的接口。那为何要定义这么多层次的接口呢？查阅这些接口的源码和说明发现，每个接口都有它的使用场合，它主要是为了区分在Spring内部在操作过程中对象的传递和转化过程时，对对象的数据访问所做的限制。例如LIstableBeanFactory接口表示这些Bean是可列表化的，而HierarchcalBeanFacotry表示的是这些Bean是有继承关系的，也就是每个Bean有可能有父Bean。AutowireCapableBeanFactory接口定义Bean自动装配规则。这三个接口共同定义了Bean的集合、Bean之间的关系、以及Bean行为。

最基本的IOC容器接口BeanFactory，来看一下它的源码：

```java
public interface BeanFactory {

   //对FactoryBean的转义定义，因为如果使用bean的名字检索FactoryBean得到的对象是工厂生成的对象，
   //如果需要得到工厂本身，需要转义
   String FACTORY_BEAN_PREFIX = "&";

   //根据bean的名字，获取在IOC容器中得到bean实例
   Object getBean(String name) throws BeansException;

   //根据bean的名字和Class类型来得到bean实例，增加了类型安全验证机制。
   <T> T getBean(String name, @Nullable Class<T> requiredType) throws BeansException;
   Object getBean(String name, Object... args) throws BeansException;
   <T> T getBean(Class<T> requiredType) throws BeansException;
   <T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

   //提供对bean的检索，看看是否在IOC容器有这个名字的bean
   boolean containsBean(String name);

   //根据bean名字得到bean实例，并同时判断这个bean是不是单例
   boolean isSingleton(String name) throws NoSuchBeanDefinitionException;
  
	 //根据bean名字得到bean实例，并同时判断这个bean是不是原型
   boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

   boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;

   boolean isTypeMatch(String name, @Nullable Class<?> typeToMatch) throws NoSuchBeanDefinitionException;

   //得到bean实例的Class类型
   @Nullable
   Class<?> getType(String name) throws NoSuchBeanDefinitionException;

   //得到bean的别名，如果根据别名检索，那么其原名也会被检索出来
   String[] getAliases(String name);

}
```

在BeanFactory里只对IOC容器的基本行为了作了定义，根本不关系你的Bean是如何定义怎样加载的。正如我们只关心工厂里得到什么的产品对象，至于工厂是怎么生产这些对象，这个基本的接口不关心。

​	而要知道工厂是如何生产对象的，我们需要看具体的IOC容器实现，Spring提供了许多IOC容器的实现。比如：GenericApplicationContext、ClasspathXmlApplicationContext等。

​	ApplicationContext是Spring提供的一个高级IOC容器，它除了能够提供IOC容器的基本功能外，还为用户提供了一下附加服务。从ApplicationContext接口实现，总结其特点：

1. 支持信息源，可以实现国际化。（实现MessageSource接口）
2. 访问资源。（实现ResourcePatternResolver接口）
3. 支持应用事件。（实现ApplicationEventPublisher接口）

### BeanDefinition

​	Spring IOC容器管理了我们定义的各种Bean对象以及相互的关系。Bean对象在Spring实现中是以BeanDefinition来描述的，无论是XML、注解、还是java配置类。最终都会被转换成BeanDefinition并保存在IOC容器中。

其继承体系如下：

![image-20190923122013684](/Users/dongwenbin/github/doc/Spring/assets/image-20190923122013684.png)

### BeanDefinitionReader

Bean的解析过程非常复杂，功能被分的很细，因为这里需要被扩展的地方很多，必须保证有足够的灵活性，以应对可能的变化。Bean的解析主要就是对Spring配置文件的解析。这个解析过程主要通过BeanDefinitionReader来完成。

Spring中BeanDefinitionReader的类结构图：

![image-20190923123332675](/Users/dongwenbin/github/doc/Spring/assets/image-20190923123332675.png)

抛开一切细节总结IOC一下加载流程：ResourceLoader将资源路径转换为resouce对象.BeanDefinitionReader使用resouce对象将配置转换为definition对象。并注册到DefaultListableBeanFactory中的beanDefinitionMap属性中。DI根据beanDefinitionMap中的BeanDefinition对象中记录的bean信息进行实例化bean。

## 基于XML的IOC容器初始化

**IOC容器的初始化包括BeanDefinition的Resource定位、加载和注册这三个基本的过程。**我们以ApplicationContext为例，ApplicationContext系列容器也许是我们最熟悉的，因为Web项目中适用的XmlWebApplicationContext就属于这个继承体系，还有ClasspathXmlApplicationContext等，其继承体系如下图：

![image-20190923132630109](/Users/dongwenbin/github/doc/Spring/assets/image-20190923132630109.png)

ApplicationContext允许上下文嵌套，通过保持父上下文可以维持一个上下文体系。对于Bean的查找可以在这个上下文体系中发生，首先检查当前上下文，其次是父上下文，逐级向上，这样为不同的Spring应用提供了一个共享的Bean定义环境。

### 源码分析

#### 入口

```java
ApplicationContext app = new ClassPathXmlApplicationContext("application.xml");
```

构造函数：

```java
public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
   this(new String[] {configLocation}, true, null);
}
```

实际调用的构造函数：

```java
public ClassPathXmlApplicationContext(
      String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
      throws BeansException {

   super(parent);
   setConfigLocations(configLocations);
   if (refresh) {
      refresh();
   }
}
```

还有像AnnotationConfigApplicationContext、FileSystemXmlApplicationContext、XmlWebApplicationContext等都继承自父容器AbstractApplicationContext主要用到了装饰器模式和策略模式，**最终都是调用refresh()方法。**

#### 获得配置路径

通过分析ClassPathXmlApplicationContext的源代码可以知道，在创建ClassPathXmlApplicationContext容器时，构造方法做一下两项重要工作：

1，调用父类容器的构造老方法```super(parent)```为容器设置好Bean资源加载器。

2，在调用AbstractRefreshableConfigApplicationContext的setConfigLocations(configLoactions)方法设置Bean配置信息的定位路径。

​	通过追踪ClassPathXmlApplicationContext的继承体系，发现其父类的父类AbstractApplicationContext中初始化IOC容器所做的主要源码如下：

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
      implements ConfigurableApplicationContext {
	//静态初始化块，在整个容器创建过程中只执行一次
	static {
		// Eagerly load the ContextClosedEvent class to avoid weird classloader issues
		// on application shutdown in WebLogic 8.1. (Reported by Dustin Woods.)
		//为了避免应用程序在Weblogic8.1关闭时出现类加载异常加载问题，加载IoC容
		//器关闭事件(ContextClosedEvent)类
		ContextClosedEvent.class.getName();
	}
	
	public AbstractApplicationContext(@Nullable ApplicationContext parent) {
		this();
		setParent(parent);
	}
	
	public AbstractApplicationContext() {
		this.resourcePatternResolver = getResourcePatternResolver();
	}
	
	//获取一个Spring Source的加载器用于读入Spring Bean定义资源文件
	protected ResourcePatternResolver getResourcePatternResolver() {
		//AbstractApplicationContext继承DefaultResourceLoader，因此也是一个资源加载器
		//Spring资源加载器，其getResource(String location)方法用于载入资源
		return new PathMatchingResourcePatternResolver(this);
	}
	...
}
```

AbstractApplicationContext的默认构造方法中又调用PathMatchingResourceResolver的构造方法创建Spring资源加载器：

```java
public PathMatchingResourcePatternResolver(ResourceLoader resourceLoader) {
   Assert.notNull(resourceLoader, "ResourceLoader must not be null");
   //设置Spring的资源加载器
   this.resourceLoader = resourceLoader;
}
```

在设置容器的资源加载器之后，接下来ClassPathXmlApplicationContext执行setConfigLocations()方法通过调用其父类AbstractRefreshableConfigApplicationContext的方法捷星对Bean配置信息的定位，该方法源码如下：

```java
/**
 * Set the config locations for this application context in init-param style,
 * i.e. with distinct locations separated by commas, semicolons or whitespace.
 * <p>If not set, the implementation may use a default as appropriate.
 */
//处理单个资源文件路径为一个字符串的情况
public void setConfigLocation(String location) {
   //String CONFIG_LOCATION_DELIMITERS = ",; /t/n";
   //即多个资源文件路径之间用” ,; \t\n”分隔，解析成数组形式
   setConfigLocations(StringUtils.tokenizeToStringArray(location, CONFIG_LOCATION_DELIMITERS));
}

/**
 * Set the config locations for this application context.
 * <p>If not set, the implementation may use a default as appropriate.
 */
//解析Bean定义资源文件的路径，处理多个资源文件字符串数组
public void setConfigLocations(@Nullable String... locations) {
   if (locations != null) {
      Assert.noNullElements(locations, "Config locations must not be null");
      this.configLocations = new String[locations.length];
      for (int i = 0; i < locations.length; i++) {
         // resolvePath为同一个类中将字符串解析为路径的方法
         this.configLocations[i] = resolvePath(locations[i]).trim();
      }
   }
   else {
      this.configLocations = null;
   }
}
```

通过这两个方法的源码我们可以看出，我们既可以使用一个字符串来配置多个 Spring Bean 配置信息， 也可以使用字符串数组，即下面两种方式都是可以的:

```java
ClassPathResource res = new ClassPathResource("a.xml,b.xml");			
```

多个资源文件路径之间可以是用” , ; \t\n”等分隔。

```java
ClassPathResource res =new ClassPathResource(new String[]{"a.xml","b.xml"});
```

至此，**SpringIOC 容器在初始化时将配置的 Bean 配置信息定位为 Spring 封装的 Resource。**

#### 开始启动

Spring IOC 容器对Bean配置资源的载入是从refresh()函数开始的，refresh()是一个模板方法，规定了IOC容器的启动流程，有些逻辑要交给其子类去实现。它对Bean配置资源进行载入ClasspathXmlApplicationContext通过调用其父类AbstractApplicationContext的refresh()函数启动这个IOC容器对Bean定义的载入过程，现在我们来详细看看refresh()中的逻辑处理：

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
      // Prepare this context for refreshing.
      //1、调用容器准备刷新的方法，获取容器的当时时间，同时给容器设置同步标识
      prepareRefresh();

      // Tell the subclass to refresh the internal bean factory.
      //2、告诉子类启动refreshBeanFactory()方法，Bean定义资源文件的载入从
      //子类的refreshBeanFactory()方法启动
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      // Prepare the bean factory for use in this context.
      //3、为BeanFactory配置容器特性，例如类加载器、事件处理器等
      prepareBeanFactory(beanFactory);

      try {
         // Allows post-processing of the bean factory in context subclasses.
         //4、为容器的某些子类指定特殊的BeanPost事件处理器
         postProcessBeanFactory(beanFactory);

         // Invoke factory processors registered as beans in the context.
         //5、调用所有注册的BeanFactoryPostProcessor的Bean
         invokeBeanFactoryPostProcessors(beanFactory);

         // Register bean processors that intercept bean creation.
         //6、为BeanFactory注册BeanPost事件处理器.
         //BeanPostProcessor是Bean后置处理器，用于监听容器触发的事件
         registerBeanPostProcessors(beanFactory);

         // Initialize message source for this context.
         //7、初始化信息源，和国际化相关.
         initMessageSource();

         // Initialize event multicaster for this context.
         //8、初始化容器事件传播器.
         initApplicationEventMulticaster();

         // Initialize other special beans in specific context subclasses.
         //9、调用子类的某些特殊Bean初始化方法
         onRefresh();

         // Check for listener beans and register them.
         //10、为事件传播器注册事件监听器.
         registerListeners();

         // Instantiate all remaining (non-lazy-init) singletons.
         //11、初始化所有剩余的单例Bean
         finishBeanFactoryInitialization(beanFactory);

         // Last step: publish corresponding event.
         //12、初始化容器的生命周期事件处理器，并发布容器的生命周期事件
         finishRefresh();
      }

      catch (BeansException ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
                  "cancelling refresh attempt: " + ex);
         }

         // Destroy already created singletons to avoid dangling resources.
         //13、销毁已创建的Bean
         destroyBeans();

         // Reset 'active' flag.
         //14、取消refresh操作，重置容器的同步标识。
         cancelRefresh(ex);

         // Propagate exception to caller.
         throw ex;
      }

      finally {
         // Reset common introspection caches in Spring's core, since we
         // might not ever need metadata for singleton beans anymore...
         //15、重设公共缓存
         resetCommonCaches();
      }
   }
}
```

refresh()方法主要为IOC容器Bean的生命周期管理提供条件，Spring IOC容器载入Bean配置信息从其子类容器的refreshBeanFactory()方法启动，所以真个refresh()中```ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();```这句以后的代码都是注册容器信息源和声明周期事件，我们前面说的载入就是从这句代码开始启动。

refresh()方法的主要作用是：在创建IOC容器前，如果已经有容器存在，则需要把已有的容器销毁和关闭，以保证在refresh之后使用的是新建立起来的IOC容器。它类似于对IOC容器的重启，在新建立好的容器中对容器进行初始化，对Bean配置资源进行载入。

#### 创建容器

obtainFreshBeanFactory()方法调用子类的refreshBeanFactory()方法，启动容器载入Bean配置信息过程，代码如下：

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
   //这里使用了委派设计模式，父类定义了抽象的refreshBeanFactory()方法，具体实现调用子类容器的refreshBeanFactory()方法
   refreshBeanFactory();
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();
   if (logger.isDebugEnabled()) {
      logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
   }
   return beanFactory;
}
```

AbstractApplicationContext类中只抽象定义了refreshBeanFactory()方法，容器真正调用的是其子类AbstractRefreshableApplicationContext实现的refreshBeanFactory()方法，方法的源码如下：

```java
@Override
protected final void refreshBeanFactory() throws BeansException {
   //如果已经有容器，销毁容器中的bean，关闭容器
   if (hasBeanFactory()) {
      destroyBeans();
      closeBeanFactory();
   }
   try {
      //创建IOC容器
      DefaultListableBeanFactory beanFactory = createBeanFactory();
      beanFactory.setSerializationId(getId());
      //对IOC容器进行定制化，如设置启动参数，开启注解的自动装配等
      customizeBeanFactory(beanFactory);
      //调用载入Bean定义的方法，主要这里又使用了一个委派模式，在当前类中只定义了抽象的loadBeanDefinitions方法，具体的实现调用子类容器
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

在这个方法中，先判断BeanFactory是否存在，如果存在则先销毁beans并关闭beanFactory，接着创建DefaultListableBeanFactory,并调用loadBeanDefinitions(beanFactory)装载bean定义。

#### 载入配置路径

AbstractRefreshableApplicationContext中只定义了抽象的loadBeanDefinitions方法，容器真正调用的是其子类AbstractXmlApplicationContext对该方法的实现，AbstractXmlApplicationContext的主要代码如下：

```java
public abstract class AbstractXmlApplicationContext extends AbstractRefreshableConfigApplicationContext {

  ...
	
	//实现父类抽象的载入Bean定义方法
	@Override
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// Create a new XmlBeanDefinitionReader for the given BeanFactory.
		//创建XmlBeanDefinitionReader，即创建Bean读取器，并通过回调设置到容器中去，容  器使用该读取器读取Bean定义资源
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		// Configure the bean definition reader with this context's
		// resource loading environment.
		//为Bean读取器设置Spring资源加载器，AbstractXmlApplicationContext的
		//祖先父类AbstractApplicationContext继承DefaultResourceLoader，因此，容器本身也是一个资源加载器
		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		//为Bean读取器设置SAX xml解析器
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// Allow a subclass to provide custom initialization of the reader,
		// then proceed with actually loading the bean definitions.
		//当Bean读取器读取Bean定义的Xml资源文件时，启用Xml的校验机制
		initBeanDefinitionReader(beanDefinitionReader);
		//Bean读取器真正实现加载的方法
		loadBeanDefinitions(beanDefinitionReader);
	}

	protected void initBeanDefinitionReader(XmlBeanDefinitionReader reader) {
		reader.setValidating(this.validating);
	}

	//Xml Bean读取器加载Bean定义资源
	protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
		//获取Bean定义资源的定位
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
			//Xml Bean读取器调用其父类AbstractBeanDefinitionReader读取定位
			//的Bean定义资源
			reader.loadBeanDefinitions(configResources);
		}
		//如果子类中获取的Bean定义资源定位为空，则获取FileSystemXmlApplicationContext构造方法中setConfigLocations方法设置的资源
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			//Xml Bean读取器调用其父类AbstractBeanDefinitionReader读取定位
			//的Bean定义资源
			reader.loadBeanDefinitions(configLocations);
		}
	}

	//这里又使用了一个委托模式，调用子类的获取Bean定义资源定位的方法
	//该方法在ClassPathXmlApplicationContext中进行实现，对于我们
	//举例分析源码的FileSystemXmlApplicationContext没有使用该方法
	@Nullable
	protected Resource[] getConfigResources() {
		return null;
	}

}

```

以XmlBean读取器的其中一种策略XmlBeanDefinitionReader为例。XmlBeanDefinitionReader调用其父类AbstractBeanDefinitionReader的reader.loadBeanDefinitions()方法读取配置资源。

由于我们使用ClassPathXmlApplictionContext作为例子分析，因此getConfigResources()的返回值为null，因此程序执行```reader.loadBeanDefinitions(configLocations)```分支；

#### 分配路径处理策略

在XmlBeanDefinitionReader的抽象父类AbstractBeanDefinitionReader中定义了载入过程。

AbstractBeanDefinitionReader的loadBeandefinitions()方法源码如下：

```java
//重载方法，调用下面的loadBeanDefinitions(String, Set<Resource>);方法
@Override
public int loadBeanDefinitions(String location) throws BeanDefinitionStoreException {
   return loadBeanDefinitions(location, null);
}

public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
   //获取在IoC容器初始化过程中设置的资源加载器
   ResourceLoader resourceLoader = getResourceLoader();
   if (resourceLoader == null) {
      throw new BeanDefinitionStoreException(
            "Cannot import bean definitions from location [" + location + "]: no ResourceLoader available");
   }

   if (resourceLoader instanceof ResourcePatternResolver) {
      // Resource pattern matching available.
      try {
         //将指定位置的Bean定义资源文件解析为Spring IOC容器封装的资源
         //加载多个指定位置的Bean定义资源文件
         Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
         //委派调用其子类XmlBeanDefinitionReader的方法，实现加载功能
         int loadCount = loadBeanDefinitions(resources);
         if (actualResources != null) {
            for (Resource resource : resources) {
               actualResources.add(resource);
            }
         }
         if (logger.isDebugEnabled()) {
            logger.debug("Loaded " + loadCount + " bean definitions from location pattern [" + location + "]");
         }
         return loadCount;
      }
      catch (IOException ex) {
         throw new BeanDefinitionStoreException(
               "Could not resolve bean definition resource pattern [" + location + "]", ex);
      }
   }
   else {
      // Can only load single resources by absolute URL.
      //将指定位置的Bean定义资源文件解析为Spring IOC容器封装的资源
      //加载单个指定位置的Bean定义资源文件
      Resource resource = resourceLoader.getResource(location);
      //委派调用其子类XmlBeanDefinitionReader的方法，实现加载功能
      int loadCount = loadBeanDefinitions(resource);
      if (actualResources != null) {
         actualResources.add(resource);
      }
      if (logger.isDebugEnabled()) {
         logger.debug("Loaded " + loadCount + " bean definitions from location [" + location + "]");
      }
      return loadCount;
   }
}

//重载方法，调用loadBeanDefinitions(String);
@Override
public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
   Assert.notNull(locations, "Location array must not be null");
   int counter = 0;
   for (String location : locations) {
      counter += loadBeanDefinitions(location);
   }
   return counter;
}
```

AbstractRefreshableConfigApplicationContext的loadBeanDefinitions(Resource... resources)方法实际上是调用AbstractBeanDefinitionReader的loadBeanDefinitions()方法。从对AbstractBeanDefinitionReader的loadBeanDefinitions()方法源码分析可以看出该方法就做了两件事：

首先，调用资源加载器的获取资源方法resourceLoader.getResource(location)，获取到要加载的资源。其次，真正执行加载功能是其子类XmlBeanDefinitionReader的loadBeanDefinitions()方法。在loadBeanDefinitions()方法中调用了AbstractApplictionContext的getResources()方法，跟进去后发现getResources()方法其实定义在ResourcePatternResolver中，此时，我们有必要来看一下ResourcePatternResolver的圈类图：

![image-20190923145622894](/Users/dongwenbin/github/doc/Spring/assets/image-20190923145622894.png)

从上面可以看到ResourceLoder与ApplicationContext的继承关系，可以看出其实际调用的是DefaultResourceLoader中的getSource()方法定位Resource，因为ClassPathXmlApplicationContext本省就的DefaultResourceLoader的实现类，所以此时又回到了ClassPathXmlApplicationContext中。

#### 解析配置文件路径

XmlBeanDefinitionReader通过调用ClassPathXmlApplicationContext的父类DefaultResourceLoader的getResource()方法获取要加载的资源，其源码如下：

```java
//获取Resource的具体实现方法
@Override
public Resource getResource(String location) {
   Assert.notNull(location, "Location must not be null");

   for (ProtocolResolver protocolResolver : this.protocolResolvers) {
      Resource resource = protocolResolver.resolve(location, this);
      if (resource != null) {
         return resource;
      }
   }
   //如果是类路径的方式，那需要使用ClassPathResource 来得到bean 文件的资源对象
   if (location.startsWith("/")) {
      return getResourceByPath(location);
   }
   else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
      return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
   }
   else {
      try {
         // Try to parse the location as a URL...
         // 如果是URL 方式，使用UrlResource 作为bean 文件的资源对象
         URL url = new URL(location);
         return (ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
      }
      catch (MalformedURLException ex) {
         // No URL -> resolve as resource path.
         //如果既不是classpath标识，又不是URL标识的Resource定位，则调用
         //容器本身的getResourceByPath方法获取Resource
         return getResourceByPath(location);
      }
   }
}
```

DefalutResourceLoder提供了getResourceByPath()方法的实现，就是为了处理既不是classpath标识，又不是URL标识的Resource定位这种情况。

```java
protected Resource getResourceByPath(String path) {
   return new ClassPathContextResource(path, getClassLoader());
}
```

在ClassPathResource中完成了对这个路径的解析。这样，就可以从类路径上对IOC配置文件进行加载，当然我们可以按照这个逻辑从任何地方加载，在Spring中我们看到它提供的各种资源抽象，比如：ClassPathResource、URLResource、FileSystemResource等来供我们使用。上面我们看到的是定位Reource的一个过程，而这只是加载过程的一部分。例如：FileSystemXmlApplicationContext容器就重写了getResourceByPath()方法：

```java
@Override
protected Resource getResourceByPath(String path) {
   if (path.startsWith("/")) {
      path = path.substring(1);
   }
   //这里使用文件系统资源对象来定义bean 文件
   return new FileSystemResource(path);
}
```

通过子类的覆盖，巧妙地完成了将类路径变为文件路径的转换。

#### 开始读取配置内容