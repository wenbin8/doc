# Spring核心概念与系统架构

## Spring的设计初衷

Spring是为解决企业级应用的复杂性而设计，它可以做很多事。但归根到底支撑Spring的仅仅是少许的基本理念，而所有的这些基本理念都能追溯到一个最根本的使命：简化开发。这是一个郑重的承诺，其实许多框架都声称在某些方面做了简化。而Spring则立志于全方面的简化Java开发。对此，它主要采取了4个关键策略：

1. 基于POJO的轻量级和最小侵入性编程。
2. 通过依赖注入和面向接口声明式编程。
3. 基于切面和惯性进行声明式编程。
4. 通过切面和模板减少样板式代码。

主要通过：面向Bean（BOP）、依赖注入（DI）、控制反转（IOC）以及面向切面（AOP）这些方式来达成的。

### BOP编程

Spring是面向Bean的编程（Bean Oriented Programming, BOP），Bean在Spring中才是真正的主角。Bean在Spring中的作用就像Object对OOP的意义一样，Spring中没有Bean也就没有Spring存在的意义。Spring提供了IOC容器通过配置文件或者注解的方式来管理对象之间的依赖关系。

控制反转，其中最常见的实现方式叫做依赖注入（Dependency Injection, DI），还有一种方式叫“依赖查找”（Dependency Lookup, DL），它在C++、Java、PHP以及.net中都有运用。在最早的Spring中是包含有依赖注入方法和依赖查询的，但因为依赖查询使用频率过低，不久就被Spring移除了，所以在Spring中控制反转也被直接称作依赖注入，它的基本概念是：不创建对象，但是描述创建对象的方式。在代码中不直接与对象和服务连接，但在配置文件中面熟哪一个组件需要那一项服务。容器（IOC容器）负责将这些联系在一起。

​	在典型的IOC场景中，容器创建了所有对象，并设置必要的属性将它们连接在一起，决定什么事件调用方法。

### 依赖注入

Spring设计的核心```org.springframework.beans```包(架构核心是```org.springframework.core```包)，它的设计目标是与JavaBean组件一起使用。这个包通常不是由用户直接使用，而是由服务器将其用作其他多数功能的底层中介。下一个高级抽象是```BeanFactory```接口，它是工厂设计模式的实现，允许通过名称创建和检索对象。```BeanFactory```也可以管理最想之间的关系。

```BeanFactory```最底层支持两个对象模型：

1. 单例：提供了具有特定名称的全局共享实例对象，可以在查询时对其进行检索。Singleton是默认的也是最常用的对象模型。
2. 原型：确保每次检索都会创建单独的实例对象。在每个用户都需要自己的对象时，采用原型模式。

Bean工厂的概念是Spring作为IOC容器的基础。IOC则将处理事情的责任从应用程序代码转移到框架。

### AOP编程

​	面向切面编程，即AOP，是一种编程思想，它允许程序员对横切关注点或者横切典型的职责分界线的行为（例如日志和事务管理）进行模块化。AOP的核心构造是方面（切面），它将那些影响多个类的行为封装到可重用的模块中。

​	AOP和IOC是补充性的技术，它们都运用模块化方式解决企业应用程序开发中的复杂问题。在典型的面向对象开发中，可能要将日志记录语句放在所有方法和Java类中才能实现日志功能。在AOP方式中，可以反过来将日志服务模块化，并以声明的方式将它们应用到需要日志的组件上。当然，优势就是Java类不需要知道日志服务的存在，也不需要考虑相关代码。所以，用Spring AOP编写的应用程序代码是松耦合的。

AOP的功能完成集成到了Spring事务管理、日志和其他各种特性的上下文中。

AOP编程的常用场景有：Authentication(权限认证)、Auto Caching(自动缓存处理)、Error Handling(统一错误处理)、Debugging（调试信息输出）、Logging（日志记录）、Transactions（事务处理）等。

## Spring5 系统架构

Spring总共大约有20个模块，由1300多个不同的文件构成。而这些组件被分别整合在核心容器（Core Container）、AOP（Aspect Oriented Programming）和设备支持（Instrmentation）、数据访问及集成（Data Access/Integeration）、Web、报文发送（Messaging）、Test，6个体模块集合中。以下是Spring5的模块结构图：

