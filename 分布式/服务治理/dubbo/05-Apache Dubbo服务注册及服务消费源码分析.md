# Apache Dubbo服务注册及服务消费源码分析

## Invoker是什么？

从前面的分析来看，服务的发布分三个阶段

第一个阶段会创造一个invoker。

第二个阶段会把经历过一系列处理的invoker（各种包装），在DubboProtocol中保存到exporterMap中。

第三个阶段把dubbo协议的url地址注册到注册中心上。

前面没有分析invoker，这里先看一下invoker是个啥东西。

invoker是dubbo领域模型中非常重要的一个概念，和ExtensionLoader的重要性是一样的，如果Invoker没有搞懂，那么就不算是看懂了Dubbo的源码。我们继续回到ServiceConfig中export的代码，这段代码还没有分析过。以这个作为入口来分析我们前面export出去的Invoker到底是个啥东西。

```java
Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(EXPORT_KEY, url.toFullString()));
```

### proxyFactory.getInvoker

这个是一个代理工程，用来生成Invoker，从它的定义来看它是一个自适应扩展点，看到这样的扩展点，我们几乎可以不假思索的想到它会存在一个动态适配器类。

```java
private static final ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
```

### ProxyFactory

这个方法的简单解读为: 它是一个spi扩展点，并且默认的扩展实现是javassit, 这个接口中有三个方 法，并且都是加了@Adaptive的自适应扩展点。所以如果调用getInvoker方法，应该会返回一个ProxyFactory$Adaptive 

```java
@SPI("javassist")
public interface ProxyFactory {

    @Adaptive({PROXY_KEY})
    <T> T getProxy(Invoker<T> invoker) throws RpcException;

    @Adaptive({PROXY_KEY})
    <T> T getProxy(Invoker<T> invoker, boolean generic) throws RpcException;

    @Adaptive({PROXY_KEY})
    <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException;

}
```

### **ProxyFactory$Adaptive** 

这个自适应扩展点，做了两件事情 

通过ExtensionLoader.getExtensionLoader(ProxyFactory.class).getExtension(extName)获取了 一个指定名称的扩展点, 在dubbo-rpc-api/resources/META-INF/com.alibaba.dubbo.rpc.ProxyFactory中，定义了 javassis=JavassisProxyFactory 

调用JavassisProxyFactory的getInvoker方法 

```java
public class ProxyFactory$Adaptive implements org.apache.dubbo.rpc.ProxyFactory {
    
    public java.lang.Object getProxy(org.apache.dubbo.rpc.Invoker arg0) throws org.apache.dubbo.rpc.RpcException {
        if (arg0 == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
        org.apache.dubbo.common.URL url = arg0.getUrl();
        String extName = url.getParameter("proxy", "javassist");
        if (extName == null)
            throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.ProxyFactory) name from url (" + url.toString() + ") use keys([proxy])");
        org.apache.dubbo.rpc.ProxyFactory extension = (org.apache.dubbo.rpc.ProxyFactory) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.ProxyFactory.class).getExtension(extName);
        return extension.getProxy(arg0);
    }

    public java.lang.Object getProxy(org.apache.dubbo.rpc.Invoker arg0, boolean arg1) throws org.apache.dubbo.rpc.RpcException {
        if (arg0 == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
        org.apache.dubbo.common.URL url = arg0.getUrl();
        String extName = url.getParameter("proxy", "javassist");
        if (extName == null)
            throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.ProxyFactory) name from url (" + url.toString() + ") use keys([proxy])");
        org.apache.dubbo.rpc.ProxyFactory extension = (org.apache.dubbo.rpc.ProxyFactory) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.ProxyFactory.class).getExtension(extName);
        return extension.getProxy(arg0, arg1);
    }

    public org.apache.dubbo.rpc.Invoker getInvoker(java.lang.Object arg0, java.lang.Class arg1, org.apache.dubbo.common.URL arg2) throws org.apache.dubbo.rpc.RpcException {
        if (arg2 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg2;
        String extName = url.getParameter("proxy", "javassist");
        if (extName == null)
            throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.ProxyFactory) name from url (" + url.toString() + ") use keys([proxy])");
        org.apache.dubbo.rpc.ProxyFactory extension = (org.apache.dubbo.rpc.ProxyFactory) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.ProxyFactory.class).getExtension(extName);
        return extension.getInvoker(arg0, arg1, arg2);
    }
}
```

### **JavassistProxyFactory.getInvoker** 

javassist是一个动态类库，用来实现动态代理的。

proxy:接口的实现: com.wenbin.dubbo.demo.dubboproviderdemo.SayHelloServiceImpl

type:接口全称： com.wenbin.demo.dubbo.api.SayHelloServicee 

url:协议地址:registry://... 

```java
@Override
public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
    // TODO Wrapper cannot handle this scenario correctly: the classname contains '$'
    final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
    return new AbstractProxyInvoker<T>(proxy, type, url) {
        @Override
        protected Object doInvoke(T proxy, String methodName,
                                  Class<?>[] parameterTypes,
                                  Object[] arguments) throws Throwable {
            return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
        }
    };
}
```

### javassist生成的动态代理代码

通过断点的方式(Wrapper258行)，在Wrapper.getWrapper中的makeWrapper，会创建一个动态代理，核心的方法invokeMethod代码如下：

```java
public Object invokeMethod(Object o, String n, Class[] p, Object[] v) throws java.lang.reflect.InvocationTargetException {
    com.wenbin.demo.dubbo.api.SayHelloService w;
    try {
        w = ((com.wenbin.demo.dubbo.api.SayHelloService) $1);
    } catch (Throwable e) {
        throw new IllegalArgumentException(e);
    }
    try {
        if ("hello".equals($2) && $3.length == 1) {
            return ($w) w.hello((java.lang.String) $4[0]);
        }
    } catch (Throwable e) {
        throw new java.lang.reflect.InvocationTargetException(e);
    }
    throw new org.apache.dubbo.common.bytecode.NoSuchMethodException("Not found method \"" + $2 + "\" in class com.wenbin.demo.dubbo.api.SayHelloService.");
}
```

构建好了代理类之后，返回一个AbstractproxyInvoker,并且它实现了doInvoke方法，这个地方似乎看 到了dubbo消费者调用过来的时候触发的影子，因为wrapper.invokeMethod本质上就是触发上面动态 代理类的方法invokeMethod. 

```java
return new AbstractProxyInvoker<T>(proxy, type, url) {
    @Override
    protected Object doInvoke(T proxy, String methodName,
                              Class<?>[] parameterTypes,
                              Object[] arguments) throws Throwable {
        return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
    }
};
```

所以，简单总结一下Invoke本质上应该是一个代理，经过层层包装最终进行了发布。当消费者发起请求 的时候，会获得这个invoker进行调用。 

最终发布出去的invoker, 也不是单纯的一个代理，也是经过多层包装 InvokerDelegate(DelegateProviderMetaDataInvoker(AbstractProxyInvoker())) 

