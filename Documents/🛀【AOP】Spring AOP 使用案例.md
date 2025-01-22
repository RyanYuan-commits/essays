# 正式使用

> 通过前面的介绍，我们已经辨析了 AOP 的基本概念，了解了控制何时执行逻辑的**通知类型（Advice）**，定义在什么位置执行的**切点表达式（PointCut）**，下面我们正式来尝试使用 AOP 来解决一些现实的问题。
> 

就举一个前面提到的日志记录的 AOP 吧

**定义一个简单的服务类**
```java
@Service
public class UserService {

    public String getUserById(int userId) {
        // 模拟方法体
        return "User: " + userId;
    }
}
```

**创建日志记录的切面类**
```java
@Aspect
@Component
public class LoggingAspect {

    @Before("execution(* com.example.service.UserService.getUserById(int))")
    public void logBeforeGetUserById(JoinPoint joinPoint) {
		    // 获取方法参数
		    Object[] args = joinPoint.getArgs();
        System.out.println("Before calling getUserById with userId: " + args[0]);
    }

    @AfterReturning(pointcut = "execution(* com.example.service.UserService.getUserById(int))", returning = "result")
    public void logAfterReturningGetUserById(JoinPoint joinPoint, Object result) {
        System.out.println("After returning from getUserById with result: " + result);
    }
}
```

然后我们去做一个简单的测试，可以看到如下的输出：
```java
Before calling getUserById with userId: 123
After returning from getUserById with result: User: 123
User retrieved: User: 123
```

# 关于 JoinPoint 和 ProceedingJoinPoint
> 在 Spring AOP 中，`JoinPoint` 和 `ProceedingJoinPoint` 是两个重要的接口，用于在切面中获取方法执行时的信息和控制方法执行；我们在书写切面逻辑的时候，需要的大部分参数或者方法信息等，都是从这里面获取的。

## JoinPoint 接口
`JoinPoint` 接口是 Spring AOP 提供的一个核心接口，用于描述正在执行的连接点（join point），它可以用来获取方法的签名、参数等信息，但是不能直接控制方法的执行流程。

**常用方法**
```java
Signature getSignature(); // 获取代表被通知方法签名的对象，可以进一步获取方法名、声明类型等信息。

Object[] getArgs(); // 获取被通知方法的参数对象数组。

Object getTarget(); // 获取目标对象，即被通知的目标类实例。

Object getThis(); // 获取代理对象的引用，即代理对象本身。

Object[] getArgs(); // 获取调用方法时传递的参数
```

其中有一个接口名为 `Signature`，方法签名接口：
```java
public interface Signature {
    String toString(); // 返回方法的字符串表示形式。

    String toShortString(); // 返回方法的简短字符串表示形式。

    String toLongString(); // 返回方法的长字符串表示形式。

    String getName();// 获取方法名。 

    int getModifiers(); // 获取方法的修饰符，返回一个整数，具体取值需要通过 java.lang.reflect.Modifier 类来解析。

    Class getDeclaringType(); // 获取声明该方法的类的 Class 对象。

    String getDeclaringTypeName(); // 获取声明该方法的类的全限定名。
}
```
大家可以自己写个方法测试一下，这里就不过多赘述了。

## ProceedingJoinPoint 接口
`ProceedingJoinPoint` 接口继承自 `JoinPoint` 接口，它扩展了 `JoinPoint` 接口，提供了控制方法执行流程的能力。通常在 Around Advice 中使用 `ProceedingJoinPoint` 来调用目标方法，并可以控制是否继续执行该方法，以及在执行前后进行额外的处理。
**常用方法**
```java
Object proceed() throws Throwable; // 继续执行连接点（即目标方法），返回方法的返回值。

Object proceed(Object[] args) throws Throwable; // 按照给定的参数继续执行连接点。
```
由于是继承，所以 JoinPoint 提供的方法，也都可以使用。
## 区别和用途
- **JoinPoint** 主要用于获取方法的元数据信息，如方法名、参数等，不具备控制方法执行流程的能力。
- **ProceedingJoinPoint** 继承自 `JoinPoint`，可以控制方法的执行流程，在 Around Advice 中使用，可以决定是否继续执行目标方法，以及在执行前后进行额外的处理。
