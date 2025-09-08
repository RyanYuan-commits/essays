Spring 的事务机制是 Spring 框架中非常重要的一部分，它为企业级应用提供了强大而灵活的事务管理能力。
### 1 事务的基本概念
事务是一组不可分割的操作序列，这些操作要么全部成功执行，要么全部失败回滚，以保证数据的一致性和完整性。事务具有四个特性，通常简称为 ACID：
- **原子性（Atomicity）**：事务是一个不可分割的工作单位，事务中的操作要么全部成功，要么全部失败。
- **一致性（Consistency）**：事务执行前后，数据库的状态必须保持一致。例如，在转账操作中，无论转账是否成功，账户的总金额应该保持不变。
- **隔离性（Isolation）**：多个事务并发执行时，一个事务的执行不应该影响其他事务的执行。不同的隔离级别可以控制事务之间的可见性和干扰程度。
- **持久性（Durability）**：事务一旦提交，其对数据库的更改应该永久保存，即使系统出现故障也不会丢失。
### 2 Spring 事务管理的方式
Spring 提供了两种事务管理方式：编程式事务管理和声明式事务管理。
#### 2.1 编程式事务管理
编程式事务管理需要在代码中显式地编写事务管理的代码，如开启事务、提交事务或回滚事务。这种方式灵活性高，但代码侵入性强，维护成本较高。通常使用 `TransactionTemplate` 或 `PlatformTransactionManager` 来实现。
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.TransactionDefinition;
import org.springframework.transaction.TransactionStatus;
import org.springframework.transaction.support.DefaultTransactionDefinition;

@Service
public class ProgrammaticTransactionService {

    @Autowired
    private PlatformTransactionManager transactionManager;

    public void doSomething() {
        TransactionDefinition def = new DefaultTransactionDefinition();
        TransactionStatus status = transactionManager.getTransaction(def);
        try {
            // 业务逻辑代码
            // ...
            transactionManager.commit(status);
        } catch (Exception e) {
            transactionManager.rollback(status);
        }
    }
}
```
#### 2.2 声明式事务管理
声明式事务管理是通过 AOP（面向切面编程）实现的，它将事务管理代码与业务逻辑代码分离，降低了代码的耦合度，提高了可维护性。声明式事务管理可以通过 XML 配置或注解的方式实现，常用的注解是 `@Transactional`。
```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class DeclarativeTransactionService {
    @Transactional
    public void doSomething() {
        // 业务逻辑代码
        // ...
    }
}
```
### 3 事务的传播行为
事务的传播行为定义了在多个事务方法相互调用时，事务如何进行传播。Spring 定义了 7 种事务传播行为：
- **REQUIRED（默认）**：如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。
- **SUPPORTS**：如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务方式执行。
- **MANDATORY**：如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。
- **NESTED**：如果当前存在事务，则在嵌套事务内执行；如果当前没有事务，则创建一个新的事务。
- **REQUIRES_NEW**：创建一个新的事务，如果当前存在事务，则将当前事务挂起。
- **NOT_SUPPORTED**：以非事务方式执行操作，如果当前存在事务，则将当前事务挂起。
- **NEVER**：以非事务方式执行操作，如果当前存在事务，则抛出异常。
### 4 事务的隔离级别
事务的隔离级别定义了一个事务对其他事务的可见性。Spring 支持以下 5 种隔离级别：
- **ISOLATION_DEFAULT**：使用数据库的默认隔离级别。
- **ISOLATION_READ_UNCOMMITTED**：允许读取未提交的数据，可能会导致脏读、不可重复读和幻读问题。
- **ISOLATION_READ_COMMITTED**：只允许读取已提交的数据，可以避免脏读，但可能会出现不可重复读和幻读问题。
- **ISOLATION_REPEATABLE_READ**：确保在同一个事务中多次读取同一数据的结果是一致的，可以避免脏读和不可重复读，但可能会出现幻读问题。
- **ISOLATION_SERIALIZABLE**：最高的隔离级别，所有事务依次串行执行，可以避免脏读、不可重复读和幻读问题，但会影响并发性能。
### 5 事务的使用示例
以下是一个使用 `@Transactional` 注解进行声明式事务管理的示例：
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    @Transactional(propagation = Propagation.REQUIRED, isolation = Isolation.READ_COMMITTED)
    public void transferMoney(long fromUserId, long toUserId, double amount) {
        // 从源账户扣除金额
        userRepository.withdraw(fromUserId, amount);

        // 模拟异常
        if (amount > 1000) {
            throw new RuntimeException("Transfer amount exceeds limit");
        }

        // 向目标账户存入金额
        userRepository.deposit(toUserId, amount);
    }
}
```
在上述示例中，`transferMoney` 方法使用 `@Transactional` 注解进行事务管理，指定了事务的传播行为为 `PROPAGATION_REQUIRED`，隔离级别为 `ISOLATION_READ_COMMITTED`。如果在方法执行过程中抛出异常，事务将自动回滚，确保数据的一致性。
综上所述，Spring 的事务机制为开发者提供了强大而灵活的事务管理能力，通过编程式和声明式事务管理方式、事务的传播行为和隔离级别等特性，开发者可以根据实际需求灵活地管理事务，保证数据的一致性和完整性。
### 6 源码分析
Spring 的事务机制是通过 AOP 实现的，当给方法加上 @Transaction 注解后，AOP 的执行链路中会多一个 TransactionInterceptor 拦截器。
```java
@Override
@Nullable
public Object invoke(MethodInvocation invocation) throws Throwable {

	Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

	// 通过事务执行方法
	return invokeWithinTransaction(invocation.getMethod(), targetClass, new CoroutinesInvocationCallback() {
		@Override
		@Nullable
		public Object proceedWithInvocation() throws Throwable {
			return invocation.proceed();
		}
		@Override
		public Object getTarget() {
			return invocation.getThis();
		}
		@Override
		public Object[] getArguments() {
			return invocation.getArguments();
		}
	});
}
```

