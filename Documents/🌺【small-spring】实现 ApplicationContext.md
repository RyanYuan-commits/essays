# 目标
果你在自己的实际工作中开发过基于 Spring 的技术组件，或者学习过关于 [SpringBoot 中间件设计和开发](https://juejin.cn/book/6940996508632219689)等内容。那么你一定会继承或者实现了 Spring 对外暴露的类或接口，在接口的实现中获取了 BeanFactory 以及 Bean 对象的获取等内容，并对这些内容做一些操作，例如：修改 Bean 的信息，添加日志打印、处理数据库路由对数据源的切换、给 RPC 服务连接注册中心等。

在对容器中 Bean 的实例化过程添加扩展机制的同时，还需要把目前关于 Spring.xml 初始化和加载策略进行优化，因为我们不太可能让面向 Spring 本身开发的 `DefaultListableBeanFactory` 服务，直接给予用户使用。修改点如下：
![](https://bugstack.cn/assets/images/spring/spring-7-01.png)

比如看一下 Spring 是如何做的：
```java
public static void main(String[] args) {  
    ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");  
    UserService service = applicationContext.getBean("userService", UserService.class);  
    UserMapper mapper = applicationContext.getBean("userMapper", UserMapper.class);  
}
```
我们只需要构建一个 `ApplicationContext`，之后不管是 XML 解析、之后对 Bean 工厂的访问，都是通过这个接口进行的。
- `DefaultListableBeanFactory`、`XmlBeanDefinitionReader`，是我们在目前 Spring 框架中对于服务功能测试的使用方式，它能很好的体现出 Spring 是如何对 xml 加载以及注册Bean对象的操作过程，但这种方式是面向 Spring 本身的，还不具备一定的扩展性。
- 就像我们现在需要提供出一个可以在 Bean 初始化过程中，完成对 Bean 对象的扩展时，就很难做到自动化处理。所以我们要把 Bean 对象扩展机制功能和对 Spring 框架上下文的包装融合起来，对外提供完整的服务。

# 设计
为了能满足于在 Bean 对象从注册到实例化的过程中执行用户的自定义操作，就需要在 Bean 的定义和初始化过程中插入接口类，这个接口再有外部去实现自己需要的服务。那么在结合对 Spring 框架上下文的处理能力，就可以满足我们的目标需求了。整体设计结构如下图：
![[实现 ApplicationContext 架构图.png]]
- 满足于对 Bean 对象扩展的两个接口，其实也是 Spring 框架中非常具有重量级的两个接口：`BeanFactoryPostProcessor` 和 `BeanPostProcessor`，也几乎是大家在使用 Spring 框架额外新增开发自己组建需求的两个必备接口。
- BeanFactoryPostProcessor，是由 Spring 框架组建提供的容器扩展机制，允许在 Bean 对象注册后但未实例化之前，对 Bean 的定义信息 `BeanDefinition` 执行修改操作。
- BeanPostProcessor，也是 Spring 提供的扩展机制，不过 BeanPostProcessor 是在 Bean 对象实例化之后修改 Bean 对象，也可以替换 Bean 对象。这部分与后面要实现的 AOP 有着密切的关系。
- 同时如果只是添加这两个接口，不做任何包装，那么对于使用者来说还是非常麻烦的。我们希望于开发 Spring 的上下文操作类，把相应的 XML 加载 、注册、实例化以及新增的修改和扩展都融合进去，让 Spring 可以自动扫描到我们的新增服务，便于用户使用。
# 代码实现
## 工程结构与核心架构图
```java
small-spring-step-06
└── src
    ├── main
    │   └── java
    │       └── cn.bugstack.springframework
    │           ├── beans
    │           │   ├── factory
    │           │   │   ├── config
    │           │   │   │   ├── AutowireCapableBeanFactory.java
    │           │   │   │   ├── BeanDefinition.java
    │           │   │   │   ├── BeanFactoryPostProcessor.java
    │           │   │   │   ├── BeanPostProcessor.java
    │           │   │   │   ├── BeanReference.java
    │           │   │   │   ├── ConfigurableBeanFactory.java
    │           │   │   │   └── SingletonBeanRegistry.java
    │           │   │   ├── support
    │           │   │   │   ├── AbstractAutowireCapableBeanFactory.java
    │           │   │   │   ├── AbstractBeanDefinitionReader.java
    │           │   │   │   ├── AbstractBeanFactory.java
    │           │   │   │   ├── BeanDefinitionReader.java
    │           │   │   │   ├── BeanDefinitionRegistry.java
    │           │   │   │   ├── CglibSubclassingInstantiationStrategy.java
    │           │   │   │   ├── DefaultListableBeanFactory.java
    │           │   │   │   ├── DefaultSingletonBeanRegistry.java
    │           │   │   │   ├── InstantiationStrategy.java
    │           │   │   │   └── SimpleInstantiationStrategy.java  
    │           │   │   ├── support
    │           │   │   │   └── XmlBeanDefinitionReader.java
    │           │   │   ├── BeanFactory.java
    │           │   │   ├── ConfigurableListableBeanFactory.java
    │           │   │   ├── HierarchicalBeanFactory.java
    │           │   │   └── ListableBeanFactory.java
    │           │   ├── BeansException.java
    │           │   ├── PropertyValue.java
    │           │   └── PropertyValues.java 
    │           ├── context // 核心拓展部分
    │           │   ├── support
    │           │   │   ├── AbstractApplicationContext.java 
    │           │   │   ├── AbstractRefreshableApplicationContext.java 
    │           │   │   ├── AbstractXmlApplicationContext.java 
    │           │   │   └── ClassPathXmlApplicationContext.java 
    │           │   ├── ApplicationContext.java 
    │           │   └── ConfigurableApplicationContext.java
    │           ├── core.io
    │           │   ├── ClassPathResource.java 
    │           │   ├── DefaultResourceLoader.java 
    │           │   ├── FileSystemResource.java 
    │           │   ├── Resource.java 
    │           │   ├── ResourceLoader.java 
    │           │   └── UrlResource.java
    │           └── utils
    │               └── ClassUtils.java
    └── test
        └── java
            └── cn.bugstack.springframework.test
                ├── bean
                │   ├── UserDao.java
                │   └── UserService.java  
                ├── common
                │   ├── MyBeanFactoryPostProcessor.java
                │   └── MyBeanPostProcessor.java
                └── ApiTest.java
```
架构关系：
![[实现 ApplicationContext 类关系图.png|900]]
## 定义 BeanPostProcessor
```java
public interface BeanPostProcessor {

    /**
     * 在 Bean 对象执行初始化方法之前，执行此方法
     *
     * @param bean
     * @param beanName
     * @return
     * @throws BeansException
     */
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;

    /**
     * 在 Bean 对象执行初始化方法之后，执行此方法
     *
     * @param bean
     * @param beanName
     * @return
     * @throws BeansException
     */
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;

}
```
- 在 Spring 源码中有这样一段描述 `Factory hook that allows for custom modification of new bean instances,e.g. checking for marker interfaces or wrapping them with proxies.`也就是提供了修改新实例化 Bean 对象的扩展点。
- 另外此接口提供了两个方法：`postProcessBeforeInitialization` 用于在 Bean 对象执行初始化方法之前，执行此方法、`postProcessAfterInitialization`用于在 Bean 对象执行初始化方法之后，执行此方法。
这个方法会在 Bean 属性填充完成后调用，具体的时机是这样的：
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
    // 调用 postProcessBeforeInitialization 的时机
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
    // 调用 postProcessAfterInitialization 的时机
       wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);  
    }  
  
    return wrappedBean;  
}
```
## 定义上下文接口
```java
public interface ApplicationContext extends ListableBeanFactory {
}
```
- `context` 是本次实现应用上下文功能新增的服务包
- `ApplicationContext`，继承于 `ListableBeanFactory`，也就继承了关于 `BeanFactory` 方法，比如一些 `getBean` 的方法。另外 `ApplicationContext` 本身是 `Central` 接口，但目前还不需要添加一些获取ID和父类上下文，所以暂时没有接口方法的定义。

```java
public interface ConfigurableApplicationContext extends ApplicationContext {

