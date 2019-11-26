# rabbitMQ进阶知识及Spring AMQP

## **RabbitMQ** 进阶知识 

### TTL(Time To Live) 消息的过期时间 

有两种设置方式:

1. 通过队列属性设置消息过期时间 ，所有队列中的消息超过时间未被消费时，都会过期。 

   ```java
   @Bean("ttlQueue")
   public Queue queue() {
       Map<String, Object> map = new HashMap<String, Object>();
       map.put("x-message-ttl", 11000); // 队列中的消息未被消费11秒后过期
       // map.put("x-expire", 30000); // 队列30秒没有使用以后会被删除
       return new Queue("TTL_QUEUE", true, false, false, map);
   }
   ```

2. 设置单条消息的过期时间，在发送消息的时候指定消息属性。 

   ```
   public class TtlSender {
   
       public static void main(String[] args) {
           AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(TtlSender.class);
           RabbitAdmin rabbitAdmin = context.getBean(RabbitAdmin.class);
           RabbitTemplate rabbitTemplate = context.getBean(RabbitTemplate.class);
   
           MessageProperties messageProperties = new MessageProperties();
           messageProperties.setExpiration("4000"); // 消息的过期属性，单位ms
           messageProperties.setDeliveryMode(MessageDeliveryMode.PERSISTENT);
           Message message = new Message("这条消息4秒后过期".getBytes(), messageProperties);
           rabbitTemplate.send("TTL_EXCHANGE", "my.ttl", message);
   
           // 随队列的过期属性过期，单位ms
           rabbitTemplate.convertAndSend("TTL_EXCHANGE", "my.ttl", "这条消息");
   
       }
   }
   ```

如果同时指定了Message TTL和Queue TTL，则小的那个时间生效。 

### 死信队列 

消息在某些情况下会变成死信(Dead Letter)。 

**队列在创建的时候可以指定一个死信交换机DLX(Dead Letter Exchange)。 死信交换机绑定的队列被称为死信队列DLQ(Dead Letter Queue)，DLX实际上也是普通的交换机，DLQ也是普通的队列(例如替补球员也是普通球员)。** 

<img src="assets/image-20191126093055611.png" alt="image-20191126093055611" style="zoom:50%;" />

#### 什么情况下消息会变成死信?

1. 消息被消费者拒绝并且未设置重回队列:(NACK || Reject ) && requeue == false 

2. 消息过期 

3. 队列达到最大长度，超过了 Max length(消息数)或者 Max length bytes (字节数)，最先入队的消息会被发送到 DLX。 

#### 死信队列如何使用? 

1. 声明原交换机(ORI_USE_EXCHANGE)、原队列(ORI_USE_QUEUE)，相互绑定。

   队列中的消息10秒钟过期，因为没有消费者，会变成死信。指定原队列的死信交换 

   机(DEAD_LETTER_EXCHANGE)。 

   ```java
   @Bean("oriUseExchange")
   public DirectExchange exchange() {
       return new DirectExchange("ORI_USE_EXCHANGE", true, false, new HashMap<>());
   }
   
   @Bean("oriUseQueue")
   public Queue queue() {
       Map<String, Object> map = new HashMap<String, Object>();
       map.put("x-message-ttl", 10000); // 10秒钟后成为死信
       map.put("x-dead-letter-exchange", "DEAD_LETTER_EXCHANGE"); // 队列中的消息变成死信后，进入死信交换机
       return new Queue("ORI_USE_QUEUE", true, false, false, map);
   }
   
   @Bean
   public Binding binding(@Qualifier("oriUseQueue") Queue queue,@Qualifier("oriUseExchange") DirectExchange exchange) {
       return BindingBuilder.bind(queue).to(exchange).with("my.ori.use");
   }
   ```

2. 声明死信交换机 (DEAD_LETTER_EXCHANGE ) 、死信队列(GP_DEAD_LETTER_QUEUE)，相互绑定：

   ```java
   /**
    * 队列的死信交换机
    * @return
    */
   @Bean("deatLetterExchange")
   public TopicExchange deadLetterExchange() {
       return new TopicExchange("DEAD_LETTER_EXCHANGE", true, false, new HashMap<>());
   }
   
   @Bean("deatLetterQueue")
   public Queue deadLetterQueue() {
       return new Queue("DEAD_LETTER_QUEUE", true, false, false, new HashMap<>());
   }
   
   @Bean
   public Binding bindingDead(@Qualifier("deatLetterQueue") Queue queue,@Qualifier("deatLetterExchange") TopicExchange exchange) {
       return BindingBuilder.bind(queue).to(exchange).with("#"); // 无条件路由
   }
   ```

