---
type: Java
sub-type: Framework
topic: Dubbo
---
## 1	deomService.sayHello : JDK 代理对象调用

## 2	InvokerInvocationHandler : JDK 代理调用 Dubbo 入口

## 3	MigrationInvoker : 迁移调用

## 4	MockClusterInvoker : 模拟集群调用

## 5	Filter : 过滤器链调用

## 6	FailoverCluster : 故障转移策略

### 6.1	源代码

```java
///////////////////////////////////////////////////                  
// org.apache.dubbo.rpc.cluster.support.FailoverClusterInvoker#doInvoke
// 故障转移策略的核心逻辑实现类
///////////////////////////////////////////////////
@Override
@SuppressWarnings({"unchecked", "rawtypes"})
public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    List<Invoker<T>> copyInvokers = invokers;
    checkInvokers(copyInvokers, invocation);
    // 获取此次调用的方法名
    String methodName = RpcUtils.getMethodName(invocation);
    // 通过方法名计算获取重试次数
    int len = calculateInvokeTimes(methodName);
    // retry loop.
    // 循环计算得到的 len 次数
    RpcException le = null; // last exception.
    List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyInvokers.size()); // invoked invokers.
    Set<String> providers = new HashSet<String>(len);
    for (int i = 0; i < len; i++) {
        //Reselect before retry to avoid a change of candidate `invokers`.
        //NOTE: if `invokers` changed, then `invoked` also lose accuracy.
        // 从第2次循环开始，会有一段特殊的逻辑处理
        if (i > 0) {
            // 检测 invoker 是否被销毁了
            checkWhetherDestroyed();
            // 重新拿到调用接口的所有提供者列表集合，
            // 粗俗理解，就是提供该接口服务的每个提供方节点就是一个 invoker 对象
            copyInvokers = list(invocation);
            // check again
            // 再次检查所有拿到的 invokes 的一些可用状态
            checkInvokers(copyInvokers, invocation);
        }
        // 选择其中一个，即采用了负载均衡策略从众多 invokers 集合中挑选出一个合适可用的
        Invoker<T> invoker = select(loadbalance, invocation, copyInvokers, invoked);
        invoked.add(invoker);
        // 设置 RpcContext 上下文
        RpcContext.getServiceContext().setInvokers((List) invoked);
        boolean success = false;
        try {
            // 得到最终的 invoker 后也就明确了需要调用哪个提供方节点了
            // 反正继续走后续调用流程就是了
            Result result = invokeWithContext(invoker, invocation);
            // 如果没有抛出异常的话，则认为正常拿到的返回数据
            // 那么设置调用成功标识，然后直接返回 result 结果
            success = true;
            return result;
        } catch (RpcException e) {
            // 如果是 Dubbo 框架层面认为的业务异常，那么就直接抛出异常
            if (e.isBiz()) { // biz exception.
                throw e;
            }
            // 其他异常的话，则不继续抛出异常，那么就意味着还可以有机会再次循环调用
            le = e;
        } catch (Throwable e) {
            le = new RpcException(e.getMessage(), e);
        } finally {
            // 如果没有正常返回拿到结果的话，那么把调用异常的提供方地址信息记录起来
            if (!success) {
                providers.add(invoker.getUrl().getAddress());
            }
        }
    }
	
    // 如果 len 次循环仍然还没有正常拿到调用结果的话，
    // 那么也不再继续尝试调用了，直接索性把一些需要开发人员关注的一些信息写到异常描述信息中，通过异常方式拋出去
    throw new RpcException(le.getCode(), "Failed to invoke the method "
            + methodName + " in the service " + getInterface().getName()
            + ". Tried " + len + " times of the providers " + providers
            + " (" + providers.size() + "/" + copyInvokers.size()
            + ") from the registry " + directory.getUrl().getAddress()
            + " on the consumer " + NetUtils.getLocalHost() + " using the dubbo version "
            + Version.getVersion() + ". Last error is: "
            + le.getMessage(), le.getCause() != null ? le.getCause() : le);
}
```

### 6.2	流程分析

从流程上来看, 是一个大的 for 循环, 循环体中进行了 select 操作, 拿到 invoker 并发起后续的逻辑调用

