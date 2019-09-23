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

继续回到XmlBeanDefinitionReader的loadBeanDefinitions(Resource...) 方法看到代表bean文件的资源定义以后载入过程。

```java
@Override
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
   //将读入的XML资源进行特殊编码处理
   return loadBeanDefinitions(new EncodedResource(resource));
}


//这里是载入XML形式Bean定义资源文件方法
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
   Assert.notNull(encodedResource, "EncodedResource must not be null");
   if (logger.isInfoEnabled()) {
      logger.info("Loading XML bean definitions from " + encodedResource.getResource());
   }

   Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
   if (currentResources == null) {
      currentResources = new HashSet<>(4);
      this.resourcesCurrentlyBeingLoaded.set(currentResources);
   }
   if (!currentResources.add(encodedResource)) {
      throw new BeanDefinitionStoreException(
            "Detected cyclic loading of " + encodedResource + " - check your import definitions!");
   }
   try {
      //将资源文件转为InputStream的IO流
      InputStream inputStream = encodedResource.getResource().getInputStream();
      try {
         //从InputStream中得到XML的解析源
         InputSource inputSource = new InputSource(inputStream);
         if (encodedResource.getEncoding() != null) {
            inputSource.setEncoding(encodedResource.getEncoding());
         }
         //这里是具体的读取过程
         return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
      }
      finally {
         //关闭从Resource中得到的IO流
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
  
  ...
    
	//从特定XML文件中实际载入Bean定义资源的方法
	protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
			//将XML文件转换为DOM对象，解析过程由documentLoader实现
			Document doc = doLoadDocument(inputSource, resource);
			//这里是启动对Bean定义解析的详细过程，该解析过程会用到Spring的Bean配置规则
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
}
```

通过源码分析，载入Bean配置信息的最后一步是将Bean配置信息转换为Document对象，该过程由documentLoader()方法实现。

#### 准备文档对象

DocumetLoader将Bean配置资源转换成Document对象的源码如下：

```java
//使用标准的JAXP将载入的Bean定义资源转换成document对象
@Override
public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
      ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {

   //创建文件解析器工厂
   DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
   if (logger.isDebugEnabled()) {
      logger.debug("Using JAXP provider [" + factory.getClass().getName() + "]");
   }
   //创建文档解析器
   DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
   //解析Spring的Bean定义资源
   return builder.parse(inputSource);
}


protected DocumentBuilderFactory createDocumentBuilderFactory(int validationMode, boolean namespaceAware)
      throws ParserConfigurationException {

   //创建文档解析工厂
   DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
   factory.setNamespaceAware(namespaceAware);

   //设置解析XML的校验
   if (validationMode != XmlValidationModeDetector.VALIDATION_NONE) {
      factory.setValidating(true);
      if (validationMode == XmlValidationModeDetector.VALIDATION_XSD) {
         // Enforce namespace aware for XSD...
         factory.setNamespaceAware(true);
         try {
            factory.setAttribute(SCHEMA_LANGUAGE_ATTRIBUTE, XSD_SCHEMA_LANGUAGE);
         }
         catch (IllegalArgumentException ex) {
            ParserConfigurationException pcex = new ParserConfigurationException(
                  "Unable to validate using XSD: Your JAXP provider [" + factory +
                  "] does not support XML Schema. Are you running on Java 1.4 with Apache Crimson? " +
                  "Upgrade to Apache Xerces (or Java 1.5) for full XSD support.");
            pcex.initCause(ex);
            throw pcex;
         }
      }
   }

   return factory;
}
```

上面的解析过程是调用JavaEE标准的JAXP标准进行处理。至此Spring IOC容器根据定位的Bean配置信息。将其加载读入并转换称为Document对象的过程完成。接下来继续分析Spring IOC容器将载入的Bean配置信息转换为Document对象之后，是如何将其解析为Spring IOC管理的Bean对象并将其注册到容器中的。

#### 分配解析策略

XmlBeanDefinitionReader类中的doLoadBeanDefinition()方法是从特定XML文件中实际载入Bean配置资源的方法，该方法在载入Bean配置资源之后将其转换为Document对象，接下来调用registerBeanDefinitions()启动Spring IOC容器对Bean定义的解析过程，registerBeanDefinitions()方法源码如下：

```java
//按照Spring的Bean语义要求将Bean定义资源解析并转换为容器内部数据结构
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
   //得到BeanDefinitionDocumentReader来对xml格式的BeanDefinition解析
   BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
   //获得容器中注册的Bean数量
   int countBefore = getRegistry().getBeanDefinitionCount();
   //解析过程入口，这里使用了委派模式，BeanDefinitionDocumentReader只是个接口,
   //具体的解析实现过程有实现类DefaultBeanDefinitionDocumentReader完成
   documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
   //统计解析的Bean数量
   return getRegistry().getBeanDefinitionCount() - countBefore;
}
```

