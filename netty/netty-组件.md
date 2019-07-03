# Netty-组件

## Channel

### 介绍

​		基本的I/O操作（bind()、connect()、read()和write()）依赖于底层网络传输所提供的原语。在基于Java的网络编程中，其基本的构造是Socket。Netty的Channel接口所提供的API，降低了直接使用Socket类的复杂性。此外，Channel也是拥有许多预定义的、专门化实现的广泛类层次结构的根，

下面是一个简短的部分清单：

- EmbeddedChannel；
- LocalServerChannel；
- NioDatagramChannel；
- NioSctpChannel；
- NioSocketChannel。

![image-20190703105119012](assets/image-20190703105119012.png)

每个Channel都将会被分配一个ChannelPipeline和ChannelConfig。

- ChannelConfig包含了该Channel的所有配置设置，并且支持热更新。由于特定的传输可能具有独特的设置，所以它可能会实现一个ChannelConfig的子类型。

- **ChannelPipeline持有所有将应用于入站和出站数据以及事件的ChannelHandler实例**，这些ChannelHandler实现了应用程序用于处理状态变化以及数据处理的逻辑。

  

​		Channel是独一无二的，所以为了保证顺序将Channel声明为java.lang.Comparable的一个子接口。如果两个不同的Channel实例都返回了相同的散列码，那么AbstractChannel中的compareTo()方法的实现将会抛出一个Error。

### 常用方法

| 方法名        | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| eventLoop     | 返回分配给Channel的EventLoop                                 |
| pipeline      | 返回分配给Channel的ChannelPipeline                           |
| isActive      | 如果Channel是活动的,则返回true.活动的意义可能依赖于底层的传输.例如,一个Socket传输一旦连接到了远程节点便是活动的,而一个Datagram传输一旦打开便是活跃的. |
| localAddress  | 返回本地的SocketAddress                                      |
| remoteAddress | 返回远程的SocketAddress                                      |
| write         | 将数据写到远程节点.这个数据将被传递给ChannelPipeline,并且排队直到被冲刷 |
| flush         | 将之前已写的数据冲刷到底层传输,如一个Socket                  |
| writeAndFlush | 等同于调用write()并借着调用flush().                          |

### 内置Channel实现

| 名称     | 包                          | 描述                                                         |
| -------- | --------------------------- | ------------------------------------------------------------ |
| NIO      | io.netty.channel.socket.nio | 使用java.io.channels包作为基础,基于选择器方式的同步非阻塞IO模型. |
| Epoll    | Io.netty.channel.epoll      | 有JNI驱动的epoll()非阻塞IO.这个Channel支持只有在Linux上可用的多种特性,比NIO更快. |
| OIO      | io.netty.channel.socket.oio | 使用java.net包作为基础-使用阻塞IO                            |
| Local    | io.netty.channel.local      | 可以在JVM内部通过管道进行通信的本地传输                      |
| Embedded | io.netty.channel.embedded   | EmbeddedChannel,允许使用ChannelHandler而又不需要一个真正的基于网络的传输.通常用来支撑单元测试. |



#### NIO-同步非阻塞I/O

选择器返回的时间类型:

| 名称       | 描述                                                         |
| ---------- | ------------------------------------------------------------ |
| OP_ACCEPT  | 在接受新连接并创建Channel时获得通知.                         |
| OP_CONNECT | 在建立一个连接时获得通知                                     |
| OP_READ    | 数据已经就绪,可以从Channel中读取时获得通知.                  |
| OP_WRITE   | 可以向Channel中写更多数据时获得通知.这处理了套接字缓冲区呗完全填满时的情况,这种情况通常发生在数据发送速度比远程节点的处理速度更快的时候. |

封装处理了如下图java原生NIO的API的处理流程:

![image-20190703113548178](assets/image-20190703113548178.png)

#### Epoll-Linux的本地非阻塞I/O