具体的执行事务的方法为：
```java
@Nullable
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
	final InvocationCallback invocation) throws Throwable {
	//  1）获取事务源和事务管理器
	TransactionAttributeSource tas = getTransactionAttributeSource();
	final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
	final TransactionManager tm = determineTransactionManager(txAttr);
	
	if (this.reactiveAdapterRegistry != null && tm instanceof ReactiveTransactionManager rtm) {
			// 处理 Reactive 事务
		});
	}
	
	PlatformTransactionManager ptm = asPlatformTransactionManager(tm);
	final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);
	
	if (txAttr == null || !(ptm instanceof CallbackPreferringPlatformTransactionManager cpptm)) {
		// 开启一个标准的事务
		TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);
	
		Object retVal;
		try {
			// 实际执行方法
			retVal = invocation.proceedWithInvocation();
		}
		catch (Throwable ex) {
			// 抛出异常则检测是否需要回滚
			completeTransactionAfterThrowing(txInfo, ex);
			throw ex;
		}
		finally {
			cleanupTransactionInfo(txInfo);
		}
	
		if (retVal != null && txAttr != null) {
			TransactionStatus status = txInfo.getTransactionStatus();
			if (status != null) {
				if (retVal instanceof Future<?> future && future.isDone()) {
					// 检查返回值，处理 Future 和 Vavr
				}
				else if (vavrPresent && VavrDelegate.isVavrTry(retVal)) {
					// 在 Vavr 失败匹配回滚规则的情况下设置仅回滚
					retVal = VavrDelegate.evaluateTryFailure(retVal, txAttr, status);
				}
			}
		}
	
		commitTransactionAfterReturning(txInfo);
		return retVal;
	}
	
	else {
		// CallbackPreferringPlatformTransactionManager 处理	
	}
}
```
通过 `invocation.proceedWithInvocation()` 执行事务方法，在事务方法中抛出的异常会被 catch 住，并最终执行 `completeTransactionAfterThrowing` 方法：
```java
	protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
		if (txInfo != null && txInfo.getTransactionStatus() != null) {
		// 检测当前抛出的异常是否需要被回滚
			if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
				try {
					txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
				}
				catch (TransactionSystemException ex2) {
					logger.error("Application exception overridden by rollback exception", ex);
					ex2.initApplicationException(ex);
					throw ex2;
				}
				catch (RuntimeException | Error ex2) {
					logger.error("Application exception overridden by rollback exception", ex);
					throw ex2;
				}
			}
			else {
				// 当前抛出的异常不需要背回滚
				// 如果当前事务状态是否设置为只能回滚，也会执行回滚逻辑
				try {
					txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
				}
				catch (TransactionSystemException ex2) {
					logger.error("Application exception overridden by commit exception", ex);
					ex2.initApplicationException(ex);
					throw ex2;
				}
				catch (RuntimeException | Error ex2) {
					logger.error("Application exception overridden by commit exception", ex);
					throw ex2;
				}
			}
		}
	}
```
### 7 注解失效情况总结
#### 1）在类的内部调用
```java
@Service
public class MyService {

    @Autowired
    private MyMapper myMapper;

    @Transactional(propagation = Propagation.REQUIRED)
    public void methodA() {
        // Some operations
        methodB(); // 直接调用 methodB
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void methodB() {
        // Some operations
    }
    
}
```
当 `methodA` 开启了事务并调用了 `methodB`，即使 `methodB` 的事务注解没有生效，`methodB` 中的 SQL 语句仍然会在 `methodA` 的事务上下文中执行。这意味着 `methodB` 的操作会被包含在 `methodA` 的事务中，因此如果 `methodA` 事务回滚，`methodB` 的操作也会回滚，反之亦然。
原因：事务代理是通过动态代理实现的，如果是直接在类内部去调用另一个方法，则不会通过代理对象去执行这个方法，而是直接调用，这就会导致另一个方法上的注解失效。
#### 2）方法为非 public 修饰
Spring AOP 提供了两种主要的代理方式：
- **JDK 动态代理**：基于接口创建代理，只能代理实现接口的 `public` 方法。
- **CGLIB 代理**：基于子类创建代理，可以代理类的 `public` 和 `protected` 方法。
虽然 CGLIB 代理方法能够代理 protected 的方法，但是 Spring 在默认配置下只会对 `public` 方法应用事务管理，以保持一致性和简单性。
#### 3）方法用 final 修饰
将一个方法定义成 final 意味着该方法是不能被重写的
- 当使用 JDK 动态代理的时候，不用考虑这一块，因为接口方法不能被 final 修饰。
- 当使用 CGLIB 代理的时候，因为 CGLIB 的代理对象是通过继承被代理类并重写方法来创建的。如果被代理类中的方法被声明为 `final`，CGLIB将无法通过继承来重写这个方法，因为`final`方法在子类中不可覆盖。这样，CGLIB就无法生成有效的代理对象，因为代理的核心机制就是方法的重写。
同样，将方法定义为 static 的时候也会因为无法代理而导致注解失效
这是因为Spring AOP的动态代理机制（无论是JDK的基于接口的代理还是CGLIB的基于类的代理）都是基于对象实例的。代理对象是在**运行时**创建的，并且它们代理的是对象的行为，而不是类的行为。
#### 4）代理类未被 Spring 管理
比如忘记了@Controller、@Service、@Component、@Repository等注解，这就导致了调用的对象根本不是代理对象。
#### 5）自己捕获并且处理了异常
```java
catch (Throwable ex) {
	// target invocation exception
	completeTransactionAfterThrowing(txInfo, ex);
	throw ex;
}
```
执行回滚的代码是当代理方法捕获到了异常的时候才会执行的，所以当我们手动捕获并且处理了异常，就不会触发事务的回滚；对于一些不需要回滚的异常，我们可以通过配置注解中的 **`rollbackFor`** 的内容，当执行 `completeTransactionAfterThrowing` 方法的时候，会检测抛出的异常是否是我们指定的异常，如果不是，则不会触发回滚。