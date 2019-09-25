# Spring-DI源码分析

## 依赖注入发生时间

当Spring IOC容器完成了Bean定义资源的定位、载入和解析注册以后，IOC容器中已经管理了Bean定义的相关数据，但是此时IOC容器还没有对所管理的Bean进行依赖注入，依赖注入在一下两种情况发生：

1. 用户第一次调用getBean()方法时，IOC容器触发依赖注入。
2. 当用户在配置文件中将```<bean>```元素配置了```lazy-init=false```属性,即让容器在解析Bean定义时进行预实例化，触发依赖注入。

BeanFactory接口定义了Spring IOC容器的基本功能规范，是Spring IOC容器所应遵守的最底层和最基本的编程规范。BeanFactory接口中定义了几个getBean()方法，就是用户向IOC容器索取管理的Bean的方法，就是用户向IOC索取管理的Bean方法，我们通过分析其子类的具体实现，理解Spring IOC容器在用户索取Bean时如何完成依赖注入。

![image-20190925081119271](/Users/dongwenbin/github/doc/Spring/assets/image-20190925081119271.png)

在BeanFactory中我们可以看到getBean(String...)方法，但它具体实现在AbstractBeanFactory中。

## 源码分析

### 获取Bean的入口

AbstractBeanFactory的getBean()相关方法的源码如下：

```java
//获取IOC容器中指定名称的Bean
@Override
public Object getBean(String name) throws BeansException {
   //doGetBean才是真正向IoC容器获取被管理Bean的过程
   return doGetBean(name, null, null, false);
}

//获取IOC容器中指定名称和类型的Bean
@Override
public <T> T getBean(String name, @Nullable Class<T> requiredType) throws BeansException {
   //doGetBean才是真正向IoC容器获取被管理Bean的过程
   return doGetBean(name, requiredType, null, false);
}

//获取IOC容器中指定名称和参数的Bean
@Override
public Object getBean(String name, Object... args) throws BeansException {
   //doGetBean才是真正向IoC容器获取被管理Bean的过程
   return doGetBean(name, null, args, false);
}

//获取IOC容器中指定名称、类型和参数的Bean
public <T> T getBean(String name, @Nullable Class<T> requiredType, @Nullable Object... args)
      throws BeansException {
   //doGetBean才是真正向IoC容器获取被管理Bean的过程
   return doGetBean(name, requiredType, args, false);
}

@SuppressWarnings("unchecked")
//真正实现向IOC容器获取Bean的功能，也是触发依赖注入功能的地方
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
      @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

   //根据指定的名称获取被管理Bean的名称，剥离指定名称中对容器的相关依赖
   //如果指定的是别名，将别名转换为规范的Bean名称
   final String beanName = transformedBeanName(name);
   Object bean;

   // Eagerly check singleton cache for manually registered singletons.
   //先从缓存中取是否已经有被创建过的单态类型的Bean
   //对于单例模式的Bean整个IOC容器中只创建一次，不需要重复创建
   Object sharedInstance = getSingleton(beanName);
   //IOC容器创建单例模式Bean实例对象
   if (sharedInstance != null && args == null) {
      if (logger.isDebugEnabled()) {
         //如果指定名称的Bean在容器中已有单例模式的Bean被创建
         //直接返回已经创建的Bean
         if (isSingletonCurrentlyInCreation(beanName)) {
            logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
                  "' that is not fully initialized yet - a consequence of a circular reference");
         }
         else {
            logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
         }
      }
      //获取给定Bean的实例对象，主要是完成FactoryBean的相关处理
      //注意：BeanFactory是管理容器中Bean的工厂，而FactoryBean是
      //创建创建对象的工厂Bean，两者之间有区别
      bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
   }

   else {
      // Fail if we're already creating this bean instance:
      // We're assumably within a circular reference.
      //缓存没有正在创建的单例模式Bean
      //缓存中已经有已经创建的原型模式Bean
      //但是由于循环引用的问题导致实例化对象失败
      if (isPrototypeCurrentlyInCreation(beanName)) {
         throw new BeanCurrentlyInCreationException(beanName);
      }

      // Check if bean definition exists in this factory.
      //对IOC容器中是否存在指定名称的BeanDefinition进行检查，首先检查是否
      //能在当前的BeanFactory中获取的所需要的Bean，如果不能则委托当前容器
      //的父级容器去查找，如果还是找不到则沿着容器的继承体系向父级容器查找
      BeanFactory parentBeanFactory = getParentBeanFactory();
      //当前容器的父级容器存在，且当前容器中不存在指定名称的Bean
      if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
         // Not found -> check parent.
         //解析指定Bean名称的原始名称
         String nameToLookup = originalBeanName(name);
         if (parentBeanFactory instanceof AbstractBeanFactory) {
            return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                  nameToLookup, requiredType, args, typeCheckOnly);
         }
         else if (args != null) {
            // Delegation to parent with explicit args.
            //委派父级容器根据指定名称和显式的参数查找
            return (T) parentBeanFactory.getBean(nameToLookup, args);
         }
         else {
            // No args -> delegate to standard getBean method.
            //委派父级容器根据指定名称和类型查找
            return parentBeanFactory.getBean(nameToLookup, requiredType);
         }
      }

      //创建的Bean是否需要进行类型验证，一般不需要
      if (!typeCheckOnly) {
         //向容器标记指定的Bean已经被创建
         markBeanAsCreated(beanName);
      }

      try {
         //根据指定Bean名称获取其父级的Bean定义
         //主要解决Bean继承时子类合并父类公共属性问题
         final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
         checkMergedBeanDefinition(mbd, beanName, args);

         // Guarantee initialization of beans that the current bean depends on.
         //获取当前Bean所有依赖Bean的名称
         String[] dependsOn = mbd.getDependsOn();
         //如果当前Bean有依赖Bean
         if (dependsOn != null) {
            for (String dep : dependsOn) {
               if (isDependent(beanName, dep)) {
                  throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
               }
               //递归调用getBean方法，获取当前Bean的依赖Bean
               registerDependentBean(dep, beanName);
               //把被依赖Bean注册给当前依赖的Bean
               getBean(dep);
            }
         }

         // Create bean instance.
         //创建单例模式Bean的实例对象
         if (mbd.isSingleton()) {
            //这里使用了一个匿名内部类，创建Bean实例对象，并且注册给所依赖的对象
            sharedInstance = getSingleton(beanName, () -> {
               try {
                  //创建一个指定Bean实例对象，如果有父级继承，则合并子类和父类的定义
                  return createBean(beanName, mbd, args);
               }
               catch (BeansException ex) {
                  // Explicitly remove instance from singleton cache: It might have been put there
                  // eagerly by the creation process, to allow for circular reference resolution.
                  // Also remove any beans that received a temporary reference to the bean.
                  //显式地从容器单例模式Bean缓存中清除实例对象
                  destroySingleton(beanName);
                  throw ex;
               }
            });
            //获取给定Bean的实例对象
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
         }

         //IOC容器创建原型模式Bean实例对象
         else if (mbd.isPrototype()) {
            // It's a prototype -> create a new instance.
            //原型模式(Prototype)是每次都会创建一个新的对象
            Object prototypeInstance = null;
            try {
               //回调beforePrototypeCreation方法，默认的功能是注册当前创建的原型对象
               beforePrototypeCreation(beanName);
               //创建指定Bean对象实例
               prototypeInstance = createBean(beanName, mbd, args);
            }
            finally {
               //回调afterPrototypeCreation方法，默认的功能告诉IOC容器指定Bean的原型对象不再创建
               afterPrototypeCreation(beanName);
            }
            //获取给定Bean的实例对象
            bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
         }

         //要创建的Bean既不是单例模式，也不是原型模式，则根据Bean定义资源中
         //配置的生命周期范围，选择实例化Bean的合适方法，这种在Web应用程序中
         //比较常用，如：request、session、application等生命周期
         else {
            String scopeName = mbd.getScope();
            final Scope scope = this.scopes.get(scopeName);
            //Bean定义资源中没有配置生命周期范围，则Bean定义不合法
            if (scope == null) {
               throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
            }
            try {
               //这里又使用了一个匿名内部类，获取一个指定生命周期范围的实例
               Object scopedInstance = scope.get(beanName, () -> {
                  beforePrototypeCreation(beanName);
                  try {
                     return createBean(beanName, mbd, args);
                  }
                  finally {
                     afterPrototypeCreation(beanName);
                  }
               });
               //获取给定Bean的实例对象
               bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
            }
            catch (IllegalStateException ex) {
               throw new BeanCreationException(beanName,
                     "Scope '" + scopeName + "' is not active for the current thread; consider " +
                     "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                     ex);
            }
         }
      }
      catch (BeansException ex) {
         cleanupAfterBeanCreationFailure(beanName);
         throw ex;
      }
   }

   // Check if required type matches the type of the actual bean instance.
   //对创建的Bean实例对象进行类型检查
   if (requiredType != null && !requiredType.isInstance(bean)) {
      try {
         T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
         if (convertedBean == null) {
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
         }
         return convertedBean;
      }
      catch (TypeMismatchException ex) {
         if (logger.isDebugEnabled()) {
            logger.debug("Failed to convert bean '" + name + "' to required type '" +
                  ClassUtils.getQualifiedName(requiredType) + "'", ex);
         }
         throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
      }
   }
   return (T) bean;
}
```