    /**
     * 刷新容器
     *
     * @throws BeansException
     */
    void refresh() throws BeansException;

}
```
- `ConfigurableApplicationContext` 继承自 `ApplicationContext`，并提供了 `refresh` 这个核心方法。 接下来也是需要在上下文的实现中完成刷新容器的操作过程。
## 应用上下文抽象类实现
```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader implements ConfigurableApplicationContext {

    @Override
    public void refresh() throws BeansException {
        // 1. 创建 BeanFactory，并加载 BeanDefinition
        refreshBeanFactory();

        // 2. 获取 BeanFactory
        ConfigurableListableBeanFactory beanFactory = getBeanFactory();

        // 3. 在 Bean 实例化之前，执行 BeanFactoryPostProcessor (Invoke factory processors registered as beans in the context.)
        invokeBeanFactoryPostProcessors(beanFactory);

        // 4. BeanPostProcessor 需要提前于其他 Bean 对象实例化之前执行注册操作
        registerBeanPostProcessors(beanFactory);

        // 5. 提前实例化单例Bean对象
        beanFactory.preInstantiateSingletons();
    }

    protected abstract void refreshBeanFactory() throws BeansException;

    protected abstract ConfigurableListableBeanFactory getBeanFactory();

    private void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
        Map<String, BeanFactoryPostProcessor> beanFactoryPostProcessorMap = beanFactory.getBeansOfType(BeanFactoryPostProcessor.class);
        for (BeanFactoryPostProcessor beanFactoryPostProcessor : beanFactoryPostProcessorMap.values()) {
            beanFactoryPostProcessor.postProcessBeanFactory(beanFactory);
        }
    }

    private void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
        Map<String, BeanPostProcessor> beanPostProcessorMap = beanFactory.getBeansOfType(BeanPostProcessor.class);
        for (BeanPostProcessor beanPostProcessor : beanPostProcessorMap.values()) {
            beanFactory.addBeanPostProcessor(beanPostProcessor);
        }
    }
    
    //... getBean、getBeansOfType、getBeanDefinitionNames 方法

}
```
- AbstractApplicationContext 继承 DefaultResourceLoader 是为了处理 `spring.xml` 配置资源的加载。
- 之后是在 refresh() 定义实现过程，包括：
    - 1. 创建 BeanFactory，并加载 BeanDefinition
    - 2. 获取 BeanFactory
    - 3. 在 Bean 实例化之前，执行 BeanFactoryPostProcessor (Invoke factory processors registered as beans in the context.)
    - 4. BeanPostProcessor 需要提前于其他 Bean 对象实例化之前执行注册操作
    - 5. 提前实例化单例Bean对象
- 另外把定义出来的抽象方法，refreshBeanFactory()、getBeanFactory() 由后面的继承此抽象类的其他抽象类实现。
## 获取 Bean 工厂和加载资源
```java
public abstract class AbstractRefreshableApplicationContext extends AbstractApplicationContext {

