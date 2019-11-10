# zookeeper的watcher机制原理

## **Watcher** 的基本流程

zookeeper的watcher机制，总的来说可以分为三个过程：

- 客户端注册Watcher。
- 服务器处理Watcher。
- 客户端回调Watcher。

客户端注册 watcher有3种方式，getData、exists、getChildren。以如下代码为例,来分析整个触发机制的原理

### 基于zkclient客户端发起一个数据操作

```xml
<dependency>
    <groupId>com.101tec</groupId> 
    <artifactId>zkclient</artifactId> 
    <version>0.10</version>
</dependency>
```



```java
public class WatcherDemo {
    private static String CONNECTION_STR = "192.168.1.4:2181,192.168.1.4:2182,192.168.1.4:2183";


    public static void main (String[] args) throws KeeperException, InterruptedException, IOException {
        ZooKeeper zookeeper = new ZooKeeper(CONNECTION_STR, 4000, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                System.out.println("event.type" + event.getType());
            }
        });
        //创建节点
        zookeeper.create("/watch1", "0".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        zookeeper.exists("/watch1", true); //注册监听
        Thread.sleep(1000);
        zookeeper.setData("/watch1", "1".getBytes(), -1); //修改节点的值触发监听
        System.in.read();
    }
}
```

### zookeeper客户端的初始化过程

```java
ZooKeeper zookeeper = new ZooKeeper(CONNECTION_STR, 4000, new Watcher() {
    @Override
    public void process(WatchedEvent event) {
        System.out.println("event.type" + event.getType());
    }
});
```

在创建一个zookeeper客户端对象实例时，我们通过new Watcher()向构造方法中传入一个默认的Watcher，这个Watcher将作为整个zookeeper会话期间默认Watcher，会一直被保存在客户端ZKWatchManager的defaultWatcher中，代码如下：

```java
public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher, boolean canBeReadOnly) throws IOException {
    this.watchManager = new ZooKeeper.ZKWatchManager();
    LOG.info("Initiating client connection, connectString=" + connectString + " sessionTimeout=" + sessionTimeout + " watcher=" + watcher);
    this.watchManager.defaultWatcher = watcher;
    ConnectStringParser connectStringParser = new ConnectStringParser(connectString);
    HostProvider hostProvider = new StaticHostProvider(connectStringParser.getServerAddresses());
    this.cnxn = new ClientCnxn(connectStringParser.getChrootPath(), hostProvider, sessionTimeout, this, this.watchManager, getClientCnxnSocket(), canBeReadOnly);
    this.cnxn.start();
}
```

代码中ClientCnxn：是zookeeper客户端和zookeeper服务器端进行通信和事件通知处理的主要类，它内部包含两个类：

1. SendThread：负责客户端和服务器端的数据通信，也包括事件信息的传输。
2. EventThread：主要在客户端回调注册的Watcher进行通知处理。

### ClientCnxn初始化

```java
public ClientCnxn(String chrootPath, HostProvider hostProvider, int sessionTimeout, ZooKeeper zooKeeper, ClientWatchManager watcher, ClientCnxnSocket clientCnxnSocket, long sessionId, byte[] sessionPasswd, boolean canBeReadOnly) {
    this.authInfo = new CopyOnWriteArraySet();
    this.pendingQueue = new LinkedList();
    this.outgoingQueue = new LinkedList();
    this.sessionPasswd = new byte[16];
    this.closing = false;
    this.seenRwServerBefore = false;
    this.eventOfDeath = new Object();
    this.xid = 1;
    this.state = States.NOT_CONNECTED;
    this.zooKeeper = zooKeeper;
    this.watcher = watcher;
    this.sessionId = sessionId;
    this.sessionPasswd = sessionPasswd;
    this.sessionTimeout = sessionTimeout;
    this.hostProvider = hostProvider;
    this.chrootPath = chrootPath;
    this.connectTimeout = sessionTimeout / hostProvider.size();
    this.readTimeout = sessionTimeout * 2 / 3;
    this.readOnly = canBeReadOnly;
    // 初始化sendThread
    this.sendThread = new ClientCnxn.SendThread(clientCnxnSocket);
    // 初始化EventThread
    this.eventThread = new ClientCnxn.EventThread();
}

// 启动两个线程
public void start() {
    this.sendThread.start();
    this.eventThread.start();
}
```

## 服务器端接收请求处理流程

### NIOServerCnxnFactory#run

