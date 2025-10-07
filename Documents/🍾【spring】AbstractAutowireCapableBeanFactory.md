---
created: 2025-09-27 22:34:06
updated: 2025-09-27 22:34:06
---
A-- This is the text! ---B
# 继承关系图
AbstractAutowireCapableBeanFactory，抽象的、具有自动装配能力的 Bean 工厂。
![[AbstractAutowireCapableBeanFactory 继承关系图.png]]
`AbstractAutowireCapableBeanFactory` 继承了 [[🍊【spring】AbstractBeanFactory]]，拥有了这些能力：单例 Bean 的缓存、对单例原型 Bean 对支持、对 Factory Bean 对支持、别名管理、Disposable Bean 的支持。
除此之外，它还实现了 `AutowirecapableBeanFactory` 接口，这个接口定义了一系列 `createBean` 接口，例如：
```java
Object createBean(Class<?> beanClass, int autowireMode, boolean dependencyCheck) throws BeansException;
```
其中 `autowireMode` 是自动注入的类型，常见的有这几种：
```java
int AUTOWIRE_NO = 0;  // 无自动注入
int AUTOWIRE_BY_NAME = 1;  // 通过名称自动注入
int AUTOWIRE_BY_TYPE = 2;  // 通过类型自动注入
int AUTOWIRE_CONSTRUCTOR = 3; // 通过构造函数自动注入
```
# 默认的 Bean 创建
类中有私有属性，名为 `private InstantiationStrategy instantiationStrategy;`，表示了实例化 Bean 的默认方式。
它有两种选择，分别是通过 java 反射创建、通过 CgLib；一般来说，除了不支持 CgLib 的某些运行环境来说，默认的实例化方式都是 CgLib：
```java
public AbstractAutowireCapableBeanFactory() {  
    super();  
    ignoreDependencyInterface(BeanNameAware.class);  
    ignoreDependencyInterface(BeanFactoryAware.class);  
    ignoreDependencyInterface(BeanClassLoaderAware.class);  
    if (NativeDetector.inNativeImage()) {  
       this.instantiationStrategy = new SimpleInstantiationStrategy();  
    }  A-- This is the text! ---BA-- This is the text! ---BA-- This is the text! ---BA-- This is the text! ---B




    else {  
       this.instantiationStrategy = new CglibSubclassingInstantiationStrategy();  
    }  
}
```
且实现了 `AbstractBeanFactory` 中创建 Bean 的重要方法：
```java
protected abstract Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)  
       throws BeanCreationException;
```
这个方法在 `AbstractBeanFactory` 的 `doGetBean` 的方法中被调用，作为 `Bean` 创建的方式；其最终会调用到 `doCreateBean` 方法：
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
  
    // Allow post-processors to modify the merged bean definition.  
    synchronized (mbd.postProcessingLock) {  
       if (!mbd.postProcessed) {  
          try {  
             applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);  
          }  
          catch (Throwable ex) {  
             throw new BeanCreationException(mbd.getResourceDescription(), beanName,  
                   "Post-processing of merged bean definition failed", ex);  
          }  
          mbd.postProcessed = true;  
       }  
    }  
  
    // Eagerly cache singletons to be able to resolve circular references  
    // even when triggered by lifecycle interfaces like BeanFactoryAware.    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&  
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
       if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {  
          throw (BeanCreationException) ex;  
       }  
       else {  
          throw new BeanCreationException(  
                mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);  
       }  
    }  
  
    if (earlySingletonExposure) {  
       Object earlySingletonReference = getSingleton(beanName, false);  
       if (earlySingletonReference != null) {  
          if (exposedObject == bean) {  
             exposedObject = earlySingletonReference;  
          }  
          else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {  
             String[] dependentBeans = getDependentBeans(beanName);  
             Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);  
             for (String dependentBean : dependentBeans) {  
                if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {  
                   actualDependentBeans.add(dependentBean);  
                }  
             }  
             if (!actualDependentBeans.isEmpty()) {  
                throw new BeanCurrentlyInCreationException(beanName,  
                      "Bean with name '" + beanName + "' has been injected into other beans [" +  
                      StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +  
                      "] in its raw version as part of a circular reference, but has eventually been " +  
                      "wrapped. This means that said other beans do not use the final version of the " +  
                      "bean. This is often the result of over-eager type matching - consider using " +  
                      "'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");  
             }  
          }  
       }  
    }  
  
    // Register bean as disposable.  
    try {  
       registerDisposableBeanIfNecessary(beanName, bean, mbd);  
    }  
    catch (BeanDefinitionValidationException ex) {  
       throw new BeanCreationException(  
             mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);  
    }  
  
    return exposedObject;  
}
```
实例化 bean：
	检查 `mbd.isSingleton()` 是否为单例 bean，如果是则从 `factoryBeanInstanceCache` 中移除并获取实例。
	如果缓存中没有实例，则调用 `createBeanInstance` 创建新的 Bean 实例。
	应用合并后的 bean 定义后处理器：
	使用同步锁 mbd.postProcessingLock 确保线程安全。
	调用 applyMergedBeanDefinitionPostProcessors 应用后处理器。
	标记 mbd.postProcessed 为 true 表示已处理。
提前暴露单例 bean：
	检查 mbd.isSingleton() 和 this.allowCircularReferences 是否允许提前暴露单例。
	如果允许且当前 bean 正在创建中，则调用 addSingletonFactory 提前暴露单例。
初始化 bean：
	调用 `populateBean` 填充属性值。
	调用 `initializeBean` 初始化 bean，包括调用初始化方法和后处理器。
处理循环依赖：
	如果存在循环依赖，检查并处理依赖关系，确保其他 bean 使用的是最终版本的 bean。
注册销毁方法：
	注册 bean 的销毁方法，以便在容器关闭时调用。
# 依赖注入
实现了一系列自动注入方法，例如：
```java
	@Override
	public Object autowire(Class<?> beanClass, int autowireMode, boolean dependencyCheck) throws BeansException {
		// Use non-singleton bean definition, to avoid registering bean as dependent bean.
		RootBeanDefinition bd = new RootBeanDefinition(beanClass, autowireMode, dependencyCheck);
		bd.setScope(SCOPE_PROTOTYPE);
		if (bd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR) {
			return autowireConstructor(beanClass.getName(), bd, null, null).getWrappedInstance();
		}
		else {
			Object bean;
			if (System.getSecurityManager() != null) {
				bean = AccessController.doPrivileged(
						(PrivilegedAction<Object>) () -> getInstantiationStrategy().instantiate(bd, null, this),
						getAccessControlContext());
			}
			else {
				bean = getInstantiationStrategy().instantiate(bd, null, this);
			}
			populateBean(beanClass.getName(), bd, new BeanWrapperImpl(bean));
			return bean;
		}
	}

