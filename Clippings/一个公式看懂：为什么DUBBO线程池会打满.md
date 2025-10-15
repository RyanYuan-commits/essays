---
title: 一个公式看懂：为什么DUBBO线程池会打满
source: https://www.cnblogs.com/ceshi2016/p/17286921.html
created: 2025-09-02
description: 本文通过“并发量 = QPS x RT”公式，深入剖析了DUBBO线程池打满的原因，主要归结为慢服务、预热不足导致的RT上升和流量激增导致的QPS上升，并提供了相应的排查和解决策略。
finished: "false"
tag: clipper
cover: https://cn.dubbo.apache.org/imgs/dubbo_colorful.png
updated: 2025-09-27 22:34:06
---
## 1	Dubbo Thread Modules

### 1.1	Classification of Thread Modules

Dubbo provides multiple thread modules, and selecting a thread module requires specifying th `dispather` attribute in the  configuration file:

```xml
<dubbo:protocol name= "dubbo" dispatcher= "all" />
<dubbo:protocol name= "dubbo" dispatcher= "direct" />
<dubbo:protocol name= "dubbo" dispatcher= "message" />
<dubbo:protocol name= "dubbo" dispatcher= "execution" />
<dubbo:protocol name= "dubbo" dispatcher= "connection" />
```

Dubbo offical documentation explains how diffenent thread modules choose between IO threads and bussiness threads:

**all**: All messages are dispatched to the business thread pool, including requests, responses, connection events, disconnection events, and heartbeats.

**direct**: All messages are _not_ dispatched to the business thread pool; they are all executed directly in the IO thread.

**message**: Only request and response messages are dispatched to the business thread pool; other messages like connection/disconnection events and heartbeats are executed directly in the IO thread.

**execution**: Only request messages are dispatched to the business thread pool; responses and other messages like connection/disconnection events and heartbeats are executed directly in the IO thread.

**connection**: Connection disconnection events are placed in a queue on the IO thread and executed sequentially one by one; other messages are dispatched to the business thread pool.

### 1.2	Determination Timing

The provider and consumer determine thread module during initialization.

```java
// Provider
public class NettyServer extends AbstractServer implements Server {
    public NettyServer(URL url, ChannelHandler handler) throws RemotingException {
        super(url, ChannelHandlerwrap(handler, ExecutorUtisetThreadName(url, SERVER_THREAD_POOL_NAME)));
    }
}

// Consumer
public class NettyClient extends AbstractClient {
    public NettyClient(final URL url, final ChannelHandler handler) throws RemotingException {
        super(url, wrapChannelHandler(url, handler));
    }
}
```

---

The default thread for both consumer and proider is **AllDispatcher**. The `ChannelHandler.wrap()` method can obtain the Dispather adaptive extension point. If we specify `dispather` in the configuration file, the extenxion loader will load corresponding thread module using the attribute value from URL:

```java
@SPI(AllDispatcher.NAME)
public interface Dispatcher {
    @Adaptive({Constants.DISPATCHER_KEY, "channel.handler"})
    ChannelHandler dispatch(ChannelHandler handler, URL url);
}
```

### 1.3	Source Code Analysis

We analyze the source code for two thread modules; pleare refer to Dubbo source code for others.

#### 1.3.1	AllDispatcher Module

```java
// org.apache.dubbo.remoting.transport.dispatcher.all.AllDispatcher
public class AllDispatcher implements Dispatcher {  
  
    public static final String NAME = "all";  
  
    @Override  
    public ChannelHandler dispatch(ChannelHandler handler, URL url) {  
        return new AllChannelHandler(handler, url);  
    }  
  
}
```