```java
//当收到客户端的请求时，会需要从这个方法里面来看-> create/delete/setdata
public void run() {
    while (!ss.socket().isClosed()) {
        try {
            selector.select(1000);
            Set<SelectionKey> selected;
            synchronized (this) {
                selected = selector.selectedKeys();
            }
            ArrayList<SelectionKey> selectedList = new ArrayList<SelectionKey>(
                    selected);
            Collections.shuffle(selectedList);
            for (SelectionKey k : selectedList) {
                if ((k.readyOps() & SelectionKey.OP_ACCEPT) != 0) {
                    SocketChannel sc = ((ServerSocketChannel) k
                            .channel()).accept();
                    InetAddress ia = sc.socket().getInetAddress();
                    int cnxncount = getClientCnxnCount(ia);
                    if (maxClientCnxns > 0 && cnxncount >= maxClientCnxns){
                        LOG.warn("Too many connections from " + ia
                                 + " - max is " + maxClientCnxns );
                        sc.close();
                    } else {
                        LOG.info("Accepted socket connection from "
                                 + sc.socket().getRemoteSocketAddress());
                        sc.configureBlocking(false);
                        SelectionKey sk = sc.register(selector,
                                SelectionKey.OP_READ);
                        NIOServerCnxn cnxn = createConnection(sc, sk);
                        sk.attach(cnxn);
                        addCnxn(cnxn);
                    }
                } else if ((k.readyOps() & (SelectionKey.OP_READ | SelectionKey.OP_WRITE)) != 0) {
                    NIOServerCnxn c = (NIOServerCnxn) k.attachment();
                    // 处理客户端发送的请求。
                    c.doIO(k);
                } else {
                    if (LOG.isDebugEnabled()) {
                        LOG.debug("Unexpected ops in select "
                                  + k.readyOps());
                    }
                }
            }
            selected.clear();
        } catch (RuntimeException e) {
            LOG.warn("Ignoring unexpected runtime exception", e);
        } catch (Exception e) {
            LOG.warn("Ignoring exception", e);
        }
    }
    closeAll();
    LOG.info("NIOServerCnxn factory exited run method");
}
```

其中```c.doIO(k);```处理客户端发送请求。



### NIOServerCnxn#doIO

```java
void doIO(SelectionKey k) throws InterruptedException {
    try {
        if (isSocketOpen() == false) {
            LOG.warn("trying to do i/o on a null socket for session:0x"
                     + Long.toHexString(sessionId));

            return;
        }
        if (k.isReadable()) {
            int rc = sock.read(incomingBuffer);
            if (rc < 0) {
                throw new EndOfStreamException(
                        "Unable to read additional data from client sessionid 0x"
                        + Long.toHexString(sessionId)
                        + ", likely client has closed socket");
            }
            if (incomingBuffer.remaining() == 0) {
                boolean isPayload;
                if (incomingBuffer == lenBuffer) { // start of next request
                    incomingBuffer.flip();
                    isPayload = readLength(k);
                    incomingBuffer.clear();
                } else {
                    // continuation
                    isPayload = true;
                }
                if (isPayload) { // not the case for 4letterword
                    // 处理请求。
                    readPayload();
                }
                else {
                    // four letter words take care
                    // need not do anything else
                    return;
                }
            }
        }
      // ....省略部分代码
}
```

### NIOServerCnxn#readPayload

```java
private void readPayload() throws IOException, InterruptedException {
    if (incomingBuffer.remaining() != 0) { // have we read length bytes?
        int rc = sock.read(incomingBuffer); // sock is non-blocking, so ok
        if (rc < 0) {
            throw new EndOfStreamException(
                    "Unable to read additional data from client sessionid 0x"
                    + Long.toHexString(sessionId)
                    + ", likely client has closed socket");
        }
    }

    if (incomingBuffer.remaining() == 0) { // have we read length bytes?
        packetReceived();
        incomingBuffer.flip();
        if (!initialized) {
            readConnectRequest();
        } else {
            // 处理请求
            readRequest();
        }
        lenBuffer.clear();
        incomingBuffer = lenBuffer;
    }
}
```

### NIOServerCnxn#readRequest

```java
private void readRequest() throws IOException {
    zkServer.processPacket(this, incomingBuffer);
}
```

### ZooKeeperServer#processPacket

处理客户端传送过来的数据包

