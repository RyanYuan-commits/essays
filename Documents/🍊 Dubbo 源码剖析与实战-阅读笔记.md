原文地址： https://time.geekbang.org/column/article/611301
Dubbo 中文手册： https://cn.dubbo.apache.org/zh-cn/overview/mannual/java-sdk/

##  1 温故知新：Dubbo基础知识你掌握得如何？

=> https://time.geekbang.org/column/article/611355

Consumer 因为 Provider 未启动导致的启动失败问题如何解决？

- 让 Provider 正常启动起来即可 => 耦合性太强，因为提供方没有发布服务而导致消费方无法启动，有点说不过去。
- 在 Consumer 方的 Dubbo XML 的配置文件中，为服务添加 `check="false"` 属性，来达到启动不检查服务的目的 => 需要针对指定的服务级别设置“启动不检查”，但一个消费方工程，会有几十上百甚至上千个提供方服务配置，一个个设置起来特别麻烦，而且一般我们也很少会逐个设置；
- 在 Consumer 的 Dubbo XML 配置文件中，为整个消费方添加 `check="false"` 属性。 => 是我们比较常用的一种设置方式，保证不论提供方的服务处于何种状态，都不能影响消费方的启动。

```xml
<!-- 引用远程服务 -->
<dubbo:reference id="demoFacade" check="false"
    interface="com.hmilyylimh.cloud.facade.demo.DemoFacade">
</dubbo:reference>

<!-- 为整个消费方添加启动不检查提供方服务是否正常 -->
<dubbo:consumer check="false"></dubbo:consumer>
```

Dubbo 的故障转移：当调用环节出现不可预估的故障时，尝试去调用其他的 Provider，尽可能保证服务的正常运行；在 Consumer 方配置 `cluster="failover"` 启动。

```xml
<!-- 引用远程服务 -->
<dubbo:reference id="demoFacade" cluster="failover" timeout="5000" retries="3"
	interface="com.hmilyylimh.cloud.facade.demo.DemoFacade">
</dubbo:reference>
```

除了故障转移外，还有其他的容错策略：

![[Dubbo-容错策略.png]]

## 2 异步化实践：莫名其妙出现线程池耗尽怎么办？

Dubbo 线程池总数默认是固定的，200 个，假设系统单位时间可以处理 500 个请求，同时系统存在一个耗时任务，而极端情况下，这 200 个线程都在处理这个耗时任务，而剩下的 300 个请求，即使是不耗时的任务，也无法拿到线程资源了；

为了让这种耗时的请求尽量不占用公共的线程池资源，我们就要开始琢磨异步了。

异步要解决的首要问题是，如何返回结果？使用 Thread 和 线程池都无法解决这个问题。

思路：在方法中开始异步化分支处理，紧接着在异步化分支中得到异步结果，然后把异步结果**存储到某个地方**，最后再看看谁可以取出这个异步结果并返回。

通过请求流程来梳理谁有能力感知到异步结果：

请求方网络请求 => 网络模块 Netty => 代理模块 Proxy => 拦截模块 Filter => 业务模块

[[🍜【Dubbo】异步化及原理解析]]

## 3 隐式传递：如何精准的找出一次请求的全部日志

问题：无法在海量日志中，串联出某一个执行流程的完整日志；

思路：使用唯一序列号在各个系统中传递。

**显式传递方案**：也就是在所有方法入参中添加唯一 ID，实践中可以定义一个抽象的 Request 类，让所有 Request 类继承；
- 不适合迭代了很久的系统，改 Request 要花很长时间；
- 接口是 Java 公共类而非业务自定义类无法使用这种方法。

**隐式传递方案**：类似 HTTP 的请求头，将需要传递的放在 body，将其他字段放在 Header 中，在 Dubbo 中，RpcContext 可以担任请求头的身份；使用 Dubbo 提供的 Filter 来实现 ID 的传递。

Consumer 过滤器：取出 ID 放入 RpcContext 中；

```java
@Override  
public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {  
    // 从上下文对象中取出跟踪序列号值  
    String existsTraceId = RpcContext.getServerAttachment()
	    .getAttachment(FilterConstant.TRACE_ID);  
  
    if (existsTraceId == null) {  
        existsTraceId = "no-trace-id";  
    }  
  
    // 然后将序列号值设置到请求对象中  
    invocation.getObjectAttachments()
	    .put(FilterConstant.TRACE_ID, existsTraceId);  
    RpcContext.getClientAttachment()
	    .setObjectAttachment(FilterConstant.TRACE_ID, existsTraceId);  
  
    // 继续后面过滤器的调用  
    return invoker.invoke(invocation);  
}
```