通过上面对向IOC容器获取Bean方法的分析，我们可以看到在Spring中，如果Bean定义的单例模式（Singleton），则容器在创建之前先从缓存中查找，以确保整个容器只存在一个实例对象。如果Bean定义是原型模式（Prototype），则容器每次都会创建一个新的实例对象。除此之外，Bean定义还可以扩展为指定其生命周期范围。

上面的源码只是定义了根据Bean定义的模式，采取不同创建Bean实例对象的策略，具体的Bean实例对象的创建过程由实现了ObejctFactory接口的匿名内部类的createBean()方法完成，ObjectFactory使用委派模式，具体的Bean实例创建过程交由其实现类AbstractAutowireCapableBeanFactory完成，我们继续分析AbstractAutoCapableBeanFactory的createBean()方法的源码，理解其创建Bean实例的具体实现过程。

### 开始实例化

AbstractAutowireCapableBeanFactory类实现了ObejctFactory接口，创建容器执行的Bean实例对象，同时还对创建的Bean实例对象进行初始化处理。其创建Bean实例对象的方法源码如下：

```java
//创建Bean实例对象
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
      throws BeanCreationException {

   if (logger.isDebugEnabled()) {
      logger.debug("Creating instance of bean '" + beanName + "'");
   }
   RootBeanDefinition mbdToUse = mbd;

   //判断需要创建的Bean是否可以实例化，即是否可以通过当前的类加载器加载
   Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
   if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
      mbdToUse = new RootBeanDefinition(mbd);
      mbdToUse.setBeanClass(resolvedClass);
   }

   //校验和准备Bean中的方法覆盖
   try {
      mbdToUse.prepareMethodOverrides();
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
            beanName, "Validation of method overrides failed", ex);
   }

   try {
      //如果Bean配置了初始化前和初始化后的处理器，则试图返回一个需要创建Bean的代理对象
      Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
      if (bean != null) {
         return bean;
      }
   }
   catch (Throwable ex) {
      throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
            "BeanPostProcessor before instantiation of bean failed", ex);
   }

   try {
      //创建Bean的入口
      Object beanInstance = doCreateBean(beanName, mbdToUse, args);
      if (logger.isDebugEnabled()) {
         logger.debug("Finished creating instance of bean '" + beanName + "'");
      }
      return beanInstance;
   }
   catch (BeanCreationException ex) {
      // A previously detected exception with proper bean creation context already...
      throw ex;
   }
   catch (ImplicitlyAppearedSingletonException ex) {
      // An IllegalStateException to be communicated up to DefaultSingletonBeanRegistry...
      throw ex;
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
   }
}

//真正创建Bean的方法
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
      throws BeanCreationException {

   //封装被创建的Bean对象
   BeanWrapper instanceWrapper = null;
   if (mbd.isSingleton()) {
      instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
   }
   if (instanceWrapper == null) {
      instanceWrapper = createBeanInstance(beanName, mbd, args);
   }
   final Object bean = instanceWrapper.getWrappedInstance();
   //获取实例化对象的类型
   Class<?> beanType = instanceWrapper.getWrappedClass();
   if (beanType != NullBean.class) {
      mbd.resolvedTargetType = beanType;
   }

   //调用PostProcessor后置处理器
   synchronized (mbd.postProcessingLock) {
      if (!mbd.postProcessed) {
         try {
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
         }
         catch (Throwable ex) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                  "Post-processing of merged bean definition failed", ex);
         }
         mbd.postProcessed = true;
      }
   }

   //向容器中缓存单例模式的Bean对象，以防循环引用
   boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
         isSingletonCurrentlyInCreation(beanName));
   if (earlySingletonExposure) {
      if (logger.isDebugEnabled()) {
         logger.debug("Eagerly caching bean '" + beanName +
               "' to allow for resolving potential circular references");
      }
      //这里是一个匿名内部类，为了防止循环引用，尽早持有对象的引用
      addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
   }

   // Initialize the bean instance.
   //Bean对象的初始化，依赖注入在此触发
   //这个exposedObject在初始化完成之后返回作为依赖注入完成后的Bean
   Object exposedObject = bean;
   try {
      //将Bean实例对象封装，并且Bean定义中配置的属性值赋值给实例对象
      populateBean(beanName, mbd, instanceWrapper);
      //初始化Bean对象
      exposedObject = initializeBean(beanName, exposedObject, mbd);
   }
   catch (Throwable ex) {
      if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
         throw (BeanCreationException) ex;
      }
      else {
         throw new BeanCreationException(
               mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
      }
   }

   if (earlySingletonExposure) {
      //获取指定名称的已注册的单例模式Bean对象
      Object earlySingletonReference = getSingleton(beanName, false);
      if (earlySingletonReference != null) {
         //根据名称获取的已注册的Bean和正在实例化的Bean是同一个
         if (exposedObject == bean) {
            //当前实例化的Bean初始化完成
            exposedObject = earlySingletonReference;
         }
         //当前Bean依赖其他Bean，并且当发生循环引用时不允许新创建实例对象
         else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
            String[] dependentBeans = getDependentBeans(beanName);
            Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
            //获取当前Bean所依赖的其他Bean
            for (String dependentBean : dependentBeans) {
               //对依赖Bean进行类型检查
               if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                  actualDependentBeans.add(dependentBean);
               }
            }
            if (!actualDependentBeans.isEmpty()) {
               throw new BeanCurrentlyInCreationException(beanName,
                     "Bean with name '" + beanName + "' has been injected into other beans [" +
                     StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                     "] in its raw version as part of a circular reference, but has eventually been " +
                     "wrapped. This means that said other beans do not use the final version of the " +
                     "bean. This is often the result of over-eager type matching - consider using " +
                     "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
            }
         }
      }
   }

   // Register bean as disposable.
   //注册完成依赖注入的Bean
   try {
      registerDisposableBeanIfNecessary(beanName, bean, mbd);
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
   }

   return exposedObject;
}
```