​		Netty的NIO传输基于Java提供的异步/非阻塞网络编程的通用抽象。虽然这保证了Netty的非阻塞API可以在任何平台上使用，但它也包含了相应的限制，因为**JDK为了在所有系统上提供相同的功能，必须做出妥协。**

​		Linux作为高性能网络编程的平台，其重要性与日俱增，这催生了大量先进特性的开发，其中包括**epoll——一个高度可扩展的I/O事件通知特性**。这个API自Linux内核版本2.5.44（2002）被引入，提供了**比旧的POSIXselect和poll系统调用[3]更好的性能**，同时现在也是***Linux上非阻塞网络编程的事实标准。***Linux JDK NIO API使用了这些epoll调用。

​		Netty为Linux提供了一组NIO API，以一种和它本身的设计更加一致的方式使用epoll，并且以一种更加轻量的方式使用中断。**如果你的应用程序旨在运行于Linux系统，那么请考虑利用这个版本的Channel；你将发现在高负载下它的性能要优于JDK的NIO实现。**

#### OIO-阻塞I/O

​		Netty是如何能够使用用于异步传输相同的API来支持OIO的呢。答案就是，Netty利用了SO_TIMEOUT这个Socket标志，它指定了等待一个I/O操作完成的最大毫秒数。如果操作在指定的时间间隔内没有完成，则将会抛出一个SocketTimeoutException。Netty将捕获这个异常并继续处理循环。在EventLoop下一次运行时，它将再次尝试。这实际上也是类似于Netty这样的异步框架能够支持OIO的唯一方式。

逻辑图:

![image-20190703114711321](assets/image-20190703114711321.png)

#### Local-Jvm内部通信的Channel

​		Netty提供了一个Local传输，用于在同一个JVM中运行的客户端和服务器程序之间的异步通信。同样，这个传输也支持对于所有Netty传输实现都共同的API。

​		在这个传输中，和服务器Channel相关联的SocketAddress并没有绑定物理网络地址；相反，只要服务器还在运行，它就会被存储在注册表里，并在Channel关闭时注销。因为这个传输并不接受真正的网络流量，所以它并不能够和其他传输实现进行互操作。因此，**客户端希望连接到（在同一个JVM中）使用了这个传输的服务器端时也必须使用它。**除了这个限制，它的使用方式和其他的传输一模一样。

- #### Embedded-单元测试


​	特殊的Channel实现——**EmbeddedChannel**，它是**Netty专门为改进针对ChannelHandler的单元测试而提供的。**

### 支持的协议

![image-20190703115219298](assets/image-20190703115219298.png)

## ByteBuf

### 介绍

​		网络数据的基本单位总是字节。JavaNIO提供了ByteBuffer作为它的字节容器，但是这个类使用起来过于复杂，而且也有些繁琐。Netty的ByteBuffer替代品是ByteBuf，一个强大的实现，既解决了JDKAPI的局限性，又为网络应用程序的开发者提供了更好的API。

ByteBufAPI的优点：

- 它可以被用户自定义的缓冲区类型扩展；

- 通过内置的复合缓冲区类型实现了透明的零拷贝；

- 容量可以按需增长（类似于JDK的StringBuilder）；

- 在读和写这两种模式之间切换不需要调用ByteBuffer的flip()方法；

- 读和写使用了不同的索引；

- 支持方法的链式调用；

- 支持引用计数；

- 支持池化。

  

​	ByteBuf维护了两个不同的索引：一个用于读取，一个用于写入。当你从ByteBuf读取时，它的readerIndex将会被递增已经被读取的字节数。同样地，当你写入ByteBuf时，它的writerIndex也会被递增。

下图展示了一个空ByteBuf的布局结构和状态。

![image-20190703133644030](assets/image-20190703133644030.png)