```java
public void processPacket(ServerCnxn cnxn, ByteBuffer incomingBuffer) throws IOException {
    // We have the request, now process and setup for next
    InputStream bais = new ByteBufferInputStream(incomingBuffer);
    BinaryInputArchive bia = BinaryInputArchive.getArchive(bais);
    RequestHeader h = new RequestHeader();
    h.deserialize(bia, "header");
    // Through the magic of byte buffers, txn will not be
    // pointing
    // to the start of the txn
    incomingBuffer = incomingBuffer.slice();
    if (h.getType() == OpCode.auth) {
        LOG.info("got auth packet " + cnxn.getRemoteSocketAddress());
        AuthPacket authPacket = new AuthPacket();
        ByteBufferInputStream.byteBuffer2Record(incomingBuffer, authPacket);
        String scheme = authPacket.getScheme();
        AuthenticationProvider ap = ProviderRegistry.getProvider(scheme);
        Code authReturn = KeeperException.Code.AUTHFAILED;
        if(ap != null) {
            try {
                authReturn = ap.handleAuthentication(cnxn, authPacket.getAuth());
            } catch(RuntimeException e) {
                LOG.warn("Caught runtime exception from AuthenticationProvider: " + scheme + " due to " + e);
                authReturn = KeeperException.Code.AUTHFAILED;                   
            }
        }
        if (authReturn!= KeeperException.Code.OK) {
            if (ap == null) {
                LOG.warn("No authentication provider for scheme: "
                        + scheme + " has "
                        + ProviderRegistry.listProviders());
            } else {
                LOG.warn("Authentication failed for scheme: " + scheme);
            }
            // send a response...
            ReplyHeader rh = new ReplyHeader(h.getXid(), 0,
                    KeeperException.Code.AUTHFAILED.intValue());
            cnxn.sendResponse(rh, null, null);
            // ... and close connection
            cnxn.sendBuffer(ServerCnxnFactory.closeConn);
            cnxn.disableRecv();
        } else {
            if (LOG.isDebugEnabled()) {
                LOG.debug("Authentication succeeded for scheme: "
                          + scheme);
            }
            LOG.info("auth success " + cnxn.getRemoteSocketAddress());
            ReplyHeader rh = new ReplyHeader(h.getXid(), 0,
                    KeeperException.Code.OK.intValue());
            cnxn.sendResponse(rh, null, null);
        }
        return;
    } else {
        if (h.getType() == OpCode.sasl) {
            Record rsp = processSasl(incomingBuffer,cnxn);
            ReplyHeader rh = new ReplyHeader(h.getXid(), 0, KeeperException.Code.OK.intValue());
            cnxn.sendResponse(rh,rsp, "response"); // not sure about 3rd arg..what is it?
            return;
        }
        else {
            // 最终进入这个代码块进行处理
            // 封装请求对象
            Request si = new Request(cnxn, cnxn.getSessionId(), h.getXid(),
              h.getType(), incomingBuffer, cnxn.getAuthInfo());
            si.setOwner(ServerCnxn.me);
            // 负责在服务端提交当前请求
            submitRequest(si);
        }
    }
    cnxn.incrOutstandingRequests(h);
}
```

### ZooKeeperServer#submitRequest

负责在服务端提交当前请求

```java
public void submitRequest(Request si) {
    //processor处理器，request过来以后会经历一系列处理器的处理过程
    if (firstProcessor == null) {
        synchronized (this) {
            try {
                // Since all requests are passed to the request
                // processor it should wait for setting up the request
                // processor chain. The state will be updated to RUNNING
                // after the setup.
                while (state == State.INITIAL) {
                    wait(1000);
                }
            } catch (InterruptedException e) {
                LOG.warn("Unexpected interruption", e);
            }
            if (firstProcessor == null || state != State.RUNNING) {
                throw new RuntimeException("Not started");
            }
        }
    }
    try {
        touch(si.cnxn);
        boolean validpacket = Request.isValid(si.type);
        if (validpacket) {
            // 调用firstProcessor发起请求，而这个firstProcess是一个接口，有多个实现类，具体的调用链是怎么样的?
            firstProcessor.processRequest(si);
            if (si.cnxn != null) {
                incInProcess();
            }
        } else {
            LOG.warn("Received packet at server of unknown type " + si.type);
            new UnimplementedRequestProcessor().processRequest(si);
        }
    } catch (MissingSessionException e) {
        if (LOG.isDebugEnabled()) {
            LOG.debug("Dropping request: " + e.getMessage());
        }
    } catch (RequestProcessorException e) {
        LOG.error("Unable to process request:" + e.getMessage(), e);
    }
}
```

