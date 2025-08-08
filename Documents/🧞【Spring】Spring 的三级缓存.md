### 1 Infra
所谓三级缓存，实际上就是三个 Map 对象，存储着 bean 名称和缓存内容的对应关系，这三个 Map 均为默认单例 Bean 注册器 `DefaultSingletonBeanRegistry` 的属性，除此之外还有一个 Set 存储着当前正在创建的 Bean 对象，方便快速确认 Bean 是否正在被创建，下面是它们的具体定义：
#### 1.1 singletonFactories 三级缓存
```java
/** Cache of singleton factories: bean name to ObjectFactory. */  
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
```
它的缓存内容是一个获取早期 Bean 的 Lambda 表达式（生产 Bean 的工厂），会根据 Bean 是否被代理来返回普通 Bean 或是代理 Bean。
#### 1.2 earlySingletonObject 二级缓存
```java
/** Cache of early singleton objects: bean name to bean instance. */
private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);
```
存放了中间状态的 Bean，也就是未执行依赖注入的 代理对象 / 普通对象。
#### 1.3 singletonObjects 一级缓存
```java
/** Cache of singleton objects: bean name to bean instance. */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
```
存储已经创建完成的对象，可以被其他 Bean 作为依赖注入使用。
#### 1.4 singletonsCurrentlyInCreation
```java
/** Names of beans that are currently in creation. */
private final Set<String> singletonsCurrentlyInCreation =
		Collections.newSetFromMap(new ConcurrentHashMap<>(16));
```
### 2 Process
三级缓存是为了解决单例 Bean 的 **循环依赖** 问题而设计的，要看它们作用的详细步骤就要从获取 Bean 的方法开始看。
#### 2.1 检查 Bean 是否在缓存中
首先定位到抽象 Bean 工厂 `org.springframework.beans.factory.support.AbstractBeanFactory` 的 `doGetBean` 方法：
```java
protected <T> T doGetBean(
			String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
			throws BeansException {

		String beanName = transformedBeanName(name);
		Object beanInstance;

		// Eagerly check singleton cache for manually registered singletons.
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isTraceEnabled()) {
				// 打印日志
			}
			beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}
		else {
			// ......
		}
		return adaptBeanInstance(name, beanInstance, requiredType);
	}
```
Spring 容器初始化中，创建单例 Bean 的过程其实就是不断调用 getBean 方法的过程，这个方法承包了 Bean 对象的创建和获取。
首先，其会调用 `getSingleton` 方法来尝试获取指定名称的 Bean 对象，Spring 中的 Factory 通过继承 `FactoryBeanRegistrySupport` 类来获得了对 Bean 的注册存储和获取能力，`getSingleton` 执行的就是其从 `DefaultSingletonBeanRegistry` 继承来的方法。
```java
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
	// Quick check for existing instance without full singleton lock
	Object singletonObject = this.singletonObjects.get(beanName);
	if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
		// ......
	}
	return singletonObject;
}
```
`getSingleton` 会先从 `singletonObjects` **一级缓存** 中读取 Bean，如果读取成功，直接返回。
#### 2.2 检查二级缓存
```java
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
	// Quick check for existing instance without full singleton lock
	Object singletonObject = this.singletonObjects.get(beanName);
	if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
		singletonObject = this.earlySingletonObjects.get(beanName);
		if (singletonObject == null && allowEarlyReference) {
			// ......
		}
	}
	return singletonObject;
}
```
在上面查询一级缓存后，Spring 会继续尝试从 `earlySingletonObjects` 二级缓存中去获取 Bean 对象，此时如果获取成功也会直接返回。

前面我们提到过，二级缓存中存放的是 "中间状态的 Bean"，在一个简单的 A -> B -> A 的循环依赖中，需要寻找一个出口，才能破除这个依赖循环，二级缓存就充当了一个 **出口** 的角色，当创建 A 的时候需要 B 对象的创建，此时 A 就处于一个 "中间状态"，在创建 B 的时候，发现其依赖 A，此时将这个中间状态的 A 注入 B，就破除了这个循环。

#### 2.3 检查三级缓存
```java
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
	// Quick check for existing instance without full singleton lock
	Object singletonObject = this.singletonObjects.get(beanName);
	if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
		singletonObject = this.earlySingletonObjects.get(beanName);
		if (singletonObject == null && allowEarlyReference) {
			synchronized (this.singletonObjects) {
				// Consistent creation of early reference within full singleton lock
				singletonObject = this.singletonObjects.get(beanName);
				if (singletonObject == null) {
					singletonObject = this.earlySingletonObjects.get(beanName);
					if (singletonObject == null) {
						ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
						if (singletonFactory != null) {
							singletonObject = singletonFactory.getObject();
							this.earlySingletonObjects.put(beanName, singletonObject);
							this.singletonFactories.remove(beanName);
						}
					}
				}
			}
		}
	}
	return singletonObject;
}
```
- 为了防止并发状态下 Bean 的重复创建，需要先上一个锁，`synchronized (this.singletonObjects)`，在 `synchronized` 代码块中继续检查 Bean 的创建状态。
- 此时 Spring 会从三级缓存 `singletonFactories` 中获取创建 Bean 的工厂，执行其中的 `getObject()` 方法来创建一个原始 Bean 对象并返回；但如果发现 `singletonFactories` 也不存有该 Bean 的创建方法，`getSingleton` 最终会返回 null。

