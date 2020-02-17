# Spring Boot的基本认识

不管是Spring Cloud Alibaba还是Spring Cloud Netflix，都是基于Spring Boot 这个微框架来构建的。本篇只对Spring Boot做一个基本认知。后续会对Spring 的自动装配做详细的源码分析。另外推荐小马哥的《Spring Boot 编程思想》（核心篇）。对Spring Boot 做了详尽的分析。

## 什么是Spring Boot

对于Spring框架，我们接触得比较多的应该是Spring MVC和Spring。而Spring的核心在于IOC（控制反转）和DI（依赖注入）。而这些框架在使用的过程中会需要配置大量的xml，或者需要做很多繁琐的配置。

Spring Boot框架是为了能够帮助使用Spring框架的开发者快速高效的构建一个基于Spring框架以及Spring生态体系的应用解决方案。它是对”约定由于配置“这个理念下的一个最佳实践。因此它是一个服务于框架的框架，服务的范围是简化配置文件。

### 约定由于配置的体现

1. Maven的目录结构
   - 默认由resources文件夹存放配置文件。
   - 默认打包方式为jar。
2. spring-boot-starter-web中默认包含Spring MVC相关依赖以及内置的tomcat容器，使得构建一个web应用更加简单。
3. 默认提供application.properties/yml文件。
4. 默认通过spring.profiles.active属性来决定运行环境时独缺的配置文件。
5. EnableAutoConfiguration默认对于依赖的starter进行自动装载。

### 从SpringBootApplication注解入手

为了揭开Spring Boot的奥秘，我们直接从Annotation入手，看着@SpringBootApplication里面，做了什么？

打开SpringBootApplication这个注解，可以看到它实际上是一个复合注解：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
      @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
@ConfigurationPropertiesScan
public @interface SpringBootApplication {
}
```

SpringBootApplication本质上是由3个注解组成，分别是：

1. @Configuration
2. @EnableAutoConfiguration
3. @ComponentScan

我们可以直接用这三个注解也可以启动SpringBoot应用，只是每次配置三个注解比较繁琐，所以直接用一个服务注解更方便一些。

仔细观察这三个注解，除了@EnableAutoConfiguration可能稍微陌生一点，其他两个注解使用得都很多。

#### 简单分析@Configuration

@Configuration这个注解大家应该有用过，它是JavaConfig形式的基于Spring IOC容器的配置类使用的一种注解。因为SpringBoot本质上就是一个Spring应用，所以通过这个注解来加载IOC容器的配置是很正常的。所以在启动类里面标注了@Configuration，意味着它其实也是一个IOC容器的配置类。

传统意义上的Spring应用都是基于xml形式来配置bean的依赖关系。然后通过Spring容器在启动的时候，把Bean进行初始化并且，如果Bean之间存在依赖关系，则分析这些已经在IOC容器中的Bean根据依赖关系进行组装。直到Java5中，引入了Annotations这个特性，Spring框架也紧随潮流退出了基于Java代码和Annotation元信息的依赖关系绑定描述的方式。也就是JavaConfig。

从Spring3开始，Spring就支持了两种Bean的配置方式，一种是基于xml文件方式，另一种是JavaConfig任何一个标注了@Configuration的Java类定义都是一个JavaConfig配置类。而在这个配置类中，任何标注了@Bean的方法，它的返回值都会作为Bean定义注册到Spring的IOC容器，方法名默认称为这个Bean的Id。

#### 简单分析@ComponentScan

@ComponentScan这个注解是大家接触得最多的了，相当于xml配置文件中的```<context:component-scan />```。它的主要作用就是扫描指定路径下标识了需要装配的类，自动装配到Spring的IOC容器中。

标识需要装配的类的形式主要是：@Component、@Repository、@Service、@Controller这类的注解标识的类。

@ComponentScan默认会扫描当前package下的所有加了相关注解标识的类到IOC容器中。

#### 简单分析@EnableAutoConfiguration

把@EnableAutoConfiguration放在最后将的目的并不是说它是一个新东西，只是他对于SpringBoot来说意义重大。

##### Enable*并不是新东西

仍然是在Spring 3.1版本中，提供了一系列的@Enable开头的注解，@Enable开头的注解应该是在JavaConfig框架上更进一步的完善，使用户在使用Spring相关的框架时，避免配置大量的代码从而降低使用的难道。

比如常见的一些@Enable注解：@EnableWebMvc（这个注解引入了MVC框架在Spring应用中需要用到的所有Bean）。

比如@EnableScheduling，开支计划任务的支持。

找到EnableAutoConfiguration，我们可以看到每一个设计到@Enable开头的注解，都会带有一个@Import的注解：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
}
```