### **firstProcessor**的请求链组成

firstProcessor的初始化是在ZookeeperServer的setupRequestProcessor中完成的，代码如下:

```java
protected void setupRequestProcessors() {
    RequestProcessor finalProcessor = new FinalRequestProcessor(this);
    RequestProcessor syncProcessor = new SyncRequestProcessor(this,
            finalProcessor);
    ((SyncRequestProcessor)syncProcessor).start();
    firstProcessor = new PrepRequestProcessor(this, syncProcessor);
    ((PrepRequestProcessor)firstProcessor).start();
}
```

从上面我们可以看到firstProcessor的实例是一个PrepRequestProcessor，而这个构造方法中又传递了一个 Processor构成了一个调用链。 

RequestProcessor syncProcessor = new SyncRequestProcessor(this, finalProcessor); 

而 syncProcessor的构造方法传递的又是一个 Processor，对应的是FinalRequestProcessor。

**所以整个调用链是 PrepRequestProcessor -> SyncRequestProcessor - >FinalRequestProcessor** 



### PrepRequestProcessor#processRequest

通过上面了解到调用链关系以后，我们继续再看firstProcessor.processRequest(si)。会调用到 PrepRequestProcessor 

```java
public void processRequest(Request request) {
    // request.addRQRec(">prep="+zks.outstandingChanges.size());
    submittedRequests.add(request);
}
```

processRequest只是把request添加到submittedRequests中，根据前面的经验，很自然的想到这里又是一个异步操作。而 subittedRequests 又是一个阻塞队列 

```java
LinkedBlockingQueue<Request> submittedRequests = new LinkedBlockingQueue<Request>();
```

而PrepRequestProcessor这个类又继承了线程类，因此我们直接找到当前类中的run方法如下:

```java
@Override
public void run() {
    try {
        while (true) {
            Request request = submittedRequests.take();
            long traceMask = ZooTrace.CLIENT_REQUEST_TRACE_MASK;
            if (request.type == OpCode.ping) {
                traceMask = ZooTrace.CLIENT_PING_TRACE_MASK;
            }
            if (LOG.isTraceEnabled()) {
                ZooTrace.logRequest(LOG, traceMask, 'P', request, "");
            }
            if (Request.requestOfDeath == request) {
                break;
            }
            // 处理代码
            pRequest(request);
        }
    } catch (RequestProcessorException e) {
        if (e.getCause() instanceof XidRolloverException) {
            LOG.info(e.getCause().getMessage());
        }
        handleException(this.getName(), e);
    } catch (Exception e) {
        handleException(this.getName(), e);
    }
    LOG.info("PrepRequestProcessor exited loop!");
}
```

### PrepRequestProcessor#pRequest

找到这段代码继续往下看：

```java
nextProcessor.processRequest(request);
```

### SyncRequestProcessor#processRequest

这里也是一个异步处理：

```java
public void processRequest(Request request) {
    // request.addRQRec(">sync");
    queuedRequests.add(request);
}
```

### SyncRequestProcessor#run

其他代码省略。继续往下走调用：```org.apache.zookeeper.server.FinalRequestProcessor#processRequest```

```java
nextProcessor.processRequest(i);
```

### FinalRequestProcessor#processRequest

FinalRequestProcessor.processRequest 方法并根据 Request 对象中的操作更新内 存中Session信息或者 znode数据。 

这个方法代码太多这里只看关键代码：

```java
case OpCode.exists: {
    lastOp = "EXIS";
    // TODO we need to figure out the security requirement for this!
    ExistsRequest existsRequest = new ExistsRequest();
    // 反序列化 (将ByteBuffer反序列化成为ExitsRequest.这个就是我们在客户端发起请求的时候传递过来的 Request对象
    ByteBufferInputStream.byteBuffer2Record(request.request,
            existsRequest);
    String path = existsRequest.getPath();
    if (path.indexOf('\0') != -1) {
        throw new KeeperException.BadArgumentsException();
    }
    // 终于找到一个很关键的代码，判断请求的getWatch是否存在，如果存在，则传递cnxn(servercnxn)
	// 对于exists请求，需要监听data变化事件，添加watcher
    Stat stat = zks.getZKDatabase().statNode(path, existsRequest
            .getWatch() ? cnxn : null);
    // 在服务端内存数据库中根据路径得到结果进行组装，设置为ExistsResponse
    rsp = new ExistsResponse(stat);
    break;
}
```

## 客户端接收服务端处理完成响应