通过上面的源码注释，可以看到具体的依赖注入实现其实就在一下两个方法中：

1. createBeanInstance()方法，生成Bean所包含的Java对象实例。
2. populateBean()方法，对Bean属性的依赖注入进行处理。

下面继续分析这两个方法的代码实现。

### 选择Bean实例化策略

在createBeanInstance()方法中，根据指定的初始化策略，使用简单工厂、工厂方法或者容器的自动装配特性生成Java实例对象，创建对象的源码如下：

```java
//创建Bean的实例对象
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
   // Make sure bean class is actually resolved at this point.
   //检查确认Bean是可实例化的
   Class<?> beanClass = resolveBeanClass(mbd, beanName);

   //使用工厂方法对Bean进行实例化
   if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
      throw new BeanCreationException(mbd.getResourceDescription(), beanName,
            "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
   }

   Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
   if (instanceSupplier != null) {
      return obtainFromSupplier(instanceSupplier, beanName);
   }

   if (mbd.getFactoryMethodName() != null)  {
      //调用工厂方法实例化
      return instantiateUsingFactoryMethod(beanName, mbd, args);
   }

   // Shortcut when re-creating the same bean...
   //使用容器的自动装配方法进行实例化
   boolean resolved = false;
   boolean autowireNecessary = false;
   if (args == null) {
      synchronized (mbd.constructorArgumentLock) {
         if (mbd.resolvedConstructorOrFactoryMethod != null) {
            resolved = true;
            autowireNecessary = mbd.constructorArgumentsResolved;
         }
      }
   }
   if (resolved) {
      if (autowireNecessary) {
         //配置了自动装配属性，使用容器的自动装配实例化
         //容器的自动装配是根据参数类型匹配Bean的构造方法
         return autowireConstructor(beanName, mbd, null, null);
      }
      else {
         //使用默认的无参构造方法实例化
         return instantiateBean(beanName, mbd);
      }
   }

   // Need to determine the constructor...
   //使用Bean的构造方法进行实例化
   Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
   if (ctors != null ||
         mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
         mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
      //使用容器的自动装配特性，调用匹配的构造方法实例化
      return autowireConstructor(beanName, mbd, ctors, args);
   }

   // No special handling: simply use no-arg constructor.
   //使用默认的无参构造方法实例化
   return instantiateBean(beanName, mbd);
}
```

