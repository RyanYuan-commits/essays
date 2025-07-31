## 1 BeanFactoryPostProcessor 及其子接口

### 1.1 BeanFactoryPostProcessor

```java
public interface BeanFactoryPostProcessor {

	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

}
```

**接口定义**: 在应用上下文的标准初始化之后, 修改其内部的 bean 工厂. 此时所有 bean 定义都已加载, 但尚未实例化任何 bean. 这允许覆盖或添加属性, 甚至对 eager-initializing Bean 也是如此.

**使用场景**: 对 `BeanDefinition` 做修改和删除. 比如 Dubbo 中对 `@DubboReference` 注解的处理就使用到 `BeanFactoryPostProcessor` 的能力, 在 `ReferenceAnnotationBeanPostProcessor` 中, 对 `@DubboReference` 标注的属性中使用到的接口基于 `ReferenceBean` 进行注册.

注入的 `ConfigurableListableBeanFactory` 的类图为:

![[ConfigurableListableBeanFactory类图.png]]

这个接口是没有实现 `BeanDefinitionRegistry` 接口的, 所以通过这个接口只能对现有的 `BeanDefinition` 进行修改, 而无法创建新的 `BeanDefinition`.

### 1.2 BeanDefinitionRegistryPostProcessor

```java
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {

	void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;

}
```

**接口定义**: 在标准初始化之后修改应用上下文的内部 bean 定义注册表. 所有常规 bean 定义都已加载, 但尚未实例化任何 bean. 这允许在下一个后处理阶段开始之前**添加更多**的 bean 定义.

继承了 `BeanFactoryPostProcessor` 接口, 并在其基础上拓展了创建新 `BeanDefinition` 的能力.

### 1.3 执行流程

执行位置: `PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors`

在这个方法负责调用 Spring 容器中的 `BeanFactoryPostProcessor`, 分阶段处理, 优先执行实现了 `BeanDefinitionRegistryPostProcessor` 接口的类, 确保后续 `BeanFactoryPostProcessor` 处理的时候, `BeanDefinition` 是全量的; 

在处理这些 `BeanFactoryPostProcessor` 的时候, 对实现了 `PriorityOrdered` 和 `Ordered` 排序接口的类进行特殊的排序处理.

## 2 Aware 接口及其子接口

### 2.1 ApplicationContextAware

```java
public interface ApplicationContextAware extends Aware {

    void setApplicationContext(ApplicationContext applicationContext) throws BeansException;  
    
}
```

**接口定义**: 设置该对象运行的 `ApplicationContext`. 通常此调用将用于初始化对象;在填充普通 bean 属性之后，但在 init 回调（例如 `InitializingBean#afterPropertiesSet()` 或自定义 `init-method`）之前调用。在 `ResourceLoaderAware#setResourceLoader`、`ApplicationEventPublisherAware#setApplicationEventPublisher` 和 `MessageSourceAware`（如果适用）之后调用。

**使用场景**: 最常用的 Aware 接口, 实现这个接口可以让 Bean 感知到外部的 `ApplicationContext`, 从而可以使用上下文来动态获取 Bean, 发布或监听自定义事件, 访问 Spring 基础设施等; 
#### ApplicationContextAwareProcessor

但是, Bean 的属性填充和初始化是在 `BeanFactory` 中进行的, 但是实际执行的过程中, `ApplicaitonContext` 和 `BeanFactory` 是依赖关系, 所以在 `BeanFactory` 无法感知外部的 `ApplicaitonContext` 对象.

所以, 必须要有一个类在进入 `BeanFactory` 的领域之前保留下对 `ApplicationContext` 的引用, 这个类就是 `ApplicationContextAwareProcessor`.

```java
class ApplicationContextAwareProcessor implements BeanPostProcessor{}
```

`ApplicationContextAwareProcessor` 实现了 `BeanPostProcessor` 接口的 `postProcessBeforeInitialization` 方法, 在 Bean 初始化之前对 Bean 进行修改, `ApplicationContext` 就是在这里被注入的.