![image-20190920161749806](/Users/dongwenbin/github/doc/Spring/assets/image-20190920161749806.png)

组成Spring框架的每个模块集合或者模块都可以单独存在，也可以一个或多个模块联合实现。

### Spring模块的组成和功能

#### 核心容器

核心容器由spring-beans、spring-core、spring-context和spring-expression,4个模块组成。

spring-core和spring-beans模块是Spring框架的核心模块，包含了控制反转（Inversion of Control）和依赖注入（Denpendency Injection,DI）。BeanFactory接口是Spring框架中的核心接口，它是工厂模式的具体实现。BeanFactory使用控制反转对应用程序的配置和依赖规范与实际应用程序代码进行了分离。但BeanFactory容器实例化后并不会自动实例化Bean，只有当Bean被使用时BeanFactory容器才会对该Bean进行实例化与依赖关系装配。

spring-context模块构架与核心模块之上，他扩展了BeanFactory，为BeanFactory添加了Bean生命周期控制、框架事件体系以及资源加载透明化等功能。此外该模块还提供了许多企业级支持，如邮件访问、远程访问、任务调度等，ApplicationContext是该模块的核心接口，他的超累是BeanFactory。与BeanFactory不同，ApplicationContext容器实例化后会自动对所有的单实例Bean进行实例化与依赖关系装配，使之处于待用状态。

spring-context-support模块是对Spring IOC容器的扩展支持，以及IOC子容器。

spring-context-indexer模块是Spring的类管理组件和Classpath扫描。

spring-expression模块是统一表达式语言（EL）的扩展模块，可以查询、管理运行中的对象，同时也可以方便的调用对象方法、操作数组、集合等。它的语法类似于传统EL，但是提供了额外的功能，最出色的要数函数调用和简单字符串的模板函数。这种语言的特性是基于Spring产品的需求而设计，它可以非常方便的和Spring IOC进行交互。

#### AOP和设备支持

AOP和设备支持由spring-aop、spring-aspects和spring-instrument，3个模块组成。

spring-aop是Spring的另一个核心模块，是AOP主要的实现模块。作为继OOP后，对程序员影响最大的编程思想之一，AOP极大的开拓了人们对于编程的思路。在Spring中，他是以JVM的动态代理技术为基础，然后设计出了一系列的AOP横切实现，比如前置通知、异常通知等，同时，PointCut接口来匹配切入点，可以使用现有的切入点来设计横切面，也可以扩展相关方法根据需求进行切入。

spring-aspects模块继承自AspectJ框架，主要是为Spring AOP提供多种AOP实现方法。

spring-instrument模块是基于JAVA SE中的```java.lang.instrument```进行设计的，应该算是AOP的一个支援模块，主要作用是在JVM启用时，生成一个代理类，程序员公国代理类在运行时修改类的字节，从而改变一个类的功能，实现AOP的功能。在分类里，我把它分在了AOP模块下，在Spring光放文档里对这个地方也有点含糊不清。

#### 数据访问与集成

数据访问与集成由spring-jdbc、spring-tx、spring-orm、spring-jms和spring-oxm，5个模块组成。

spring-jdbc模块是Spring提供的JDBC抽象框架的主要实现模块，用于简化Spring JDBC操作。主要是提供JDBC模板方式、关系数据库对象化凡是、SimpleJdbc方式、事务管理来简化JDBC编程，主要实现类是JdbcTemplate、SimpleJdbcTemplate以及NamedParameterJdbcTemplate。

spring-tx模块是Spring JDBC事务控制的实现模块。使用Spring框架，它对事务做了很好的封装，通过它的AOP配置，可以灵活的配置在任何一层；但是在很多的需求和应用，直接使用JDBC事务控制还是很有优势的。其实，事务是也业务逻辑为基础的；一个完整的业务应该对应业务层里的一个方法；如果业务操作失败，则整个事务回滚；所以，事务控制是绝对应该放在业务层的；但是，持久层的设计应该遵循一个很重要的原则：保证操作的原子性，即持久层里的每个方法都应该是不可以分割的。所以，在使用Spring JDBC事务控制是，应该注意其特殊性。