关于服务发布这一条线分析完成了 。

## **服务注册流程** 

再来了解一下服务注册的过程，希望大家还记得我们之所以走到这一步，是因为我们在RegistryProtocol这个类中，看到了服务发布的流程。 

```java
//实现服务的注册和发布
@Override
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
    // registryUrl -> zookeeper://ip:port
    URL registryUrl = getRegistryUrl(originInvoker);
    // providerUrl -> dubbo:// ip:port
    URL providerUrl = getProviderUrl(originInvoker);

    // Subscribe the override data
    // FIXME When the provider subscribes, it will affect the scene : a certain JVM exposes the service and call
    //  the same service. Because the subscribed is cached key with the name of the service, it causes the
    //  subscription information to cover.
    final URL overrideSubscribeUrl = getSubscribedOverrideUrl(providerUrl);
    final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
    overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);

    providerUrl = overrideUrlWithConfig(providerUrl, overrideSubscribeListener);

    /***********************************/
    //doLocalExport 本质就是去启动一个netty服务
    final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker, providerUrl);

    // 把dubbo:// url注册到zk上
    final Registry registry = getRegistry(originInvoker);
    final URL registeredProviderUrl = getRegisteredProviderUrl(providerUrl, registryUrl);
    ProviderInvokerWrapper<T> providerInvokerWrapper = ProviderConsumerRegTable.registerProvider(originInvoker,
            registryUrl, registeredProviderUrl);
    //to judge if we need to delay publish
    boolean register = registeredProviderUrl.getParameter("register", true);
    if (register) {
        register(registryUrl, registeredProviderUrl);
        providerInvokerWrapper.setReg(true);
    }

    // Deprecated! Subscribe to override rules in 2.6.x or before.
    registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);

    exporter.setRegisterUrl(registeredProviderUrl);
    exporter.setSubscribeUrl(overrideSubscribeUrl);
    //Ensure that a new exporter instance is returned every time export
    return new DestroyableExporter<>(exporter);
}
```

### **服务注册核心代码** 

从export方法中抽离出来的部分代码，就是服务注册的流程 

```java
	// 把dubbo:// url注册到zk上
    final Registry registry = getRegistry(originInvoker);
    final URL registeredProviderUrl = getRegisteredProviderUrl(providerUrl, registryUrl);
    ProviderInvokerWrapper<T> providerInvokerWrapper = ProviderConsumerRegTable.registerProvider(originInvoker,
            registryUrl, registeredProviderUrl);
    //to judge if we need to delay publish
    boolean register = registeredProviderUrl.getParameter("register", true);
    if (register) {
        register(registryUrl, registeredProviderUrl);
        providerInvokerWrapper.setReg(true);
    }
```

### getRegistry

1. 把url转化为对应配置的注册中心的具体协议
2. 根据具体协议，从registryFactory中获得指定的注册中心实现 

那么这个registryFactory具体是怎么赋值的呢? 

```java
private Registry getRegistry(final Invoker<?> originInvoker) {
    URL registryUrl = getRegistryUrl(originInvoker);
    return registryFactory.getRegistry(registryUrl);
}
```

在RegistryProtocol中存在一段这样的代码，很明显这是通过依赖注入来实现的扩展点。 

```java
private RegistryFactory registryFactory;

public void setRegistryFactory(RegistryFactory registryFactory) {
        this.registryFactory = registryFactory;
    }
```

按照扩展点的加载规则，我们可以先看看/META-INF/dubbo/internal路径下找到RegistryFactory的配 置文件.这个factory有多个扩展点的实现。 

```properties
dubbo=org.apache.dubbo.registry.dubbo.DubboRegistryFactory 
multicast=org.apache.dubbo.registry.multicast.MulticastRegistryFactory 
zookeeper=org.apache.dubbo.registry.zookeeper.ZookeeperRegistryFactory 
redis=org.apache.dubbo.registry.redis.RedisRegistryFactory 
consul=org.apache.dubbo.registry.consul.ConsulRegistryFactory

etcd3=org.apache.dubbo.registry.etcd.EtcdRegistryFactory	
```

接着，找到RegistryFactory的实现, 发现它里面有一个自适应的方法，根据url中protocol传入的值进行适配:

```java
@SPI("dubbo")
public interface RegistryFactory {
    @Adaptive({"protocol"})
    Registry getRegistry(URL url);
}
```

### **RegistryFactory$Adaptive** 

由于在前面的代码中，url中的protocol已经改成了zookeeper，那么这个时候根据zookeeper获得的spi扩展点应该是ZookeeperRegistryFactory 

```java
import org.apache.dubbo.common.extension.ExtensionLoader;

public class RegistryFactory$Adaptive implements org.apache.dubbo.registry.RegistryFactory {
    public org.apache.dubbo.registry.Registry getRegistry(org.apache.dubbo.common.URL arg0) {
        if (arg0 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg0;
        String extName = (url.getProtocol() == null ? "dubbo" :
                url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Failed to get extension (org.apache.dubbo.registry.RegistryFactory) name from url (" + url.toString() + ") use keys([protocol])");
        org.apache.dubbo.registry.RegistryFactory extension = (org.apache.dubbo.registry.RegistryFactory) ExtensionLoader.getExtensionLoader(org.apache.dubbo.registry.RegistryFactory.class).getExtension(extName);
        return extension.getRegistry(arg0);
    }
}
```

### **ZookeeperRegistryFactory** 

这个方法中并没有getRegistry方法，而是在父类AbstractRegistryFactory 

- 从缓存REGISTRIES中，根据key获得对应的Registry 
- 如果不存在，则创建Registry 

```java
@Override
public Registry getRegistry(URL url) {
    url = URLBuilder.from(url)
            .setPath(RegistryService.class.getName())
            .addParameter(INTERFACE_KEY, RegistryService.class.getName())
            .removeParameters(EXPORT_KEY, REFER_KEY)
            .build();
    String key = url.toServiceStringWithoutResolving();
    // Lock the registry access process to ensure a single instance of the registry
    LOCK.lock();
    try {
        Registry registry = REGISTRIES.get(key);
        if (registry != null) {
            return registry;
        }
        //create registry by spi/ioc
        // 创建注册中心
        registry = createRegistry(url);
        if (registry == null) {
            throw new IllegalStateException("Can not create registry " + url);
        }
        REGISTRIES.put(key, registry);
        return registry;
    } finally {
        // Release the lock
        LOCK.unlock();
    }
}
```

### createRegistry

创建一个zookeeperRegistry，把url和zookeeperTransporter作为参数传入。zookeeperTransporter这个属性也是基于依赖注入来赋值的，具体的流程就不再分析了，这个的值应该是CuratorZookeeperTransporter表示具体使用什么框架来和zk产生连接。

```java
@Override
public Registry createRegistry(URL url) {
    return new ZookeeperRegistry(url, zookeeperTransporter);
}
```

### ZookeeperRegistry

这个方法中使用了CuratorZookeeperTransport来实现zk的连接 