```java
//使用默认的无参构造方法实例化Bean对象
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
   try {
      Object beanInstance;
      final BeanFactory parent = this;
      //获取系统的安全管理接口，JDK标准的安全管理API
      if (System.getSecurityManager() != null) {
         //这里是一个匿名内置类，根据实例化策略创建实例对象
         beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
               getInstantiationStrategy().instantiate(mbd, beanName, parent),
               getAccessControlContext());
      }
      else {
         //将实例化的对象封装起来
         beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
      }
      BeanWrapper bw = new BeanWrapperImpl(beanInstance);
      initBeanWrapper(bw);
      return bw;
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
   }
}
```

经过对上面的代码分析，我们可以看出，对使用工厂方法和自动装配特性的Bean的实例化相对比较清楚，调用相应的工厂方法或者参数匹配的构造方法即可完成实例化对象的工作，但是对于我们最常用的默认无参构造方法就需要使用相应的初始化策略（JDK的反射机制或者CGLib）来进行初始化了，在方法getInstantiationStrategy().instantiate()中就实现了使用初始策略实例化对象。

### 执行Bean实例化

在使用默认的无参构造方法创建Bean的实例化对象时，方法getInstantiationStrategy().instantiate()调用了SimpleInstantiationStrategy类中的实例化Bean的方法，其源码如下：

```java
//使用初始化策略实例化Bean对象
@Override
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
   // Don't override the class with CGLIB if no overrides.
   //如果Bean定义中没有方法覆盖，则就不需要CGLIB父类类的方法
   if (!bd.hasMethodOverrides()) {
      Constructor<?> constructorToUse;
      synchronized (bd.constructorArgumentLock) {
         //获取对象的构造方法或工厂方法
         constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
         //如果没有构造方法且没有工厂方法
         if (constructorToUse == null) {
            //使用JDK的反射机制，判断要实例化的Bean是否是接口
            final Class<?> clazz = bd.getBeanClass();
            if (clazz.isInterface()) {
               throw new BeanInstantiationException(clazz, "Specified class is an interface");
            }
            try {
               if (System.getSecurityManager() != null) {
                  //这里是一个匿名内置类，使用反射机制获取Bean的构造方法
                  constructorToUse = AccessController.doPrivileged(
                        (PrivilegedExceptionAction<Constructor<?>>) () -> clazz.getDeclaredConstructor());
               }
               else {
                  constructorToUse = clazz.getDeclaredConstructor();
               }
               bd.resolvedConstructorOrFactoryMethod = constructorToUse;
            }
            catch (Throwable ex) {
               throw new BeanInstantiationException(clazz, "No default constructor found", ex);
            }
         }
      }
      //使用BeanUtils实例化，通过反射机制调用”构造方法.newInstance(arg)”来进行实例化
      return BeanUtils.instantiateClass(constructorToUse);
   }
   else {
      // Must generate CGLIB subclass.
      //使用CGLIB来实例化对象
      return instantiateWithMethodInjection(bd, beanName, owner);
   }
}
```

通过上面的代码分析，我们看到了如果Bean有方法被覆盖了，则使用JDK的反射机制进行实例化，否则，使用CGLib进行实例化。



instantiateWithMethodInjection() 方 法 调 用 SimpleInstantiationStrategy 的 子 类 CGLibSubclassingInstantiationStrategy 使用 CGLib 来进行初始化，其源码如下:

```java
		//使用CGLIB进行Bean对象实例化
		public Object instantiate(@Nullable Constructor<?> ctor, @Nullable Object... args) {
			//创建代理子类
			Class<?> subclass = createEnhancedSubclass(this.beanDefinition);
			Object instance;
			if (ctor == null) {
				instance = BeanUtils.instantiateClass(subclass);
			}
			else {
				try {
					Constructor<?> enhancedSubclassConstructor = subclass.getConstructor(ctor.getParameterTypes());
					instance = enhancedSubclassConstructor.newInstance(args);
				}
				catch (Exception ex) {
					throw new BeanInstantiationException(this.beanDefinition.getBeanClass(),
							"Failed to invoke constructor for CGLIB enhanced subclass [" + subclass.getName() + "]", ex);
				}
			}
			// SPR-10785: set callbacks directly on the instance instead of in the
			// enhanced class (via the Enhancer) in order to avoid memory leaks.
			Factory factory = (Factory) instance;
			factory.setCallbacks(new Callback[] {NoOp.INSTANCE,
					new LookupOverrideMethodInterceptor(this.beanDefinition, this.owner),
					new ReplaceOverrideMethodInterceptor(this.beanDefinition, this.owner)});
			return instance;
		}

		private Class<?> createEnhancedSubclass(RootBeanDefinition beanDefinition) {
			//CGLIB中的类
			Enhancer enhancer = new Enhancer();
			//将Bean本身作为其基类
			enhancer.setSuperclass(beanDefinition.getBeanClass());
			enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
			if (this.owner instanceof ConfigurableBeanFactory) {
				ClassLoader cl = ((ConfigurableBeanFactory) this.owner).getBeanClassLoader();
				enhancer.setStrategy(new ClassLoaderAwareGeneratorStrategy(cl));
			}
			enhancer.setCallbackFilter(new MethodOverrideCallbackFilter(beanDefinition));
			enhancer.setCallbackTypes(CALLBACK_TYPES);
			//使用CGLIB的createClass方法生成实例对象
			return enhancer.createClass();
		}
```