3. 最终消费者监听死信队列。 

4. 生产者发送消息。 

**消息流转图：**

![image-20191126094526333](assets/image-20191126094526333.png)

### 延迟队列

我们在实际业务中有一些需要延时发送消息的场景，例如: 

1. 家里有一台智能热水器，需要在 30 分钟后启动
2. 未付款的订单，15 分钟后关闭。

RabbitMQ 本身不支持延迟队列，总的来说有三种实现方案: 

1. 先存储到数据库，用定时任务扫描 
2. 利用RabbitMQ的死信队列(Dead Letter Queue)实现 
3. 利用rabbitmq-delayed-message-exchange插件 



#### TTL + DLX的实现

基于消息 TTL，我们来看一下如何利用死信队列(DLQ)实现延迟队列。

总体步骤: 

1. 创建一个交换机 
2. 创建一个队列，与上述交换机绑定，并且通过属性指定队列的死信交换机。 
3. 创建一个死信交换机
4. 创建一个死信队列
5. 将死信交换机绑定到死信队列
6. 消费者监听死信队列 

消息的流转流程: 

生产者——原交换机——原队列(超过 TTL 之后)——死信交换机——死信队列— —最终消费者。



使用死信队列实现延时消息的缺点：

1. 如果统一用队列来设置消息的 TTL，当梯度非常多的情况下，比如1分钟，2分钟，5分钟，10分钟，20分钟，30分钟......需要创建很多交换机和队列来路由消息。 
2. 如果单独设置消息的TTL，则可能会造成队列中的消息阻塞——前一条消息没 有出队(没有被消费)，后面的消息无法投递(比如第一条消息过期TTL是30min，第二条消息TTL是10min。10分钟后，即使第二条消息应该投递了，但是由于第一条消息还未出队，所以无法投递)。 
3. 可能存在一定的时间误差。 



#### 基于延迟队列插件的实现(Linux) 

在RabbitMQ 3.5.7及以后的版本提供了一个插件 (rabbitmq-delayed-message-exchange)来实现延时队列功能。同时插件依赖Erlang/OPT 18.0 及以上。 

插件源码地址: https://github.com/rabbitmq/rabbitmq-delayed-message-exchange 

插件下载地址: https://bintray.com/rabbitmq/community-plugins/rabbitmq_delayed_message_exchange 

1、进入插件目录 

```shell
whereis rabbitmq
cd /usr/lib/rabbitmq/lib/rabbitmq_server-3.6.12/plugins
```

2、下载插件 

```shell
wget
https://bintray.com/rabbitmq/community-plugins/download_file?file_path=rabbitmq_delayed_message_exchange-0.0.1.ez
```

如果下载的文件名带问号则需要改名，如图: 

![image-20191126095511526](assets/image-20191126095511526.png)

```shell
mv download_file?file_path=rabbitmq_delayed_message_exchange-0.0.1.ez rabbitmq_delayed_message_exchange-0.0.1.ez
```

![image-20191126095539863](assets/image-20191126095539863.png)

3、启用插件 

```shell
rabbitmq-plugins enable rabbitmq_delayed_message_exchange
```

4、停用插件 

```
rabbitmq-plugins disable rabbitmq_delayed_message_exchange
```

5、插件使用 

通过声明一个x-delayed-message类型的Exchange来使用delayed-messaging特性。x-delayed-message是插件提供的类型，并不是rabbitmq本身的(区别于 direct、 topic、fanout、headers)。 

![image-20191126095658803](assets/image-20191126095658803.png)

```java
@Bean("delayExchange")
public TopicExchange exchange() {
    Map<String, Object> argss = new HashMap<String, Object>();
    argss.put("x-delayed-type", "direct");
    return new TopicExchange("DELAY_EXCHANGE", true, false, argss);
}
```

生产者:

消息属性中指定 x-delay 参数。 

```java
// 延迟的间隔时间，目标时刻减去当前时刻
messageProperties.setHeader("x-delay", delayTime.getTime() - now.getTime());
Message message = new Message(msg.getBytes(), messageProperties);

// 不能在本地测试，必须发送消息到安装了插件的服务端
rabbitTemplate.send("DELAY_EXCHANGE", "#", message);
```

### 服务端流控(Flow Control) 

当RabbitMQ生产MQ消息的速度远大于消费消息的速度时，会产生大量的消息堆积，占用系统资源，导致机器的性能下降。我们想要控制服务端接收的消息的数量，应该怎么做呢? 

队列有两个控制长度的属性: 