Bean配置资源的载入解析分为以下两个过程：

首先，通过调用XML解析器将Bean配置信息转换成Document对象，但是这些Document对象并没有按照Spring的Bean规则进行解析。这一步是载入过程

其次，在完成通用XML解析之后，按照Spring Bean的定义规则对Document对象进行解析，其解析过程是在接口BeanDefinitionDocumentReader的实现类DefaultBeanDefinitionDoumentReader中实现。

#### 将配置载入内存

BeanDefinitionDocumentReader接口通过registerBeanDefinitions()方法调用其实现类DefaultBeanDefinitionDocumentReader对Document对象进行解析，解析的代码如下:

```java
//根据Spring DTD对Bean的定义规则解析Bean定义Document对象
	@Override
	public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		//获得XML描述符
		this.readerContext = readerContext;
		logger.debug("Loading bean definitions");
		//获得Document的根元素
		Element root = doc.getDocumentElement();
		doRegisterBeanDefinitions(root);
	}

	...
    
	protected void doRegisterBeanDefinitions(Element root) {
    
		//具体的解析过程由BeanDefinitionParserDelegate实现，
		//BeanDefinitionParserDelegate中定义了Spring Bean定义XML文件的各种元素
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
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

		//在解析Bean定义之前，进行自定义的解析，增强解析过程的可扩展性
		preProcessXml(root);
		//从Document的根元素开始进行Bean定义的Document对象
		parseBeanDefinitions(root, this.delegate);
		//在解析Bean定义之后，进行自定义的解析，增加解析过程的可扩展性
		postProcessXml(root);

		this.delegate = parent;
	}

	//创建BeanDefinitionParserDelegate，用于完成真正的解析过程
	protected BeanDefinitionParserDelegate createDelegate(
			XmlReaderContext readerContext, Element root, @Nullable BeanDefinitionParserDelegate parentDelegate) {

		BeanDefinitionParserDelegate delegate = new BeanDefinitionParserDelegate(readerContext);
		//BeanDefinitionParserDelegate初始化Document根元素
		delegate.initDefaults(root, parentDelegate);
		return delegate;
	}

	//使用Spring的Bean规则从Document的根元素开始进行Bean定义的Document对象
	protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		//Bean定义的Document对象使用了Spring默认的XML命名空间
		if (delegate.isDefaultNamespace(root)) {
			//获取Bean定义的Document对象根元素的所有子节点
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				//获得Document节点是XML元素节点
				if (node instanceof Element) {
					Element ele = (Element) node;
					//Bean定义的Document的元素节点使用的是Spring默认的XML命名空间
					if (delegate.isDefaultNamespace(ele)) {
						//使用Spring的Bean规则解析元素节点
						parseDefaultElement(ele, delegate);
					}
					else {
						//没有使用Spring默认的XML命名空间，则使用用户自定义的解//析规则解析元素节点
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			//Document的根节点没有使用Spring默认的命名空间，则使用用户自定义的
			//解析规则解析Document根节点
			delegate.parseCustomElement(root);
		}
	}

	//使用Spring的Bean规则解析Document元素节点
	private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		//如果元素节点是<Import>导入元素，进行导入解析
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
		//如果元素节点是<Alias>别名元素，进行别名解析
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
		//元素节点既不是导入元素，也不是别名元素，即普通的<Bean>元素，
		//按照Spring的Bean规则解析元素
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
			processBeanDefinition(ele, delegate);
		}
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// recurse
			doRegisterBeanDefinitions(ele);
		}
	}

	//解析<Import>导入元素，从给定的导入路径加载Bean定义资源到Spring IoC容器中
	protected void importBeanDefinitionResource(Element ele) {
		//获取给定的导入元素的location属性
		String location = ele.getAttribute(RESOURCE_ATTRIBUTE);
		//如果导入元素的location属性值为空，则没有导入任何资源，直接返回
		if (!StringUtils.hasText(location)) {
			getReaderContext().error("Resource location must not be empty", ele);
			return;
		}

		// Resolve system properties: e.g. "${user.dir}"
		//使用系统变量值解析location属性值
		location = getReaderContext().getEnvironment().resolveRequiredPlaceholders(location);

		Set<Resource> actualResources = new LinkedHashSet<>(4);

		// Discover whether the location is an absolute or relative URI
		//标识给定的导入元素的location是否是绝对路径
		boolean absoluteLocation = false;
		try {
			absoluteLocation = ResourcePatternUtils.isUrl(location) || ResourceUtils.toURI(location).isAbsolute();
		}
		catch (URISyntaxException ex) {
			// cannot convert to an URI, considering the location relative
			// unless it is the well-known Spring prefix "classpath*:"
			//给定的导入元素的location不是绝对路径
		}

		// Absolute or relative?
		//给定的导入元素的location是绝对路径
		if (absoluteLocation) {
			try {
				//使用资源读入器加载给定路径的Bean定义资源
				int importCount = getReaderContext().getReader().loadBeanDefinitions(location, actualResources);
				if (logger.isDebugEnabled()) {
					logger.debug("Imported " + importCount + " bean definitions from URL location [" + location + "]");
				}
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error(
						"Failed to import bean definitions from URL location [" + location + "]", ele, ex);
			}
		}
		else {
			// No URL -> considering resource location as relative to the current file.
			//给定的导入元素的location是相对路径
			try {
				int importCount;
				//将给定导入元素的location封装为相对路径资源
				Resource relativeResource = getReaderContext().getResource().createRelative(location);
				//封装的相对路径资源存在
				if (relativeResource.exists()) {
					//使用资源读入器加载Bean定义资源
					importCount = getReaderContext().getReader().loadBeanDefinitions(relativeResource);
					actualResources.add(relativeResource);
				}
				//封装的相对路径资源不存在
				else {
					//获取Spring IOC容器资源读入器的基本路径
					String baseLocation = getReaderContext().getResource().getURL().toString();
					//根据Spring IOC容器资源读入器的基本路径加载给定导入路径的资源
					importCount = getReaderContext().getReader().loadBeanDefinitions(
							StringUtils.applyRelativePath(baseLocation, location), actualResources);
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Imported " + importCount + " bean definitions from relative location [" + location + "]");
				}
			}
			catch (IOException ex) {
				getReaderContext().error("Failed to resolve current resource location", ele, ex);
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to import bean definitions from relative location [" + location + "]",
						ele, ex);
			}
		}
		Resource[] actResArray = actualResources.toArray(new Resource[actualResources.size()]);
		//在解析完<Import>元素之后，发送容器导入其他资源处理完成事件
		getReaderContext().fireImportProcessed(location, actResArray, extractSource(ele));
	}

	//解析<Alias>别名元素，为Bean向Spring IoC容器注册别名
	protected void processAliasRegistration(Element ele) {
		//获取<Alias>别名元素中name的属性值
		String name = ele.getAttribute(NAME_ATTRIBUTE);
		//获取<Alias>别名元素中alias的属性值
		String alias = ele.getAttribute(ALIAS_ATTRIBUTE);
		boolean valid = true;
		//<alias>别名元素的name属性值为空
		if (!StringUtils.hasText(name)) {
			getReaderContext().error("Name must not be empty", ele);
			valid = false;
		}
		//<alias>别名元素的alias属性值为空
		if (!StringUtils.hasText(alias)) {
			getReaderContext().error("Alias must not be empty", ele);
			valid = false;
		}
		if (valid) {
			try {
				//向容器的资源读入器注册别名
				getReaderContext().getRegistry().registerAlias(name, alias);
			}
			catch (Exception ex) {
				getReaderContext().error("Failed to register alias '" + alias +
						"' for bean with name '" + name + "'", ele, ex);
			}
			//在解析完<Alias>元素之后，发送容器别名处理完成事件
			getReaderContext().fireAliasRegistered(name, alias, extractSource(ele));
		}
	}

	//解析Bean定义资源Document对象的普通元素
	protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		// BeanDefinitionHolder是对BeanDefinition的封装，即Bean定义的封装类
		//对Document对象中<Bean>元素的解析由BeanDefinitionParserDelegate实现
		// BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// Register the final decorated instance.
				//向Spring IOC容器注册解析得到的Bean定义，这是Bean定义向IOC容器注册的入口
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// Send registration event.
			//在完成向Spring IOC容器注册解析得到的Bean定义之后，发送注册事件
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}
```

