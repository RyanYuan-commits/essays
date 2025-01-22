要了解 SpringCloud Alibaba ，先得了解 SpringCloud netflix。
为啥？ SpringCloud Alibaba 仅仅是在 SpringCloud netflix 的基础上，替换了部分组件。 比如说注册中心，比如 RPC 组件。所以，咱们得从 SpringCloud netflix 开始。
SpringCloud Netflix 全家桶是 Pivotal 团队提供的一整套微服务开源解决方案，包括服务注册与发现、配置中心、全链路监控、服务网关、负载均衡、断路器等组件，以上的组件主要通过对 NetFilx 的 NetFlix OSS 套件中的组件通过整合完成的，其中，比较重要的整合组件有:
	（1）spring-cloud-netflix-Eureka 注册中心
	（2）spring-cloud-netflix-hystrix RPC保护组件
	（3）spring-cloud-netflix-ribbon 客户端负载均衡组件
	（4）spring-cloud-netflix-zuul 内部网关组件
	（6）spring-cloud-config 配置中心
SpringCloud 全家桶技术栈除了对 NetFlix OSS的开源组件做整合之外，还有整合了一些选型中立的开源组件。比如，SpringCloud Zookeeper 组件整合了 Zookeeper，提供了另一种方式的服务发现和配置管理。
SpringCloud 架构中的单体业务服务是基于 SpringBoot 应用进行启动和执行的。SpringBoot 是由 Pivotal 团队提供的全新框架，其设计目的是用来简化新 Spring 应用的初始搭建以及开发过程。
SpringCloud 利用 SpringBoot 是什么关系呢？
	（1）首先 SpringCloud 利用 SpringBoot 开发便利性巧妙地简化了分布式系统基础设施的开发；
	（2）其次 SpringBoot 专注于快速方便地开发单体微服务提供者，而 SpringCloud 解决的是各微服务提供者之间的协调治理关系；
	（3）第三 SpringBoot 可以离开 SpringCloud 独立使用开发项目，但是 SpringCloud 离不开 SpringBoot，其依赖 SpringBoot 而存在。
最终，SpringCloud 将 SpringBoot 开发的一个个单体微服务整合并管理起来，为各单体微服务提供配置管理、服务发现、断路器、路由、微代理、事件总线、全局锁、决策竞选、分布式会话等等基础的分布式协助能力。