```java
public ZookeeperRegistry(URL url, ZookeeperTransporter zookeeperTransporter) {
    super(url);
    if (url.isAnyHost()) {
        throw new IllegalStateException("registry address == null");
    }
    //获得group名称
    String group = url.getParameter(GROUP_KEY, DEFAULT_ROOT);
    if (!group.startsWith(PATH_SEPARATOR)) {
        group = PATH_SEPARATOR + group;
    }
    this.root = group;
    //产生一个zookeeper连接
    zkClient = zookeeperTransporter.connect(url);
    //添加zookeeper状态变化事件
    zkClient.addStateListener(state -> {
        if (state == StateListener.RECONNECTED) {
            try {
                recover();
            } catch (Exception e) {
                logger.error(e.getMessage(), e);
            }
        }
    });
}
```

### RegistryProtocol#register

继续往下分析，会调用RegistryProtocol.register去讲dubbo://的协议地址注册到zookeeper上 

```java
public void register(URL registryUrl, URL registeredProviderUrl) {
    Registry registry = registryFactory.getRegistry(registryUrl);
    registry.register(registeredProviderUrl);
}
```

这个方法会调用FailbackRegistry类中的register. 为什么呢?因为ZookeeperRegistry这个类中并没有register这个方法，但是他的父类FailbackRegistry中存在register方法，而这个类又重写了 AbstractRegistry类中的register方法。所以我们可以直接定位到FailbackRegistry这个类中的register方法中 

### FailbackRegistry#register

- FailbackRegistry，从名字上来看，是一个失败重试机制 
- 调用父类的register方法，将当前url添加到缓存集合中 

```java
@Override
public void register(URL url) {
    super.register(url);
    removeFailedRegistered(url);
    removeFailedUnregistered(url);
    try {
        // Sending a registration request to the server side
        doRegister(url);
    } catch (Exception e) {
        Throwable t = e;

        // If the startup detection is opened, the Exception is thrown directly.
        boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                && url.getParameter(Constants.CHECK_KEY, true)
                && !CONSUMER_PROTOCOL.equals(url.getProtocol());
        boolean skipFailback = t instanceof SkipFailbackWrapperException;
        if (check || skipFailback) {
            if (skipFailback) {
                t = t.getCause();
            }
            throw new IllegalStateException("Failed to register " + url + " to registry " + getUrl().getAddress() + ", cause: " + t.getMessage(), t);
        } else {
            logger.error("Failed to register " + url + ", waiting for retry, cause: " + t.getMessage(), t);
        }

        // Record a failed registration request to a failed list, retry regularly
        addFailedRegistered(url);
    }
}
```

调用doRegister方法，这个方法很明显，是一个抽象方法，会由ZookeeperRegistry子类实现。 

### **ZookeeperRegistry.doRegister** 

最终调用curator的客户端把服务地址注册到zk 

```java
@Override
public void doRegister(URL url) {
    try {
        zkClient.create(toUrlPath(url), url.getParameter(DYNAMIC_KEY, true));
    } catch (Throwable e) {
        throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}
```

服务注册的流程就完成了。

## 服务消费

消费端的代码解析是从下面这段代码开始的：

```java
<dubbo:reference id="XXXService" interface="XXXSerivce"  />
```

注解的方式的初始化入口是

`ReferenceAnnotationBeanPostProcessor->ReferenceBeanInvocationHandler.init- >ReferenceConfig.get()` 获得一个远程代理类 

### ReferenceConfig.get

```java
//返回一个代理对象（）
public synchronized T get() {
    checkAndUpdateSubConfigs(); //检查配置和完善配置

    if (destroyed) {
        throw new IllegalStateException("The invoker of ReferenceConfig(" + url + ") has already destroyed!");
    }
    if (ref == null) { //ref还没有生成的时候，init
        init();
    }
    return ref;
}
```

### init

初始化的过程，和服务发布的过程类似，会有特别多的判断以及参数的组装. 我们只需要关注createProxy，创建代理类的方法。 

```java
private void init() {
    // ....省略代码
    ref = createProxy(map); //真正意义上去构建proxy
    // ....省略代码
}
```

### createProxy

代码比较长，但是逻辑相对比较清晰：

1. 判断是否为本地调用，如果是则使用injvm协议进行调用
2. 判断是否为点对点调用，如果是则把url保存到urls集合中，如果url为1，则进入步骤4，如果urls>1，则执行5。
3. 如果是配置了注册中心，遍历注册中心，把url添加到urls集合，url为1，进入步骤4，如果urls>1，则执行5。
4. 直接构建Invoker
5. 构建invoker集合，通过cluster合并多个Invoker。
6. 最后调用ProxyFacotry生成代理类。