CGLib 是一个常用的字节码生成器的类库，它提供了一系列 API 实现 Java 字节码的生成和转换功能。 我们在学习 JDK 的动态代理时都知道，JDK 的动态代理只能针对接口，如果一个类没有实现任何接口， 要对其进行动态代理只能使用 CGLib。

### 准备依赖注入

在前面的分析中我们已经了解到Bean的依赖注入主要分为两个步骤：首先调用createBeanInstance()方法生成Bean所包含的Java对象实例。然后调用populateBean()方法，对Bean属性的依赖注入进行处理。

上面我们已经分析了容器初始化生成Bean所包含的Java实例对象的过程，现在我们继续分析生成对象后，Spring IOC容器是如何将Bean的属性依赖关系注入Bean实例对象中并设置好的，回到AbstractAutowireCapableBeanFactory的popolateBean方法，对属性依赖注入的代码如下：

```java
//将Bean属性设置到生成的实例对象上
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
   if (bw == null) {
      if (mbd.hasPropertyValues()) {
         throw new BeanCreationException(
               mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
      }
      else {
         // Skip property population phase for null instance.
         return;
      }
   }

   // Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
   // state of the bean before properties are set. This can be used, for example,
   // to support styles of field injection.
   boolean continueWithPropertyPopulation = true;

   if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
         if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
               continueWithPropertyPopulation = false;
               break;
            }
         }
      }
   }

   if (!continueWithPropertyPopulation) {
      return;
   }
   //获取容器在解析Bean定义资源时为BeanDefiniton中设置的属性值
   PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

   //对依赖注入处理，首先处理autowiring自动装配的依赖注入
   if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
         mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
      MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

      // Add property values based on autowire by name if applicable.
      //根据Bean名称进行autowiring自动装配处理
      if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
         autowireByName(beanName, mbd, bw, newPvs);
      }

      // Add property values based on autowire by type if applicable.
      //根据Bean类型进行autowiring自动装配处理
      if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
         autowireByType(beanName, mbd, bw, newPvs);
      }

      pvs = newPvs;
   }

   //对非autowiring的属性进行依赖注入处理

   boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
   boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

   if (hasInstAwareBpps || needsDepCheck) {
      if (pvs == null) {
         pvs = mbd.getPropertyValues();
      }
      PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
      if (hasInstAwareBpps) {
         for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
               InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
               pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
               if (pvs == null) {
                  return;
               }
            }
         }
      }
      if (needsDepCheck) {
         checkDependencies(beanName, mbd, filteredPds, pvs);
      }
   }

   if (pvs != null) {
      //对属性进行注入
      applyPropertyValues(beanName, mbd, bw, pvs);
   }
}
```

```java
//解析并注入依赖属性的过程
protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
   if (pvs.isEmpty()) {
      return;
   }

   //封装属性值
   MutablePropertyValues mpvs = null;
   List<PropertyValue> original;

   if (System.getSecurityManager() != null) {
      if (bw instanceof BeanWrapperImpl) {
         //设置安全上下文，JDK安全机制
         ((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
      }
   }

   if (pvs instanceof MutablePropertyValues) {
      mpvs = (MutablePropertyValues) pvs;
      //属性值已经转换
      if (mpvs.isConverted()) {
         // Shortcut: use the pre-converted values as-is.
         try {
            //为实例化对象设置属性值
            bw.setPropertyValues(mpvs);
            return;
         }
         catch (BeansException ex) {
            throw new BeanCreationException(
                  mbd.getResourceDescription(), beanName, "Error setting property values", ex);
         }
      }
      //获取属性值对象的原始类型值
      original = mpvs.getPropertyValueList();
   }
   else {
      original = Arrays.asList(pvs.getPropertyValues());
   }

   //获取用户自定义的类型转换
   TypeConverter converter = getCustomTypeConverter();
   if (converter == null) {
      converter = bw;
   }
   //创建一个Bean定义属性值解析器，将Bean定义中的属性值解析为Bean实例对象的实际值
   BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

   // Create a deep copy, resolving any references for values.

   //为属性的解析值创建一个拷贝，将拷贝的数据注入到实例对象中
   List<PropertyValue> deepCopy = new ArrayList<>(original.size());
   boolean resolveNecessary = false;
   for (PropertyValue pv : original) {
      //属性值不需要转换
      if (pv.isConverted()) {
         deepCopy.add(pv);
      }
      //属性值需要转换
      else {
         String propertyName = pv.getName();
         //原始的属性值，即转换之前的属性值
         Object originalValue = pv.getValue();
         //转换属性值，例如将引用转换为IOC容器中实例化对象引用
         Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
         //转换之后的属性值
         Object convertedValue = resolvedValue;
         //属性值是否可以转换
         boolean convertible = bw.isWritableProperty(propertyName) &&
               !PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
         if (convertible) {
            //使用用户自定义的类型转换器转换属性值
            convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
         }
         // Possibly store converted value in merged bean definition,
         // in order to avoid re-conversion for every created bean instance.
         //存储转换后的属性值，避免每次属性注入时的转换工作
         if (resolvedValue == originalValue) {
            if (convertible) {
               //设置属性转换之后的值
               pv.setConvertedValue(convertedValue);
            }
            deepCopy.add(pv);
         }
         //属性是可转换的，且属性原始值是字符串类型，且属性的原始类型值不是
         //动态生成的字符串，且属性的原始值不是集合或者数组类型
         else if (convertible && originalValue instanceof TypedStringValue &&
               !((TypedStringValue) originalValue).isDynamic() &&
               !(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
            pv.setConvertedValue(convertedValue);
            //重新封装属性的值
            deepCopy.add(pv);
         }
         else {
            resolveNecessary = true;
            deepCopy.add(new PropertyValue(pv, convertedValue));
         }
      }
   }
   if (mpvs != null && !resolveNecessary) {
      //标记属性值已经转换过
      mpvs.setConverted();
   }

   // Set our (possibly massaged) deep copy.
   //进行属性依赖注入
   try {
      bw.setPropertyValues(new MutablePropertyValues(deepCopy));
   }
   catch (BeansException ex) {
      throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Error setting property values", ex);
   }
}
```

