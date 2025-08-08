Spring aop 按照 AspectJ 规范, 定义了一系列相应的内部基础设施(infra).

`AbstractAutoProxyCreator#isInfrastructureClass`
```java
protected boolean isInfrastructureClass(Class<?> beanClass) {  
    boolean retVal = Advice.class.isAssignableFrom(beanClass) ||  
          Pointcut.class.isAssignableFrom(beanClass) ||  
          Advisor.class.isAssignableFrom(beanClass) ||  
          AopInfrastructureBean.class.isAssignableFrom(beanClass);  
          
    // do log
    
    return retVal;  
}
```

上面展示的是 Spring aop 的 infra 判断方法; 这些 infra 在 Spring 中都是以 Bean 对象的形式存在的, 下面来分别介绍一下这几个接口.

## 1 核心功能接口

### 1.1 Advice 通知

```java
public interface Advice {  
  
}
```

标记接口, 标识实现这个接口的类具有封装通知逻辑的能力, 之前提到的 `BeforeAdvice`, `AfterAdvice` 等都是它的子接口.

### 1.2 Pointcut 切点

`org.springframework.aop.Pointcut`

```java
public interface Pointcut {

	ClassFilter getClassFilter();

	MethodMatcher getMethodMatcher();

	Pointcut TRUE = TruePointcut.INSTANCE;

}
```

定义了一个完整的, 用于筛选连接的切点; 核心设计思想是一个切点由类过滤器 ClassFilter 和方法匹配器 MethodMatcher 组成.

接口定义了三个成员, 分别是:

- `getClassFilter`: 返回的是一个 `ClassFilter` 对象, 判断目标类本身是否匹配切点规则;
	
- `getMethodMatcher`: 判断某个方法是否匹配切点规则;
	
- `Pointcut TRUE`: 静态常量, 表示一个永远匹配的切点.

### 1.3 Advisor 访问者

`org.springframework.aop.Advisor`

```java
public interface Advisor {

	Advice EMPTY_ADVICE = new Advice() {};

	Advice getAdvice();
	
	boolean isPerInstance();
	
}
```

核心设计思想是将 Advice 与过滤逻辑绑定在一起，形成一个完整的横切关注点.

接口定义了三个成员，分别是：

- `getAdvice()`: 返回的是一个 `Advice` 对象, 代表了 `Advisor` 所包含的具体通知逻辑. 当切点匹配成功时, Spring 会执行这个 `Advice` 中定义的行为.
    
- `isPerInstance()`: 用于判断该 `Advisor` 是针对每个实例都创建一个新的通知实例, 还是所有实例共享同一个通知. 由于 Spring 框架的发展, 这个方法目前已不再被框架使用, 其功能已被 Spring Bean 的作用域机制所取代.
    
- `EMPTY_ADVICE`: 一个静态常量, 代表一个空的 `Advice` 实现, 用于占位.

其子接口 `PointcutAdvisor`, 就是通过 PointCut 进行过滤的 Advisor:

```java
public interface PointcutAdvisor extends Advisor {  

	Pointcut getPointcut();  

}
```

## 2 核心创建者接口

### 2.1 AopInfrastructureBean 接口

```java
public interface AopInfrastructureBean {  
  
}
```

标记接口, 表示该 bean 是 Spring AOP 基础设施的一部分. 这意味着即使切入点匹配, 此类 bean 也不会受到自动代理的影响.

之前提到, Spring AOP 是通过动态代理来实现的, 更明确的来说是通过实现 `BeanPostProcessor` 接口, 在 Bean 填充属性后, 创建一个代理对象来取代原始的 Bean, 这些与代理对象创建有关的各种 Bean 都实现了 AopInfrastructureBean 接口.