通过上述Spring IOC容器对载入的Bean定义Document解析可以看出，我们使用Spring时，在Spring配置文件中可以使用```<import>```元素来导入IOC容器所需要的其他资源，Spring IOC容器在解析时会首先将指定导入的资源加载进容器中。使用```<ailas>```别名时，Spring IOC容器首先将别名元素所定义的别名注册到容器中。

对于既不是```<import>```元素，又不是```<ailas>```元素的元素，即Spring配置文件中普通的```<bean>```元素的解析由BeanDefinitionParserDelegate类的parseBeanDefinitionElement()方法来实现。这个解析的过程非常复杂。

#### 载入```<bean>```元素

Bean配置信息中的```<import>```和```<alias>```元素解析在DefaultBeanDefinitionDocumentReader中已经完成，对Bean配置信息中适用最多的```<bean>```元素交由BeanDefinitionParserDelegate来解析，其解析代码的实现源码如下：

```java
//解析<Bean>元素的入口
@Nullable
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
   return parseBeanDefinitionElement(ele, null);
}

/**
 * Parses the supplied {@code <bean>} element. May return {@code null}
 * if there were errors during parse. Errors are reported to the
 * {@link org.springframework.beans.factory.parsing.ProblemReporter}.
 */
//解析Bean定义资源文件中的<Bean>元素，这个方法中主要处理<Bean>元素的id，name和别名属性
@Nullable
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
   //获取<Bean>元素中的id属性值
   String id = ele.getAttribute(ID_ATTRIBUTE);
   //获取<Bean>元素中的name属性值
   String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);

   //获取<Bean>元素中的alias属性值
   List<String> aliases = new ArrayList<>();

   //将<Bean>元素中的所有name属性值存放到别名中
   if (StringUtils.hasLength(nameAttr)) {
      String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
      aliases.addAll(Arrays.asList(nameArr));
   }

   String beanName = id;
   //如果<Bean>元素中没有配置id属性时，将别名中的第一个值赋值给beanName
   if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
      beanName = aliases.remove(0);
      if (logger.isDebugEnabled()) {
         logger.debug("No XML 'id' specified - using '" + beanName +
               "' as bean name and " + aliases + " as aliases");
      }
   }

   //检查<Bean>元素所配置的id或者name的唯一性，containingBean标识<Bean>
   //元素中是否包含子<Bean>元素
   if (containingBean == null) {
      //检查<Bean>元素所配置的id、name或者别名是否重复
      checkNameUniqueness(beanName, aliases, ele);
   }

   //详细对<Bean>元素中配置的Bean定义进行解析的地方
   AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
   if (beanDefinition != null) {
      if (!StringUtils.hasText(beanName)) {
         try {
            if (containingBean != null) {
               //如果<Bean>元素中没有配置id、别名或者name，且没有包含子元素
               //<Bean>元素，为解析的Bean生成一个唯一beanName并注册
               beanName = BeanDefinitionReaderUtils.generateBeanName(
                     beanDefinition, this.readerContext.getRegistry(), true);
            }
            else {
               //如果<Bean>元素中没有配置id、别名或者name，且包含了子元素
               //<Bean>元素，为解析的Bean使用别名向IOC容器注册
               beanName = this.readerContext.generateBeanName(beanDefinition);
               // Register an alias for the plain bean class name, if still possible,
               // if the generator returned the class name plus a suffix.
               // This is expected for Spring 1.2/2.0 backwards compatibility.
               //为解析的Bean使用别名注册时，为了向后兼容
               //Spring1.2/2.0，给别名添加类名后缀
               String beanClassName = beanDefinition.getBeanClassName();
               if (beanClassName != null &&
                     beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
                     !this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
                  aliases.add(beanClassName);
               }
            }
            if (logger.isDebugEnabled()) {
               logger.debug("Neither XML 'id' nor 'name' specified - " +
                     "using generated bean name [" + beanName + "]");
            }
         }
         catch (Exception ex) {
            error(ex.getMessage(), ele);
            return null;
         }
      }
      String[] aliasesArray = StringUtils.toStringArray(aliases);
      return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
   }
   //当解析出错时，返回null
   return null;
}

/**
 * Validate that the specified bean name and aliases have not been used already
 * within the current level of beans element nesting.
 */
protected void checkNameUniqueness(String beanName, List<String> aliases, Element beanElement) {
   String foundName = null;

   if (StringUtils.hasText(beanName) && this.usedNames.contains(beanName)) {
      foundName = beanName;
   }
   if (foundName == null) {
      foundName = CollectionUtils.findFirstMatch(this.usedNames, aliases);
   }
   if (foundName != null) {
      error("Bean name '" + foundName + "' is already used in this <beans> element", beanElement);
   }

   this.usedNames.add(beanName);
   this.usedNames.addAll(aliases);
}

/**
 * Parse the bean definition itself, without regard to name or aliases. May return
 * {@code null} if problems occurred during the parsing of the bean definition.
 */
//详细对<Bean>元素中配置的Bean定义其他属性进行解析
//由于上面的方法中已经对Bean的id、name和别名等属性进行了处理
//该方法中主要处理除这三个以外的其他属性数据
@Nullable
public AbstractBeanDefinition parseBeanDefinitionElement(
      Element ele, String beanName, @Nullable BeanDefinition containingBean) {
   //记录解析的<Bean>
   this.parseState.push(new BeanEntry(beanName));

   //这里只读取<Bean>元素中配置的class名字，然后载入到BeanDefinition中去
   //只是记录配置的class名字，不做实例化，对象的实例化在依赖注入时完成
   String className = null;

   //如果<Bean>元素中配置了parent属性，则获取parent属性的值
   if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
      className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
   }
   String parent = null;
   if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
      parent = ele.getAttribute(PARENT_ATTRIBUTE);
   }

   try {
      //根据<Bean>元素配置的class名称和parent属性值创建BeanDefinition
      //为载入Bean定义信息做准备
      AbstractBeanDefinition bd = createBeanDefinition(className, parent);

      //对当前的<Bean>元素中配置的一些属性进行解析和设置，如配置的单态(singleton)属性等
      parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
      //为<Bean>元素解析的Bean设置description信息
      bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));

      //对<Bean>元素的meta(元信息)属性解析
      parseMetaElements(ele, bd);
      //对<Bean>元素的lookup-method属性解析
      parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
      //对<Bean>元素的replaced-method属性解析
      parseReplacedMethodSubElements(ele, bd.getMethodOverrides());

      //解析<Bean>元素的构造方法设置
      parseConstructorArgElements(ele, bd);
      //解析<Bean>元素的<property>设置
      parsePropertyElements(ele, bd);
      //解析<Bean>元素的qualifier属性
      parseQualifierElements(ele, bd);

      //为当前解析的Bean设置所需的资源和依赖对象
      bd.setResource(this.readerContext.getResource());
      bd.setSource(extractSource(ele));

      return bd;
   }
   catch (ClassNotFoundException ex) {
      error("Bean class [" + className + "] not found", ele, ex);
   }
   catch (NoClassDefFoundError err) {
      error("Class that bean class [" + className + "] depends on not found", ele, err);
   }
   catch (Throwable ex) {
      error("Unexpected failure during bean definition parsing", ele, ex);
   }
   finally {
      this.parseState.pop();
   }

   //解析<Bean>元素出错时，返回null
   return null;
}
```

