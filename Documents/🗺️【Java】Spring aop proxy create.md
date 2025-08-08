## 1 案例准备

依赖:

```java
// Spring Boot Starter - 包含核心功能
implementation 'org.springframework.boot:spring-boot-starter'

// Spring Boot AOP Starter - 包含 AOP 功能
implementation 'org.springframework.boot:spring-boot-starter-aop'
```

接口与接口实现类:

```java
public interface UserService {  
  
    String queryUserName(Long uId);  
  
    Long queryUserId(String name);  
  
    void testThrowable() throws Exception;  
  
}

@Service  
public class UserServiceImpl implements UserService {  
  
    @Override  
    public String queryUserName(Long uId) {  
        System.out.println("queryUserName: " + uId);  
        return "Ryan";  
    }  
  
    @Override  
    public Long queryUserId(String name) {  
        System.out.println("queryUserId: " + name);  
        return 10086L;  
    }  
  
    @Override  
    public void testThrowable() throws Exception {  
        throw new Exception("testThrowable");  
    }  
  
}
```

Aspect 切面类:

```java
@Aspect  
@Component  
public class LogAspect {  
  
    @Pointcut("execution(* com.ryan.service.impl.*.*(..))")  
    public void pointcut() {}  
  
    @Before("pointcut()")  
    public void before(JoinPoint joinPoint) {  
        System.out.println("before");  
    }  
  
    @AfterThrowing(value = "pointcut()", throwing = "e")  
    public void afterThrowing(JoinPoint joinPoint, Throwable e) {  
        System.out.println("afterThrowing");  
    }  
  
}
```

Application 启动类:

```java
@EnableAspectJAutoProxy(  
        proxyTargetClass = true,  
        exposeProxy = true  
)  
@SpringBootApplication  
public class SpringDemoApplication implements CommandLineRunner {  
  
    @Resource  
    UserService userService;  
  
    public static void main(String[] args) {  
        SpringApplication.run(SpringDemoApplication.class, args);  
    }  
  
    @Override  
    public void run(String... args) {  
        System.out.println(userService);  
        System.out.println(userService.queryUserId("ryan"));  
        System.out.println(userService.queryUserName(10086L));  
        try {  
            userService.testThrowable();  
        } catch (Exception ignore) {}  
    }  
  
}
```

## 2 ProxyCreator 加载流程

```java
@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Import(AspectJAutoProxyRegistrar.class)  
public @interface EnableAspectJAutoProxy {}
```

当添加 `@EnableAspectJAutoProxy` 注解到启动类后, 会自动引入 `AspectJAutoProxyRegistrar` 注册类, `AnnotationAwareAspectJAutoProxyCreator` 类就是在这里被创建的.

最终会调用到 `AopConfigUtils#registerOrEscalateApcAsRequired`, 构建出 `ProxyCreator` 的 `BeanDefinition`, 并将其添加到 `BeanDefinitionRegistry` 中.

## 3 Advisor 加载流程

`AbstractAutoProxyCreator#postProcessBeforeInstantiation`

```java
@Override
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
		Object cacheKey = getCacheKey(beanClass, beanName);

		if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
			if (this.advisedBeans.containsKey(cacheKey)) {
				return null;
			}
			// 判断是否应该跳过
			if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
				this.advisedBeans.put(cacheKey, Boolean.FALSE);
				return null;
			}
		}

		// 处理 TargetSource ...
	}
```

在 Bean 实例化前处理一些基础类的构造逻辑, 其中调用的 `shouldSkip` 方法是这个抽象类的拓展点, 它允许子类定义更具体的规则来判断哪些 Bean 应该被排除在自动代理之外.

`AspectJAwareAdvisorAutoProxyCreator#shouldSkip`

```java
@Override  
protected boolean shouldSkip(Class<?> beanClass, String beanName) {   
    List<Advisor> candidateAdvisors = findCandidateAdvisors();  

	// 判断该 Bean 是否是一个 Advisor

    return super.shouldSkip(beanClass, beanName);  
}
```

在上面的 `findCandidateAdvisors` 会从各种位置找到 `Advisors`, 比如实现了 `Advisor` 的 Bean, xml 文件中配置的 `Advisor`, 还有通过 `@Aspect` 注解定义的切面类中的 `Advisor`.

`AnnotationAwareAspectJAutoProxyCreator#findCandidateAdvisors`

```java
protected List<Advisor> findCandidateAdvisors() {  
    // Add all the Spring advisors found according to superclass rules.  
    List<Advisor> advisors = super.findCandidateAdvisors();  
    // Build Advisors for all AspectJ aspects in the bean factory.  
    if (this.aspectJAdvisorsBuilder != null) {  
       advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());  
    }  
    return advisors;  
}
```