```java
private T createProxy(Map<String, String> map) {
    if (shouldJvmRefer(map)) {//判断是否是在同一个jvm进程中调用
        URL url = new URL(LOCAL_PROTOCOL, LOCALHOST_VALUE, 0, interfaceClass.getName()).addParameters(map);
        invoker = REF_PROTOCOL.refer(interfaceClass, url);
        if (logger.isInfoEnabled()) {
            logger.info("Using injvm service " + interfaceClass.getName());
        }
    } else {
        urls.clear(); // reference retry init will add url to urls, lead to OOM
        //url 如果不为空，说明是点对点通信
        if (url != null && url.length() > 0) { // user specified URL, could be peer-to-peer address, or register center's address.
            String[] us = SEMICOLON_SPLIT_PATTERN.split(url);
            if (us != null && us.length > 0) {
                for (String u : us) {
                    URL url = URL.valueOf(u);
                    if (StringUtils.isEmpty(url.getPath())) {
                        url = url.setPath(interfaceName);
                    }
                    // 检测url协议是否为registry，若是，表明用户想使用指定的注册中心
                    if (REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                        // 将map转换为查询字符串，并作为refer参数的值添加到url中
                        urls.add(url.addParameterAndEncoded(REFER_KEY, StringUtils.toQueryString(map)));
                    } else {
                        // 合并url，移除服务提供者的一些配置(这些配置来源于用户配置的url属性)，
                        // 比如线程池相关配置。并保留服务提供者的部分配置，
                        // 比如版本group，时间戳等，最后将合并后的配置设置为url查询字符串中。
                        urls.add(ClusterUtils.mergeUrl(url, map));
                    }
                }
            }
        } else {//从注册中心去获得服务地址
            // if protocols not injvm checkRegistry
            if (!LOCAL_PROTOCOL.equalsIgnoreCase(getProtocol())){
                // 校验注册中心的配置以及是否有必要从配置中心组装url
                checkRegistry();
                //这里的代码实现和服务端类似，也是根据注册中心配置进行解析得到URL
                //这里的URL肯定也是:
				//registry://ip:port/org.apache.dubbo.service.RegsitryService
                List<URL> us = loadRegistries(false);
                if (CollectionUtils.isNotEmpty(us)) {
                    for (URL u : us) {
                        URL monitorUrl = loadMonitor(u);
                        if (monitorUrl != null) {
                            map.put(MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
                        }
                        urls.add(u.addParameterAndEncoded(REFER_KEY, StringUtils.toQueryString(map)));
                    }
                }
                //如果没有配置注册中心，则报错
                if (urls.isEmpty()) {
                    throw new IllegalStateException("No such any registry to reference " + interfaceName + " on the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", please config <dubbo:registry address=\"...\" /> to your spring config.");
                }
            }
        }
        //urls.size=1;()
        //如果值配置了一个注册中心或者一个服务提供者，直接使用refprotocol.refer
        if (urls.size() == 1) {
            //构建一个invoker(Protocol)
            //Protocol$Adaptive -> getExtension("registry")->Qos(listener(filter(RegisterProtol
            //RegisterProtocol.refer()
            invoker = REF_PROTOCOL.refer(interfaceClass, urls.get(0));
        } else {
            List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
            URL registryURL = null;
            //遍历urls生成多个invoker
            for (URL url : urls) {
                invokers.add(REF_PROTOCOL.refer(interfaceClass, url));
                if (REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                    registryURL = url; // use last registry url
                }
            }
            //如果registryUrl不为空，构建静态directory
            if (registryURL != null) { // registry url is available
                // 使用RegistryAwareCluster
                URL u = registryURL.addParameter(CLUSTER_KEY, RegistryAwareCluster.NAME);
                // The invoker wrap relation would be: RegistryAwareClusterInvoker(StaticDirectory) -> FailoverClusterInvoker(RegistryDirectory, will execute route) -> Invoker
                invoker = CLUSTER.join(new StaticDirectory(u, invokers));
            } else { // not a registry url, must be direct invoke.
                invoker = CLUSTER.join(new StaticDirectory(invokers));
            }
        }
    }

    //检查invoker的有效性
    if (shouldCheck() && !invoker.isAvailable()) {
        throw new IllegalStateException("Failed to check the status of the service " + interfaceName + ". No provider available for the service " + (group == null ? "" : group + "/") + interfaceName + (version == null ? "" : ":" + version) + " from the url " + invoker.getUrl() + " to the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion());
    }
    if (logger.isInfoEnabled()) {
        logger.info("Refer dubbo service " + interfaceClass.getName() + " from url " + invoker.getUrl());
    }
    /**
     * @since 2.7.0
     * ServiceData Store
     */
    MetadataReportService metadataReportService = null;
    if ((metadataReportService = getMetadataReportService()) != null) {
        URL consumerURL = new URL(CONSUMER_PROTOCOL, map.remove(REGISTER_IP_KEY), 0, map.get(INTERFACE_KEY), map);
        metadataReportService.publishConsumer(consumerURL);
    }
    // create service proxy
    return (T) PROXY_FACTORY.getProxy(invoker);
}
```

### **protocol.refer** 

这里通过指定的协议来调用refer生成一个invoker对象，invoker前面讲过，它是一个代理对象。那么在当前的消费端而言，invoker主要用于执行远程调用。 

这个protocol，又是一个自适应扩展点，它得到的是一个Protocol$Adaptive。

```java
private static final Protocol REF_PROTOCOL = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```

这段代码中，根据当前的协议url，得到一个指定的扩展点，传递进来的参数中，协议地址为registry://，所以，我们可以直接定位到RegistryProtocol.refer代码：

```java
    public org.apache.dubbo.rpc.Invoker refer(Class arg0, org.apache.dubbo.common.URL arg1) throws org.apache.dubbo.rpc.RpcException {
        if (arg1 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg1;
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.Protocol) name from url (" + url.toString() + ") use keys([protocol])");
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.refer(arg0, arg1);
    }
```

根据当前的协议扩展名registry, 获得一个被包装过的RegistryProtocol 

### **RegistryProtocol.refer** 

这里面的代码逻辑比较简单

- 组装注册中心协议的url 
- 判断是否配置legroup，如果有，则cluster=getMergeableCluster()，构建invoker 
- doRefer构建invoker 

```java
@Override
@SuppressWarnings("unchecked")
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    //这段代码也很熟悉，就是根据配置的协议，生成注册中心的url: zookeeper://
    url = URLBuilder.from(url)
            .setProtocol(url.getParameter(REGISTRY_KEY, DEFAULT_REGISTRY))
            .removeParameter(REGISTRY_KEY)
            .build();
    //ZookeeperRegistery
    Registry registry = registryFactory.getRegistry(url);
    //register://
    if (RegistryService.class.equals(type)) {
        return proxyFactory.getInvoker((T) registry, type, url);
    }

    // 解析group参数，根据group决定cluster的类型
    Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(REFER_KEY));
    String group = qs.get(GROUP_KEY);
    if (group != null && group.length() > 0) {
        if ((COMMA_SPLIT_PATTERN.split(group)).length > 1 || "*".equals(group)) {
            return doRefer(getMergeableCluster(), registry, type, url);
        }
    }
    //Cluster->?
    return doRefer(cluster, registry, type, url);
}
```

### doRefer

doRefer里面就稍微复杂一些，涉及到比较多的东西，我们先关注主线 

- 构建一个RegistryDirectory 
- 构建一个consumer://协议的地址注册到注册中心 
- 订阅zookeeper中节点的变化 
- 调用cluster.join方法 

```java
//首先从zk上获得provider url
//建立连接
private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {

    /******************************************/
    // 构建一个consumer://ip:port  保存到zk   provider /configurator/consumer/router
    // 1. 连接到注册中心 ->curator
    // 2. 从注册中心拿到地址（providerUrl）
    // 3. 基于provider地址建立通信
    RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
    directory.setRegistry(registry); //registry -> 连接zk的api   ->获得url地址
    directory.setProtocol(protocol); //protocol -> DubboProtocol()  -> 建立通信
    // all attributes of REFER_KEY
    Map<String, String> parameters = new HashMap<String, String>(directory.getUrl().getParameters());
    URL subscribeUrl = new URL(CONSUMER_PROTOCOL, parameters.remove(REGISTER_IP_KEY), 0, type.getName(), parameters);
    if (!ANY_VALUE.equals(url.getServiceInterface()) && url.getParameter(REGISTER_KEY, true)) {
        directory.setRegisteredConsumerUrl(getRegisteredConsumerUrl(subscribeUrl, url));
        registry.register(directory.getRegisteredConsumerUrl()); //consumer://
    }
    directory.buildRouterChain(subscribeUrl);
    //订阅
    directory.subscribe(subscribeUrl.addParameter(CATEGORY_KEY,
            PROVIDERS_CATEGORY + "," + CONFIGURATORS_CATEGORY + "," + ROUTERS_CATEGORY));
    /******************************************/


    //MockClusterWrapper(FailoverCluster(
    //Directory
    //构建invoker
    Invoker invoker = cluster.join(directory);
    ProviderConsumerRegTable.registerConsumer(invoker, url, subscribeUrl, directory);
    return invoker;
}
```

## Cluster是什么

