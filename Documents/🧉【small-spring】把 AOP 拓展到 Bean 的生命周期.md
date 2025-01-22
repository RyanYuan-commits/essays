# 目标
在上一章节我们通过基于 Proxy.newProxyInstance 代理操作中处理方法匹配和方法拦截，对匹配的对象进行自定义的处理操作。并把这样的技术核心内容拆解到 Spring 中，用于实现 AOP 部分，通过拆分后基本可以明确各个类的职责，包括你的代理目标对象属性、拦截器属性、方法匹配属性，以及两种不同的代理操作 JDK 和 CGlib 的方式。
再有了一个 AOP 核心功能的实现后，我们可以通过单元测试的方式进行验证切面功能对方法进行拦截，但如果这是一个面向用户使用的功能，就不太可能让用户这么复杂且没有与 Spring 结合的方式单独使用 AOP，虽然可以满足需求，但使用上还是过去分散。
因此我们需要在本章节完成 AOP 核心功能与 Spring 框架的整合，最终能通过在 Spring 配置的方式完成切面的操作。

本章只是实现了一个简单的代理类，并没有填充属性。

# 方案
其实在有了AOP的核心功能实现后，把这部分功能服务融入到 Spring 其实也不难，只不过要解决几个问题，包括：怎么借着 BeanPostProcessor 把动态代理融入到 Bean 的生命周期中，以及如何组装各项切点、拦截、前置的功能和适配对应的代理器。整体设计结构如下图：
![[【small-spring】把 AOP 拓展到 Bean 的生命周期 架构图.png]]
# 实现
![[【small-spring】把 AOP 拓展到 Bean 的生命周期 核心类图.png|900]]
- 整个类关系图中可以看到，在以 BeanPostProcessor 接口实现继承的 InstantiationAwareBeanPostProcessor 接口后，做了一个自动代理创建的类 DefaultAdvisorAutoProxyCreator，这个类的就是用于处理整个 AOP 代理融入到 Bean 生命周期中的核心类。
- DefaultAdvisorAutoProxyCreator 会依赖于拦截器、代理工厂和Pointcut与Advisor的包装服务 AspectJExpressionPointcutAdvisor，由它提供切面、拦截方法和表达式。
- Spring 的 AOP 把 Advice 细化了 BeforeAdvice、AfterAdvice、AfterReturningAdvice、ThrowsAdvice，目前我们做的测试案例中只用到了 BeforeAdvice，这部分可以对照 Spring 的源码进行补充测试。
## 定义 Advice 拦截器链
```java
public interface BeforeAdvice extends Advice {

}
```

```java
public interface MethodBeforeAdvice extends BeforeAdvice {

    /**
     * Callback before a given method is invoked.
     *
     * @param method method being invoked
     * @param args   arguments to the method
     * @param target target of the method invocation. May be <code>null</code>.
     * @throws Throwable if this object wishes to abort the call.
     *                   Any exception thrown will be returned to the caller if it's
     *                   allowed by the method signature. Otherwise the exception
     *                   will be wrapped as a runtime exception.
     */
    void before(Method method, Object[] args, Object target) throws Throwable;

}
```
- 在Spring 框架中，Advice 都是通过方法拦截器 MethodInterceptor 实现的。环绕 Advice 类似一个拦截器的链路，Before Advice、After Advice 等，不过暂时我们需要那么多就只定义了一个 MethodBeforeAdvice 的接口定义。
## 定义 Advisor 访问者
```java
public interface Advisor {

    /**
     * Return the advice part of this aspect. An advice may be an
     * interceptor, a before advice, a throws advice, etc.
     * @return the advice that should apply if the pointcut matches
     * @see org.aopalliance.intercept.MethodInterceptor
     * @see BeforeAdvice
     */
    Advice getAdvice();

}
```

```java
public interface PointcutAdvisor extends Advisor {

    /**
     * Get the Pointcut that drives this advisor.
     */
    Pointcut getPointcut();

}
```
- PointcutAdvisor 承担了 Pointcut 和 Advice 的组合，Pointcut 用于获取 JoinPoint，而 Advice 决定于 JoinPoint 执行什么操作。