首先会调用到其父类 `AbstractAdvisorAutoProxyCreator` 的对应方法

`AbstractAdvisorAutoProxyCreator#findCandidateAdvisors`, 这个方法最终会通过不断调用 `BeanFactory` 的 `getBean` 方法, 将所有 `Advisor` 类型的 Bean 获取并返回.

然后会调用 `BeanFactoryAspectJAdvisorsBuilder#buildAspectJAdvisors` 方法, 这个方法会获取到 `BeanFactory` 中所有的 BeanName, 逐一遍历, 将所有切面类中定义的 Advisor 都注册并保存.

## 4 创建代理类

`AbstractAutoProxyCreator#postProcessAfterInitialization`

```java
@Override  
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
    if (bean != null) {
       Object cacheKey = getCacheKey(bean.getClass(), beanName);
       if (this.earlyProxyReferences.remove(cacheKey) != bean) {
          return wrapIfNecessary(bean, beanName, cacheKey);
       }
    }
    return bean;  
}
```

在 Bean 初始化后执行, 对符合条件的 Bean 执行 `wrapIfNecessary` 方法创建其代理类.

`AbstractAutoProxyCreator#wrapIfNecessary`

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {  

    // ......
  
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);  
    if (specificInterceptors != DO_NOT_PROXY) {  
       this.advisedBeans.put(cacheKey, Boolean.TRUE);  
       Object proxy = createProxy(  
             bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));  
       this.proxyTypes.put(cacheKey, proxy.getClass());  
       return proxy;  
    }  
  
    this.advisedBeans.put(cacheKey, Boolean.FALSE);  
    return bean;  
}
```

通过 `getAdvicesAndAdvisorsForBean` 获取该类的适配的各种 `Advisor`, 具体来说就是获取 `Advisor` 接口的类和方法匹配器, 将匹配的 `Advisor` 选择出来;

在返回之前, 还会将 `ExposeInvocationInterceptor` 添加到拦截器数组中, 来支持代理 Bean 的暴露能力.

如果 `specificInterceptors` 不为空, 就会调用 `createProxy` 方法创建代理类.

`DefaultAopProxyFactory#createAopProxy`

```java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {  
    if (!NativeDetector.inNativeImage() &&  
          (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config))) {  
       Class<?> targetClass = config.getTargetClass();  
       // ......
       if (targetClass.isInterface() || Proxy.isProxyClass(targetClass) || ClassUtils.isLambdaClass(targetClass)) {  
          return new JdkDynamicAopProxy(config);  
       }  
       return new ObjenesisCglibAopProxy(config);  
    }  
    else {  
       return new JdkDynamicAopProxy(config);  
    }  
}
```

这里总结一下什么条件下会使用 JDK 代理, 什么条件下使用 CgLib 代理:

| 条件                      | 代理类型     | 解释                            |
| ----------------------- | -------- | ----------------------------- |
| 目标类实现了接口，且未强制使用 CGLIB   | JDK 动态代理 | Spring 的默认策略，最常见的情况。          |
| 目标类没有实现任何接口             | CGLIB    | JDK 代理无法工作，Spring 必须回退。       |
| proxyTargetClass 为 true | CGLIB    | 强制使用 CGLIB，无论是否实现了接口。         |
| optimize 为 true         | CGLIB    | 强制使用 CGLIB 以优化性能。             |
| 目标类是接口、已是代理类或 Lambda    | JDK 动态代理 | 即使满足 CGLIB 条件，也优先使用 JDK 动态代理。 |
| 在原生镜像（Native Image）环境中  | JDK 动态代理 | 默认选择，以确保兼容性。                  |

## 5 代理类逻辑

```java
@Override
@Nullable
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	Object oldProxy = null;
	boolean setProxyContext = false;

	TargetSource targetSource = this.advised.targetSource;
	Object target = null;

	try {
	
		// ... 处理特殊方法

		Object retVal;

		if (this.advised.exposeProxy) {
			oldProxy = AopContext.setCurrentProxy(proxy);
			setProxyContext = true;
		}

		target = targetSource.getTarget();
		Class<?> targetClass = (target != null ? target.getClass() : null);

		List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

		if (chain.isEmpty()) {
			// ......
		}
		else {
			MethodInvocation invocation =
					new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
			retVal = invocation.proceed();
		}

		// ......
		return retVal;
	}
	finally {
		// ......
	}
}
```

核心思路就是获取所有的 `Interceptors` 然后按照顺序执行, 通知顺序最终会映射到拦截器在数组中的顺序.