#### 2.4 创建 Bean
首次执行某个 doGetBean 方法时，getSingleton 方法会返回 null，此时就需要执行 Bean 对象的创建，最终会调用到 doCreateBean 方法：
```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
		throws BeanCreationException {

	// Instantiate the bean.
	BeanWrapper instanceWrapper = null;
	if (mbd.isSingleton()) {
		instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
	}
	if (instanceWrapper == null) {
		instanceWrapper = createBeanInstance(beanName, mbd, args);
	}
	Object bean = instanceWrapper.getWrappedInstance();
	Class<?> beanType = instanceWrapper.getWrappedClass();
	if (beanType != NullBean.class) {
		mbd.resolvedTargetType = beanType;
	}

	// ... BeanPostProcessor 执行

	// 缓存单例，以便能够解析循环引用
	boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
			isSingletonCurrentlyInCreation(beanName));
	if (earlySingletonExposure) {
		if (logger.isTraceEnabled()) {
			logger.trace("Eagerly caching bean '" + beanName +
					"' to allow for resolving potential circular references");
		}
		addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
	}

	// Initialize the bean instance.
	Object exposedObject = bean;
	try {
		populateBean(beanName, mbd, instanceWrapper);
		exposedObject = initializeBean(beanName, exposedObject, mbd);
	}
	catch (Throwable ex) {
		// ... 异常处理
	}

	// ......

	// ...注册 Disposable Bean

	return exposedObject;
}
```
上面方法有这么几个关键步骤需要关注：
- `instanceWrapper = createBeanInstance(beanName, mbd, args);` 创建 Bean 对象实例
- `addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));` 将获取中间状态 Bean 的方法作为 Value 存入到 **三级缓存** 中。
- `populateBean(beanName, mbd, instanceWrapper);` 填充 Bean
当执行 populateBean 中，如果 Bean A 需要填充 Bean B，此时将需要调用 getBean(B) 方法去创建 Bean B，如果 Bean B 又依赖了 Bean A，此时就会通过 getBean(A) 方法获取 Bean A，此时就会回到前面 2.1 中的步骤，去检查 Bean 是否位于缓存中；
此时，由于在创建 Bean 的时候，Spring 已经将获取中间状态 Bean 的方法存入三级缓存中：
```java
singletonObject = singletonFactory.getObject();
this.earlySingletonObjects.put(beanName, singletonObject);
this.singletonFactories.remove(beanName);
```
执行三级缓存中的对象创建方法，然后将其存放入二级缓存中并返回这个中间状态 Bean。

此时这个循环就被打破了，Bean B 的 getBean(A) 方法调用栈最终得到返回，B 可以完成它的创建，并最终将 Bean 存放到一级缓存中：
```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
	Assert.notNull(beanName, "Bean name must not be null");
	synchronized (this.singletonObjects) {
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null) {
			// ......
			if (newSingleton) {
				addSingleton(beanName, singletonObject);
			}
		}
		return singletonObject;
	}
}
```
当 B 对象创建完毕，也就意味着 A -> B -> A 的循环依赖被打破了，A 的 getBean(B) 的方法调用栈也会返回，此时两个对象都创建完成。
### 3 Question
#### 3.1 三级缓存的作用是什么？
在创建 Bean 的时候，向三级缓存中存放的是获取中间状态的 Bean 的方法，这不是多次一举吗？为什么此时不直接创建然后将中间状态的 Bean 放入二级缓存？
答：为了 AOP 代理对象的延迟创建。
Spring 根据是否对 Bean 执行了 AOP 代理，分为普通 Bean 和 代理 Bean，而代理 Bean 的构建是在后面的 `initializeBean` 方法中执行的，顺序应该是 实例化 -> 填充依赖 -> 初始化 Bean，Spring 为了不打破这个既定顺序，使用了三级缓存，执行的是 AOP 模块提供的 `getEarlyBeanReference` 方法，而非是正常执行下的 `postProcessAfterInitialization` 方法。
#### 3.2 为什么构造器和原型 Bean 不支持循环依赖？
因为三级缓存是在实例化后再作用的，这两种方式创建的 Bean 没有进行实例化。