从入参和返回值来看, 入参是 invocation, invokers, loadbalance 三个参数, 猜测是通过负载对衡对象, 从 invokers 中选取一个合适的 invoker 然后执行调用, 返回 Result 结果

具体细节是通过计算 retries 属性值, 得到重试次数并循环, 每次循环都使用负载均衡器选择一个继续调用, 如果出现业务异常, 就继续循环, 直到所有次数都消耗完成, 就抛出 RpcException 异常

### 6.3	其他

重试次数的计算方式为: 

- 获取 URL 中的 retires 参数 (没有则取默认值 2);

- 加上 RpcContext 中的 `CommonConstants.RETRIES` 配置;

- 最后加上 1 作为初始调用的次数.

## 7	DubboInvoker

### 7.1	源码分析

```java
///////////////////////////////////////////////////                  
// org.apache.dubbo.rpc.protocol.dubbo.DubboInvoker#doInvoke
// 按照dubbo协议发起调用实现类
///////////////////////////////////////////////////
@Override
protected Result doInvoke(final Invocation invocation) throws Throwable {
    // 获取此次调用的方法名
    RpcInvocation inv = (RpcInvocation) invocation;
    final String methodName = RpcUtils.getMethodName(invocation);
    inv.setAttachment(PATH_KEY, getUrl().getPath());
    inv.setAttachment(VERSION_KEY, version);
    // 获取发送数据的客户端
    ExchangeClient currentClient;
    if (clients.length == 1) {
        currentClient = clients[0];
    } else {
        currentClient = clients[index.getAndIncrement() % clients.length];
    }
    try {
        // 看看是单程发送不需要等待响应，还是发送完了后需要等待响应
        boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
        // 获取超时时间，这个之前在“配置加载顺序”接触过这个方法
        int timeout = calculateTimeout(invocation, methodName);
        invocation.setAttachment(TIMEOUT_KEY, timeout);
        // 单程发送不需要等待响应
        if (isOneway) {
            boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
            currentClient.send(inv, isSent);
            return AsyncRpcResult.newDefaultAsyncResult(invocation);
        } else {
            // 发送完了之后需要等待响应
            ExecutorService executor = getCallbackExecutor(getUrl(), inv);
            // 操作 currentClient 发送了一个 request 请求，
            // 然后接收了一个 CompletableFuture 对象，说明这里存在异步操作
            CompletableFuture<AppResponse> appResponseFuture =
                    currentClient.request(inv, timeout, executor).thenApply(obj -> (AppResponse) obj);
            // save for 2.6.x compatibility, for example, TraceFilter in Zipkin uses com.alibaba.xxx.FutureAdapter
            FutureContext.getContext().setCompatibleFuture(appResponseFuture);
            AsyncRpcResult result = new AsyncRpcResult(appResponseFuture, inv);
            result.setExecutor(executor);
            return result;
        }
    } catch (TimeoutException e) {
        throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
    } catch (RemotingException e) {
        throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}
```

### 7.2	流程分析

从代码流程上看, 拿到了一个交换数据的 Client 类, 然后根据是否需要响应 (isOneway) 来走不同的分支

从入参和返回值来看, 是根据请求数据发起调用, 拿到 Result 结果;

再看一些具体的实现细节, 先是计算了 timeout  的值, 如果是 Oneway, 则直接调用 send 方法, 需要有响应的方法还定义了一个线程池, 调用的是 request 方法来发送

但是两个方法返回的都是一个异步结果对象, 屏蔽了不同调用之间的差异.

## 8	ReferenceCountExchangeClient

分析需要响应的情况

### 8.1	代码分析

```java
///////////////////////////////////////////////////                  
// org.apache.dubbo.rpc.protocol.dubbo.ReferenceCountExchangeClient#request(java.lang.Object, int, java.util.concurrent.ExecutorService)
// 这里将构建好的 request 对象发送出去，然后拿到了一个 CompletableFuture 异步化的对象
///////////////////////////////////////////////////
@Override
public CompletableFuture<Object> request(Object request, int timeout, ExecutorService executor) throws RemotingException {
    // client为：HeaderExchangeClient
    return client.request(request, timeout, executor);
}
```