```java
private void invokeAwareInterfaces(Object bean) {  
    if (bean instanceof EnvironmentAware) {  
       ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());  
    }  
    if (bean instanceof EmbeddedValueResolverAware) {  
       ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);  
    }  
    if (bean instanceof ResourceLoaderAware) {  
       ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);  
    }  
    if (bean instanceof ApplicationEventPublisherAware) {  
       ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);  
    }  
    if (bean instanceof MessageSourceAware) {  
       ((MessageSourceAware) bean).setMessageSource(this.applicationContext);  
    }  
    if (bean instanceof ApplicationStartupAware) {  
       ((ApplicationStartupAware) bean).setApplicationStartup(this.applicationContext.getApplicationStartup());  
    }  
    if (bean instanceof ApplicationContextAware) {  
       ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);  
    }  
}
```

#### 其他相关 Aware

在 `ApplicationContextAwareProcessor` 的 `invokeAwareInterfaces` 中, 还调用了其他 `Aware` 接口的 `set` 方法, 这里一并提及.

`EnvironmentAware`: 应用运行环境感知类, 运行环境一般指的是配置文件 profiles 和 各种属性 properties, 实现这个接口能让 Bean 获取到当前的 `Environment` 对象.

`EmbeddedValueResolverAware`: 注入字符串值解析器, 解析占位符和表达式，如 `${property}`.

`ResourceLoaderAware`: 注入资源加载器, 加载 classpath 或文件系统中的资源.

`ApplicationEventPublisherAware`: 注入事件发布器, 如果想要发布事件但是又不想依赖于整个 `ApplicationContext` 的时候, 可以实现这个接口.

`MessageSourceAware`: 注入消息源, 支持国际化消息解析.

### 2.2 BeanFactoryAware

```java
public interface BeanFactoryAware extends Aware {  

    void setBeanFactory(BeanFactory beanFactory) throws BeansException;  

}
```

**接口定义**:  回调方法，向 bean 实例提供其所属的工厂。在填充普通 bean 属性之后，但在初始化回调（例如 `InitializingBean#afterPropertiesSet()` 或自定义 `init-method` 之前调用。

**使用场景**: 需要直接与底层的 `BeanFactory` 交互，进行更低级别的 bean 管理，比如动态获取原型（`prototype`）作用域的 bean 实例。

#### 注入位置

在 `AbstractAutowireCapableBeanFactory#initializeBean` 中执行注入方法: 

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
  
    // ......
}

private void invokeAwareMethods(String beanName, Object bean) {  
    if (bean instanceof Aware) {  
       if (bean instanceof BeanNameAware) {  
          // ......
       }  
       if (bean instanceof BeanClassLoaderAware) {  
          // ...
       }  
       if (bean instanceof BeanFactoryAware) {  
          ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);  
       }  
    }  
}
```

#### 其他 Aware

在这里还能看到其他的 Aware, 这里也一并提及:

`BeanNameAware`: 让当前 Bean 感知到自己的 Bean Name, name 是工厂中实际使用的 bean 名称，可能与最初指定的名称不同, 例如内部 bean 名称可能会通过添加 `"#..."` 后缀来确保唯一性。如需提取原始名称(无后缀), 可使用`BeanFactoryUtils#originalBeanName(String)` 方法.

`BeanClassLoaderAware`: 让当前 Bean 感知到加载自己的类加载器.

## 3 BeanPostProcessor

```java
public interface BeanPostProcessor {  

    default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {  
       return bean;  
    }  
  
    @Nullable  
    default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {  
       return bean;  
    }  
  
}
```

**接口定义**: 允许在 Bean 初始化前后对 Bean 进行修改, 此时的 Bean 已经完成了实例化.

**使用场景**: 对已经实例化好的 Bean 进行属性修改, 动态代理等; 比如上面提到的 `ApplicationContextAwareProcessor` 就是 `BeanPostProcessor` 的实现类.

---

### 3.1 调用位置

`AbstractAutowireCapableBeanFactory#initializeBean` 方法:

```java
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {  
    // ......
  
    Object wrappedBean = bean;  
    if (mbd == null || !mbd.isSynthetic()) {  
       wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);  
    }  
  
    try {  
       invokeInitMethods(beanName, wrappedBean, mbd);  
    }  
    catch (Throwable ex) {  
       // ......
    }  
    if (mbd == null || !mbd.isSynthetic()) {  
       wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);  
    }  
  
    return wrappedBean;  
}
```

`applyBeanPostProcessorsBeforeInitialization` 在实例化方法 `invokeInitMethods` 之前调用, 而 `applyBeanPostProcessorsAfterInitialization` 在其之后调用.

这两个方法都是通过 `getBeanPostProcessors()` 方法获取所有的 `BeanPostProcessor` 遍历执行.