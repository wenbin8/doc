



一步一步建立起从分布式到服务化在到云原生的一套完整技术栈.

### Java8

- [x] [Lambda表达式](https://github.com/wenbin8/doc/blob/master/java8/java8的流(Stream)操作.md)
- [x] [流操作(Stream)](https://github.com/wenbin8/doc/blob/master/java8/java8的流(Stream)操作.md)
- [ ] HashMap源码分析

### 并发

- [x] [并发编程的基础](https://github.com/wenbin8/doc/blob/master/并发/01-并发编程的基础.md)
- [x] [synchronize](https://github.com/wenbin8/doc/blob/master/并发/02-synchronized.md)
- [x] [volatile](https://github.com/wenbin8/doc/blob/master/并发/03-volatile.md)
- [x] [AQS底层原理分析](https://github.com/wenbin8/doc/blob/master/并发/04-AQS底层原理分析.md)
- [x] [Condition及源码分析](https://github.com/wenbin8/doc/blob/master/并发/05-Condition及源码分析.md)
- [x] [CountDwonLatch及源码分析](https://github.com/wenbin8/doc/blob/master/并发/06-CountDwonLatch及源码分析.md)
- [x] [Semaphore及源码分析](https://github.com/wenbin8/doc/blob/master/并发/07-Semaphore及源码分析.md)
- [x] [CyclicBarrier及原理](https://github.com/wenbin8/doc/blob/master/并发/08-CyclicBarrier及原理.md)
- [x] [ConcurrentHashMap源码分析](https://github.com/wenbin8/doc/blob/master/并发/09-ConcurrentHashMap源码分析.md)
- [x] [阻塞队列、原子操作的原理分析](https://github.com/wenbin8/doc/blob/master/并发/10-阻塞队列、原子操作的原理分析.md)
- [x] [线程池、forkjoin的原理分析](https://github.com/wenbin8/doc/blob/master/并发/11-线程池、forkjoin的原理分析.md)

### 常用设计模式

​	用好设计模式能帮助我们更好的解决实际问题，设计模式最重要的是解耦。设计模式天天都在用，但自己却无感知。把设计模式作为一个专题，主要是学习设计模式是如何总结经验的，把经验为自己所用。同时也为阅读框架源码打下坚实的基础。在学习设计模式之前，一定要先了解软件设计原则。

示例代码地址：git@github.com:wenbin8/design-pattern.git

- [x] [软件设计的七大原则](https://github.com/wenbin8/doc/blob/master/设计模式/01-软件设计的七大原则.md)
- [x] [工厂模式](https://github.com/wenbin8/doc/blob/master/设计模式/02-工厂模式.md)
- [x] [单例模式](https://github.com/wenbin8/doc/blob/master/设计模式/03-单例模式.md)
- [x] [原型模式](https://github.com/wenbin8/doc/blob/master/设计模式/04-原型模式.md)
- [x] [代理模式](https://github.com/wenbin8/doc/blob/master/设计模式/05-代理模式.md)
- [x] [委派模式](https://github.com/wenbin8/doc/blob/master/设计模式/06-委派模式.md)
- [x] [策略模式](https://github.com/wenbin8/doc/blob/master/设计模式/07-策略模式.md)
- [x] [模板模式](https://github.com/wenbin8/doc/blob/master/设计模式/08-模板模式.md)
- [x] [适配器模式](https://github.com/wenbin8/doc/blob/master/设计模式/09-适配器模式.md)
- [x] [装饰者模式](https://github.com/wenbin8/doc/blob/master/设计模式/10-装饰者模式.md)
- [x] [观察者模式](https://github.com/wenbin8/doc/blob/master/设计模式/11-观察者模式.md)

### 框架源码分析

Spring:

- [x] [Spring5的系统架构和源码构建](https://github.com/wenbin8/doc/blob/master/Spring/01-Spring核心概念与系统架构.md)
- [x] [Spring的IOC源码分析](https://github.com/wenbin8/doc/blob/master/Spring/02-Spring-IOC源码分析.md)
- [x] [Spring的DI源码分析](https://github.com/wenbin8/doc/blob/master/Spring/03-Spring-DI源码分析.md)
- [x] [Spring的AOP源码分析](https://github.com/wenbin8/doc/blob/master/Spring/04-Spring-AOP源码分析.md)
- [x] [Spring MVC源码分析](https://github.com/wenbin8/doc/blob/master/Spring/05-Spring-MVC源码分析.md)

### 数据库

- [ ] b+Tree的原理与Mysql的innoDB索引原理
- [ ] mySql数据查询性能优化

### 数据库分库分表

- [ ] Mycat

### JVM

- [ ] JVM运行时数据区的内存管理机制
- [ ] 垃圾收集算法与垃圾收集器
- [ ] 类加载机制与编译优化
- [ ] 性能监控与故障处理

### 分布式

#### 分布式基础

- [x] [分布式架构演进](https://github.com/wenbin8/doc/blob/master/分布式/分布式基础/01-分布式架构演进.md)
- [x] [通信协议Tcp/ip](https://github.com/wenbin8/doc/blob/master/分布式/分布式基础/02-通信协议总结-TCPIP.md)
- [x] [HTTP及HTTPS协议原理](https://github.com/wenbin8/doc/blob/master/分布式/分布式基础/03-HTTP及HTTPS协议原理.md)
- [ ] 序列化与反序列化

#### 分布式协调服务

- [ ] zookeeper
- [ ] etcd

#### 分布式服务治理

- [ ] Dubbo

#### 分布式消息通信

- [ ] kafka
- [ ] rabbitMQ

#### 分布式缓存

- [ ] MongoDB
- [ ] Redis


### 微服务(Spring boot/Spring Cloud)

#### Spring Boot

- [ ] 自动配置\起步依赖\Actuator

#### Spring Cloud

- [ ] Eureka注册中心
- [ ] Ribbon 负载均衡
- [ ] Fegion 声明式服务调用
- [ ] Hystrix 服务熔断降级方式
- [ ] Zuul 实现微服务网关
- [ ] Config 分布式统一配置中心
- [ ] Sleuth 调用链路跟踪
- [ ] BUS 消息总线
- [ ] Spring Boot 与 Spring Cloud 整合

### 云原生

#### Docker

- [ ] 理解docker实现原理(内核支持,和联合文件系统)
- [ ] docker镜像和镜像仓库
- [ ] docker容器启动与资源分配
- [ ] docker数据卷(volumes)
- [ ] docker的网络
- [ ] dockerfile的使用
- [ ] docker Compose 

#### K8S