## 方法拦截器
```java
public class MethodBeforeAdviceInterceptor implements MethodInterceptor {

    private MethodBeforeAdvice advice;

    public MethodBeforeAdviceInterceptor(MethodBeforeAdvice advice) {
        this.advice = advice;
    }

    public MethodBeforeAdvice getAdvice() {
        return advice;
    }

    public void setAdvice(MethodBeforeAdvice advice) {
        this.advice = advice;
    }

    @Override
    public Object invoke(MethodInvocation methodInvocation) throws Throwable {
        this.advice.before(methodInvocation.getMethod(), methodInvocation.getArguments(), methodInvocation.getThis());
        return methodInvocation.proceed();
    }

}

```
- MethodBeforeAdviceInterceptor 实现了 MethodInterceptor 接口，在 invoke 方法中调用 advice 中的 before 方法，传入对应的参数信息。
- 而这个 advice.before 则是用于自己实现 MethodBeforeAdvice 接口后做的相应处理。_其实可以看到具体的 MethodInterceptor 实现类，其实和我们之前做的测试是一样的，只不过现在交给了 Spring 来处理_
## 代理工厂
```java
public class ProxyFactory {

    private AdvisedSupport advisedSupport;

    public ProxyFactory(AdvisedSupport advisedSupport) {
        this.advisedSupport = advisedSupport;
    }

    public Object getProxy() {
        return createAopProxy().getProxy();
    }

    private AopProxy createAopProxy() {
        if (advisedSupport.isProxyTargetClass()) {
            return new Cglib2AopProxy(advisedSupport);
        }

        return new JdkDynamicAopProxy(advisedSupport);
    }

}
```
# 融入 Bean 生命周期的自动代理创建者
```java
public class DefaultAdvisorAutoProxyCreator implements InstantiationAwareBeanPostProcessor, BeanFactoryAware {

    private DefaultListableBeanFactory beanFactory;

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = (DefaultListableBeanFactory) beanFactory;
    }

    @Override
    public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {

        if (isInfrastructureClass(beanClass)) return null;

        Collection<AspectJExpressionPointcutAdvisor> advisors = beanFactory.getBeansOfType(AspectJExpressionPointcutAdvisor.class).values();

        for (AspectJExpressionPointcutAdvisor advisor : advisors) {
            ClassFilter classFilter = advisor.getPointcut().getClassFilter();
            if (!classFilter.matches(beanClass)) continue;

            AdvisedSupport advisedSupport = new AdvisedSupport();

            TargetSource targetSource = null;
            try {
                targetSource = new TargetSource(beanClass.getDeclaredConstructor().newInstance());
            } catch (Exception e) {
                e.printStackTrace();
            }
            advisedSupport.setTargetSource(targetSource);
            advisedSupport.setMethodInterceptor((MethodInterceptor) advisor.getAdvice());
            advisedSupport.setMethodMatcher(advisor.getPointcut().getMethodMatcher());
            advisedSupport.setProxyTargetClass(false);

            return new ProxyFactory(advisedSupport).getProxy();

        }

        return null;
    }
    
}
```
- 因为创建的是代理对象不是之前流程里的普通对象，所以我们需要前置于其他对象的创建，即需要在 AbstractAutowireCapableBeanFactory#createBean 优先完成 Bean 对象的判断，是否需要代理，有则直接返回代理对象。
# 测试
```java
public class UserServiceInterceptor implements MethodInterceptor {  
    @Override  
    public Object invoke(MethodInvocation invocation) throws Throwable {  
        long start = System.currentTimeMillis();  
        try {  
            return invocation.proceed();  
        } finally {  
            System.out.println("监控 - Begin By AOP");  
            System.out.println("方法名称：" + invocation.getMethod());  
            System.out.println("方法耗时：" + (System.currentTimeMillis() - start) + "ms");  
            System.out.println("监控 - End\r\n");  
        }  
    }  
}
```

```java
public class UserServiceBeforeAdvice implements MethodBeforeAdvice {  
  
    @Override  
    public void before(Method method, Object[] args, Object target) throws Throwable {  
        System.out.println("拦截方法：" + method.getName());  
    }  
  
}
```

`spring.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>  
<beans>  
    <bean id="userService" class="cn.bugstack.springframework.test.bean.UserService"/>  
  
    <bean class="cn.bugstack.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"/>  
  
    <bean id="beforeAdvice" class="cn.bugstack.springframework.test.bean.UserServiceBeforeAdvice"/>  
  
    <bean id="methodInterceptor" class="cn.bugstack.springframework.aop.framework.adapter.MethodBeforeAdviceInterceptor">  
        <property name="advice" ref="beforeAdvice"/>  
    </bean>  
  
    <bean id="pointcutAdvisor" class="cn.bugstack.springframework.aop.aspectj.AspectJExpressionPointcutAdvisor">  
        <property name="expression" value="execution(* cn.bugstack.springframework.test.bean.IUserService.*(..))"/>  
        <property name="advice" ref="methodInterceptor"/>  
    </bean>  
</beans>
```