Provider 过滤器：取出 ID 放入 MDC 中使用；

```java
@Override  
public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {  
    // 获取入参的跟踪序列号值  
    Map<String, Object> attachments = invocation.getObjectAttachments();  
    String reqTraceId = attachments != null ? (String) attachments
	    .get(FilterConstant.TRACE_ID) : null;  
  
    // 若 reqTraceId 为空则重新生成一个序列号值，序列号在一段相对长的时间内唯一足够了  
    reqTraceId = reqTraceId == null ? generateTraceId() : reqTraceId;  
  
    // 将序列号值设置到上下文对象中  
    RpcContext.getServerAttachment()
	    .setObjectAttachment(FilterConstant.TRACE_ID, reqTraceId);  
  
    // 并且将序列号设置到日志打印器中，方便在日志中体现出来  
    MDC.put(FilterConstant.TRACE_ID, reqTraceId);  
    log.info("provider log");  
  
    // 继续后面过滤器的调用  
    return invoker.invoke(invocation);  
}
```

总结自定义 Filter 的步骤：

- 创建一个实现了 `org.apache.dubbo.rpc.Filter` 接口的类；
- 在类上通过 `@Activate` 注解标明生效范围；
- 实现 `Filter` 接口的 `invoke` 方法；
- 将类路径添加到 `META-INF/dubbo/org.apache.dubbo.rpc.Filter`，并取别名。

## 4 泛化调用：三步教你搭建通用的泛化调用框架

平常架构：前端请求 Web 服务器 => Web 服务器请求后端系统 => 组装返回

后端的一个简单模拟代码：

```java
@RestController
public class UserController {
    // 响应码为成功时的值
    public static final String SUCC = "000000";
    
    // 定义访问下游查询用户服务的字段
    @DubboReference
    private UserQueryFacade userQueryFacade;
    
    // 定义URL地址
    @PostMapping("/queryUserInfo")
    public String queryUserInfo(@RequestBody QueryUserInfoReq req){
        // 将入参的req转为下游方法的入参对象，并发起远程调用
        QueryUserInfoResp resp = userQueryFacade
			.queryUserInfo(convertReq(req));
        
        // 判断响应对象的响应码，不是成功的话，则组装失败响应
        if(!SUCC.equals(resp.getRespCode())){
            return RespUtils.fail(resp);
        }
        
        // 如果响应码为成功的话，则组装成功响应
        return RespUtils.ok(resp);
    }
}
```

得出结论：步骤是固定的、带有遗传性质的，可以简化成一个泛化方法。

### 4.1 反射调用方式

通过反射调用 DubboReference 对象：

```java
@RestController  
@RequestMapping("/facade")  
public class FacadeReflectionController {  
  
    @DubboReference(check = false)  
    private UserService userService;  
  
    @GetMapping("/queryUserName")  
    public String queryUserName() {  
        return commonInvoke(userService, "queryUserName", null);  
    }  
  
    public static String commonInvoke(Object reqObj, String mtdName, 
									  Object[] reqParam) {  
        Method method = ReflectionUtils
					        .findMethod(reqObj.getClass(), mtdName);  
        if (method == null) {  
            return "404 not found";  
        }  
        method.setAccessible(true);  
        try {  
            return (String) ReflectionUtils.invoke(reqObj, method, reqParam);  
        } catch (Exception e) {  
            throw new RuntimeException(e);  
        }  
    }  
  
}
```

### 4.2 泛化调用方式

思路：在链接中指定要调用的 Dubbo Service 和方法，使用占位符型的 URL。