### 8.2	流程分析

从代码流程上看, 方法将入参全部交给类成员变量 client 处理.

从方法的入参和返回值来看, 入参是请求对象, 超时时间, 线程池对象, 返回值是 CompletableFuture 对象, 这个方法完成了同步转异步的操作.

再看一些具体的实现细节, 从断点可以看到 client 的类型是 HeaderExchangeClient, 意味着这个对象完成了同步转异步的操作.

## 9	HeaderExchangeClient

```java
// org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeChannel#request
@Override
public CompletableFuture<Object> request(Object request, int timeout, ExecutorService executor)
		throws RemotingException {
	if (closed) {
		throw new RemotingException(
				this.getLocalAddress(),
				null,
				"Failed to send request " + request + ", cause: The channel " + this + " is closed!");
	}
	Request req;
	if (request instanceof Request) {
		req = (Request) request;
	} else {
		// create request.
		req = new Request();
		req.setVersion(Version.getProtocolVersion());
		req.setTwoWay(true);
		req.setData(request);
	}
	DefaultFuture future = DefaultFuture.newFuture(channel, req, timeout, executor);
	try {
		channel.send(req);
	} catch (RemotingException e) {
		future.cancel();
		throw e;
	}
	return future;
}
```

从代码流程上看, 是获取或者构建了 Request 对象, 然后调用类成员变量 channel 的 send 方法, 将 Request 发送出去.

从方法的入参和返回值来看, 入参是请求对象, 超时时间, 线程池对象, 返回值是一个 CompletableFuture. 可以看到, 在这个方法中, 完成了实际的同步到异步的转变.

再看一些具体的实现细节, 返回的 CompletableFuture 对象的具体类型是 DefaultFuture, 后续远程调用结束后, 会将调用结果存储到这个 Future 对象中, 等待业务线程使用.

## 10	NettyClient

```java
// org.apache.dubbo.remoting.transport.AbstractClient#send
@Override  
public void send(Object message, boolean sent) throws RemotingException {  
    if (needReconnect && !isConnected()) {  
        connect();  
    }  
    Channel channel = getChannel();  
    // TODO Can the value returned by getChannel() be null? need improvement.  
    if (channel == null || !channel.isConnected()) {  
        throw new RemotingException(this, "message can not send, because channel is closed . url:" + getUrl());  
    }  
    channel.send(message, sent);  
}
```

从代码流程上看, 是将 Message 发送出去.

从方法的入参和返回值来看, 方法没有返回值, 入参是 Message 对象和是否已经发送, 方法没有返回值,

调用 AbstractClient 的 getChannel 方法, 获取到 Channel 对象, 然后调用 send 方法发送, send 方法是 Endpoint 接口定义的方法.


## 11	NettyCodecAdapter

```java
// org.apache.dubbo.remoting.transport.netty4.NettyCodecAdapter.InternalEncoder#encode
protected void encode(ChannelHandlerContext ctx, Object msg, ByteBuf out) throws Exception {  
	boolean encoded = false;  
	if (msg instanceof ByteBuf) {  
		out.writeBytes(((ByteBuf) msg));  
		encoded = true;  
	} else if (msg instanceof MultiMessage) {  
		for (Object singleMessage : ((MultiMessage) msg)) {  
			if (singleMessage instanceof ByteBuf) {  
				ByteBuf buf = (ByteBuf) singleMessage;  
				out.writeBytes(buf);  
				encoded = true;  
				buf.release();  
			}  
		}  
	}  
	
	if (!encoded) {  
		ChannelBuffer buffer = new NettyBackedChannelBuffer(out);  
		Channel ch = ctx.channel();  
		NettyChannel channel = NettyChannel.getOrAddChannel(ch, url, handler);  
		codec.encode(channel, buffer, msg);  
	}  
} 
```

调用 codec 编码器, 对入参进行编码处理

方法的入参 msg 的请求对象, out 是 ByteBuf 缓冲区, 是将 msg 编码之后的数据流

实现细节比较简单, 就是将传入的对象进行编码, 然后放入到 ByteBuf 中.