分析上述代码，我们可以看出，对属性的注入过程分一下两种情况：

1. 属性值类型不需要强制转换时，不需要解析属性值，直接准备进行依赖注入。
2. 属性值需要进行类型强制装换时，如对其他对象的引用等，首先需要解析属性值，然后对解析后的属性值进行依赖注入。

对实行值的解析是在BeanDefinitionValueResolver类中的resolveValueIfNecessary()方法中进行的，对属性值的依赖注入是通过bw.setPropertyValues()方法实现的，在分析属性值的依赖注入之前，我们先分析一下对属性值的解析过程。



### 解析属性注入规则

当容器在对属性进行依赖注入时，如果发现属性值需要进行类型转换，如属性值是容器中另一个Bean实例对象的引用，则容器首先需要根据属性值解析出所引用的对象，然后才能将该引用对象注入到目标实例对象的属性上去，对属性进行解析的由resolveValueIfNecessary()方法实现，其源码如下：

```java
//解析属性值，对注入类型进行转换
@Nullable
public Object resolveValueIfNecessary(Object argName, @Nullable Object value) {
   // We must check each value to see whether it requires a runtime reference
   // to another bean to be resolved.
   //对引用类型的属性进行解析
   if (value instanceof RuntimeBeanReference) {
      RuntimeBeanReference ref = (RuntimeBeanReference) value;
      //调用引用类型属性的解析方法
      return resolveReference(argName, ref);
   }
   //对属性值是引用容器中另一个Bean名称的解析
   else if (value instanceof RuntimeBeanNameReference) {
      String refName = ((RuntimeBeanNameReference) value).getBeanName();
      refName = String.valueOf(doEvaluate(refName));
      //从容器中获取指定名称的Bean
      if (!this.beanFactory.containsBean(refName)) {
         throw new BeanDefinitionStoreException(
               "Invalid bean name '" + refName + "' in bean reference for " + argName);
      }
      return refName;
   }
   //对Bean类型属性的解析，主要是Bean中的内部类
   else if (value instanceof BeanDefinitionHolder) {
      // Resolve BeanDefinitionHolder: contains BeanDefinition with name and aliases.
      BeanDefinitionHolder bdHolder = (BeanDefinitionHolder) value;
      return resolveInnerBean(argName, bdHolder.getBeanName(), bdHolder.getBeanDefinition());
   }
   else if (value instanceof BeanDefinition) {
      // Resolve plain BeanDefinition, without contained name: use dummy name.
      BeanDefinition bd = (BeanDefinition) value;
      String innerBeanName = "(inner bean)" + BeanFactoryUtils.GENERATED_BEAN_NAME_SEPARATOR +
            ObjectUtils.getIdentityHexString(bd);
      return resolveInnerBean(argName, innerBeanName, bd);
   }
   //对集合数组类型的属性解析
   else if (value instanceof ManagedArray) {
      // May need to resolve contained runtime references.
      ManagedArray array = (ManagedArray) value;
      //获取数组的类型
      Class<?> elementType = array.resolvedElementType;
      if (elementType == null) {
         //获取数组元素的类型
         String elementTypeName = array.getElementTypeName();
         if (StringUtils.hasText(elementTypeName)) {
            try {
               //使用反射机制创建指定类型的对象
               elementType = ClassUtils.forName(elementTypeName, this.beanFactory.getBeanClassLoader());
               array.resolvedElementType = elementType;
            }
            catch (Throwable ex) {
               // Improve the message by showing the context.
               throw new BeanCreationException(
                     this.beanDefinition.getResourceDescription(), this.beanName,
                     "Error resolving array type for " + argName, ex);
            }
         }
         //没有获取到数组的类型，也没有获取到数组元素的类型
         //则直接设置数组的类型为Object
         else {
            elementType = Object.class;
         }
      }
      //创建指定类型的数组
      return resolveManagedArray(argName, (List<?>) value, elementType);
   }
   //解析list类型的属性值
   else if (value instanceof ManagedList) {
      // May need to resolve contained runtime references.
      return resolveManagedList(argName, (List<?>) value);
   }
   //解析set类型的属性值
   else if (value instanceof ManagedSet) {
      // May need to resolve contained runtime references.
      return resolveManagedSet(argName, (Set<?>) value);
   }
   //解析map类型的属性值
   else if (value instanceof ManagedMap) {
      // May need to resolve contained runtime references.
      return resolveManagedMap(argName, (Map<?, ?>) value);
   }
   //解析props类型的属性值，props其实就是key和value均为字符串的map
   else if (value instanceof ManagedProperties) {
      Properties original = (Properties) value;
      //创建一个拷贝，用于作为解析后的返回值
      Properties copy = new Properties();
      original.forEach((propKey, propValue) -> {
         if (propKey instanceof TypedStringValue) {
            propKey = evaluate((TypedStringValue) propKey);
         }
         if (propValue instanceof TypedStringValue) {
            propValue = evaluate((TypedStringValue) propValue);
         }
         if (propKey == null || propValue == null) {
            throw new BeanCreationException(
                  this.beanDefinition.getResourceDescription(), this.beanName,
                  "Error converting Properties key/value pair for " + argName + ": resolved to null");
         }
         copy.put(propKey, propValue);
      });
      return copy;
   }
   //解析字符串类型的属性值
   else if (value instanceof TypedStringValue) {
      // Convert value to target type here.
      TypedStringValue typedStringValue = (TypedStringValue) value;
      Object valueObject = evaluate(typedStringValue);
      try {
         //获取属性的目标类型
         Class<?> resolvedTargetType = resolveTargetType(typedStringValue);
         if (resolvedTargetType != null) {
            //对目标类型的属性进行解析，递归调用
            return this.typeConverter.convertIfNecessary(valueObject, resolvedTargetType);
         }
         //没有获取到属性的目标对象，则按Object类型返回
         else {
            return valueObject;
         }
      }
      catch (Throwable ex) {
         // Improve the message by showing the context.
         throw new BeanCreationException(
               this.beanDefinition.getResourceDescription(), this.beanName,
               "Error converting typed String value for " + argName, ex);
      }
   }
   else if (value instanceof NullBean) {
      return null;
   }
   else {
      return evaluate(value);
   }
}
```

