---
type: Java
sub-type: Dubbo
finished: "false"
source: https://cn.dubbo.apache.org/zh-cn/docsv2.7/dev/source/service-invoking-process/
created: 2025-09-27 22:34:06
updated: 2025-09-27 22:34:06
---
```embed
title: "服务调用过程"
image: "https://cn.dubbo.apache.org/imgs/dev/send-request-process.jpg"
description: "本文介绍了服务调用过程的原理和实现细节"
url: "https://cn.dubbo.apache.org/zh-cn/docsv2.7/dev/source/service-invoking-process/"
favicon: ""
aspectRatio: "24.269005847953213"
```

Dubbo 的服务调用过程图

![[Dubbo 的服务调用过程图.png]]

## 1 服务调用方式

```java
@DubboReference(check = false)  
UserService userService;  
  
// main......
  
@Override  
public void run(String... args) throws InterruptedException {  
    while (true) {  
        userService.queryUserName();  
        Thread.sleep(10 * 1000);  
    }  
}
```

调用的是 Dubbo 生成的代理类, 反编译后, 具体调用的方法为:

```java
public String queryUserName() {
	Object[] objectArray = new Object[]{};
	Object object = this.handler.invoke(this, methods[0], objectArray);
	return (String)object;
}
```

将运行参数存储到数组中, 然后调用 `InvocationHandler` 的 `invoke` 方法.

---

`org.apache.dubbo.rpc.proxy.InvokerInvocationHandler#invoke`

```java
@Override  
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {   
    
    // 对非 RPC 方法的特殊处理......
     
    RpcInvocation rpcInvocation = new RpcInvocation(serviceModel, method, invoker.getInterface().getName(), protocolServiceKey, args);  
    String serviceKey = url.getServiceKey();  
    rpcInvocation.setTargetServiceUniqueName(serviceKey);  
  
    RpcServiceContext.setRpcContext(url);  
  
    if (serviceModel instanceof ConsumerModel) {  
        rpcInvocation.put(Constants.CONSUMER_MODEL, serviceModel);  
        rpcInvocation.put(Constants.METHOD_MODEL, ((ConsumerModel) serviceModel).getMethodModel(method));  
    }  
  
    return invoker.invoke(rpcInvocation).recreate();  
}
```

`InvokerInvocationHandler` 中的 `invoker` 成员变量的类型为 `MockClusterInvoker`, 内部封装了服务降级的逻辑.

---

`org.apache.dubbo.rpc.protocol.dubbo.DubboInvoker#doInvoke`

```java
@Override
protected Result doInvoke(final Invocation invocation) throws Throwable {
	// 1 参数填充
    RpcInvocation inv = (RpcInvocation) invocation;
    final String methodName = RpcUtils.getMethodName(invocation);
    inv.setAttachment(PATH_KEY, getUrl().getPath());
    inv.setAttachment(VERSION_KEY, version);

    // 2 轮询复用与 Server 端的连接
    ExchangeClient currentClient;
    if (clients.length == 1) {
        currentClient = clients[0];
    } else {
        currentClient = clients[index.getAndIncrement() % clients.length];
    }

    try {
    	// 3 是否是单向调用?  
        boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
        int timeout = calculateTimeout(invocation, methodName);
        invocation.put(TIMEOUT_KEY, timeout);
        if (isOneway) {
        	// 3.1 单向调用, 发送后直接返回
            boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
            currentClient.send(inv, isSent);
            return AsyncRpcResult.newDefaultAsyncResult(invocation);
        } else {
        	// 3.2 需要获取结果, 设置回调执行
            ExecutorService executor = getCallbackExecutor(getUrl(), inv);
            CompletableFuture<AppResponse> appResponseFuture =
                    currentClient.request(inv, timeout, executor).thenApply(obj -> (AppResponse) obj);
            FutureContext.getContext().setCompatibleFuture(appResponseFuture);
            AsyncRpcResult result = new AsyncRpcResult(appResponseFuture, inv);
            result.setExecutor(executor);
            return result;
        }
    } catch (TimeoutException e) {
        // log1 ......
    } catch (RemotingException e) {
        // log2 ......
    }
}
```

第 3 部分, 根据是否是单向调用 (是否需要获取返回值) 来执行不同的方法, 而在需要返回值的场景下则会根据是否是同步调用执行不同的策略.

具体体现在 `getCallbackExecutor` 方法返回值上:

```java
protected ExecutorService getCallbackExecutor(URL url, Invocation inv) { 
	// 获取共享线程池 
    ExecutorService sharedExecutor = url.getOrDefaultApplicationModel()
	    .getExtensionLoader(ExecutorRepository.class)  
        .getDefaultExtension()  
        .getExecutor(url);

    if (InvokeMode.SYNC == RpcUtils.getInvokeMode(getUrl(), inv)) {  
        return new ThreadlessExecutor(sharedExecutor);  
    } else {  
        return sharedExecutor;  
    }  
}
```

