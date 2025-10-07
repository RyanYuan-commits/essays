---
type: Java
sub-type: Dubbo
finished: "true"
created: 2025-09-27 22:34:06
updated: 2025-09-27 22:34:06
---
![[Dubbo 整体设计.png]]

## 1 各层说明

### Service 层

消费方使用接口来进行调用, 提供方提供接口和实现, 这些都和实际业务息息相关, 和框架底层没有太大关系, Dubbo 将这一层称为服务层, 即 Service. 

### Config 配置层

调用是相互的, 调用方有一堆的配置要设置, 提供方也一样, 同样会需要针对方法、接口、实例设置一堆的配置. 

这些配置, 站在代码层面来说, 就是平常接触的标签、注解、API; 具体来说, 是指 `ServiceConfig` 和 `ReferenceConfig`.

Dubbo 把这样专门存储与读取配置打交道的层次称为配置层, 即 Config. 

### Proxy 代理层

通过动态代理的方式, 根据各种配置信息完成一次完整的远程方法调用. 

Dubbo 把这种代理接口发起远程调用, 或代理接收请求进行实例分发处理的层次,称为服务代理层, 即 Proxy. 

拓展接口为 `ProxyFactory`.

### Registry 注册中心层

帮助我们注册服务、发现服务的层级. 

Dubbo 把这种专门与注册中心打交道的层次, 称为注册中心层, 即 Registry. 

### Cluster 路由层

从注册中心提供的 Provider 列表中, 筛选出最终调用的 Service 对象的层级. 

Dubbo 将这种封装多个提供者并承担路由过滤和负载均衡的层次, 称为路由层, 即 Cluster. 

扩展接口为 `Cluster`, `Directory`, `Router`, `LoadBalance`.

### Monitor 监控层

然而一次远程调用, 总归是要有结果的, 正常也好, 异常也好, 都是一种结果. 比如某个方法调用成功了多少次, 失败了多少次, 调用前后所花费的时间是多少. 这些看似和业务逻辑无关紧要, 实际, 对我们开发人员在分析问题或者预估未来趋势时有着无与伦比的价值. 

于是诞生了一个监控模块来专门处理这种事情, Dubbo 将这种同步调用结果的层次称为监控层, 即 Monitor. 

扩展接口为 `MonitorFactory`, `Monitor`, `MonitorService`.

### Protocol 远程调用层

然而远程调用也是一个过程, 出于增强框架灵活扩展业务的需求, 我们有时候需要在调用之前做点什么, 在调用之后做点什么, 我们前面接触过很多次的过滤器就是过程中的一个环节. 

如果把远程调用看作一个“实体对象”, 拿着这个“实体对象”就能调出去拿到结果, 就好像“实体对象”封装了 RPC 调用的细节, 我们只需要感知“实体对象”的存在就好了. 

那么封装调用细节, 取回调用结果, Dubbo 将这种封装调用过程的层次称为远程调用层, 即 Protocol. 

扩展接口为 `Protocol`, `Invoker`, `Exporter`.

### Exchange 信息交换层

对于我们平常接触的 HTTP 请求来说, 开发人员感知的是调用五花八门的 URL 地址, 但发送 HTTP 报文的逻辑最终归到一个抽象的发送数据的口子, 统一处理. 

对 Dubbo 框架来说也是一样, 消费方的五花八门的业务请求数据最终会封装为 Request、Response 对象, 至于拿着 Request 对象是进行同步调用, 还是直接转异步调用通过 Future.get 拿结果, 那是底层要做的事情, 因此 Dubbo 将这种封装请求并根据同步异步模式获取响应结果的层次, 称为信息交换层, 即 Exchange. 

扩展接口为 `Exchanger`, `ExchangeChannel`, `ExchangeClient`, `ExchangeServer`.

### Transport 网络传输层

当 `Request` 请求对象准备好了, 不管是同步发送, 还是异步发送, 最终都是需要发送出去的, 但是对象通过谁来发到网络中的呢？

这就需要网络通信框架出场了. 网络通信框架, 封装了网络层面的各种细节, 只暴露一些发送对象的简单接口, 上层只需要放心把 Request 对象交给网络通信框架就可以了. 

Dubbo 把这种能将数据通过网络发送至对端服务的层次称为网络传输层, 即 Transport. 

抽象 Mina 和 Netty 为统一接口, 以 Message 为中心, 拓展接口为`Channel`, `Transporter`, `Client`, `Server`, `Codec`.

`NettyChannel` 类图:

![[NettyChannel 类图.png|500]]

### Serialize 数据序列化层

网络通信框架最终要把对象转成二进制才能往网卡中发送, 那么谁来将这些实打实的 Request、Response 对象翻译成网络中能识别的二进制数据呢？

将对象转成二进制或将二进制转成对象的模块, Dubbo 将这种能把对象与二进制进行相互转换的正反序列化的层次称为数据序列化层, 即 Serialize. 

扩展接口为 `Serialization`, `ObjectInput`, `ObjectOutput`, `ThreadPool`.

## 2 层级之间的依赖关系

![[Dubbo 层级之间的依赖关系.png]]

## 3 调用链

展开总设计图的红色调用链, 如下:

![[Dubbo 调用链.png]]

## 4 服务暴露和引用

### 服务暴露时序图

![[Dubbo 服务暴露时序图.png]]

### 服务引用时序图

![[Dubbo 服务引用时序图.png]]