我们只关注一下Invoker这个代理类的创建过程,其他的暂且不关心 

```java
Invoker invoker = cluster.join(directory);
```

cluster其实是在RegistryProtocol中通过set方法完成依赖注入的，并且，它还是一个被包装的 

```java
public void setCluster(Cluster cluster) {
    this.cluster = cluster;
}
```

Cluster扩展点的定义, 由于它是一个自适应扩展点，那么会动态生成一个Cluster$Adaptive的动态代理 类 

```java
@SPI(FailoverCluster.NAME)
public interface Cluster {

    @Adaptive
    <T> Invoker<T> join(Directory<T> directory) throws RpcException;

}
```

**Cluster$Adaptive** 

在动态适配的类中会基于extName，选择一个合适的扩展点进行适配，由于默认情况下cluster:failover，所以getExtension("failover")理论上应该返回FailOverCluster。但实际上，这里做了包装 MockClusterWrapper(FailOverCluster) 

```java
public class Cluster$Adaptive implements org.apache.dubbo.rpc.cluster.Cluster {
    
    public org.apache.dubbo.rpc.Invoker join(org.apache.dubbo.rpc.cluster.Directory arg0) 
            throws org.apache.dubbo.rpc.RpcException {
        if (arg0 == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.cluster.Directory argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("org.apache.dubbo.rpc.cluster.Directory argument getUrl() == null");
        org.apache.dubbo.common.URL url = arg0.getUrl();
        String extName = url.getParameter("cluster", "failover");
        if (extName == null)
            throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.cluster.Cluster) name from url (" + url.toString() + ") use keys([cluster])");
        org.apache.dubbo.rpc.cluster.Cluster extension = (org.apache.dubbo.rpc.cluster.Cluster) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.cluster.Cluster.class).getExtension(extName);
        return extension.join(arg0);
    }
}
```

### **cluster.join** 

所以再回到doRefer方法，下面这段代码, 实际是调用MockClusterWrapper(FailOverCluster.join) 

```java
Invoker invoker = cluster.join(directory);
```

所以这里返回的invoker，应该是MockClusterWrapper(FailOverCluster(directory)) 

接着回到ReferenceConfig.createProxy方法中的最后一行 

### **proxyFactory.getProxy** 

拿到invoker之后，会调用获得一个动态代理类 

```
return (T) PROXY_FACTORY.getProxy(invoker);
```

而这里的proxyFactory又是一个自适应扩展点，所以会进入下面的方法 

### **JavassistProxyFactory.getProxy** 

通过这个方法生成了一个动态代理类，并且对invoker再做了一层处理，InvokerInvocationHandler。 意味着后续发起服务调用的时候，会由InvokerInvocationHandler来进行处理。 

```java
@Override
@SuppressWarnings("unchecked")
public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
    return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
}
```

### **proxy.getProxy** 

在proxy.getProxy这个方法中会生成一个动态代理类，通过debug的形式可以看到动态代理类的原貌 在getProxy这个方法位置加一个断点 

```java
public static Proxy getProxy(ClassLoader cl, Class<?>... ics) {
    	// 省略代码....
    	ClassGenerator ccp = null, ccm = null;
    	// 省略代码....
        proxy = (Proxy) pc.newInstance();
      	// 省略代码....
}
```

然后在debug窗口，找到ccp这个变量 -> mMethods。 

![image-20191118200827922](assets/image-20191118200827922.png)

调整一下格式。 

```java
public java.lang.String sayHello(java.lang.String arg0) {
    Object[] args = new Object[1];
    args[0] = ($w) $1;
    Object ret = handler.invoke(this, methods[0], args);
    return (java.lang.String) ret;
}
```

从这个sayHello方法可以看出，我们通过 @Reference注入的一个对象实例本质上就是一个动态代理类，通过调用这个类中的方法，会触发 handler.invoke(), 而这个handler就是InvokerInvocationHandler 

## **网络连接的建立** 

前面分析的逻辑中，只讲到了动态代理类的生成，那么目标服务地址信息以及网络通信的建立在哪里实现的呢?我们继续回到doRefer这个方法中 

```java
//首先从zk上获得provider url
//建立连接
private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {

    /******************************************/
    // 构建一个consumer://ip:port  保存到zk   provider /configurator/consumer/router
    // 1. 连接到注册中心 ->curator
    // 2. 从注册中心拿到地址（providerUrl）
    // 3. 基于provider地址建立通信
    RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
    directory.setRegistry(registry); //registry -> 连接zk的api   ->获得url地址
    directory.setProtocol(protocol); //protocol -> DubboProtocol()  -> 建立通信
    // all attributes of REFER_KEY
    Map<String, String> parameters = new HashMap<String, String>(directory.getUrl().getParameters());
    URL subscribeUrl = new URL(CONSUMER_PROTOCOL, parameters.remove(REGISTER_IP_KEY), 0, type.getName(), parameters);
    if (!ANY_VALUE.equals(url.getServiceInterface()) && url.getParameter(REGISTER_KEY, true)) {
        directory.setRegisteredConsumerUrl(getRegisteredConsumerUrl(subscribeUrl, url));
        registry.register(directory.getRegisteredConsumerUrl()); //consumer://
    }
    directory.buildRouterChain(subscribeUrl);
    //订阅
    directory.subscribe(subscribeUrl.addParameter(CATEGORY_KEY,
            PROVIDERS_CATEGORY + "," + CONFIGURATORS_CATEGORY + "," + ROUTERS_CATEGORY));
    
    // ....省略部分代码
    return invoker;
}
```

### directory.subscribe

订阅注册中心指定节点的变化，如果发生变化，则通知到RegistryDirectory。Directory其实和服务的注册以及服务的发现有非常大的关联. 

```java
public void subscribe(URL url) {
	//设置consumerUrl
    setConsumerUrl(url);
    //把当前RegistryDirectory作为listener，去监听zk上节点的变化
    CONSUMER_CONFIGURATION_LISTENER.addNotifyListener(this);
    serviceConfigurationListener = new ReferenceConfigurationListener(this, url);
    //ZookeeperRegistry  ; listener: this ->RegistryDirectory
    registry.subscribe(url, this);//订阅 -> 这里的registry是zookeeperRegsitry
}
```

这里的registry 是ZookeeperRegistry ，会去监听并获取路径下面的节点。监听的路径是: 

`/dubbo/org.apache.dubbo.demo.DemoService/providers` 、

`/dubbo/org.apache.dubbo.demo.DemoService/configurators`、 

`dubbo/org.apache.dubbo.demo.DemoService/router `、

节点下面的子节点变动。

### **FailbackRegistry.subscribe** 

listener为RegistryDirectory，后续要用到。

移除失效的listener，调用doSubscribe进行订阅 。