x-max-length:队列中最大存储最大消息数，超过这个数量，队头的消息会被丢弃。

x-max-length-bytes:队列中存储的最大消息容量(单位 bytes)，超过这个容量，队头的消息会被丢弃。  

**需要注意的是，设置队列长度只在消息堆积的情况下有意义，而且会删除先入队的消息，不能真正地实现服务端限流。** 

![image-20191126100732682](assets/image-20191126100732682.png)

有没有其他办法实现服务端限流呢?

#### 内存控制 

RabbitMQ会在启动时检测机器的物理内存数值。默认当MQ占用40%以上内存时，MQ会主动抛出一个内存警告并阻塞所有连接(Connections)。可以通过修改rabbitmq.config 文件来调整内存阈值，默认值是 0.4，如下所示:  `[{rabbit, [{vm_memory_high_watermark, 0.4}]}]`，也可以用命令动态设置，如果设置成 0，则所有的消息都不能发布。 `rabbitmqctl set_vm_memory_high_watermark 0.3 `

#### 磁盘控制 

另一种方式是通过磁盘来控制消息的发布。当磁盘空间低于指定的值时(默认50MB)，触发流控措施。 

例如:指定为磁盘的 30%或者 2GB: 

https://www.rabbitmq.com/configure.html 

```properties
disk_free_limit.relative = 3.0 
disk_free_limit.absolute = 2GB 
```

### 消费端限流 

默认情况下，如果不进行配置，RabbitMQ 会尽可能快速地把队列中的消息发送到消费者。因为消费者会在本地缓存消息，如果消息数量过多，可能会导致OOM或者影响其他进程的正常运行。 

在消费者处理消息的能力有限，例如消费者数量太少，或者单条消息的处理时间过长的情况下，如果我们希望在一定数量的消息消费完之前，不再推送消息过来，就要用到消费端的流量限制措施。 

可以基于Consumer或者channel设置prefetch count的值，含义为Consumer端的最大的unacked messages 数目。当超过这个数值的消息未被确认，RabbitMQ 会停止投递新的消息给该消费者。 

```java
channel.basicQos(2); // 如果超过 2 条消息没有发送 ACK，当前消费者不再接受队列消息 channel.basicConsume(QUEUE_NAME, false, consumer);
```

**SimpleMessageListenerContainer** 

```java
container.setPrefetchCount(2);	
```

**Spring Boot 配置:** 

```java
spring.rabbitmq.listener.simple.prefetch=2
```

举例: 

channel的prefetch count设置为 5。当消费者有5条消息没有给Broker发送ACK后，RabbitMQ不再给这个消费者投递消息。 

## **Spring AMQP** 

### Spring AMQP 介绍 

Spring AMQP是对Spring基于AMQP的消息收发解决方案，它是一个抽象层，不依赖于特定的AMQP Broker 实现和客户端的抽象，所以可以很方便地替换。比如我们可以使用spring-rabbit来实现。 

```xml
<dependency> 
    <groupId>org.springframework.amqp</groupId> 
    <artifactId>spring-rabbit</artifactId> 
    <version>1.3.5.RELEASE</version>
</dependency>
```

包括3个jar包: Amqp-client-3.3.4.jar,Spring-amqp.jar,Spring.rabbit.jar 

### Spring AMQP核心组件 

#### ConnectionFactory 

Spring AMQP的连接工厂接口，用于创建连接。CachingConnectionFactory是ConnectionFactory的一个实现类。 

#### RabbitAdmin 

RabbitAdmin是AmqpAdmin的实现，封装了对RabbitMQ的基础管理操作，比如对交换机、队列、绑定的声明和删除等。 

```java
@Configuration
public class AmqpConfig {

    @Bean
    public ConnectionFactory connectionFactory() throws Exception {
        CachingConnectionFactory cachingConnectionFactory = new CachingConnectionFactory();
        cachingConnectionFactory.setUri(ResourceUtil.getKey("rabbitmq.uri"));
        return cachingConnectionFactory;
    }

    @Bean
    public RabbitAdmin amqpAdmin(ConnectionFactory connectionFactory) {
        RabbitAdmin admin = new RabbitAdmin(connectionFactory);
        // admin.setAutoStartup(true);
        return admin;
    }

    @Bean
    public SimpleMessageListenerContainer container(ConnectionFactory connectionFactory) {
        SimpleMessageListenerContainer container = new SimpleMessageListenerContainer(connectionFactory);
        container.setConsumerTagStrategy(new ConsumerTagStrategy() {
            public String createConsumerTag(String queue) {
                return null;
            }
        });
        return container;
    }

}
```