```
# Bean 初始化方法的调用 
实现了 InitializeBean 方法：
```java
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {  
    if (System.getSecurityManager() != null) {  
       AccessController.doPrivileged((PrivilegedAction<Object>) () -> {  
          invokeAwareMethods(beanName, bean);  
          return null;  
       }, getAccessControlContext());  
    }  
    else {  
       invokeAwareMethods(beanName, bean);  
    }  
  
    Object wrappedBean = bean;  
    if (mbd == null || !mbd.isSynthetic()) {  
       wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);  
    }  
  
    try {  
       invokeInitMethods(beanName, wrappedBean, mbd);  
    }  
    catch (Throwable ex) {  
       throw new BeanCreationException(  
             (mbd != null ? mbd.getResourceDescription() : null),  
             beanName, "Invocation of init method failed", ex);  
    }  
    if (mbd == null || !mbd.isSynthetic()) {  
       wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);  
    }  
  
    return wrappedBean;  
}
```
首先，initializeBean 会调用 invokeAwareMethods 方法，检查并注入 Spring 特定的回调接口。如果 Bean 实现了以下接口，Spring 会通过该方法自动注入相关依赖：
	• **BeanNameAware**：设置 Bean 的名称。
	• **BeanClassLoaderAware**：设置 Bean 的类加载器。
	• **BeanFactoryAware**：设置 BeanFactory 实例，以便 Bean 在初始化时可以使用 BeanFactory。
调用 BeanPostProcessor 的 postProcessBeforeInitialization 方法。这是一个扩展点，允许开发者在 Bean 的初始化方法调用之前，对 Bean 实例进行自定义处理。所有实现了 BeanPostProcessor 接口的类都会参与这个过程。
在 Spring 中，Bean 可以定义自定义的初始化方法，initializeBean 方法会调用该初始化方法。如果 Bean 使用了以下方式定义初始化方法，Spring 会在此阶段调用它们：
	• **实现** InitializingBean **接口**：如果 Bean 实现了 InitializingBean 接口，则会调用其 afterPropertiesSet 方法。
	• **定义** init-method **属性**：在 XML 配置中指定 init-method，或在 @Bean 注解的 initMethod 属性中指定方法名称。
	• **使用** @PostConstruct **注解**：如果类中包含带 @PostConstruct 注解的方法，也会在此阶段执行。
调用 BeanPostProcessor 的 postProcessAfterInitialization 方法。这是一个扩展点，允许开发者在 Bean 完成初始化之后进行自定义处理。通常会在这里应用 AOP 代理，将 Bean 包装成代理对象。