    private DefaultListableBeanFactory beanFactory;

    @Override
    protected void refreshBeanFactory() throws BeansException {
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        loadBeanDefinitions(beanFactory);
        this.beanFactory = beanFactory;
    }

    private DefaultListableBeanFactory createBeanFactory() {
        return new DefaultListableBeanFactory();
    }

    protected abstract void loadBeanDefinitions(DefaultListableBeanFactory beanFactory);

    @Override
    protected ConfigurableListableBeanFactory getBeanFactory() {
        return beanFactory;
    }

}
```
- 在 refreshBeanFactory() 中主要是获取了 `DefaultListableBeanFactory` 的实例化以及对资源配置的加载操作 `loadBeanDefinitions(beanFactory)`，在加载完成后即可完成对 spring.xml 配置文件中 Bean 对象的定义和注册，同时也包括实现了接口 `BeanFactoryPostProcessor`、`BeanPostProcessor` 的配置 Bean 信息。
- 但此时资源加载还只是定义了一个抽象类方法 `loadBeanDefinitions(DefaultListableBeanFactory beanFactory)`，继续由其他抽象类继承实现。
## 应用上下文中对配置信息对加载
```java
public abstract class AbstractXmlApplicationContext extends AbstractRefreshableApplicationContext {

    @Override
    protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) {
        XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory, this);
        String[] configLocations = getConfigLocations();
        if (null != configLocations){
            beanDefinitionReader.loadBeanDefinitions(configLocations);
        }
    }

    protected abstract String[] getConfigLocations();

}
```
- 在 AbstractXmlApplicationContext 抽象类的 loadBeanDefinitions 方法实现中，使用 XmlBeanDefinitionReader 类，处理了关于 XML 文件配置信息的操作。
- 同时这里又留下了一个抽象类方法，getConfigLocations()，此方法是为了从入口上下文类，拿到配置信息的地址描述。
## 应用上下文最终实现类
```java
public class ClassPathXmlApplicationContext extends AbstractXmlApplicationContext {