###### Import注解

xml形式下有一个```<improt resource/>```标签，@Import就是和这个标签的作用是一样的。可以把多个分开的配置合并在一个配置中。

#### 深入分析@EnableAutoConfiguration

@EnableAutoCOnfiguration的主要作用其实就是帮助Spring Boot应用把所有符合条件的@Configuration配置都加载到当前SpringBoot创建并使用的IOC容器中。

在回到@EnableAutoConfiguration这个注解中，我们发现它的import是这样的：

```java
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
```

##### AutoConfigurationImportSelector 

@Enable注解不仅仅可以很简单的实现多个Configuration的整合，还可以实现一些复杂的场景，比如可以根据上下文来激活不同类型的Bean，@Import注解可以配置三种不同的class:

1. 第一种就是前面演示过的，基于普通Bean或者带有@Configuration的Bean进行注入。
2. 实现ImportSelector接口进行动态注入。
3. 实现ImprotBeanDefinitionRegistrar接口进行动态注入。

##### @EnableAutoConfiguration注解实现原理

了解了ImportSelector和ImprotBeanDefinitionRegistrar后，对于@EnableAutoConfiguration的理解就容易一些了。 它会通过improt导入第三方提供的Bean配置类：

```java
@Import(AutoConfigurationImportSelector.class)
```

从名字来看，可以猜到它是基于ImportSelector来实现基于动态Bean的加载功能。之前我们讲过ImportSelector接口selectImports方法返回的数组（类的全类名）都会被纳入到Spring容器中。

那么可以猜想到这里的实现原理也是一样的，定位到AutoConfigurationImportSelector这个类中的selectImport方法：

```java
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
   if (!isEnabled(annotationMetadata)) {
      return NO_IMPORTS;
   }
   AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
         .loadMetadata(this.beanClassLoader);
   AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(autoConfigurationMetadata,
         annotationMetadata);
   return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
}
```

这里返回的数组就是需要纳入到Spring IOC容器中的类。

本质上说，其实EnableAutoConfiguration会帮助Spring Boot应用把所有符合@Configuration配置都加载到当前SpringBoot创建的IOC容器，而这里面借助了Spring框架提供的一个工具类SpringFactoriesLoader的支持。以及用到了Spring提供的条件注解。

@Conditional，选择性的针对需要加载的Bean进行条件过滤

。

##### SpringFactoriesLoader

SpringFactoriesLoader其实和Java中的SPI机制的原理是一样的，不过它比SPI更好的点在于不会一次性加载所有的类，而是根据key进行加载。

首先，SpringFactoriesLoader的作用是从classpath/META-INF/spring.factories文件中，根据key来加载对应的累到Spring IOC容器中。

##### 深入理解条件过滤

AutoConfigurationSelector的源码实现中，会先扫描Spring-autoconfiguration-metadata.properties文件，最后在扫描spring.factories对应的类，会结合前面的元数据进行过滤，为什么要过滤呢？原因是很多的@Configuration其实是依托于其他的框架来加载的，如果当前的classpath环境下没有相关的依赖，则意味着这些类没必要进行加载，所以，通过这种条件过滤可以有效的减少@Configuration类的数量从而降低SpringBott的启动时间。

##### Conditional中的其他注解

| Conditions                   | 描述                                    |
| ---------------------------- | --------------------------------------- |
| @ConditionalOnBean           | IOC容器中存在某个Bean的时候             |
| @ConditionalOnMissBean       | IOC容器中不存在某个Bean的时候           |
| @ConditionalOnClass          | 当前classpath可以找到某个类时           |
| @ConditionalOnMissClass      | 当前classpath不可以找到某个类时         |
| @ConditionalOnResource       | 当前 classpath 是否存在某个资源文件     |
| @ConditionalOnProperty       | 当前 jvm 是否包含某个系统属性为某个值   |
| @ConditionalOnWebApplication | 当前 spring context 是否是 web 应用程序 |

### 什么是Starter

Strater是Spring Boot中的一个非常重要的概念，Starter相当于模块，它能将模块所需的依赖整合起来并对模块内的Bean根据环境（条件）进行自动配置。使用者只需要依赖响应功能的Starter，无需做过多的配置和依赖，SpringBoot就能自动扫描并加载相应的模块。