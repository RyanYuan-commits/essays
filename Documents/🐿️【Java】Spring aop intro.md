## 1 什么是 AOP?

AOP (Aspect Oriented Programming), 意为 **面向切面编程**, 通过预编译方式和运行期动态代理实现 **程序功能的统一维护** 的一种技术.

利用AOP可以对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的耦合度降低，提高程序的可重用性，同时提高了开发的效率。

## 2 AOP 核心概念

**连接点(JoinPoint)**: 程序中一个潜在的, 可以被拦截的点:

- 想象你的代码像一条时间线, 连接点就是时间线上一个个可以被"打断"或"插入"的点. 例如, 方法的调用, 异常的抛出, 字段的读取等, 都是潜在的连接点;
- 后面可以了解到, 在 spring aop 中, 连接点被特化成了方法的调用或执行.

**切点(PointCut)**: 筛选连接点的规则或筛选器:

- 连接点是所有能处理的点, 而切点是真正要处理的点;
- 在 spring aop 中, PointCut 就是要在哪些方法上引用通知.

**通知(Advice)**: 在切点上执行的具体的行为或逻辑.

**切面(Aspect)**: 一个模块化的单元, 它将切点和通知封装在一起.

## 3 AspectJ 与 Spring aop

AspectJ 是一个独立的, 功能全面的 AOP 框架, 同时定义了各种关于 AOP 的规范, 比如上面提到的各种概念; AspectJ 的原理是字节码增强, 通过 **直接修改目标类的字节码**, 将切面代码植入.

Spring aop 是 Spring 对 AspectJ 规范的实现, 使用了 AspectJ 定义的一系列注解规范, 但是底层是通过 JDK Proxy 或者 CgLib 动态代理来实现的.

## 4 通知类型

上面提到, 通知是在切点上执行的具体行为或者逻辑, spring aop 中, 根据通知的 **执行时机** 将其分为了五类.

**前置通知**: 在目标方法执行之前执行的通知. 可以用来执行日志记录, 安全检查等.

```java
@Aspect
@Component
public class LoggingAspect {
    @Before("execution(* com.example.service.*.*(..))")
    public void beforeAdvice() {
        System.out.println("前置通知：方法调用之前执行");
    }
}
```

**后置通知**: 在目标方法执行之后执行的通知, 无论方法是成功返回还是抛出异常.

```java
@Aspect
@Component
public class LoggingAspect {
    @After("execution(* com.example.service.*.*(..))")
    public void afterAdvice() {
        System.out.println("后置通知：方法调用之后执行");
    }
}
```

**返回后通知**:在目标方法成功返回结果之后执行的通知.可以用来记录返回值或对返回值进行处理.

```java
@Aspect
@Component
public class LoggingAspect {
    @AfterReturning(pointcut = "execution(* com.example.service.*.*(..))", returning = "result")
    public void afterReturningAdvice(Object result) {
        System.out.println("返回后通知：方法返回值为 " + result);
    }
}
```

**抛出异常后通知**: 在目标方法抛出异常后执行的通知, 可以用来记录异常信息或执行异常处理逻辑.

```java
@Aspect
@Component
public class LoggingAspect {
    @AfterThrowing(pointcut = "execution(* com.example.service.*.*(..))", throwing = "exception")
    public void afterThrowingAdvice(Exception exception) {
        System.out.println("抛出异常后通知：异常为 " + exception.getMessage());
    }
}
```

**环绕通知**: 环绕通知在目标方法执行的前后都执行, 可以完全控制目标方法的执行, 包括决定是否执行目标方法, 以及在目标方法执行前后添加自定义逻辑.

```java
@Aspect
@Component
public class LoggingAspect {
    @Around("execution(* com.example.service.*.*(..))")
    public Object aroundAdvice(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println("环绕通知：方法调用之前");
        Object result = joinPoint.proceed();  // 执行目标方法
        System.out.println("环绕通知：方法调用之后");
        return result;
    }
}
```

其中环绕通知可以接收可执行的连接点参数 ProceedingJoinPoint, 来自由的控制方法的执行, 其余的通知接收的是连接点参数 JoinPoint.

这五种通知类型的执行顺序为: 前置通知, 环绕通知的前半部分, 目标方法执行, 环绕通知的后半部分, 返回后通知(如果目标方法成功返回), 抛出异常后通知(如果目标方法抛出异常), 后置通知.

其中返回后通知和抛出异常后通知是有条件的, 其他通知则无论什么情况均会执行.
