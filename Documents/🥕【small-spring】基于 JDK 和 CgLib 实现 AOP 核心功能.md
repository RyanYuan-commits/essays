# 目标
到本章节我们将要从 IOC 的实现，转入到关于 AOP(`Aspect Oriented Programming`) 内容的开发。
在软件行业，AOP 意为：面向切面编程，通过预编译的方式和运行期间动态代理实现程序功能功能的统一维护。其实 AOP 也是 OOP 的延续，在 Spring 框架中是一个非常重要的内容，使用 AOP 可以对业务逻辑的各个部分进行隔离，从而使各模块间的业务逻辑耦合度降低，提高代码的可复用性，同时也能提高开发效率。
关于 AOP 的核心技术实现主要是动态代理的使用，就像你可以给一个接口的实现类，使用代理的方式替换掉这个实现类，使用代理类来处理你需要的逻辑。比如：
```java
@Test
public void test_proxy_class() {
    IUserService userService = (IUserService) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), new Class[]{IUserService.class}, (proxy, method, args) -> "你被代理了！");
    String result = userService.queryUserInfo();
    System.out.println("测试结果：" + result);
}
```
# 方案
在把 AOP 整个切面设计融合到 Spring 前，我们需要解决两个问题，包括：`如何给符合规则的方法做代理`，`以及做完代理方法的案例后，把类的职责拆分出来`。
而这两个功能点的实现，都是以切面的思想进行设计和开发。如果不是很清楚 AOP 是啥，你可以把切面理解为用刀切韭菜，一根一根切总是有点慢，那么用手(`代理`)把韭菜捏成一把，用菜刀或者斧头这样不同的拦截操作来处理。而程序中其实也是一样，只不过韭菜变成了方法，菜刀变成了拦截方法。整体设计结构如下图：
![[【small-spring】基于 JDK 和 CgLib 实现 AOP 核心功能 架构图.png]]
- 就像你在使用 Spring 的 AOP 一样，只处理一些需要被拦截的方法。在拦截方法后，执行你对方法的扩展操作。
- 那么我们就需要先来实现一个可以代理方法的 Proxy，其实代理方法主要是使用到方法拦截器类处理方法的调用 `MethodInterceptor#invoke`，而不是直接使用 invoke 方法中的入参 Method method 进行 `method.invoke(targetObj, args)` 这块是整个使用时的差异。
- 除了以上的核心功能实现，还需要使用到 `org.aspectj.weaver.tools.PointcutParser` 处理拦截表达式 `"execution(* cn.bugstack.springframework.test.bean.IUserService.*(..))"`，有了方法代理和处理拦截，我们就可以完成设计出一个 AOP 的雏形了。
# 实现
AOP 切点表达式和使用以及基于 JDK 和 CGLIB 的动态代理类关系，如图 12-2
![[【small-spring】基于 JDK 和 CgLib 实现 AOP 核心功能 核心类图.png|800]]
- 整个类关系图就是 AOP 实现核心逻辑的地方，上面部分是关于方法的匹配实现，下面从 AopProxy 开始是关于方法的代理操作。
- AspectJExpressionPointcut 的核心功能主要依赖于 aspectj 组件并处理 Pointcut、ClassFilter,、MethodMatcher 接口实现，专门用于处理类和方法的匹配过滤操作。
- AopProxy 是代理的抽象对象，它的实现主要是基于 JDK 的代理和 Cglib 代理。在前面章节关于对象的实例化 CglibSubclassingInstantiationStrategy，我们也使用过 Cglib 提供的功能。
## 代理方法案例
在实现 AOP 的核心功能之前，我们先做一个代理方法的案例，通过这样一个可以概括代理方法的核心全貌，可以让大家更好的理解后续拆解各个方法，设计成解耦功能的 AOP 实现过程。
```java
@Test
public void test_proxy_method() {
    // 目标对象(可以替换成任何的目标对象)
    Object targetObj = new UserService();
    // AOP 代理
    IUserService proxy = (IUserService) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), targetObj.getClass().getInterfaces(), new InvocationHandler() {
        // 方法匹配器
        MethodMatcher methodMatcher = new AspectJExpressionPointcut("execution(* cn.bugstack.springframework.test.bean.IUserService.*(..))");
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            if (methodMatcher.matches(method, targetObj.getClass())) {
                // 方法拦截器
                MethodInterceptor methodInterceptor = invocation -> {
                    long start = System.currentTimeMillis();
                    try {
                        return invocation.proceed();
                    } finally {
                        System.out.println("监控 - Begin By AOP");
                        System.out.println("方法名称：" + invocation.getMethod().getName());
                        System.out.println("方法耗时：" + (System.currentTimeMillis() - start) + "ms");
                        System.out.println("监控 - End\r\n");
                    }
                };
                // 反射调用
                return methodInterceptor.invoke(new ReflectiveMethodInvocation(targetObj, method, args));
            }
            return method.invoke(targetObj, args);
        }
    });
    String result = proxy.queryUserInfo();
    System.out.println("测试结果：" + result);
}
```
- 首先整个案例的目标是给一个 UserService 当成目标对象，对类中的所有方法进行拦截添加监控信息打印处理。
- 从案例中你可以看到有代理的实现 Proxy.newProxyInstance，有方法的匹配 MethodMatcher，有反射的调用 invoke(Object proxy, Method method, Object[] args)，也用用户自己拦截方法后的操作。这样一看其实和我们使用的 AOP 就非常类似了，只不过你在使用 AOP 的时候是框架已经提供更好的功能，这里是把所有的核心过程给你展示出来了。
## 切点表达式
```java
public interface Pointcut {

    /**
     * Return the ClassFilter for this pointcut.
     * @return the ClassFilter (never <code>null</code>)
     */
    ClassFilter getClassFilter();

    /**
     * Return the MethodMatcher for this pointcut.
     * @return the MethodMatcher (never <code>null</code>)
     */
    MethodMatcher getMethodMatcher();

}
```
- 切入点接口，定义用于获取 ClassFilter、MethodMatcher 的两个类，这两个接口获取都是切点表达式提供的内容，通过确定 类 和 方法，能够精确的定位到一个切点