```java
@Override
public void subscribe(URL url, NotifyListener listener) {
    super.subscribe(url, listener);
    removeFailedSubscribed(url, listener);
    try {
        // Sending a subscription request to the server side
        doSubscribe(url, listener);
    } catch (Exception e) {
        Throwable t = e;

        List<URL> urls = getCacheUrls(url);
        if (CollectionUtils.isNotEmpty(urls)) {
            notify(url, listener, urls);
            logger.error("Failed to subscribe " + url + ", Using cached list: " + urls + " from cache file: " + getUrl().getParameter(FILE_KEY, System.getProperty("user.home") + "/dubbo-registry-" + url.getHost() + ".cache") + ", cause: " + t.getMessage(), t);
        } else {
            // If the startup detection is opened, the Exception is thrown directly.
            boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                    && url.getParameter(Constants.CHECK_KEY, true);
            boolean skipFailback = t instanceof SkipFailbackWrapperException;
            if (check || skipFailback) {
                if (skipFailback) {
                    t = t.getCause();
                }
                throw new IllegalStateException("Failed to subscribe " + url + ", cause: " + t.getMessage(), t);
            } else {
                logger.error("Failed to subscribe " + url + ", waiting for retry, cause: " + t.getMessage(), t);
            }
        }

        // Record a failed registration request to a failed list, retry regularly
        addFailedSubscribed(url, listener);
    }
}
```

### **ZookeeperRegistry.doSubscribe** 

这个方法是订阅，逻辑实现比较多，可以分两段来看，这里的实现把所有Service层发起的订阅以及指定的Service层发起的订阅分开处理。所有Service层类似于监控中心发起的订阅。指定的Service层发起的订阅可以看作是服务消费者的订阅。我们只需要关心指定service层发起的订阅即可 

```java
@Override
public void doSubscribe(final URL url, final NotifyListener listener) {
    try {
        if (ANY_VALUE.equals(url.getServiceInterface())) {
            // ....省略部分代码
        } else {
            List<URL> urls = new ArrayList<>();
            //configurator/ consumer/ router
            for (String path : toCategoriesPath(url)) {
                ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
                // 如果之前该路径没有添加过listener，则创建一个map来放置listener
                if (listeners == null) {
                    zkListeners.putIfAbsent(url, new ConcurrentHashMap<>());
                    listeners = zkListeners.get(url);
                }
                ChildListener zkListener = listeners.get(listener);
                if (zkListener == null) {
                    // 如果没有添加过对于子节点的listener，则创建,通知服务变化回调NotifyListener
                    listeners.putIfAbsent(listener, (parentPath, currentChilds) -> ZookeeperRegistry.this.notify(url, listener, toUrlsWithEmpty(url, parentPath, currentChilds)));
                    zkListener = listeners.get(listener);
                }
                zkClient.create(path, false);
                //添加path节点的当前节点及子节点监听，并且获取子节点信息 
                //也就是dubbo://ip:port/...
                List<String> children = zkClient.addChildListener(path, zkListener);
                if (children != null) {
                    urls.addAll(toUrlsWithEmpty(url, path, children));
                }
            }
            //调用notify进行通知，对已经可用的列表进行通知
            notify(url, listener, urls);
        }
    } catch (Throwable e) {
        throw new RpcException("Failed to subscribe " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}
```

### **FailbackRegistry.notify** 

调用FailbackRegistry.notify， 对参数进行判断。 然后调用AbstractRegistry.notify方法 

```java
@Override
protected void notify(URL url, NotifyListener listener, List<URL> urls) {
    if (url == null) {
        throw new IllegalArgumentException("notify url == null");
    }
    if (listener == null) {
        throw new IllegalArgumentException("notify listener == null");
    }
    try {
        doNotify(url, listener, urls);
    } catch (Exception t) {
        // Record a failed registration request to a failed list, retry regularly
        addFailedNotified(url, listener, urls);
        logger.error("Failed to notify for subscribe " + url + ", waiting for retry, cause: " + t.getMessage(), t);
    }
}
```

```java
protected void doNotify(URL url, NotifyListener listener, List<URL> urls) {
    super.notify(url, listener, urls);
}
```

### **AbstractRegistry.notify** 

这里面会针对每一个category，调用listener.notify进行通知，然后更新本地的缓存文件 

```java
protected void notify(URL url, NotifyListener listener, List<URL> urls) {
    // 省略部分代码....
    // keep every provider's category.
    Map<String, List<URL>> result = new HashMap<>();
    for (URL u : urls) {
        if (UrlUtils.isMatch(url, u)) {
            String category = u.getParameter(CATEGORY_KEY, DEFAULT_CATEGORY);
            List<URL> categoryList = result.computeIfAbsent(category, k -> new ArrayList<>());
            categoryList.add(u);
        }
    }
    if (result.size() == 0) {
        return;
    }
    Map<String, List<URL>> categoryNotified = notified.computeIfAbsent(url, u -> new ConcurrentHashMap<>());
    for (Map.Entry<String, List<URL>> entry : result.entrySet()) {
        String category = entry.getKey();
        List<URL> categoryList = entry.getValue();
        categoryNotified.put(category, categoryList);
        // 通知？ RegistryDirectory
        listener.notify(categoryList);
        // We will update our cache file after each notification.
        // When our Registry has a subscribe failure due to network jitter, we can return at least the existing cache URL.
        //dubbo:// 保存到文件中
        saveProperties(url);
    }
}
```

消费端的listener是最开始传递过来的RegistryDirectory，所以这里会触发RegistryDirectory.notify 

### **RegistryDirectory.notify** 

Invoker的网络连接以及后续的配置变更，都会调用这个notify方法。

urls: zk的path数据，这里表示的是dubbo:// 

```java
@Override
public synchronized void notify(List<URL> urls) {
    //对url列表进行校验、过滤，然后分成 config、router、provider 3个分组map
    Map<String, List<URL>> categoryUrls = urls.stream()
            .filter(Objects::nonNull)
            .filter(this::isValidCategory)
            .filter(this::isNotCompatibleFor26x)
            .collect(Collectors.groupingBy(url -> {
                if (UrlUtils.isConfigurator(url)) {
                    return CONFIGURATORS_CATEGORY;
                } else if (UrlUtils.isRoute(url)) {
                    return ROUTERS_CATEGORY;
                } else if (UrlUtils.isProvider(url)) {
                    return PROVIDERS_CATEGORY;
                }
                return "";
            }));

    List<URL> configuratorURLs = categoryUrls.getOrDefault(CONFIGURATORS_CATEGORY, Collections.emptyList());
    this.configurators = Configurator.toConfigurators(configuratorURLs).orElse(this.configurators);

    // 如果router 路由节点有变化，则从新将router 下的数据生成router
    List<URL> routerURLs = categoryUrls.getOrDefault(ROUTERS_CATEGORY, Collections.emptyList());
    toRouters(routerURLs).ifPresent(this::addRouters);

    // providers
    // 获得provider URL，然后调用refreshOverrideAndInvoker进行刷新
    List<URL> providerURLs = categoryUrls.getOrDefault(PROVIDERS_CATEGORY, Collections.emptyList());
    //刷新或者覆盖invoker
    //providerURLs:  dubboo://ip:port
    refreshOverrideAndInvoker(providerURLs);
}
```