只要对Spring配置文件比较熟悉的人，通过对上述源码的分析，就会明白我们在Spring配置文件中```<bean>```元素中配置的属性就是通过该方法解析和设置到Bean中去的。

**注意：在解析```<bean>```元素过程中没有创建和实例化Bean对象，只是创建了Bean对象的定义类BeanDefinition，将```<bean>```元素中的配置信息设置到BeanDefinition中作为记录，当依赖注入时才使用这些记录信息创建和实例化具体的Bean对象。**

上面方法中对一些配置如：元信息（meta）、qualifier等的计息，我们在Spring配置时候使用的也不多，我们在使用Spring的```<bean>```元素时，配置最多的是```<property>```属性，因此我们下面继续分析源码，了解Bean的属性在解析时是如何设置的。

#### 载入```<property>```元素

BeanDefinitionParserDelegate在解析```<bean>```调用parsePropertyElements()方法解析```<bean>```元素中的```<property>```属性子元素，解析源码如下：

```java
//解析<Bean>元素中的<property>子元素
public void parsePropertyElements(Element beanEle, BeanDefinition bd) {
   //获取<Bean>元素中所有的子元素
   NodeList nl = beanEle.getChildNodes();
   for (int i = 0; i < nl.getLength(); i++) {
      Node node = nl.item(i);
      //如果子元素是<property>子元素，则调用解析<property>子元素方法解析
      if (isCandidateElement(node) && nodeNameEquals(node, PROPERTY_ELEMENT)) {
         parsePropertyElement((Element) node, bd);
      }
   }
}
```