```java
public interface ClassFilter {

    /**
     * Should the pointcut apply to the given interface or target class?
     * @param clazz the candidate target class
     * @return whether the advice should apply to the given target class
     */
    boolean matches(Class<?> clazz);

}
```
- 定义类匹配类，用于切点找到给定的接口和目标类。

```java
public interface MethodMatcher {

    /**
     * Perform static checking whether the given method matches. If this
     * @return whether or not this method matches statically
     */
    boolean matches(Method method, Class<?> targetClass);
    
}
```
- 方法匹配，找到表达式范围内匹配下的目标类和方法。在上文的案例中有所体现：`methodMatcher.matches(method, targetObj.getClass())`

```java
public class AspectJExpressionPointcut implements Pointcut, ClassFilter, MethodMatcher {

    private static final Set<PointcutPrimitive> SUPPORTED_PRIMITIVES = new HashSet<PointcutPrimitive>();

    static {
        SUPPORTED_PRIMITIVES.add(PointcutPrimitive.EXECUTION);
    }

    private final PointcutExpression pointcutExpression;

    public AspectJExpressionPointcut(String expression) {
        PointcutParser pointcutParser = PointcutParser.getPointcutParserSupportingSpecifiedPrimitivesAndUsingSpecifiedClassLoaderForResolution(SUPPORTED_PRIMITIVES, this.getClass().getClassLoader());
        pointcutExpression = pointcutParser.parsePointcutExpression(expression);
    }

    @Override
    public boolean matches(Class<?> clazz) {
        return pointcutExpression.couldMatchJoinPointsInType(clazz);
    }

    @Override
    public boolean matches(Method method, Class<?> targetClass) {
        return pointcutExpression.matchesMethodExecution(method).alwaysMatches();
    }

    @Override
    public ClassFilter getClassFilter() {
        return this;
    }

    @Override
    public MethodMatcher getMethodMatcher() {
        return this;
    }

}
```
- 切点表达式实现了 Pointcut、ClassFilter、MethodMatcher，三个接口定义方法，同时这个类主要是对 aspectj 包提供的表达式校验方法使用。
- 匹配 matches：`pointcutExpression.couldMatchJoinPointsInType(clazz)`、`pointcutExpression.matchesMethodExecution(method).alwaysMatches()`，这部分内容可以单独测试验证。