    private String[] configLocations;

    public ClassPathXmlApplicationContext() {
    }

    /**
     * 从 XML 中加载 BeanDefinition，并刷新上下文
     *
     * @param configLocations
     * @throws BeansException
     */
    public ClassPathXmlApplicationContext(String configLocations) throws BeansException {
        this(new String[]{configLocations});
    }

    /**
     * 从 XML 中加载 BeanDefinition，并刷新上下文
     * @param configLocations
     * @throws BeansException
     */
    public ClassPathXmlApplicationContext(String[] configLocations) throws BeansException {
        this.configLocations = configLocations;
        refresh();
    }

    @Override
    protected String[] getConfigLocations() {
        return configLocations;
    }

}
```
## 在 Bean 创建完成后完成前置和后置处理
```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory implements AutowireCapableBeanFactory {

    private InstantiationStrategy instantiationStrategy = new CglibSubclassingInstantiationStrategy();

    @Override
    protected Object createBean(String beanName, BeanDefinition beanDefinition, Object[] args) throws BeansException {
        Object bean = null;
        try {
            bean = createBeanInstance(beanDefinition, beanName, args);
            // 给 Bean 填充属性
            applyPropertyValues(beanName, bean, beanDefinition);
            // 执行 Bean 的初始化方法和 BeanPostProcessor 的前置和后置处理方法
            bean = initializeBean(beanName, bean, beanDefinition);
        } catch (Exception e) {
            throw new BeansException("Instantiation of bean failed", e);
        }

        addSingleton(beanName, bean);
        return bean;
    }

    public InstantiationStrategy getInstantiationStrategy() {
        return instantiationStrategy;
    }

    public void setInstantiationStrategy(InstantiationStrategy instantiationStrategy) {
        this.instantiationStrategy = instantiationStrategy;
    }

    private Object initializeBean(String beanName, Object bean, BeanDefinition beanDefinition) {
        // 1. 执行 BeanPostProcessor Before 处理
        Object wrappedBean = applyBeanPostProcessorsBeforeInitialization(bean, beanName);

        // 待完成内容：invokeInitMethods(beanName, wrappedBean, beanDefinition);
        invokeInitMethods(beanName, wrappedBean, beanDefinition);

        // 2. 执行 BeanPostProcessor After 处理
        wrappedBean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
        return wrappedBean;
    }

    private void invokeInitMethods(String beanName, Object wrappedBean, BeanDefinition beanDefinition) {

    }

    @Override
    public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName) throws BeansException {
        Object result = existingBean;
        for (BeanPostProcessor processor : getBeanPostProcessors()) {
            Object current = processor.postProcessBeforeInitialization(result, beanName);
            if (null == current) return result;
            result = current;
        }
        return result;
    }

    @Override
    public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName) throws BeansException {
        Object result = existingBean;
        for (BeanPostProcessor processor : getBeanPostProcessors()) {
            Object current = processor.postProcessAfterInitialization(result, beanName);
            if (null == current) return result;
            result = current;
        }
        return result;
    }

}
```
- 实现 `BeanPostProcessor` 接口后，会涉及到两个接口方法，`postProcessBeforeInitialization`、`postProcessAfterInitialization`，分别作用于 Bean 对象执行初始化前后的额外处理。
- 也就是需要在创建 Bean 对象时，在 `createBean` 方法中添加 `initializeBean(beanName, bean, beanDefinition);` 操作。而这个操作主要主要是对于方法 `applyBeanPostProcessorsBeforeInitialization`、`applyBeanPostProcessorsAfterInitialization` 的使用。
- 另外需要提一下，`applyBeanPostProcessorsBeforeInitialization`、`applyBeanPostProcessorsAfterInitialization` 两个方法是在接口类 `AutowireCapableBeanFactory` 中新增加的。