```java
//解析<property>元素
public void parsePropertyElement(Element ele, BeanDefinition bd) {
   //获取<property>元素的名字
   String propertyName = ele.getAttribute(NAME_ATTRIBUTE);
   if (!StringUtils.hasLength(propertyName)) {
      error("Tag 'property' must have a 'name' attribute", ele);
      return;
   }
   this.parseState.push(new PropertyEntry(propertyName));
   try {
      //如果一个Bean中已经有同名的property存在，则不进行解析，直接返回。
      //即如果在同一个Bean中配置同名的property，则只有第一个起作用
      if (bd.getPropertyValues().contains(propertyName)) {
         error("Multiple 'property' definitions for property '" + propertyName + "'", ele);
         return;
      }
      //解析获取property的值
      Object val = parsePropertyValue(ele, bd, propertyName);
      //根据property的名字和值创建property实例
      PropertyValue pv = new PropertyValue(propertyName, val);
      //解析<property>元素中的属性
      parseMetaElements(ele, pv);
      pv.setSource(extractSource(ele));
      bd.getPropertyValues().addPropertyValue(pv);
   }
   finally {
      this.parseState.pop();
   }
}
```

```java
//解析获取property值
@Nullable
public Object parsePropertyValue(Element ele, BeanDefinition bd, @Nullable String propertyName) {
   String elementName = (propertyName != null) ?
               "<property> element for property '" + propertyName + "'" :
               "<constructor-arg> element";

   // Should only have one child element: ref, value, list, etc.
   //获取<property>的所有子元素，只能是其中一种类型:ref,value,list,etc等
   NodeList nl = ele.getChildNodes();
   Element subElement = null;
   for (int i = 0; i < nl.getLength(); i++) {
      Node node = nl.item(i);
      //子元素不是description和meta属性
      if (node instanceof Element && !nodeNameEquals(node, DESCRIPTION_ELEMENT) &&
            !nodeNameEquals(node, META_ELEMENT)) {
         // Child element is what we're looking for.
         if (subElement != null) {
            error(elementName + " must not contain more than one sub-element", ele);
         }
         else {
            //当前<property>元素包含有子元素
            subElement = (Element) node;
         }
      }
   }

   //判断property的属性值是ref还是value，不允许既是ref又是value
   boolean hasRefAttribute = ele.hasAttribute(REF_ATTRIBUTE);
   boolean hasValueAttribute = ele.hasAttribute(VALUE_ATTRIBUTE);
   if ((hasRefAttribute && hasValueAttribute) ||
         ((hasRefAttribute || hasValueAttribute) && subElement != null)) {
      error(elementName +
            " is only allowed to contain either 'ref' attribute OR 'value' attribute OR sub-element", ele);
   }

   //如果属性是ref，创建一个ref的数据对象RuntimeBeanReference
   //这个对象封装了ref信息
   if (hasRefAttribute) {
      String refName = ele.getAttribute(REF_ATTRIBUTE);
      if (!StringUtils.hasText(refName)) {
         error(elementName + " contains empty 'ref' attribute", ele);
      }
      //一个指向运行时所依赖对象的引用
      RuntimeBeanReference ref = new RuntimeBeanReference(refName);
      //设置这个ref的数据对象是被当前的property对象所引用
      ref.setSource(extractSource(ele));
      return ref;
   }
   //如果属性是value，创建一个value的数据对象TypedStringValue
   //这个对象封装了value信息
   else if (hasValueAttribute) {
      //一个持有String类型值的对象
      TypedStringValue valueHolder = new TypedStringValue(ele.getAttribute(VALUE_ATTRIBUTE));
      //设置这个value数据对象是被当前的property对象所引用
      valueHolder.setSource(extractSource(ele));
      return valueHolder;
   }
   //如果当前<property>元素还有子元素
   else if (subElement != null) {
      //解析<property>的子元素
      return parsePropertySubElement(subElement, bd);
   }
   else {
      // Neither child element nor "ref" or "value" attribute found.
      //propery属性中既不是ref，也不是value属性，解析出错返回null
      error(elementName + " must specify a ref or value", ele);
      return null;
   }
}
```