spring-orm模块是ORM框架支持模块，主要集成Hibernate,Java Persistence API (JPA)和Java Data Objects(JDO)用于资源管理、数据访问对象（DAO）的实现和事务策略。

spring-oxm模块主要提供一个抽象层以支撑OXM（OXM是Object-to-XML-Mapping，的缩写，它是一个O/M-mapper，将Java对象映射成xml数据，或者将xml数据映射成Java对象），例如：JAXB，Castor,xmlBeans,XStream等。

spring-jms模块（Java Messaging Service）能够发送和接受信息，自Spring Framework 4.1以后，它还提供了对spring-messaging模块的支撑。

#### Web组件

​	由spring-web、spring-webmvc、spring-websocket和spring-webflux，4个模块组成。

​	spring-web模块为Spring提供了最基础的Web支持，主要简历与核心容器之上，通过Servlet或者Listeners来初始化IOC容器，也包含一些与Web相关的支持。

​	spring-webmvc模是一个Web-Servlet模块，试想了Spring MVC（model-view-controller）的Web应用。

​	spring-webflux是一个新的非阻塞函数式Reactive Web框架，可以用来建立异步的，非阻塞，事件驱动的服务，并且扩展性非常好。

#### 通信报文

​	即 spring-messaging 模块，是从 Spring4 开始新加入的一个模块，主要职责是为 Spring 框架集 成一些基础的报文传送应用。

#### 集成测试

​	即 spring-test 模块，主要为测试提供支持的，毕竟在不需要发布(程序)到你的应用服务器或者连接 到其他企业设施的情况下能够执行一些集成测试或者其他测试对于任何企业都是非常重要的。

#### 集成兼容

​	即 spring-framework-bom 模块（Bill of Materials，BOM）解决 Spring 的不同模块依赖版本不同问题。

#### 各模块之间的依赖关系

Spring 官网对 Spring5 各模块之间的关系也做了详细说明:

![image-20190920173044052](/Users/dongwenbin/github/doc/Spring/assets/image-20190920173044052.png)

对 Spring5 各模块做了一次系统的总结，描述模块之间的依赖关系，希望能对小伙伴们有所 帮助。

![image-20190920173110832](/Users/dongwenbin/github/doc/Spring/assets/image-20190920173110832.png)

## Spring源码构建

首先你的 JDK 需要升级到 1.8 以上。Spring3.0 开始,Spring 源码采用 github 托管，不再提供官网下载 链接。这里不做过多赘述，大家可自行去 github 网站下载，我们使用的版本下载链接为: https://github.com/spring-projects/spring-framework/archive/v5.0.2.RELEASE.zip

### **基于 Gradle 的源码构建**

安装Gradle:

```bash
brew install gradle
```

查看Gardle安装地址：brew info gradle

```bash
dongwenbindeMacBook-Pro:spring-framework-5.0.2.RELEASE dongwenbin$ brew info gradle
gradle: stable 5.6.2
Open-source build automation tool based on the Groovy and Kotlin DSL
https://www.gradle.org/
/usr/local/Cellar/gradle/5.6.2 (14,326 files, 245MB) *
  Built from source on 2019-09-20 at 11:40:31
From: https://github.com/Homebrew/homebrew-core/blob/master/Formula/gradle.rb
==> Requirements
Required: java >= 1.8 ✔
==> Analytics
install: 59,381 (30 days), 161,358 (90 days), 613,309 (365 days)
install_on_request: 56,061 (30 days), 152,650 (90 days), 574,222 (365 days)
build_error: 0 (30 days)
dongwenbindeMacBook-Pro:spring-framework-5.0.2.RELEASE dongwenbin$

```

在信息里看到Gradle被brew安装到了：/usr/local/Cellar/gradle/5.6.2目录下。记录下来就好后面构建的时候回用到。

### 导入代码

![image-20190920174210839](/Users/dongwenbin/github/doc/Spring/assets/image-20190920174210839.png)

![image-20190920173738844](/Users/dongwenbin/github/doc/Spring/assets/image-20190920173738844.png)

点击next

![image-20190920173906663](/Users/dongwenbin/github/doc/Spring/assets/image-20190920173906663.png)

等待构建完成，在网络良好的情况下大约需要 10 分钟便可自动构建完成。