1. ByteBuf维护了readerIndex和writerIndex索引
2. 当readerIndex > writerIndex时，则抛出IndexOutOfBoundsException
3. ByteBuf容量 = writerIndex。
4. ByteBuf可读容量 = writerIndex - readerIndex
5. readXXX()和writeXXX()方法将会推进其对应的索引。自动推进
6. getXXX()和setXXX()方法将对writerIndex和readerIndex无影响

### ByteBuf使用模式

​		**ByteBuf本质是: 一个由不同的索引分别控制读访问和写访问的字节数组。**

​		ByteBuf共有三种模式: 堆缓冲区模式(Heap Buffer)、直接缓冲区模式(Direct Buffer)和复合缓冲区模式(Composite Buffer)

#### 堆缓冲区模式(Heap Buffer)

堆缓冲区模式又称为：支撑数组(backing array)。将数据存放在JVM的堆空间，通过将数据存储在数组中实现

- 堆缓冲的优点: 由于数据存储在Jvm堆中可以快速创建和快速释放，并且提供了数组直接快速访问的方法
- 堆缓冲的缺点: 每次数据与I/O进行传输时，都需要将数据拷贝到直接缓冲区

```java
public static void heapBuffer() {
  ByteBuf heapBuf = Unpooled.buffer(1024);
  heapBuf.writeBytes("heapBuffer".getBytes());
  // 检查ByteBuf是有支撑数组
  if (heapBuf.hasArray()) {
    // 获取支撑数组的引用
    // 当heapBuf.hasArray()返回false时,尝试访问支撑数组将处罚UnsupportedOperationException
    byte[] array = heapBuf.array();
    // 计算第一个字节的偏移量,也就是可以开始读的字节
    int offset = heapBuf.arrayOffset() + heapBuf.readerIndex();
    // 获得可读字节数
    int length = heapBuf.readableBytes();
    handleArray(array, offset, length);
  }
}

```

handleArray方法的实现:

```
private static void handleArray(byte[] array, int offset, int length) {
    for (int i = offset; i < length; i++) {
        byte b = array[i];
        System.out.print((char) b);

    }
    System.out.println("");
    System.out.println("------------------");
}
```

执行结果:

```
heapBuffer
------------------
```



#### 直接缓冲区模式(Direct Buffer)

Direct Buffer属于堆外分配的直接内存，不会占用堆的容量。适用于套接字传输过程，避免了数据从内部缓冲区拷贝到直接缓冲区的过程，性能较好

- Direct Buffer的优点: 使用Socket传递数据时性能很好，避免了数据从Jvm堆内存拷贝到直接缓冲区的过程。提高了性能
- Direct Buffer的缺点: 相对于堆缓冲区而言，Direct Buffer分配内存空间和释放更为昂贵
- 对于涉及大量I/O的数据读写，建议使用Direct Buffer。而对于用于后端的业务消息编解码模块建议使用Heap Buffer

```java
public static void directBuffer() {
  ByteBuf directBuf = Unpooled.directBuffer();
  directBuf.writeBytes("directBuffer".getBytes());
  // 检查ByteBuf是否由数组支撑.如果不是,则这是一个直接缓存区
  if (!directBuf.hasArray()) {
    // 获取可读字节数
    int length = directBuf.readableBytes();
    // 声明一个新的数组来保存具有该长度的字节数据
    byte[] array = new byte[length];
    // 将字节复制到该数组
    directBuf.getBytes(directBuf.readerIndex(), array);
    handleArray(array, 0, length);
  }
}

```

执行结果:

```
directBuffer
------------------
```



#### 复合缓冲区模式(Composite Buffer)

Composite Buffer是Netty特有的缓冲区。**本质上类似于提供一个或多个ByteBuf的组合视图，可以根据需要添加和删除不同类型的ByteBuf。**

