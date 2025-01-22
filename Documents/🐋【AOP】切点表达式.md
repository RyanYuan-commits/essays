#aop

切点则表示**一组** joint point，这些 joint point 或是通过逻辑关系组合起来，或是通过通配、正则表达式等方式集中起来，它定义了相应的 Advice 将要发生的地方。

切点表达式的格式是这样的(`?` 表示可选参数)

```java
execution(modifiers-pattern? ret-type-pattern declaring-type-pattern? name-pattern(param-pattern) throws-pattern?)
```

参数具体的含义为：
	`execution`：指定切点类型为方法执行。
	`modifiers-pattern`：可选，方法的访问修饰符，如 public、protected、private。通常省略不写，表示任意访问修饰符。
	`ret-type-pattern`：方法的返回类型模式，例如 void、String、``（任意返回类型）。
	`declaring-type-pattern`：可选，方法所在类的全限定名称模式，如 com.example.service.\*。指定要匹配的类或包。
	`name-pattern`：方法名称模式，如 Service、get*。支持通配符 ``（匹配任意字符序列）和 ..（匹配任意数量和类型的参数）。
	`param-pattern`：方法参数模式，如 ()（无参数）、(\*)（一个任意类型的参数）、(..)（任意数量和类型的参数）。
	`throws-pattern`：可选，方法可能抛出的异常模式。

来看几个案例：
1）指定 `com.example.service` 包下的所有类的所有方法：
```java
@Pointcut("execution(* com.example.service.*.*(..))")
private void serviceMethods() {}
```
单独将上面的切点表达式摘出来是这样的：
![[切点表达式案例.png]]
2）com.example.service 包下所有类的方法中第一个参数为 String 类型的方法
```java
@Pointcut("execution(* com.example.service.*.*(String, ..))")
private void stringParamMethods() {}
```

除了上面简单的案例，切点表达式还有一些特殊的方式：
	`within`：限定匹配位置的连接点。within(com.example.service.\*) 匹配 com.example.service 包及其子包中所有方法。
	`this`：限定匹配特定类型的 bean。this(com.example.service.MyService) 匹配实现 com.example.service.MyService 接口的 bean。
	`target`：限定匹配特定类型的目标对象。target(com.example.service.MyService) 匹配目标对象是 com.example.service.MyService 的连接点。
	`args`：限定匹配特定参数类型的方法。args(String, ..) 匹配第一个参数是 String 类型的方法。
	`@annotation`：限定匹配特定注解的方法。@annotation(org.springframework.transaction.annotation.Transactional) 匹配标注有 @Transactional 注解的方法。

但同时，这些表达式其实是可以共用的，比如通过这样的方式：
```java
@Before("execution(* com.example.service.UserService.getUserById(int)) && args(userId)")
    public void logBeforeGetUserById(JoinPoint joinPoint, int userId) {
        System.out.println("Before calling getUserById with userId: " + userId);
    }
```
上面的 userId 参数是通过切点表达式中的 args(userId) 指定的，所以在方法体内可以直接使用 userId 参数来获取方法执行时的具体值；但如果仅仅使用 joinPoint 的话就需要 getArgs() 再拿取参数了