### refreshOverrideAndInvoker

- 逐个调用注册中心里面的配置，覆盖原来的url，组成最新的url放入overrideDirectoryUrl存储
- 根据provider urls，重新刷新Invoker 

```java
private void refreshOverrideAndInvoker(List<URL> urls) {
    // mock zookeeper://xxx?mock=return null
    overrideDirectoryUrl(); //覆盖
    refreshInvoker(urls); //刷新
}
```

### **refreshInvoker** 

```java
private void refreshInvoker(List<URL> invokerUrls) {
    Assert.notNull(invokerUrls, "invokerUrls should not be null");

    if (invokerUrls.size() == 1
            && invokerUrls.get(0) != null
            && EMPTY_PROTOCOL.equals(invokerUrls.get(0).getProtocol())) {
        this.forbidden = true; // Forbid to access
        this.invokers = Collections.emptyList();
        routerChain.setInvokers(this.invokers);
        destroyAllInvokers(); // Close all invokers
    } else {
        this.forbidden = false; // Allow to access
        Map<String, Invoker<T>> oldUrlInvokerMap = this.urlInvokerMap; // local reference
        
        if (invokerUrls == Collections.<URL>emptyList()) {
            invokerUrls = new ArrayList<>();
        }
        if (invokerUrls.isEmpty() && this.cachedInvokerUrls != null) {
            invokerUrls.addAll(this.cachedInvokerUrls);
        } else {
            this.cachedInvokerUrls = new HashSet<>();
            this.cachedInvokerUrls.addAll(invokerUrls);//Cached invoker urls, convenient for comparison
        }
        //如果url为空，则直接返回
        if (invokerUrls.isEmpty()) {
            return;
        }
        //根据provider url，生成新的invoker
        Map<String, Invoker<T>> newUrlInvokerMap = toInvokers(invokerUrls);// Translate url list to Invoker map

        /**
         * If the calculation is wrong, it is not processed.
         *
         * 1. The protocol configured by the client is inconsistent with the protocol of the server.
         *    eg: consumer protocol = dubbo, provider only has other protocol services(rest).
         * 2. The registration center is not robust and pushes illegal specification data.
         *
         */
        if (CollectionUtils.isEmptyMap(newUrlInvokerMap)) {
            logger.error(new IllegalStateException("urls to invokers error .invokerUrls.size :" + invokerUrls.size() + ", invoker.size :0. urls :" + invokerUrls
                    .toString()));
            return;
        }

        //转化为list
        List<Invoker<T>> newInvokers = Collections.unmodifiableList(new ArrayList<>(newUrlInvokerMap.values()));
        // pre-route and build cache, notice that route cache should build on original Invoker list.
        // toMergeMethodInvokerMap() will wrap some invokers having different groups, those wrapped invokers not should be routed.
        routerChain.setInvokers(newInvokers);
        //如果服务配置了分组，则把分组下的provider包装成StaticDirectory,组成一个invoker
        //实际上就是按照group进行合并
        this.invokers = multiGroup ? toMergeInvokerList(newInvokers) : newInvokers;
        this.urlInvokerMap = newUrlInvokerMap;

        try {
            //旧的url是否在新map里面存在，不存在，就是销毁url对应的Invoker
            destroyUnusedInvokers(oldUrlInvokerMap, newUrlInvokerMap); // Close the unused Invoker
        } catch (Exception e) {
            logger.warn("destroyUnusedInvokers error. ", e);
        }
    }
}
```

### **toInvokers** 

这个方法中有比较长的判断和处理逻辑，我们只需要关心invoker是什么时候初始化的就行。 这里用到了protocol.refer来构建了一个invoker 

`invoker = new InvokerDelegate<>(protocol.refer(serviceType, url), url, providerUrl);`

构建完成之后，会保存在`Map<String, Invoker<T>> newUrlInvokerMap = new HashMap<>();`这个集合中 

```java
private Map<String, Invoker<T>> toInvokers(List<URL> urls) {
    Map<String, Invoker<T>> newUrlInvokerMap = new HashMap<>();
    if (urls == null || urls.isEmpty()) {
        return newUrlInvokerMap;
    }
    Set<String> keys = new HashSet<>();
    String queryProtocols = this.queryMap.get(PROTOCOL_KEY);
    for (URL providerUrl : urls) {
        // If protocol is configured at the reference side, only the matching protocol is selected
        if (queryProtocols != null && queryProtocols.length() > 0) {
            boolean accept = false;
            String[] acceptProtocols = queryProtocols.split(",");
            for (String acceptProtocol : acceptProtocols) {
                if (providerUrl.getProtocol().equals(acceptProtocol)) {
                    accept = true;
                    break;
                }
            }
            if (!accept) {
                continue;
            }
        }
        if (EMPTY_PROTOCOL.equals(providerUrl.getProtocol())) {
            continue;
        }
        if (!ExtensionLoader.getExtensionLoader(Protocol.class).hasExtension(providerUrl.getProtocol())) {
            logger.error(new IllegalStateException("Unsupported protocol " + providerUrl.getProtocol() +
                    " in notified url: " + providerUrl + " from registry " + getUrl().getAddress() +
                    " to consumer " + NetUtils.getLocalHost() + ", supported protocol: " +
                    ExtensionLoader.getExtensionLoader(Protocol.class).getSupportedExtensions()));
            continue;
        }
        URL url = mergeUrl(providerUrl);

        String key = url.toFullString(); // The parameter urls are sorted
        if (keys.contains(key)) { // Repeated url
            continue;
        }
        keys.add(key);
        // Cache key is url that does not merge with consumer side parameters, regardless of how the consumer combines parameters, if the server url changes, then refer again
        Map<String, Invoker<T>> localUrlInvokerMap = this.urlInvokerMap; // local reference
        Invoker<T> invoker = localUrlInvokerMap == null ? null : localUrlInvokerMap.get(key);
        if (invoker == null) { // Not in the cache, refer again
            try {
                boolean enabled = true;
                if (url.hasParameter(DISABLED_KEY)) {
                    enabled = !url.getParameter(DISABLED_KEY, false);
                } else {
                    enabled = url.getParameter(ENABLED_KEY, true);
                }
                if (enabled) {
                    invoker = new InvokerDelegate<>(protocol.refer(serviceType, url), url, providerUrl);
                }
            } catch (Throwable t) {
                logger.error("Failed to refer invoker for interface:" + serviceType + ",url:(" + url + ")" + t.getMessage(), t);
            }
            if (invoker != null) { // Put new invoker in cache
                newUrlInvokerMap.put(key, invoker);
            }
        } else {
            newUrlInvokerMap.put(key, invoker);
        }
    }
    keys.clear();
    return newUrlInvokerMap;
}
```

## **protocol.refer** 

调用指定的协议来进行远程引用。protocol是一个Protocol$Adaptive类 而真正的实现应该是: 