- 想要理解Composite Buffer，请记住：它是一个组合视图。它提供一种访问方式让使用者自由的组合多个ByteBuf，避免了拷贝和分配新的缓冲区。
- **Composite Buffer不支持访问其支撑数组。**因此如果要访问，需要先将内容拷贝到堆内存中，再进行访问
- 下图是将两个ByteBuf：头部+Body组合在一起，没有进行任何复制过程。仅仅创建了一个视图

![image-20190703143218808](assets/image-20190703143218808.png)

```java
public static void byteBufComposite() {
        // Netty使用了CompositeByteBuf来优化套接字的I/O操作，
        // 尽可能地消除了由JDK的缓冲区实现所导致的性能以及内存使用率的惩罚。
        CompositeByteBuf messageBuf = Unpooled.compositeBuffer();
        ByteBuf headerBuf = Unpooled.buffer(1024); // can be backing or direct
        headerBuf.writeBytes("headerBuf".getBytes());
        ByteBuf bodyBuf = Unpooled.directBuffer();   // can be backing or direct
        bodyBuf.writeBytes("bodyBuf".getBytes());
        // 将ByteBuf实例追加到CompositeByteBuf
        messageBuf.addComponents(true, headerBuf, bodyBuf);

        // 访问CompositeByteBuf中的数据和访问直接缓冲区的模式相同,应为混合模式不支持访问支撑数组
        int length = messageBuf.readableBytes();
        byte[] array = new byte[length];
        messageBuf.getBytes(messageBuf.readerIndex(), array);
        handleArray(array, 0, length);

        // CompositeByteBuf像是一个ByteBuf的容器
        // 循环遍历所有ByteBuf
        messageBuf.removeComponent(0); // remove the header
        for (ByteBuf buf : messageBuf) {
            length = buf.readableBytes();
            byte[] array1 = new byte[length];
            messageBuf.getBytes(buf.readerIndex(), array1);
            handleArray(array1, 0, length);
        }

        // 重新写入ByteBuff
        messageBuf.writeBytes("byteBufComposite".getBytes());
  
        length = messageBuf.readableBytes();
        byte[] array1 = new byte[length];
        messageBuf.getBytes(messageBuf.readerIndex(), array1);
        handleArray(array1, 0, length);

    }
```

### 字节级操作

​		ByteBuf提供了许多超出基本读、写操作的方法用于修改它的数据。

#### 随机访问索引

 		ByteBuf的索引是从零开始的：第一个字节的索引是0，最后一个字节的索引总是capacity() - 1。

- readXXX()和writeXXX()方法将会推进其对应的索引readerIndex和writerIndex。自动推进
- getXXX()和setXXX()方法用于访问数据，对writerIndex和readerIndex无影响

```java
public static void byteBufRelativeAccess() {
  // readXXX()和writeXXX()方法将会推进其对应的索引readerIndex和writerIndex。自动推进
  // getXXX()和setXXX()方法用于访问数据，对writerIndex和readerIndex无影响
  ByteBuf buffer = BYTE_BUF_FROM_SOMEWHERE;

  // writeXXX方法会推进writetIndex
  buffer.writeBytes("byteBufRelativeAccess".getBytes());
  buffer.writeBytes("1234".getBytes());

  // setXXX方法对writerIndex无影响
  buffer.setBytes(30, "setXXX".getBytes());
  getPrint(buffer);

  // 再次通过write方法写入. 使用set方法写入的数据被覆盖
  buffer.writeBytes("1234567890".getBytes());
  getPrint(buffer);

  // 使用read方法剩余一个未被覆盖的X字符,不应该被输出
  readPrint(buffer);
}

private static void getPrint(ByteBuf buffer) {
  for (int i = 0; i < buffer.capacity(); i++) {
    // getXXX()方法用于访问数据对readerIndex无影响
    byte b = buffer.getByte(i);
    System.out.print((char) b);
  }

  System.out.println("");
  System.out.println("------------------");
}

private static void readPrint(ByteBuf buffer) {

  while (buffer.readableBytes() >= 1) {
    byte b = buffer.readByte();
    System.out.print((char) b);
  }

  System.out.println("");
  System.out.println("------------------");
}
```