```java
public class AdminTest {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AdminTest.class);
        RabbitAdmin rabbitAdmin = context.getBean(RabbitAdmin.class);

        // 声明一个交换机
        rabbitAdmin.declareExchange(new DirectExchange("ADMIN_EXCHANGE", false, false));

        // 声明一个队列
        rabbitAdmin.declareQueue(new Queue("ADMIN_QUEUE", false, false, false));

        // 声明一个绑定
        rabbitAdmin.declareBinding( new Binding("ADMIN_QUEUE", Binding.DestinationType.QUEUE,
                "ADMIN_EXCHANGE", "admin", null));

    }
}
```

为什么我们在配置文件(Spring)或者配置类(SpringBoot)里面定义了交换机、 队列、绑定关系，并没有直接调用Channel的declare的方法，Spring在启动的时候就可以帮我们创建这些元数据?这些事情就是由RabbitAdmin完成的。 

RabbitAdmin实现了InitializingBean接口，里面有唯一的一个方法afterPropertiesSet()，这个方法会在RabbitAdmin的属性值设置完的时候被调用。 

在 afterPropertiesSet()方法中，调用了一个initialize()方法。这里面创建了三个Collection，用来盛放交换机、队列、绑定关系。

最后依次声明返回类型为Exchange、Queue和Binding这些Bean，底层还是调用了Channel的declare的方法。 

```java
declareExchanges(channel, exchanges.toArray(new Exchange[exchanges.size()])); declareQueues(channel, queues.toArray(new Queue[queues.size()])); declareBindings(channel, bindings.toArray(new Binding[bindings.size()]));	
```

#### Message 

Message 是 Spring AMQP 对消息的封装。 

两个重要的属性: 

- body:消息内容。 
- messageProperties:消息属性。 

![image-20191126102507491](assets/image-20191126102507491.png)

#### RabbitTemplate 消息模板 

RabbitTemplate是AmqpTemplate的一个实现(目前为止也是唯一的实现)，用来简化消息的收发，支持消息的确认(Confirm)与返回(Return)。 跟JDBCTemplate一样，它封装了创建连接、创建消息信道、收发消息、消息格式转换 (ConvertAndSend→Message)、关闭信道、关闭连接等等操作。 

针对于多个服务器连接，可以定义多个 Template。可以注入到任何需要收发消息的地方使用。 

确认与回发：

```java
public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
    RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
    rabbitTemplate.setMandatory(true);
    rabbitTemplate.setReturnCallback(new RabbitTemplate.ReturnCallback(){
        public void returnedMessage(Message message,
                                    int replyCode,
                                    String replyText,
                                    String exchange,
                                    String routingKey){
            System.out.println("回发的消息：");
            System.out.println("replyCode: "+replyCode);
            System.out.println("replyText: "+replyText);
            System.out.println("exchange: "+exchange);
            System.out.println("routingKey: "+routingKey);
        }
    });

    rabbitTemplate.setChannelTransacted(true);

    rabbitTemplate.setConfirmCallback(new RabbitTemplate.ConfirmCallback() {
        public void confirm(CorrelationData correlationData, boolean ack, String cause) {
            if (!ack) {
                System.out.println("发送消息失败：" + cause);
                throw new RuntimeException("发送异常：" + cause);
            }
        }
    });


    return rabbitTemplate;
}
```

#### MessageListener 消息侦听 

##### MessageListener 

MessageListener是Spring AMQP异步消息投递的监听器接口，它只有一个方法onMessage，用于处理消息队列推送来的消息，作用类似于Java API中的 Consumer。 

##### MessageListenerContainer 

MessageListenerContainer可以理解为MessageListener的容器，一个Container只有一个Listener，但是可以生成多个线程使用相同的MessageListener同时消费消息。 

Container可以管理Listener的生命周期，可以用于对于消费者进行配置。 

例如:动态添加移除队列、对消费者进行设置，例如ConsumerTag、Arguments、并发、消费者数量、消息确认模式等等。 

```java
    @Bean
    public SimpleMessageListenerContainer container(ConnectionFactory connectionFactory) {
        SimpleMessageListenerContainer container = new SimpleMessageListenerContainer(connectionFactory);
        container.setQueues(getSecondQueue(), getThirdQueue());//监听的队列
        container.setConcurrentConsumers(1);// 最小消费者数
        container.setMaxConcurrentConsumers(5);// 最大的消费者数量
        container.setDefaultRequeueRejected(false);//是否重回队列
        container.setAcknowledgeMode(AcknowledgeMode.AUTO);//签收模式
        container.setExposeListenerChannel(true);
        container.setConsumerTagStrategy(new ConsumerTagStrategy() {
            public String createConsumerTag(String queue) {
                return null;
            }
        });
        return container;
    }
```