## 测试匹配验证
```java
@Test
public void test_aop() throws NoSuchMethodException {
    AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut("execution(* cn.bugstack.springframework.test.bean.UserService.*(..))");
    Class<UserService> clazz = UserService.class;
    Method method = clazz.getDeclaredMethod("queryUserInfo");   

    System.out.println(pointcut.matches(clazz));
    System.out.println(pointcut.matches(method, clazz));          
    
    // true、true
}
```

## 包装切面 Advice
```java
public class AdvisedSupport {

    // 被代理的目标对象
    private TargetSource targetSource;
    // 方法拦截器
    private MethodInterceptor methodInterceptor;
    // 方法匹配器(检查目标方法是否符合通知条件)
    private MethodMatcher methodMatcher;
    
    // ...get/set
}
```
- `AdvisedSupport`，主要是用于把代理、拦截、匹配的各项属性包装到一个类中，方便在 Proxy 实现类进行使用。这和业务开发中包装入参是一个道理
- `TargetSource`，是一个目标对象，在目标对象类中提供 Object 入参属性，以及获取目标类 TargetClass 信息。
- `MethodInterceptor`，是一个具体拦截方法实现类，由用户自己实现 MethodInterceptor#invoke 方法，做具体的处理。
- `MethodMatcher`，是一个匹配方法的操作，这个对象由 AspectJExpressionPointcut 提供服务。

## 代理接口的抽象实现
```java
public interface AopProxy {

    Object getProxy();

}
```
- 定义一个标准接口，用于获取代理类。因为具体实现代理的方式可以有 JDK 方式，也可以是 Cglib 方式，所以定义接口会更加方便管理实现类。

```java
public class JdkDynamicAopProxy implements AopProxy, InvocationHandler {

    private final AdvisedSupport advised;

    public JdkDynamicAopProxy(AdvisedSupport advised) {
        this.advised = advised;
    }

    @Override
    public Object getProxy() {
        return Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), advised.getTargetSource().getTargetClass(), this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (advised.getMethodMatcher().matches(method, advised.getTargetSource().getTarget().getClass())) {
            MethodInterceptor methodInterceptor = advised.getMethodInterceptor();
            return methodInterceptor.invoke(new ReflectiveMethodInvocation(advised.getTargetSource().getTarget(), method, args));
        }
        return method.invoke(advised.getTargetSource().getTarget(), args);
    }

}
```
- 基于 JDK 实现的代理类，需要实现接口 AopProxy、InvocationHandler，这样就可以把代理对象 getProxy 和反射调用方法 invoke 分开处理了。
- getProxy 方法中的是代理一个对象的操作，需要提供入参 ClassLoader、AdvisedSupport、和当前这个类 this，因为这个类提供了 invoke 方法。
- invoke 方法中主要处理匹配的方法后，使用用户自己提供的方法拦截实现，做反射调用 methodInterceptor.invoke 。
- 这里还有一个 ReflectiveMethodInvocation，其他它就是一个入参的包装信息，提供了入参对象：目标对象、方法、入参。
# 测试
```java
@Test  
public void test_dynamic() {  
    // 目标对象  
    IUserService userService = new UserService();  
    // 组装代理信息  
    AdvisedSupport advisedSupport = new AdvisedSupport();  
    advisedSupport.setTargetSource(new TargetSource(userService));  
    advisedSupport.setMethodInterceptor(new UserServiceInterceptor());  
    advisedSupport.setMethodMatcher(new AspectJExpressionPointcut("execution(* cn.bugstack.springframework.test.bean.IUserService.*(..))"));  
  
    // 代理对象(JdkDynamicAopProxy)  
    IUserService proxy_jdk = (IUserService) new JdkDynamicAopProxy(advisedSupport).getProxy();  
    // 测试调用  
    System.out.println("测试结果：" + proxy_jdk.queryUserInfo());  
  
    // 代理对象(Cglib2AopProxy)  
    IUserService proxy_cglib = (IUserService) new Cglib2AopProxy(advisedSupport).getProxy();  
    // 测试调用  
    System.out.println("测试结果：" + proxy_cglib.register("花花"));  
}
```