执行结果:

```
byteBufRelativeAccess1234     setXXX                        
------------------
byteBufRelativeAccess12341234567890X                        
------------------
byteBufRelativeAccess12341234567890
------------------
```

#### 顺序访问索引

Netty的ByteBuf同时具有读索引和写索引，但JDK的ByteBuffer只有一个索引，所以JDK需要调用flip()方法在读模式和写模式之间切换。

-  ByteBuf被读索引和写索引划分成3个区域：可丢弃字节区域，可读字节区域和可写字节区域

![image-20190703164521238](assets/image-20190703164521238.png)

##### 可丢弃字节区

可丢弃字节区域是指:[0，readerIndex)之间的区域。可调用discardReadBytes()方法丢弃已经读过的字节。

    1. discardReadBytes()效果 ----- 将可读字节区域(CONTENT)[readerIndex, writerIndex)往前移动readerIndex位，同时修改读索引和写索引。
  2. discardReadBytes()方法会移动可读字节区域内容(CONTENT)。如果频繁调用，会有多次数据复制开销，对性能有一定的影响

![image-20190703172153807](assets/image-20190703172153807.png)

```java
public static void byteBufDiscardReadBytes() {

    ByteBuf buffer = BYTE_BUF_FROM_SOMEWHERE;
    buffer.writeBytes("1234".getBytes());
    // 使用read读取,移动readerIndex索引,
    readPrint(buffer);

    // 写入一些未被读取过的数据
    buffer.writeBytes("5678".getBytes());

    System.out.println("此时第一个1234为<可丢弃字节区域>");
    getPrint(buffer);
    System.out.println("调用discardReadBytes(),1234被未被读取");
    buffer.discardReadBytes();
    getPrint(buffer);
    // 写入abcd,在之前存入5678的地方会被新数据覆盖掉
    buffer.writeBytes("abcd".getBytes());
    getPrint(buffer);

}
```

执行结果:

```
1234
------------------
此时第一个1234为<可丢弃字节区域>
12345678                                                    
------------------
调用discardReadBytes()
56785678                                                    
------------------
5678abcd                                                    
------------------
```

##### 可读字节区

​		ByteBuf的可读字节分段存储了实际数据。新分配的、包装的或者复制的缓冲区的默认的readerIndex值为0。任何名称以read或者skip开头的操作都将检索或者跳过位于当前readerIndex的数据，并且将它增加已读字节数。

​	如果被调用的方法需要一个ByteBuf参数作为写入的目标，并且没有指定目标索引参数，那么该目标缓冲区的writerIndex也将被增加

例如:

![image-20190703173426479](assets/image-20190703173426479.png)

如果尝试在缓冲区的可读字节数已经耗尽时从中读取数据，那么将会引发一个IndexOutOfBoundsException。



##### 可写字节区

​		可写字节分段是指一个拥有未定义内容的、写入就绪的内存区域。新分配的缓冲区的writerIndex的默认值为0。任何名称以write开头的操作都将从当前的writerIndex处开始写数据，并将它增加已经写入的字节数。

​		如果写操作的目标也是ByteBuf，并且没有指定源索引的值，则源缓冲区的readerIndex也同样会被增加相同的大小。

例如:

![image-20190703173724554](assets/image-20190703173724554.png)

​		如果尝试往目标写入超过目标容量的数据，将会引发一个IndexOutOfBoundException[5]。

```java
public static void writeAndGetPrint() {
    ByteBuf buffer = BYTE_BUF_FROM_SOMEWHERE;
    String str = "123456789";
    // 确定写缓冲区是否还有足够的空间
    while (buffer.writableBytes() >= 9) {
        buffer.writeBytes(str.getBytes());
    }
    getPrint(buffer);
}
```

输出结果:

```
123456789123456789123456789123456789123456789123456789      
------------------
```

#### 索引管理

1. markReaderIndex()+resetReaderIndex() ----- markReaderIndex()是先备份当前的readerIndex，resetReaderIndex()则是将刚刚备份的readerIndex恢复回来。常用于dump ByteBuf的内容，又不想影响原来ByteBuf的readerIndex的值
2. readerIndex(int) ----- 设置readerIndex为固定的值
3. writerIndex(int) ----- 设置writerIndex为固定的值
4. clear() ----- 效果是: readerIndex=0, writerIndex(0)。不会清除内存
5. 调用clear()比调用discardReadBytes()轻量的多。仅仅重置readerIndex和writerIndex的值，不会拷贝任何内存，开销较小。



```java
public static void byteBufIndexManager() {
    ByteBuf buffer = BYTE_BUF_FROM_SOMEWHERE;
    buffer.writeBytes("1234567890".getBytes());
    getPrint(buffer);
    // markReaderIndex()+resetReaderIndex() ----- markReaderIndex()是先备份当前的readerIndex，
    // resetReaderIndex()则是将刚刚备份的readerIndex恢复回来。常用于dump ByteBuf的内容，又不想影响原
    // 来ByteBuf的readerIndex的值
    buffer.markReaderIndex();       // 读取前标记
    readPrint(buffer);              // 读取一次
    buffer.resetReaderIndex();      // reset复位ReaderIndex
    readPrint(buffer);              // 在读取一次


    buffer.writeBytes("123456789".getBytes());
    getPrint(buffer);
    // 设置readerIndex为固定的值
    buffer.readerIndex(12);
    readPrint(buffer);

    // clear() ----- 效果是: readerIndex=0, writerIndex(0)。不会清除内存
    buffer.clear();
    // 使用read不会独处任何数据
    readPrint(buffer);
    // 使用get方法
    getPrint(buffer);
}
```

#### 查找操作(indexOf)

​		查找ByteBuf指定的值。类似于，String.indexOf("str")操作