通过上面的代码分析，我们明白了Spring是如何将引用类型，内部类以及集合类型等进行解析的，属性值解析完成后就可以进行依赖注入了，依赖注入的过程就是Bean对象实例设置到它所依赖的Bean对象属性上去。而真正的依赖注入是通过bw.setPropertyValues()方法试想，该方法也使用了委托模式，在BeanWrapper接口中至少定义了方法声明，依赖注入的具体实现交由其实现类BeanWrapperImpl来完成，下面我们就分析BeanWrapperImpl中依赖注入相关的源码。

### 注入赋值

BeanWrapperImpl类主要是对容器中完成初始化的Bean实例对象进行属性的依赖注入，即把Bean对象设置到它所依赖的另一个Bean的属性中去。然而，BeanWrapperImpl中的注入方法实际上有AbstractNestablePropertyAccessor来实现的，其相关源码如下：

```java
//实现属性依赖注入功能
protected void setPropertyValue(PropertyTokenHolder tokens, PropertyValue pv) throws BeansException {
   if (tokens.keys != null) {
      processKeyedProperty(tokens, pv);
   }
   else {
      processLocalProperty(tokens, pv);
   }
}

//实现属性依赖注入功能
@SuppressWarnings("unchecked")
private void processKeyedProperty(PropertyTokenHolder tokens, PropertyValue pv) {
   //调用属性的getter方法，获取属性的值
   Object propValue = getPropertyHoldingValue(tokens);
   PropertyHandler ph = getLocalPropertyHandler(tokens.actualName);
   if (ph == null) {
      throw new InvalidPropertyException(
            getRootClass(), this.nestedPath + tokens.actualName, "No property handler found");
   }
   Assert.state(tokens.keys != null, "No token keys");
   String lastKey = tokens.keys[tokens.keys.length - 1];

   //注入array类型的属性值
   if (propValue.getClass().isArray()) {
      Class<?> requiredType = propValue.getClass().getComponentType();
      int arrayIndex = Integer.parseInt(lastKey);
      Object oldValue = null;
      try {
         if (isExtractOldValueForEditor() && arrayIndex < Array.getLength(propValue)) {
            oldValue = Array.get(propValue, arrayIndex);
         }
         Object convertedValue = convertIfNecessary(tokens.canonicalName, oldValue, pv.getValue(),
               requiredType, ph.nested(tokens.keys.length));
         //获取集合类型属性的长度
         int length = Array.getLength(propValue);
         if (arrayIndex >= length && arrayIndex < this.autoGrowCollectionLimit) {
            Class<?> componentType = propValue.getClass().getComponentType();
            Object newArray = Array.newInstance(componentType, arrayIndex + 1);
            System.arraycopy(propValue, 0, newArray, 0, length);
            setPropertyValue(tokens.actualName, newArray);
            //调用属性的getter方法，获取属性的值
            propValue = getPropertyValue(tokens.actualName);
         }
         //将属性的值赋值给数组中的元素
         Array.set(propValue, arrayIndex, convertedValue);
      }
      catch (IndexOutOfBoundsException ex) {
         throw new InvalidPropertyException(getRootClass(), this.nestedPath + tokens.canonicalName,
               "Invalid array index in property path '" + tokens.canonicalName + "'", ex);
      }
   }

   //注入list类型的属性值
   else if (propValue instanceof List) {
      //获取list集合的类型
      Class<?> requiredType = ph.getCollectionType(tokens.keys.length);
      List<Object> list = (List<Object>) propValue;
      //获取list集合的size
      int index = Integer.parseInt(lastKey);
      Object oldValue = null;
      if (isExtractOldValueForEditor() && index < list.size()) {
         oldValue = list.get(index);
      }
      //获取list解析后的属性值
      Object convertedValue = convertIfNecessary(tokens.canonicalName, oldValue, pv.getValue(),
            requiredType, ph.nested(tokens.keys.length));
      int size = list.size();
      //如果list的长度大于属性值的长度，则多余的元素赋值为null
      if (index >= size && index < this.autoGrowCollectionLimit) {
         for (int i = size; i < index; i++) {
            try {
               list.add(null);
            }
            catch (NullPointerException ex) {
               throw new InvalidPropertyException(getRootClass(), this.nestedPath + tokens.canonicalName,
                     "Cannot set element with index " + index + " in List of size " +
                     size + ", accessed using property path '" + tokens.canonicalName +
                     "': List does not support filling up gaps with null elements");
            }
         }
         list.add(convertedValue);
      }
      else {
         try {
            //将值添加到list中
            list.set(index, convertedValue);
         }
         catch (IndexOutOfBoundsException ex) {
            throw new InvalidPropertyException(getRootClass(), this.nestedPath + tokens.canonicalName,
                  "Invalid list index in property path '" + tokens.canonicalName + "'", ex);
         }
      }
   }

   //注入map类型的属性值
   else if (propValue instanceof Map) {
      //获取map集合key的类型
      Class<?> mapKeyType = ph.getMapKeyType(tokens.keys.length);
      //获取map集合value的类型
      Class<?> mapValueType = ph.getMapValueType(tokens.keys.length);
      Map<Object, Object> map = (Map<Object, Object>) propValue;
      // IMPORTANT: Do not pass full property name in here - property editors
      // must not kick in for map keys but rather only for map values.
      TypeDescriptor typeDescriptor = TypeDescriptor.valueOf(mapKeyType);
      //解析map类型属性key值
      Object convertedMapKey = convertIfNecessary(null, null, lastKey, mapKeyType, typeDescriptor);
      Object oldValue = null;
      if (isExtractOldValueForEditor()) {
         oldValue = map.get(convertedMapKey);
      }
      // Pass full property name and old value in here, since we want full
      // conversion ability for map values.
      //解析map类型属性value值
      Object convertedMapValue = convertIfNecessary(tokens.canonicalName, oldValue, pv.getValue(),
            mapValueType, ph.nested(tokens.keys.length));
      //将解析后的key和value值赋值给map集合属性
      map.put(convertedMapKey, convertedMapValue);
   }

   else {
      throw new InvalidPropertyException(getRootClass(), this.nestedPath + tokens.canonicalName,
            "Property referenced in indexed property path '" + tokens.canonicalName +
            "' is neither an array nor a List nor a Map; returned value was [" + propValue + "]");
   }
}

private Object getPropertyHoldingValue(PropertyTokenHolder tokens) {
   // Apply indexes and map keys: fetch value for all keys but the last one.
   Assert.state(tokens.keys != null, "No token keys");
   PropertyTokenHolder getterTokens = new PropertyTokenHolder(tokens.actualName);
   getterTokens.canonicalName = tokens.canonicalName;
   getterTokens.keys = new String[tokens.keys.length - 1];
   System.arraycopy(tokens.keys, 0, getterTokens.keys, 0, tokens.keys.length - 1);

   Object propValue;
   try {
      //获取属性值
      propValue = getPropertyValue(getterTokens);
   }
   catch (NotReadablePropertyException ex) {
      throw new NotWritablePropertyException(getRootClass(), this.nestedPath + tokens.canonicalName,
            "Cannot access indexed value in property referenced " +
            "in indexed property path '" + tokens.canonicalName + "'", ex);
   }

   if (propValue == null) {
      // null map value case
      if (isAutoGrowNestedPaths()) {
         int lastKeyIndex = tokens.canonicalName.lastIndexOf('[');
         getterTokens.canonicalName = tokens.canonicalName.substring(0, lastKeyIndex);
         propValue = setDefaultValue(getterTokens);
      }
      else {
         throw new NullValueInNestedPathException(getRootClass(), this.nestedPath + tokens.canonicalName,
               "Cannot access indexed value in property referenced " +
               "in indexed property path '" + tokens.canonicalName + "': returned null");
      }
   }
   return propValue;
}
```

通过对上面注入依赖代码的分析，我们已经明白了Sping IOC容器是如何将属性的值注入到Bean实例对象中去的：

1. 对于集合类型的属性，将其属性值解析为目标类型的集合后直接赋值给属性。
2. 对于非集合类型的属性，大量使用了JDK的反射机制，通过属性的getter()方法获取指定属性注入以前的值，同时调用属性的setter()方法为属性设置注入后的值。看到这里相信很多人都明白了Spring的setter()注入原理。

至此 Spring IOC 容器对 Bean 定义资源文件的定位，载入、解析和依赖注入已经全部分析完毕，现在 Spring IOC 容器中管理了一系列靠依赖关系联系起来的 Bean，程序不需要应用自己手动创建所需的对 象，Spring IOC 容器会在我们使用的时候自动为我们创建，并且为我们注入好相关的依赖，这就是 Spring 核心功能的控制反转和依赖注入的相关功能。