```java
@RestController  
@RequestMapping("/facade/general")  
public class GeneralizedController {  
  
    @RequestMapping("/gateway/{className}/{mtdName}/{parType}/request")  
    public String queryUserName(@PathVariable String clsName, 
							    @PathVariable String mtdName,  
                                @PathVariable String parType, 
                                @RequestBody Object reqParam) {  
        return commandInvoke(clsName, mtdName, parType, reqParam);  
    }  
  
    private static String commandInvoke(String clsName, String mtdName, String parType, Object reqParam) {  
        ReferenceConfig<GenericService> referenceConfig = 
									        createReferenceConfig(clsName);  
  
        GenericService genericService = referenceConfig.get();  
        Object o = genericService.$invoke(  
                mtdName,  
                new String[]{parType},  
                new Object[]{JSON.parseObject(reqParam.toString(), 
							                  Map.class)}  
        );  
  
        return JSON.toJSONString(o);  
    }  
  
    private static ReferenceConfig<GenericService> createReferenceConfig(String clsName) {  
        DubboBootstrap dubboBootstrap = DubboBootstrap.getInstance();  
  
	        ApplicationConfig applicationConfig = new ApplicationConfig();  
	        
	        applicationConfig.setName(dubboBootstrap.getApplicationModel()
											        .getApplicationName());  
  
        // 设置注册中心的地址  
        String address = dubboBootstrap.getConfigManager()  
                .getRegistries()  
                .iterator()  
                .next()  
                .getAddress();  
        RegistryConfig registryConfig = new RegistryConfig(address);  
        ReferenceConfig<GenericService> referenceConfig = 
										        new ReferenceConfig<>();  
        referenceConfig.setApplication(applicationConfig);  
        referenceConfig.setRegistry(registryConfig);  
        referenceConfig.setInterface(clsName);  
  
        referenceConfig.setGeneric(true);  
        referenceConfig.setTimeout(5000);  
  
        return referenceConfig;  
    }  
      
}
```

### 4.3 RpcContext 整理

RpcContext 作为关键的上下文对象，有这么几个重要的属性：

| 属性                  | 作用范围    | 数据传输方向  | 典型场景             |
| ------------------- | ------- | ------- | ---------------- |
| `SERVER_LOCAL`      | 服务端本地   | 不传输     | 服务端临时存储请求元数据     |
| `CLIENT_ATTACHMENT` | 客户端→服务端 | 请求中传递   | 传递 TraceID、鉴权信息等 |
| `SERVER_ATTACHMENT` | 服务端→客户端 | 响应中传递   | 返回令牌、扩展元数据等      |
| `SERVICE_CONTEXT`   | 跨线程     | 手动保存/恢复 | 异步调用、线程池切换       |

#### SERVER_LOCAL

```java
private static final InternalThreadLocal<RpcContextAttachment> SERVER_LOCAL = new InternalThreadLocal<RpcContextAttachment>() {  
    @Override  
    protected RpcContextAttachment initialValue() {  
        return new RpcContextAttachment();  
    }  
};
```

## 5 点点直连：点对点搭建产线“后门”的万能管控

## 6 事件通知：一招打败各种神乎其神的回调事件

代码解耦思路：

1. **功能相关性**：将一些功能非常相近的汇聚到一起，既是对资源的聚焦整合，也降低了他这人的学习成本；
2. **密切相关性**：按照与主流程的密切相关性，将一个个小功能分为密切与非密切；
3. **状态变更性**：按照是否有明显业务状态的先后变更，将一个个小功能再归类。


6W 分析模型，分析事件通知：

1. Who 谁产生的事件？是功能本身业务实现体产生，还是功能归属源码框架来产生。
2. What 产生什么功能的事件？事件数据对象包含业务信息和事件凭证信息。
3. When 什么时候发生的事件？在业务发生之前、之后，还是业务发生异常时发布。
4. Where 在哪个系统、哪个功能、哪段代码位置发生的事件？
5. Why 为什么要有这个事件？解决某类问题，提供一种扩展机制来丰富事件价值。
6. How 怎么把事件发布给需要关注的消费者？是自研框架，还是扩展已有框架中具有拦截或过滤特性的机制。

背景：要在某个功能返回后去做一些处理；

思路：利用 AOP 代理，添加一些事件和处理逻辑；

Dubbo 提供了事件回调的机制，具体的使用方法为：

