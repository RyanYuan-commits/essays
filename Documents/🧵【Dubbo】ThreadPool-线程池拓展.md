---
type: Java
sub-type: Dubbo
finished: "true"
created: 2025-09-27 22:34:06
updated: 2025-09-27 22:34:06
---
服务提供方线程池实现策略, 当服务器收到一个请求时, 需要在线程池中创建一个线程去执行服务提供方业务逻辑.

## 1 扩展接口

### 1.1 对应接口

对应接口为: `org.apache.dubbo.common.threadpool.ThreadPool`

```java
@SPI(value = "fixed", scope = ExtensionScope.FRAMEWORK)
public interface ThreadPool {

    /**
     * Thread pool
     *
     * @param url URL contains thread parameter
     * @return thread pool
     */
    @Adaptive({THREADPOOL_KEY})
    Executor getExecutor(URL url);

}
```

提供获取线程池的 `getExecutor` 方法, 可以通过 URL 参数 `threadpool` 来指定, 默认实现为 `org.apache.dubbo.common.threadpool.support.fixed.FixedThreadPool`.

### 1.2 默认参数

```java
public class FixedThreadPool implements ThreadPool {

    @Override
    public Executor getExecutor(URL url) {
        String name = url.getParameter(THREAD_NAME_KEY, (String) url.getAttribute(THREAD_NAME_KEY, DEFAULT_THREAD_NAME));
        int threads = url.getParameter(THREADS_KEY, DEFAULT_THREADS);
        int queues = url.getParameter(QUEUES_KEY, DEFAULT_QUEUES);
        return new ThreadPoolExecutor(threads, threads, 0, TimeUnit.MILLISECONDS,
                queues == 0 ? new SynchronousQueue<Runnable>() :
                        (queues < 0 ? new LinkedBlockingQueue<Runnable>()
                                : new LinkedBlockingQueue<Runnable>(queues)),
                new NamedInternalThreadFactory(name, true), new AbortPolicyWithReport(name, url));
    }
    
}
```

`corePoolSize` 核心线程数: 根据 URL Param `threads` 获取, 如果没有, 取默认值为 200;

`maximumPoolSize`: 最大线程数, 配置同上;

`keepAliveTime`: 0s, 但未配置核心线程可销毁, 创建的线程将会一直存活;

`unit`: 时间单位, `TimeUnit.MILLISECONDS` 同上, 无效的配置;

`workQueue`: 任务队列, 默认值为 `SynchronousQueue`, 根据 URL Param `threads` 配置改变, 若不为 0, 使用 `LinkedBlockingQueue`, 小于零最大任务数为 `Integer.MAX_VALUE`, 反之, 使用传入的配置;

`threadFactory`: 线程工厂, 使用 `NamedInternalThreadFactory`;

```java
@Override  
public Thread newThread(Runnable runnable) {  
    String name = mPrefix + mThreadNum.getAndIncrement();  
    InternalThread ret = new InternalThread(mGroup, InternalRunnable.Wrap(runnable), name, 0);  
    ret.setDaemon(mDaemon);  
    return ret;  
}
```

`handler`: 拒绝策略, 使用 `AbortPolicyWithReport`.

### 1.3 拓展示例

项目目录:

```
src
 |-main
    |-java
        |-com
            |-xxx
                |-XxxThreadPool.java
    |-resources
        |-META-INF
            |-dubbo
                |-org.apache.dubbo.common.threadpool.ThreadPool
```

`XxxThreadPool.java`

```java
package com.xxx;
 
import org.apache.dubbo.common.threadpool.ThreadPool;
import java.util.concurrent.Executor;
 
public class XxxThreadPool implements ThreadPool {
    public Executor getExecutor() {
        // ...
    }
}
```

`META-INF/dubbo/org.apache.dubbo.common.threadpool.ThreadPool`

```
xxx=com.xxx.XxxThreadPool
```

## 2 初始化流程

Dubbo 在服务导出时, 如果发现 Netty 线程池没有被创建, 则会调用 `ThreadPool` 的 `getExecutor` 方法创建一个线程池, 主要的代码链路大致为:

`org.apache.dubbo.config.ServiceConfig#doExportUrl`

```java
private void doExportUrl(URL url, boolean withMetaData) {  
    Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);  
    if (withMetaData) {  
        invoker = new DelegateProviderMetaDataInvoker(invoker, this);  
    }  
    // protocol, 默认为 dubbo
    Exporter<?> exporter = protocolSPI.export(invoker);  
    exporters.add(exporter);  
}
```

调用 `protocolSPI` 的 `export` 方法, `protocol` 默认为 `dubbo` 实现.

---

`org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol#openServer`

```java
private void openServer(URL url) {  
    checkDestroyed();  
    // find server.  
    String key = url.getAddress();  
    // client can export a service which only for server to invoke  
    boolean isServer = url.getParameter(IS_SERVER_KEY, true);  
    if (isServer) {  
        ProtocolServer server = serverMap.get(key);  
        if (server == null) {  
            synchronized (this) {  
                server = serverMap.get(key);  
                if (server == null) {  
	                // 创建 server
                    serverMap.put(key, createServer(url));  
                }else {  
                    server.reset(url);  
                }  
            }  
        } else {  
            // server supports reset, use together with override  
            server.reset(url);  
        }  
    }  
}
```

先尝试在 `serverMap` 中获取, 已创建则调用 reset 方法; 反之, 则创建后将其放置在 `serverMap` 中.

---

`org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol#createServer`

```java
private ProtocolServer createServer(URL url) {
    // ......
  
    ExchangeServer server;  
    try {  
        server = Exchangers.bind(url, requestHandler);  
    } catch (RemotingException e) {  
        throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);  
    }  
  
    // ......
}
```

`Protocol` 层调用其下层 `Exchange` 层来完成最终的注册和端口监听。

---

`org.apache.dubbo.remoting.exchange.support.header.HeaderExchanger#bind`

```java
@Override  
public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {  
    return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));  
}
```

Exchange 层又会调用下层 Transporter 的 `bind` 方法.

---

`org.apache.dubbo.remoting.transport.netty4.NettyTransporter#bind`

```java
@Override  
public RemotingServer bind(URL url, ChannelHandler handler) throws RemotingException {  
    return new NettyServer(url, handler);  
}
```

Transporter 层默认实现为 NettyTransporter.

---

`org.apache.dubbo.remoting.transport.AbstractServer#AbstractServer`

```java
public AbstractServer(URL url, ChannelHandler handler) throws RemotingException {  
    super(url, handler);  
    executorRepository = url.getOrDefaultApplicationModel().getExtensionLoader(ExecutorRepository.class).getDefaultExtension();  
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
        // ......
        }  
    } catch (Throwable t) {  
        // ......  
    }  
    executor = executorRepository.createExecutorIfAbsent(url);  
}
```

通过 Dubbo 的 SPI 机制, 获取 `ExecutorRepository` 的默认实现, 使用其创建方法 `createExecutorIfAbsent` 来创建线程池.

---
`org.apache.dubbo.common.threadpool.manager.DefaultExecutorRepository#createExecutor`

```java
private ExecutorService createExecutor(URL url) {  
    return (ExecutorService) extensionAccessor.getExtensionLoader(ThreadPool.class).getAdaptiveExtension().getExecutor(url);  
}
```

最终会调用到 `org.apache.dubbo.common.threadpool.ThreadPool` 的 `getExecutor` 方法, 默认实现就是上面的 `FixedThreadPool`.