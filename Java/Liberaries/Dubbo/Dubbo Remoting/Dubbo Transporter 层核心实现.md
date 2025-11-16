---
type: Java
sub-type: Framework
topic: Dubbo
---
> 官方介绍: 抽象 mina 和 netty 为统一接口, 以 Message 为中心, 扩展接口为 Channel, Transporter, Client, Server, Codec.

![[Dubbo Transport.png]]

## 1	AbstractPeer

AbstractPeer 抽象类同时实现了 [[Dubbo Remoting 概览#2.1 Endpoint|Endpoint]] 和 [[Dubbo Remoting 概览#2.3 ChannelHandler|ChannelHandler]] 接口, 代表在网络通信中一个对等的, 可交互的端点. AbstractPeer 是 AbstractChannel 和 AbstractEndpoint 的父类.

```java
public abstract class AbstractPeer implements Endpoint, ChannelHandler {

	private final ChannelHandler handler;

	private volatile URL url;

	// closing closed means the process is being closed and close is finished
	private volatile boolean closing;

	private volatile boolean closed;

}
```

AbstractPeer 中有四个字段, 一个是表示该端点自身 URL 状态的字段, 还有两个 Boolean 类型的字段, 用来记录当前端点的状态, 这三个字段都有 Endpoint 接口相关;

还有有个字段指向 ChannelHandler 对象, AbstractPeer 对于 ChannelHandler 的所有实现都委托给了这个 ChannelHandler 对象.

## 2	AbstractEndpoint

AbstractEndpoint 继承了 AbstractPeer 抽象类. 在 AbstractEndpoint 中维护了一个 Codec2 对象 和 超时时间, 构造函数中根据传入的 URL 初始化这三个字段.

```java
public AbstractEndpoint(URL url, ChannelHandler handler) {
	super(url, handler);
	this.codec = getChannelCodec(url);
	this.connectTimeout =
		url.getPositiveParameter(Constants.CONNECT_TIMEOUT_KEY, Constants.DEFAULT_CONNECT_TIMEOUT);
}
```

另外, AbstractEndpoint 还实现了 Resetable 接口, 能够根据传入的 URL 重置其内部:

```java
public void reset(URL url) {
	// 检测当前AbstractEndpoint是否已经关闭(略)
	// 省略重置timeout、connectTimeout两个字段的逻辑

	try {
		if (url.hasParameter(Constants.CODEC_KEY)) {
			this.codec = getChannelCodec(url);
		}
	} catch (Throwable t) {
		logger.error(t.getMessage(), t);
	}
}
```

## 3	Server

![[NettyServer.png|600]]

### 3.1	AbstractServer 

AbstractServer 是对服务端的抽象, 实现了服务端的公共逻辑, AbstractServer 的核心字段有:

- localAddress, bindAddress (InetSocketAddress 类型): 分别对应该 Server 的本地地址和绑定的地址, 都是从 URL 中的参数中获取, bindAddress 默认值与 localAddress 一致;
	
- accepts (int 类型): 该 Server 能接收的最大连接数, 从 URL 的 accepts 参数中获取, 默认值为 0, 表示没有限制;
	
- executorRepository (ExecutorRepository 类型): 负责管理线程池, 后面我们会深入介绍 ExecutorRepository 的具体实现;
	
- executor (ExecutorService 类型): 当前 Server 关联的线程池, 由上面的 ExecutorRepository 创建并管理.

在 AbstractServer 的构造方法中, 会根据传入的 URL 初始化上述字段, 并调用 doOpen 方法完成 Server 的启动:

```java
public AbstractServer(URL url, ChannelHandler handler) throws RemotingException {
	super(url, handler);
	executorRepository = ExecutorRepository.getInstance(url.getOrDefaultApplicationModel());
	localAddress = getUrl().toInetSocketAddress();

	String bindIp = getUrl().getParameter(Constants.BIND_IP_KEY, getUrl().getHost());
	int bindPort = getUrl().getParameter(Constants.BIND_PORT_KEY, getUrl().getPort());
	if (url.getParameter(ANYHOST_KEY, false) || NetUtils.isInvalidLocalHost(bindIp)) {
		bindIp = ANYHOST_VALUE;
	}
	bindAddress = new InetSocketAddress(bindIp, bindPort);
	this.accepts = url.getParameter(ACCEPTS_KEY, DEFAULT_ACCEPTS);
	try {
		doOpen();
		if (logger.isInfoEnabled()) {
			logger.info("[SERVICE_PUBLISH][METADATA_REGISTER] Start "
					+ getClass().getSimpleName() + " bind " + getBindAddress() + ", export " + getLocalAddress());
		}
	} catch (Throwable t) {
		throw new RemotingException(
				url.toInetSocketAddress(),
				null,
				"Failed to bind " + getClass().getSimpleName() + " on " + bindAddress + ", cause: "
						+ t.getMessage(),
				t);
	}
	executors.add(
			executorRepository.createExecutorIfAbsent(ExecutorUtil.setThreadName(url, SERVER_THREAD_POOL_NAME)));
}
```

### 3.2	ExecutorRepository

AbstractServer 中的 executor 是 ExecutorRepository 实例, 负责创建并管理 Dubbo 中的线程池; 在其默认实现类 DefaultExecutorRepository 中维护了一个 ConcurrentMap 集合, 用于缓存已有的线程池:

```java
private final ConcurrentMap<String, ConcurrentMap<String, ExecutorService>> data = new ConcurrentHashMap<>();
```

第一层的 key 表示线程池属于 Provider 还是 Consumer 端, 第二层 Key 表示线程池关联服务的端口.

DefaultExecutorRepository.createExecutorIfAbsent 方法会根据 URL 参数创建相应的线程池并存放在合适的位置:

```java
@Override  
public synchronized ExecutorService createExecutorIfAbsent(URL url) {  
    // 第一层, Provider or Consumer  
    String executorKey = getExecutorKey(url);  
    ConcurrentMap<String, ExecutorService> executors =  
            ConcurrentHashMapUtils.computeIfAbsent(data, executorKey, k -> new ConcurrentHashMap<>());  
  
    // 第二层, Consumer 共享, Provider by port  
    String executorCacheKey = getExecutorSecondKey(url);  
  
    url = setThreadNameIfAbsent(url, executorCacheKey);  
  
    URL finalUrl = url;  
    ExecutorService executor =  
            ConcurrentHashMapUtils.computeIfAbsent(executors, executorCacheKey, k -> createExecutor(finalUrl));  
    // If executor has been shut down, create a new one  
    if (executor.isShutdown() || executor.isTerminated()) {  
        executors.remove(executorCacheKey);  
        executor = createExecutor(url);  
        executors.put(executorCacheKey, executor);  
    }  
    dataStore.put(executorKey, executorCacheKey, executor);  
    return executor;  
}
```

### 3.3	ThreadPool


### 3.4	NettyServer

NettyServer 继承自 AbstractServer, 实现了 doOpen 和 doClose 方法.

```java
@Override  
protected void doOpen() throws Throwable {  
    bootstrap = new ServerBootstrap();  
  
    // initialize serverShutdownTimeoutMills before potential usage to avoid NPE.  
    // read config before destroy    
    serverShutdownTimeoutMills = ConfigurationUtils.getServerShutdownTimeout(getUrl().getOrDefaultModuleModel());  
  
    bossGroup = createBossGroup();  
    workerGroup = createWorkerGroup();  
  
    final NettyServerHandler nettyServerHandler = createNettyServerHandler();  
    channels = nettyServerHandler.getChannels();  
  
    initServerBootstrap(nettyServerHandler);  
  
    // bind  
    try {  
        ChannelFuture channelFuture = bootstrap.bind(getBindAddress());  
        channelFuture.syncUninterruptibly();  
        channel = channelFuture.channel();  
    } catch (Throwable t) {  
        closeBootstrap();  
        throw t;  
    }  
	
	// Metrics ......
  
}
```

Dubbo 在 NettyServer 的 doOpen 方法中初始化 ServerBootstrap, 创建 Boss 和 Worker EventLoopGroup, 创建 ChannelInitializer 等一系列标准的 Netty 流程.

```java
@Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        int closeTimeout = UrlUtils.getCloseTimeout(getUrl());
                        NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
                        ch.pipeline().addLast("negotiation", new SslServerTlsHandler(getUrl()));
                        ch.pipeline()
                                .addLast("decoder", adapter.getDecoder())
                                .addLast("encoder", adapter.getEncoder())
                                .addLast("server-idle-handler", new IdleStateHandler(0, 0, closeTimeout, MILLISECONDS))
                                .addLast("handler", nettyServerHandler);
                    }
```