在SpringBoot2.0中新增了一个DirectMessageListenerContainer。 

##### MessageListenerContainerFactory 

可以在消费者上指定，当我们需要监听多个RabbitMQ的服务器的时候，指定不同的 MessageListenerContainerFactory。 

```java
@Bean
public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(ConnectionFactory connectionFactory) {
    SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
    factory.setConnectionFactory(connectionFactory);
    factory.setMessageConverter(new Jackson2JsonMessageConverter());
    factory.setAcknowledgeMode(AcknowledgeMode.NONE);
    factory.setAutoStartup(true);
    return factory;
}
```

```java
@Component
@PropertySource("classpath:rabbitmq.properties")
@RabbitListener(queues = "${com.wenbin.firstqueue}", containerFactory="rabbitListenerContainerFactory")
public class FirstConsumer {

    @RabbitHandler
    public void process(@Payload Merchant merchant){
        System.out.println("First Queue received msg : " + merchant.getName());
    }

}
```

关系：

MessageListenerContainerFactory——MessageListenerContainer——MessageListener 

继承关系: 

![image-20191126104534962](assets/image-20191126104534962.png)

![image-20191126104544981](assets/image-20191126104544981.png)

整合演示：

```java
public static void main(String[] args) throws Exception {
    ConnectionFactory connectionFactory = new CachingConnectionFactory(new URI("amqp://guest:guest@localhost:5672"));
    SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
    factory.setConnectionFactory(connectionFactory);
    SimpleMessageListenerContainer container = factory.createListenerContainer();
    // 不用工厂模式也可以创建
    // SimpleMessageListenerContainer container = new SimpleMessageListenerContainer(connectionFactory);
    container.setConcurrentConsumers(1);
    container.setQueueNames("BASIC_SECOND_QUEUE");
    container.setMessageListener(new MessageListener() {
        @Override
        public void onMessage(Message message) {
            System.out.println("收到消息："+message);
        }
    });
    container.start();

    AmqpTemplate template = new RabbitTemplate(connectionFactory);
    template.convertAndSend("BASIC_SECOND_QUEUE", "msg 1");
}
```

#### 转换器 MessageConvertor 

##### MessageConvertor 的作用 

RabbitMQ的消息在网络传输中需要转换成 `byte[]`(字节数组)进行发送，消费者需要对字节数组进行解析。 

在Spring AMQP中，消息会被封装为 `org.springframework.amqp.core.Message` 对象。消息的序列化和反序列化，就是处理Message的消息体body对象。 

如果消息已经是 byte[]格式，就不需要转换。

如果是 String，会转换成 `byte[]`。

如果是 Java 对象，会使用 JDK 序列化将对象转换为 `byte[]`(体积大，效率差)。 

在调用RabbitTemplate的convertAndSend()方法发送消息时，会使用MessageConvertor进行消息的序列化，默认使用SimpleMessageConverter。 

**在某些情况下，我们需要选择其他的高效的序列化工具。如果我们不想在每次发送消息时自己处理消息，就可以直接定义一个MessageConvertor。** 

```java
@Bean
public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(ConnectionFactory connectionFactory) {
    SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
    factory.setConnectionFactory(connectionFactory);
    factory.setMessageConverter(new Jackson2JsonMessageConverter());
    factory.setAcknowledgeMode(AcknowledgeMode.NONE);
    factory.setAutoStartup(true);
    return factory;
}
```

##### MessageConvertor如何工作 

调用了RabbitTemplate的convertAndSend()方法时会使用对应的MessageConvertor进行消息的序列化和反序列化。 

序列化:Object —— Json —— Message(body) —— byte[] 

反序列化:byte[] ——Message —— Json —— Object 

##### 有哪些 MessageConvertor 

在Spring中提供了一个默认的转换器:SimpleMessageConverter。

Jackson2JsonMessageConverter(RbbitMQ自带):将对象转换为 json，然后再转换成字节数组进行传递。 

<img src="assets/image-20191126110835608.png" alt="image-20191126110835608" style="zoom:30%;" />

##### 如何自定义 MessageConverter 

例如:我们要使用Gson格式化消息: 

创建一个类，实现MessageConverter接口，重写toMessage()和fromMessage() 方法。 

toMessage(): Java对象转换为Message 

fromMessage(): Message对象转换为Java对象 