通过对上述源码的分析，我们可以了解在Spring配置文件中，```<bean>```元素中```<property>```元素相关配置是如何处理的：

1. ref被封装为指向依赖对象的一个引用。
2. value配置都会封装成一个字符串类型的对象。
3. ref和value都通过“解析的数据类型属性值.setSource(extractSource(ele));”方法将属性值、引用于所引用的属性关联起来。

在方法的最后对于```<property>```元素的子元素通过parsePropertySubElement()解析，继续分析该方法的源码。

#### 载入```<property>```的子元素

在BeanDefinitionParserDelegate类中的parsePropertySubElement()方法对```<property>```中的子元素解析，源码如下：

```java
@Nullable
public Object parsePropertySubElement(Element ele, @Nullable BeanDefinition bd) {
   return parsePropertySubElement(ele, bd, null);
}
```

```java
//解析<property>元素中ref,value或者集合等子元素
@Nullable
public Object parsePropertySubElement(Element ele, @Nullable BeanDefinition bd, @Nullable String defaultValueType) {
   //如果<property>没有使用Spring默认的命名空间，则使用用户自定义的规则解析内嵌元素
   if (!isDefaultNamespace(ele)) {
      return parseNestedCustomElement(ele, bd);
   }
   //如果子元素是bean，则使用解析<Bean>元素的方法解析
   else if (nodeNameEquals(ele, BEAN_ELEMENT)) {
      BeanDefinitionHolder nestedBd = parseBeanDefinitionElement(ele, bd);
      if (nestedBd != null) {
         nestedBd = decorateBeanDefinitionIfRequired(ele, nestedBd, bd);
      }
      return nestedBd;
   }
   //如果子元素是ref，ref中只能有以下3个属性：bean、local、parent
   else if (nodeNameEquals(ele, REF_ELEMENT)) {
      // A generic reference to any name of any bean.
      //可以不再同一个Spring配置文件中，具体请参考Spring对ref的配置规则
      String refName = ele.getAttribute(BEAN_REF_ATTRIBUTE);
      boolean toParent = false;
      if (!StringUtils.hasLength(refName)) {
         // A reference to the id of another bean in a parent context.
         //获取<property>元素中parent属性值，引用父级容器中的Bean
         refName = ele.getAttribute(PARENT_REF_ATTRIBUTE);
         toParent = true;
         if (!StringUtils.hasLength(refName)) {
            error("'bean' or 'parent' is required for <ref> element", ele);
            return null;
         }
      }
      if (!StringUtils.hasText(refName)) {
         error("<ref> element contains empty target attribute", ele);
         return null;
      }
      //创建ref类型数据，指向被引用的对象
      RuntimeBeanReference ref = new RuntimeBeanReference(refName, toParent);
      //设置引用类型值是被当前子元素所引用
      ref.setSource(extractSource(ele));
      return ref;
   }
   //如果子元素是<idref>，使用解析ref元素的方法解析
   else if (nodeNameEquals(ele, IDREF_ELEMENT)) {
      return parseIdRefElement(ele);
   }
   //如果子元素是<value>，使用解析value元素的方法解析
   else if (nodeNameEquals(ele, VALUE_ELEMENT)) {
      return parseValueElement(ele, defaultValueType);
   }
   //如果子元素是null，为<property>设置一个封装null值的字符串数据
   else if (nodeNameEquals(ele, NULL_ELEMENT)) {
      // It's a distinguished null value. Let's wrap it in a TypedStringValue
      // object in order to preserve the source location.
      TypedStringValue nullHolder = new TypedStringValue(null);
      nullHolder.setSource(extractSource(ele));
      return nullHolder;
   }
   //如果子元素是<array>，使用解析array集合子元素的方法解析
   else if (nodeNameEquals(ele, ARRAY_ELEMENT)) {
      return parseArrayElement(ele, bd);
   }
   //如果子元素是<list>，使用解析list集合子元素的方法解析
   else if (nodeNameEquals(ele, LIST_ELEMENT)) {
      return parseListElement(ele, bd);
   }
   //如果子元素是<set>，使用解析set集合子元素的方法解析
   else if (nodeNameEquals(ele, SET_ELEMENT)) {
      return parseSetElement(ele, bd);
   }
   //如果子元素是<map>，使用解析map集合子元素的方法解析
   else if (nodeNameEquals(ele, MAP_ELEMENT)) {
      return parseMapElement(ele, bd);
   }
   //如果子元素是<props>，使用解析props集合子元素的方法解析
   else if (nodeNameEquals(ele, PROPS_ELEMENT)) {
      return parsePropsElement(ele);
   }
   //既不是ref，又不是value，也不是集合，则子元素配置错误，返回null
   else {
      error("Unknown property sub-element: [" + ele.getNodeName() + "]", ele);
      return null;
   }
}
```

