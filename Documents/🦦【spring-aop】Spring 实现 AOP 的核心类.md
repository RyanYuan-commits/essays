# infrastructure
## 什么是 infrastructure？
AOP 中的核心概念，例如 Advice、PointCut 等，在 Spring 的实现中，被视为 AOP 的基础设施（infrastructure）。

`org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#isInfrastructureClass`
```java
protected boolean isInfrastructureClass(Class<?> beanClass) {  
    boolean retVal = Advice.class.isAssignableFrom(beanClass) ||  
          Pointcut.class.isAssignableFrom(beanClass) ||  
          Advisor.class.isAssignableFrom(beanClass) ||  
          AopInfrastructureBean.class.isAssignableFrom(beanClass);  
    if (retVal && logger.isTraceEnabled()) {  
       logger.trace("Did not attempt to auto-proxy infrastructure class [" + beanClass.getName() + "]");  
    }  
    return retVal;  
}
```
上面的是在 `AbstractAutoProxyCreator` 中判断基础设施的方法；
在 Spring 中，无论是 `Pointcut` 或者 `Advisor` 等都会被构造为 Bean 对象；而代理也是要给 Bean 对象创建的，上面的方法就为区分普通 Bean 还是 AOP 基础设施 Bean 提供了支持。
## Advisor 访问者接口
全限定类名：`org.springframework.aop.Advisor`
```java
public interface Advisor {
	Advice EMPTY_ADVICE = new Advice() {};

	Advice getAdvice();
	
	boolean isPerInstance();
}
```
这个类用于 **封装AOP（面向切面编程）中的建议（advice）和过滤器（filter）**，最常见的过滤器就是上面提到的 PointCut。
主要提供了两个功能：
	获取通知逻辑
	判断通知是否与适用于特定的实例
	
在 Spring AOP 中，根据 Filter 的不同，为 `Advisor` 创建了一系列的子接口
其中 Filter 为 PointCut 的接口就是 `PointcutAdvisor`：
全限定名：`org.springframework.aop.PointcutAdvisor`，其在 `Advisor` 的基础上提供了获取切点的功能
```java
public interface PointcutAdvisor extends Advisor {  
  
    /**  
     * Get the Pointcut that drives this advisor.     
     */   
    Pointcut getPointcut();  
  
}
```