1.  最简单的方法 ----- indexOf(）
2.  利用ByteProcessor作为参数来查找某个指定的值。

```java
public static void byteProcessor() {
    ByteBuf buffer = Unpooled.buffer(); //get reference form somewhere
    byte[] b = new byte[]{(byte) 8,(byte) 9,(byte) 10};
    buffer.writeBytes(b);

    // 使用indexOf()方法来查找
    int i = buffer.indexOf(buffer.readerIndex(), buffer.writerIndex(), (byte)9);
    System.out.println(i);
    // 使用ByteProcessor查找给定的值
    int index = buffer.forEachByte(ByteProcessor.FIND_CR);
    System.out.println(index);
}
```

输出结果:

```
1
-1
```

#### 派生缓冲区

​		派生缓冲区为ByteBuf提供了以专门的方式来呈现其内容的视图。

这类视图是通过以下方法被创建的：

1. duplicate()；
2. slice()；
3. slice(int,int)；
4. Unpooled.unmodifiableBuffer(…)；
5. order(ByteOrder)；
6. readSlice(int)。

​		每个这些方法都将返回一个新的ByteBuf实例，它具有自己的读索引、写索引和标记索引。其内部存储和JDK的ByteBuffer一样也是共享的。这使得派生缓冲区的创建成本是很低廉的，但是这也意味着，如果你修改了它的内容，也同时修改了其对应的源实例，所以要小心。

视图代码:

```java
public static void byteBufSlice() {
    Charset utf8 = Charset.forName("UTF-8");
    ByteBuf buf = Unpooled.copiedBuffer("Netty in Action rocks!", utf8);
    ByteBuf sliced = buf.slice(0, 15);
    System.out.println(sliced.toString(utf8));
    // 改变下标0位置的判断两个ByteBuf值是否一样
    buf.setByte(0, (byte)'J');
    // 是同一套数据,所以相等.
    System.out.println(buf.getByte(0) == sliced.getByte(0));
}
```

执行结果:

```
Netty in Action
true
```

复制代码:

```java
public static void byteBufCopy() {
    Charset utf8 = Charset.forName("UTF-8");
    ByteBuf buf = Unpooled.copiedBuffer("Netty in Action rocks!", utf8);
    ByteBuf copy = buf.copy(0, 15);
    System.out.println(copy.toString(utf8));
    // 改变下标0位置的判断两个ByteBuf值是否一样
    buf.setByte(0, (byte)'J');
    // 不是同一数据.所以为false
    System.out.println(buf.getByte(0) == copy.getByte(0));

    getPrint(buf);
    getPrint(copy);
}
```

执行结果:

```
Netty in Action
false
Jetty in Action rocks!                                            
------------------
Netty in Action
------------------
```

#### 读/写操作

两种类别的读/写操作：

1. get()和set()操作，从给定的索引开始，并且保持索引不变；
2. read()和write()操作，从给定的索引开始，并且会根据已经访问过的字节数对索引进行调整。
3. get()操作，set()操作、read()操作和write()操作完整的列表可参考书籍或API

#### 更多操作

​		由ByteBuf提供的其他有用操作。

| 名称            | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| isReadable()    | 如果至少有一个字节可供读取,则返回true                        |
| isWritable()    | 如果至少有一个字节可被写入,则返回true                        |
| readableBytes() | 返回可被读取的字节数                                         |
| writableBytes() | 返回可被写入的字节数                                         |
| capacity()      | 返回ByteBuf可容纳的字节数.在此之后,它会尝试再次扩展直到达到maxCapacity() |
| maxCapacity()   | 返回ByteBuf可以容乃的最大字节数.                             |
| hasArray()      | 如果ByteBuf由一个字节数组支撑,则返回true                     |
| array()         | 如果ByteBuf由一个字节数组支撑则返回该数组;否则,它将抛出一个UnsupportedOperationException |

### ByteBufHolder接口

ByteBufHolder为Netty的高级特性提供了支持，如缓冲区池化，可以从池中借用ByteBuf，并且在需要时自动释放。

1. ByteBufHolder是ByteBuf的容器，可以通过子类实现ByteBufHolder接口，根据自身需要添加自己需要的数据字段。可以用于自定义缓冲区类型扩展字段。
2. Netty提供了一个默认的实现DefaultByteBufHolder。

ByteBufHolder的操作

| 名称        | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| content()   | 返回由这个ByteBufHolder所持有的ByteBuf                       |
| copy()      | 返回这个ByteBufHolder的一个深拷贝,包括一个其所包含的ByteBuf的非共享拷贝. |
| duplicate() | 返回这个ByteBufHolder的一个浅拷贝，包括一个其所包含的ByteBuf的共享拷贝. |

代码:

```java
public class DefaultByteBufHolder implements ByteBufHolder {

    private final ByteBuf data;

    public DefaultByteBufHolder(ByteBuf data) {
        if (data == null) {
            throw new NullPointerException("data");
        }
        this.data = data;
    }

    @Override
    public ByteBuf content() {
        if (data.refCnt() <= 0) {
            throw new IllegalReferenceCountException(data.refCnt());
        }
        return data;
    }

    /**
     * {@inheritDoc}
     * <p>
     * This method calls {@code replace(content().copy())} by default.
     */
    @Override
    public ByteBufHolder copy() {
        return replace(data.copy());
    }

    /**
     * {@inheritDoc}
     * <p>
     * This method calls {@code replace(content().duplicate())} by default.
     */
    @Override
    public ByteBufHolder duplicate() {
        return replace(data.duplicate());
    }
    // ...
}
```

### ByteBuf分配

#### 按需分配:ByteBufAllocator接口

#### Unpooled缓冲区

#### ByteBufUtil类



### 引用计数