通过上述源码分析，我们看到了在Spring配置文件中，对```<property>```元素中配置的array、list、set、map、prop等各种集合子元素的都通过上述方法解析，生成对应的数据对象，比如ManagedList、ManagedArray、ManagedSet等，这些Meanaged类是Spring对象BeanDefinition的数据封装，对集合数据类型的具体解析由各自的解析方法实现，解析方法的命名非常规范，一目了然，我们对```<list>```集合元素的解析方法进行源码分析，了解其实现过程。

#### 载入```<list>```的子元素

在BeanDefinitionParserDelegate类中的parseListElement()方法就是具体实现解析```<property>```元素中的```<list>```集合子元素，源码如下：

```java
//解析<list>集合子元素
public List<Object> parseListElement(Element collectionEle, @Nullable BeanDefinition bd) {
   //获取<list>元素中的value-type属性，即获取集合元素的数据类型
   String defaultElementType = collectionEle.getAttribute(VALUE_TYPE_ATTRIBUTE);
   //获取<list>集合元素中的所有子节点
   NodeList nl = collectionEle.getChildNodes();
   //Spring中将List封装为ManagedList
   ManagedList<Object> target = new ManagedList<>(nl.getLength());
   target.setSource(extractSource(collectionEle));
   //设置集合目标数据类型
   target.setElementTypeName(defaultElementType);
   target.setMergeEnabled(parseMergeAttribute(collectionEle));
   //具体的<list>元素解析
   parseCollectionElements(nl, target, bd, defaultElementType);
   return target;
}
```

```java
//具体解析<list>集合元素，<array>、<list>和<set>都使用该方法解析
protected void parseCollectionElements(
      NodeList elementNodes, Collection<Object> target, @Nullable BeanDefinition bd, String defaultElementType) {
   //遍历集合所有节点
   for (int i = 0; i < elementNodes.getLength(); i++) {
      Node node = elementNodes.item(i);
      //节点不是description节点
      if (node instanceof Element && !nodeNameEquals(node, DESCRIPTION_ELEMENT)) {
         //将解析的元素加入集合中，递归调用下一个子元素
         target.add(parsePropertySubElement((Element) node, bd, defaultElementType));
      }
   }
}
```

经过对Spring Bean配置信息转换的Document对象中的元素层层解析，Spring IOC现在已经将XML形式定义的Bean配置信息转换为Spring IOC所识别的数据结构——BeanDefinition，它是Bean配置信息中配置的POJO对象在Spring IOC容器中的映射，我们可以通过AbstractBeanDefinition为入口，看到了IOC容器进行索引、查询和操作。

#### 分配注册策略

让我们继续跟踪程序的执行顺序，接下来我们来分析DefaultBeanDefinitionDocumentReader对Bean定义转换的Document对象解析的流程中，在其parseDefaultElement()方法中完成对Document对象的解析后得到封装BeanDefinition的BeanDefinitionHold对象，然后调用BeanDefinitionReaderUtils的registerBeanDefinition()方法向IOC容器注册解析的Bean，BeanDefnitionReaderUtils的注册源码如下：