```java
@DubboService
@Component
public class PayFacadeImpl implements PayFacade {
    @Autowired
    @DubboReference(
            /** 为 DemoRemoteFacade 的 sayHello 方法设置事件通知机制 **/
            methods = {@Method(
                    name = "sayHello",
                    oninvoke = "eventNotifyService.onInvoke",
                    onreturn = "eventNotifyService.onReturn",
                    onthrow = "eventNotifyService.onThrow")}
    )
    private DemoRemoteFacade demoRemoteFacade;
    
    // 商品支付功能：一个大方法
    @Override
    public PayResp recvPay(PayReq req){
        // 支付核心业务逻辑处理
        method1();
        // 返回支付结果
        return buildSuccResp();
    }
    private void method1() {
        // 省略其他一些支付核心业务逻辑处理代码
        demoRemoteFacade.sayHello(buildSayHelloReq());
    }
}

// 专门为 demoRemoteFacade.sayHello 该Dubbo接口准备的事件通知处理类
@Component("eventNotifyService")
public class EventNotifyServiceImpl implements EventNotifyService {
    // 调用之前
    @Override
    public void onInvoke(String name) {
        System.out.println("[事件通知][调用之前] onInvoke 执行.");
    }
    // 调用之后
    @Override
    public void onReturn(String result, String name) {
        System.out.println("[事件通知][调用之后] onReturn 执行.");
        // 埋点已支付的商品信息
        method2();
        // 发送支付成功短信给用户
        method3();
        // 通知物流派件
        method4();
    }
    // 调用异常
    @Override
    public void onThrow(Throwable ex, String name) {
        System.out.println("[事件通知][调用异常] onThrow 执行.");
    }
}
```

## 7 参数验证：写个参数校验居然也会被训？

### 7.1 参数校验思路

场景：使用 RPC 调用下游服务之前，需要对参数进行校验，对不符合预期的参数进行拦截。

参数校验的验证思路：

1. 直接写在代码中。最简单的处理方式，但是需要改动位置太多；
2. 使用上一节讲的通知 Filter。该过滤器不支持对不符合预期的请求进行拦截；
3. 使用 Dubbo 的 ValidationFilter。效果符合预期。

ValidationFilter 的类注释信息：

```
ValidationFilter invoke the validation by finding the right Validator instance based on the configured validation attribute value of invoker url before the actual method invocation.
```

简单理解就是，ValidationFilter 会根据 url 配置的 validation 属性值找到正确的校验器，在方法真正执行之前触发调用校验器执行参数验证逻辑。

使用方式为：

```java
e.g. <dubbo:method name="save" validation="jvalidation" />
```

除此之外，还有一些特殊的设置方式：

```java
e.g. <dubbo:method name="save" validation="special" />
where "special" is representing a validator for special character.
special=xxx.yyy.zzz.SpecialValidation under META-INF folders org.apache.dubbo.validation.Validation file.
```

可以在 validation 属性的值上，填充一个自定义的校验类名，并且嘱咐我们记得将自定义的校验类名添加到 META-INF 文件夹下的 org.apache.dubbo.validation.Validation 文件中。

### 7.2 代码改造

梳理一下改造的步骤：

1. 为下游接口添加 validation 属性；
2. 从源码中寻找能提供校验规则的标准产物，也就是注解；
3. 再下游的方法入参对象中，为需要校验的字段添加注解。

根据源码得出，Dubbo 原生提供的过滤器依赖的是 javax.validation 包的能力，注解也使用 javax.validation 的校验注解。

## 8 缓存操作：如何为接口优雅地提供缓存功能？

## 9 流量控制：控制接口调用请求流量的三个秘诀

## 10 服务认证：被异构系统侵入调用了，怎么办？

## 11 配置加载顺序：为什么你设置的超时时间不生效？

### 11.1 Dubbo 属性加载流程

![[Dubbo 属性加载流程.png|500]]
- 第一阶段为 DubboBootstrap 初始化之前，在 Spring context 启动时解析处理XML配置/注解配置/Java-config 或者是执行API配置代码，创建 config bean 并且加入到ConfigManager 中。
- 第二阶段为 DubboBootstrap 初始化过程，从配置中心读取外部配置，依次处理实例级属性配置和应用级属性配置，最后刷新所有配置实例的属性，也就是属性覆盖。

### 11.2 属性覆盖

属性配置源有四个层级关系：

- System Properties，最高优先级，启动的时候，通过 JVM 的 -D 进行指定；
- Externalized Configuration，优先级次之，外部化配置，从配置中心加载的配置；
- API / XML / 注解，优先级再次降低；
- Local File，优先级最低，项目中的默认配置。