ProtocolListenerWrapper(ProtocolFilterWrapper(QosProtocolWrapper(DubboProtocol.refer) 前面的包装过程，在服务发布的时候已经分析过了，我们直接进入DubboProtocol.refer方法 

### AbstractProtocol#refer

```
@Override
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    return new AsyncToSyncInvoker<>(protocolBindingRefer(type, url));
}
```

### DubboProtocol#protocolBindingRefer

- 优化序列化 
- 构建DubboInvoker 

在构建DubboInvoker时，会构建一个ExchangeClient，通过getClients(url)方法，这里基本可以猜到到是服务的通信建立 

```java
@Override
public <T> Invoker<T> protocolBindingRefer(Class<T> serviceType, URL url) throws RpcException {
    optimizeSerialization(url);

    // create rpc invoker.
    //getClients(url)
    DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
    invokers.add(invoker);

    return invoker;
}
```

### getClients

这里面是获得客户端连接的方法

- 判断是否为共享连接，默认是共享同一个连接进行通信 
- 是否配置了多个连接通道connections，默认只有一个 

```java
private ExchangeClient[] getClients(URL url) {
    // whether to share connection

    boolean useShareConnect = false;

    int connections = url.getParameter(CONNECTIONS_KEY, 0);
    List<ReferenceCountExchangeClient> shareClients = null;
    // if not configured, connection is shared, otherwise, one connection for one service
    //如果没有配置连接数，则默认为共享连接
    if (connections == 0) {
        useShareConnect = true;

        /**
         * The xml configuration should have a higher priority than properties.
         */
        String shareConnectionsStr = url.getParameter(SHARE_CONNECTIONS_KEY, (String) null);
        connections = Integer.parseInt(StringUtils.isBlank(shareConnectionsStr) ? ConfigUtils.getProperty(SHARE_CONNECTIONS_KEY,
                DEFAULT_SHARE_CONNECTIONS) : shareConnectionsStr);
        // 获得一个共享连接 
        shareClients = getSharedClient(url, connections);
    }

    ExchangeClient[] clients = new ExchangeClient[connections];
    for (int i = 0; i < clients.length; i++) {
        if (useShareConnect) {
            clients[i] = shareClients.get(i);

        } else {
            clients[i] = initClient(url);
        }
    }

    return clients;
}
```

### **getSharedClient** 

获得一个共享连接 

```java
private List<ReferenceCountExchangeClient> getSharedClient(URL url, int connectNum) {
    String key = url.getAddress();
    List<ReferenceCountExchangeClient> clients = referenceClientMap.get(key);
	//检查当前的key检查连接是否已经创建过并且可用，如果是，则直接返回并且增加连接的个数的统计
    if (checkClientCanUse(clients)) {
        batchClientRefIncr(clients);
        return clients;
    }

    //如果连接已经关闭或者连接没有创建过
    locks.putIfAbsent(key, new Object());
    synchronized (locks.get(key)) {
        clients = referenceClientMap.get(key);
        // 在创建连接之前，在做一次检查，防止连接并发创建
        if (checkClientCanUse(clients)) {
            batchClientRefIncr(clients);
            return clients;
        }

        // connectNum must be greater than or equal to 1
        // 连接数必须大于等于1
        connectNum = Math.max(connectNum, 1);

        // If the clients is empty, then the first initialization is
        //如果当前消费者还没有和服务端产生连接，则初始化
        if (CollectionUtils.isEmpty(clients)) {
            clients = buildReferenceCountExchangeClientList(url, connectNum);
            //创建clients之后，保存到map中
            referenceClientMap.put(key, clients);

        } else {//如果clients不为空，则从clients数组中进行遍历
            for (int i = 0; i < clients.size(); i++) {
                ReferenceCountExchangeClient referenceCountExchangeClient = clients.get(i);
                // If there is a client in the list that is no longer available, create a new one to replace him.
                // 如果在集合中存在一个连接但是这个连接处于closed状态，则重新构建一个进行替换
                if (referenceCountExchangeClient == null || referenceCountExchangeClient.isClosed()) {
                    clients.set(i, buildReferenceCountExchangeClient(url));
                    continue;
                }

                //增加个数
                referenceCountExchangeClient.incrementAndGetCount();
            }
        }

        /**
         * I understand that the purpose of the remove operation here is to avoid the expired url key
         * always occupying this memory space.
         */
        locks.remove(key);

        return clients;
    }
}
```

### buildReferenceCountExchangeClient

根据连接数配置，来构建指定个数的链接。默认为1 

```java
private ReferenceCountExchangeClient buildReferenceCountExchangeClient(URL url) {
    ExchangeClient exchangeClient = initClient(url);

    return new ReferenceCountExchangeClient(exchangeClient);
}
```

### initClient

终于进入到初始化客户端连接的方法了，猜测应该是根据url中配置的参数进行远程通信的构建 

```java
private ExchangeClient initClient(URL url) {

    // client type setting.
    // 获得连接类型
    String str = url.getParameter(CLIENT_KEY, url.getParameter(SERVER_KEY, DEFAULT_REMOTING_CLIENT));

    //添加默认序列化方式
    url = url.addParameter(CODEC_KEY, DubboCodec.NAME);
    // enable heartbeat by default
    //设置心跳时间
    url = url.addParameterIfAbsent(HEARTBEAT_KEY, String.valueOf(DEFAULT_HEARTBEAT));

    // BIO is not allowed since it has severe performance issue.
    if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
        throw new RpcException("Unsupported client type: " + str + "," +
                " supported client type is " + StringUtils.join(ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions(), " "));
    }

    ExchangeClient client;
    try {
        // connection should be lazy.
        // 是否需要延迟创建连接，注意哦，这里的requestHandler是一个适配器
        if (url.getParameter(LAZY_CONNECT_KEY, false)) {
            client = new LazyConnectExchangeClient(url, requestHandler);

        } else {
            client = Exchangers.connect(url, requestHandler);
        }

    } catch (RemotingException e) {
        throw new RpcException("Fail to create remoting client for service(" + url + "): " + e.getMessage(), e);
    }

    return client;
}
```

### Exchangers.connect

创建一个客户端连接 

```java
public static ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    if (handler == null) {
        throw new IllegalArgumentException("handler == null");
    }
    url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
    return getExchanger(url).connect(url, handler);
}
```

### **HeaderExchange.connect** 

主要关注transporters.connect 

```java
@Override
public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
    return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))), true);
}
```

```java
public static Client connect(URL url, ChannelHandler... handlers) throws RemotingException {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    ChannelHandler handler;
    if (handlers == null || handlers.length == 0) {
        handler = new ChannelHandlerAdapter();
    } else if (handlers.length == 1) {
        handler = handlers[0];
    } else {
        handler = new ChannelHandlerDispatcher(handlers);
    }
    return getTransporter().connect(url, handler);
}
```

### **NettyTransport.connect** 

使用netty构建了一个客户端连接 

```java
@Override
public Client connect(URL url, ChannelHandler listener) throws RemotingException {
    return new NettyClient(url, listener);
}
```