```java
//将解析的BeanDefinitionHold注册到容器中
public static void registerBeanDefinition(
      BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
      throws BeanDefinitionStoreException {

   // Register bean definition under primary name.
   //获取解析的BeanDefinition的名称
   String beanName = definitionHolder.getBeanName();
   //向IOC容器注册BeanDefinition
   registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

   // Register aliases for bean name, if any.
   //如果解析的BeanDefinition有别名，向容器为其注册别名
   String[] aliases = definitionHolder.getAliases();
   if (aliases != null) {
      for (String alias : aliases) {
         registry.registerAlias(beanName, alias);
      }
   }
}
```

当调用BeanDefinitionReaderUtils向IOC容器注册解析的BeanDefinition时，真正完成注册功能的是DefaultListableBeanFactory。

#### 向容器注册

DefaultListableBeanFactory中适用一个HashMap的集合对象存放IOC容器中注册解析的BeanDefinition，向IOC容器注册主要源码如下：

```java
	//存储注册信息的BeanDefinition
	private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);

//向IOC容器注册解析的BeanDefiniton
@Override
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
      throws BeanDefinitionStoreException {

   Assert.hasText(beanName, "Bean name must not be empty");
   Assert.notNull(beanDefinition, "BeanDefinition must not be null");

   //校验解析的BeanDefiniton
   if (beanDefinition instanceof AbstractBeanDefinition) {
      try {
         ((AbstractBeanDefinition) beanDefinition).validate();
      }
      catch (BeanDefinitionValidationException ex) {
         throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
               "Validation of bean definition failed", ex);
      }
   }

   BeanDefinition oldBeanDefinition;

   oldBeanDefinition = this.beanDefinitionMap.get(beanName);

   if (oldBeanDefinition != null) {
      if (!isAllowBeanDefinitionOverriding()) {
         throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
               "Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
               "': There is already [" + oldBeanDefinition + "] bound.");
      }
      else if (oldBeanDefinition.getRole() < beanDefinition.getRole()) {
         // e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
         if (this.logger.isWarnEnabled()) {
            this.logger.warn("Overriding user-defined bean definition for bean '" + beanName +
                  "' with a framework-generated bean definition: replacing [" +
                  oldBeanDefinition + "] with [" + beanDefinition + "]");
         }
      }
      else if (!beanDefinition.equals(oldBeanDefinition)) {
         if (this.logger.isInfoEnabled()) {
            this.logger.info("Overriding bean definition for bean '" + beanName +
                  "' with a different definition: replacing [" + oldBeanDefinition +
                  "] with [" + beanDefinition + "]");
         }
      }
      else {
         if (this.logger.isDebugEnabled()) {
            this.logger.debug("Overriding bean definition for bean '" + beanName +
                  "' with an equivalent definition: replacing [" + oldBeanDefinition +
                  "] with [" + beanDefinition + "]");
         }
      }
      this.beanDefinitionMap.put(beanName, beanDefinition);
   }
   else {
      if (hasBeanCreationStarted()) {
         // Cannot modify startup-time collection elements anymore (for stable iteration)
         //注册的过程中需要线程同步，以保证数据的一致性
         synchronized (this.beanDefinitionMap) {
            this.beanDefinitionMap.put(beanName, beanDefinition);
            List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
            updatedDefinitions.addAll(this.beanDefinitionNames);
            updatedDefinitions.add(beanName);
            this.beanDefinitionNames = updatedDefinitions;
            if (this.manualSingletonNames.contains(beanName)) {
               Set<String> updatedSingletons = new LinkedHashSet<>(this.manualSingletonNames);
               updatedSingletons.remove(beanName);
               this.manualSingletonNames = updatedSingletons;
            }
         }
      }
      else {
         // Still in startup registration phase
         this.beanDefinitionMap.put(beanName, beanDefinition);
         this.beanDefinitionNames.add(beanName);
         this.manualSingletonNames.remove(beanName);
      }
      this.frozenBeanDefinitionNames = null;
   }

   //检查是否有同名的BeanDefinition已经在IOC容器中注册
   if (oldBeanDefinition != null || containsSingleton(beanName)) {
      //重置所有已经注册过的BeanDefinition的缓存
      resetBeanDefinition(beanName);
   }
}
```

至此，Bean配置信息中配置Bean被解析过后，已经注册到IOC容器中，被容器管理起来，真正完成了IOC容器初始化所做的全部工作。现在IOC容器中已经建立了整个Bean的配置信息，这些BeanDefinition信息已经可以使用，并且可以被检索，IOC容器的作用就是对这些注册Bean定义信息进行处理和维护。这些的注册的Bean定义信息是IOC容器控制反转的基础，正是有了这些注册的数据，容器才可以进行依赖注入。