高配置源的配置项会覆盖低配置源的配置项，最终保留下来的配置项根据匹配格式匹配：

```properties
#1. 指定id的实例级配置
dubbo.{config-type}s.{config-id}.{config-item}={config-item-value}

#2. 指定name的实例级配置
dubbo.{config-type}s.{config-name}.{config-item}={config-item-value}

#3. 应用级配置（单数配置）
dubbo.{config-type}.{config-item}={config-item-value}
```

## 12 源码框架：框架在源码层面如何体现分层？

### 12.1 模块流程图

![[Dubbo-模块流程图.png]]

### 12.2 模块串联

#### Service 服务层

消费方使用接口来进行调用，提供方提供接口和实现，这些都和实际业务息息相关，和框架底层没有太大关系，Dubbo 将这一层称为服务层，即 Service。

#### Config 配置层

调用是相互的，调用方有一堆的配置要设置，提供方也一样，同样会需要针对方法、接口、实例设置一堆的配置。

这些配置，站在代码层面来说，就是平常接触的标签、注解、API，更实际点说，落地到类层面就是代码中各种 XxxConfig 类，比如 ServiceConfig、ReferenceConfig，都是我们比较熟悉的配置类。

Dubbo 把这样专门存储与读取配置打交道的层次称为配置层，即 Config。

#### Proxy 代理层

通过动态代理的方式，根据各种配置信息完成一次完整的远程方法调用。

Dubbo 把这种代理接口发起远程调用，或代理接收请求进行实例分发处理的层次，称为服务代理层，即 Proxy。

#### Registry 注册中心层

帮助我们注册服务、发现服务的层级。

Dubbo 把这种专门与注册中心打交道的层次，称为注册中心层，即 Registry。

#### Cluster 路由层

从注册中心提供的 Provider 列表中，筛选出最终调用的 Service 对象的层级。

Dubbo 将这种封装多个提供者并承担路由过滤和负载均衡的层次，称为路由层，即 Cluster。

#### Monitor 监控层

然而一次远程调用，总归是要有结果的，正常也好，异常也好，都是一种结果。比如某个方法调用成功了多少次，失败了多少次，调用前后所花费的时间是多少。这些看似和业务逻辑无关紧要，实际，对我们开发人员在分析问题或者预估未来趋势时有着无与伦比的价值。

于是诞生了一个监控模块来专门处理这种事情，Dubbo 将这种同步调用结果的层次称为监控层，即 Monitor。

#### Protocol 远程调用层

然而远程调用也是一个过程，出于增强框架灵活扩展业务的需求，我们有时候需要在调用之前做点什么，在调用之后做点什么，我们前面接触过很多次的过滤器就是过程中的一个环节。

如果把远程调用看作一个“实体对象”，拿着这个“实体对象”就能调出去拿到结果，就好像“实体对象”封装了 RPC 调用的细节，我们只需要感知“实体对象”的存在就好了。

那么封装调用细节，取回调用结果，Dubbo 将这种封装调用过程的层次称为远程调用层，即 Protocol。

#### Exchange 信息交换层

对于我们平常接触的 HTTP 请求来说，开发人员感知的是调用五花八门的 URL 地址，但发送 HTTP 报文的逻辑最终归到一个抽象的发送数据的口子，统一处理。

对 Dubbo 框架来说也是一样，消费方的五花八门的业务请求数据最终会封装为 Request、Response 对象，至于拿着 Request 对象是进行同步调用，还是直接转异步调用通过 Future.get 拿结果，那是底层要做的事情，因此 Dubbo 将这种封装请求并根据同步异步模式获取响应结果的层次，称为信息交换层，即 Exchange。

#### Transport 网络传输层

当 Request 请求对象准备好了，不管是同步发送，还是异步发送，最终都是需要发送出去的，但是对象通过谁来发到网络中的呢？

这就需要网络通信框架出场了。网络通信框架，封装了网络层面的各种细节，只暴露一些发送对象的简单接口，上层只需要放心把 Request 对象交给网络通信框架就可以了。

Dubbo 把这种能将数据通过网络发送至对端服务的层次称为网络传输层，即 Transport。

#### Serialize 数据序列化层

网络通信框架最终要把对象转成二进制才能往网卡中发送，那么谁来将这些实打实的 Request、Response 对象翻译成网络中能识别的二进制数据呢？

将对象转成二进制或将二进制转成对象的模块，Dubbo 将这种能把对象与二进制进行相互转换的正反序列化的层次称为数据序列化层，即 Serialize。

### 12.3 代码分包架构

![[Dubbo-代码分包架构.png|500]]

还有些未圈出来的模块，这里我们也举 3 个平常比较关注的 Module：

1. **dubbo-common** 是 Dubbo 的公共逻辑模块，包含了许多工具类和一些通用模型对象。
2. **dubbo-configcenter** 是专门来与外部各种配置中心进行交互的，在上一讲“配置加载”中，我们配置过 标签，标签内容填写的 address 是 nacos 配置中心的地址， 其实就是该模块屏蔽了与外部配置中心的各种交互的细节逻辑。
3. **dubbo-filter** 是一些与过滤器有着紧密关联的功能，目前有缓存过滤器、校验过滤器两个功能。

## 13 集成框架：框架如何与Spring有机结合？

### 13.1 原始方式

开发 Dubbo 框架的消费方系统，一般会将调用下游的提供方系统的功能放在 integration 层，举一个简单的案例：

```java
public interface SamplesFacade {
    QueryOrderRes queryOrder(QueryOrderReq req);
}
```

```java
public interface SamplesFacadeClient {
    QueryOrderResponse queryRemoteOrder(QueryOrderRequest req);
}

public class SamplesFacadeClientImpl implements SamplesFacadeClient {
    @DubboReference
    private SamplesFacade samplesFacade;
    @Override
    public QueryOrderResponse queryRemoteOrder(QueryOrderRequest req){
        // 构建下游系统需要的请求入参对象
        QueryOrderReq integrationReq = buildIntegrationReq(req);
        
        // 调用 Dubbo 接口访问下游提供方系统
        QueryOrderRes resp = samplesFacade.queryOrder(integrationReq);
        
        // 判断返回的错误码是否成功
        if(!"000000".equals(resp.getRespCode())){
            throw new RuntimeException("下游系统 XXX 错误信息");
        }
        
        // 将下游的对象转换为当前系统的对象
        return convert2Response(resp);
    }
}
```

问题：integration 层的代码重复度较高，基本结构都是请求对象组装、远程调用、返回数据的判断和转换。

### 13.2 改善 integration 代码

**背景**：integration 层有很多和 SamplesFacadeClientImpl 相似的类，而且每个方法的实现逻辑都和 queryRemoteOrder 方法差不多。

**根据**：“ 抽象、封装、继承、多态 “ 的面向对象思想。

==封装的思路==，可以把相似流程中变化的因素提取成变量；在当前场景下，有这几个因素：

- 调用的下游接口的信息，比如类名、方法名、方法入参等；
- 接口级别的 Dubbo 参数，比如 timeout、retries 等；
- 返回参数的处理，错误码的判断因素；
- 拿到下游接口对象后的转化因素。

==抽象的思路==，保留代码中相对不会变化的基本流程；

在这个场景下，基本的流程就是先构建调用下游系统的请求对象，并将请求对象传入下游系统的接口中，然后接收返参并针对错误码进行判断，最后转成结果对象。

---

基本的流程已经确认了，现在要思考的是怎么将变化的因素交给业务实现类去实现：

- 简单的将这些因素的构建交给类似 SamplesFacadeClientImpl 的实现类去做，又回到之前的老路了；
- 利用在接口上添加注解的思路，将变化因素配置在直接上。

```java
@DubboFeignClient(
        remoteClass = SamplesFacade.class,
        needResultJudge = true,
        resultJudge = (remoteCodeNode = "respCode", 
	        remoteCodeSuccValueList = "000000", remoteMsgNode = "respMsg")
)
public interface SamplesFacadeClient {
    @DubboMethod(
            timeout = "5000",
            retries = "3",
            loadbalance = "random",
            remoteMethodName = "queryRemoteOrder",
            remoteMethodParamsTypeName = {"com.hmily.QueryOrderReq"}
     )
    QueryOrderResponse queryRemoteOrderInfo(QueryOrderRequest req);
}
```

### 13.3 将类添加 Spring 管理

文章中提供了一个重写 Spring 的 ClassPathBeanDefinitionScanner 的方法；

其实可以给这些类加上 @Component 注解，然后通过