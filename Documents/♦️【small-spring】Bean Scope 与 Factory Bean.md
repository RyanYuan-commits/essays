# 目标
交给 Spring 管理的 Bean 对象，一定就是我们用类创建出来的 Bean 吗？创建出来的 Bean 就永远是单例的吗，没有可能是原型模式吗？

在集合 Spring 框架下，我们使用的 MyBatis 框架中，它的核心作用是可以满足用户不需要实现 Dao 接口类，就可以通过 xml 或者注解配置的方式完成对数据库执行 CRUD 操作，那么在实现这样的 ORM 框架中，是怎么把一个数据库操作的 Bean 对象交给 Spring 管理的呢。

因为我们在使用 Spring、MyBatis 框架的时候都可以知道，并没有手动的去创建任何操作数据库的 Bean 对象，有的仅仅是一个接口定义，而这个接口定义竟然可以被注入到其他需要使用 Dao 的属性中去了，那么这一过程最核心待解决的问题，就是需要完成把复杂且以代理方式动态变化的对象，注册到 Spring 容器中。而为了满足这样的一个扩展组件开发的需求，就需要我们在现有手写的 Spring 框架中，添加这一能力。

# 方案
关于提供一个能让使用者定义复杂的 Bean 对象，功能点非常不错，意义也非常大，因为这样做了之后 Spring 的生态种子孵化箱就此提供了，谁家的框架都可以在此标准上完成自己服务的接入。

但这样的功能逻辑设计上并不复杂，因为整个 Spring 框架在开发的过程中就已经提供了各项扩展能力的接茬，你只需要在合适的位置提供一个接茬的处理接口调用和相应的功能逻辑实现即可，像这里的目标实现就是对外提供一个可以二次从 FactoryBean 的 getObject 方法中获取对象的功能即可，这样所有实现此接口的对象类，就可以扩充自己的对象功能了。
MyBatis 就是实现了一个 MapperFactoryBean 类，在 getObject 方法中提供 SqlSession 对执行 CRUD 方法的操作整体设计结构如下图：
![[small-spring】Bean Scope 与 Factory Bean 架构图.png]]
- 整个的实现过程包括了两部分，一个解决单例还是原型对象，另外一个处理 Factory Bean 类型对象创建过程中关于获取具体调用对象的 `getObject` 操作。
- `SCOPE_SINGLETON`、`SCOPE_PROTOTYPE`，对象类型的创建获取方式，主要区分在于 `AbstractAutowireCapableBeanFactory#createBean` 创建完成对象后是否放入到内存中，如果不放入则每次获取都会重新创建。
- createBean 执行对象创建、属性填充、依赖加载、前置后置处理、初始化等操作后，就要开始做执行判断整个对象是否是一个 FactoryBean 对象，如果是这样的对象，就需要再继续执行获取 FactoryBean 具体对象中的 `getObject` 对象了。整个 getBean 过程中都会新增一个单例类型的判断 `factory.isSingleton()`，用于决定是否使用内存存放对象信息。

# 实现
![[【small-spring】Bean Scope 与 Factory Bean 核心类图.png|900]]
## BeanDefinition 拓展 scope 属性
```java
public class BeanDefinition {

    String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;

    String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;

    private Class beanClass;

    private PropertyValues propertyValues;

    private String initMethodName;

    private String destroyMethodName;

    private String scope = SCOPE_SINGLETON;

    private boolean singleton = true;

    private boolean prototype = false;
    
    //在xml注册Bean定义时，通过scope字段来判断是单例还是原型
    public void setScope(String scope) {
        this.scope = scope;
        this.singleton = SCOPE_SINGLETON.equals(scope);
        this.prototype = SCOPE_PROTOTYPE.equals(scope);
    }
    
    // ...get/set
}
```
- singleton、prototype，是本次在 BeanDefinition 类中新增加的两个属性信息，用于把从 spring.xml 中解析到的 Bean 对象作用范围填充到属性中。

`XmlBeanDefinitionReader`：拓展解析
```java
public class XmlBeanDefinitionReader extends AbstractBeanDefinitionReader {

    protected void doLoadBeanDefinitions(InputStream inputStream) throws ClassNotFoundException {
      
        for (int i = 0; i < childNodes.getLength(); i++) {
            // 判断元素
            if (!(childNodes.item(i) instanceof Element)) continue;
            // 判断对象
            if (!"bean".equals(childNodes.item(i).getNodeName())) continue;

            // 解析标签
            Element bean = (Element) childNodes.item(i);
            String id = bean.getAttribute("id");
            String name = bean.getAttribute("name");
            String className = bean.getAttribute("class");
            String initMethod = bean.getAttribute("init-method");
            String destroyMethodName = bean.getAttribute("destroy-method");
            String beanScope = bean.getAttribute("scope");

            // 获取 Class，方便获取类中的名称
            Class<?> clazz = Class.forName(className);
            // 优先级 id > name
            String beanName = StrUtil.isNotEmpty(id) ? id : name;
            if (StrUtil.isEmpty(beanName)) {
                beanName = StrUtil.lowerFirst(clazz.getSimpleName());
            }

            // 定义Bean
            BeanDefinition beanDefinition = new BeanDefinition(clazz);
            beanDefinition.setInitMethodName(initMethod);
            beanDefinition.setDestroyMethodName(destroyMethodName);

            if (StrUtil.isNotEmpty(beanScope)) {
                beanDefinition.setScope(beanScope);
            }
            
            // ...
            
            // 注册 BeanDefinition
            getRegistry().registerBeanDefinition(beanName, beanDefinition);
        }
    }

}
```
- 在解析 XML 处理类 XmlBeanDefinitionReader 中，新增加了关于 Bean 对象配置中 scope 的解析，并把这个属性信息填充到 Bean 定义中。`beanDefinition.setScope(beanScope)`
## 创建和修改 Bean 对象的时候判断单例和原型模式
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

        // 注册实现了 DisposableBean 接口的 Bean 对象
        registerDisposableBeanIfNecessary(beanName, bean, beanDefinition);

        // 判断 SCOPE_SINGLETON、SCOPE_PROTOTYPE
        if (beanDefinition.isSingleton()) {
            addSingleton(beanName, bean);
        }
        return bean;
    }

    protected void registerDisposableBeanIfNecessary(String beanName, Object bean, BeanDefinition beanDefinition) {
        // 非 Singleton 类型的 Bean 不执行销毁方法
        if (!beanDefinition.isSingleton()) return;

        if (bean instanceof DisposableBean || StrUtil.isNotEmpty(beanDefinition.getDestroyMethodName())) {
            registerDisposableBean(beanName, new DisposableBeanAdapter(bean, beanName, beanDefinition));
        }
    }
    // ... 其他功能
}
```
- 例模式和原型模式的区别就在于是否存放到内存中，如果是原型模式那么就不会存放到内存中，每次获取都重新创建对象，另外非 Singleton 类型的 Bean 不需要执行销毁方法。
- 所以这里的代码会有两处修改，一处是 createBean 中判断是否添加到 addSingleton(beanName, bean);，另外一处是 registerDisposableBeanIfNecessary 销毁注册中的判断 `if (!beanDefinition.isSingleton()) return;`。
## 定义 Factory Bean 接口
```java
public interface FactoryBean<T> {

    T getObject() throws Exception;

    Class<?> getObjectType();

    boolean isSingleton();

}
```
- FactoryBean 中需要提供3个方法，获取对象、对象类型，以及是否是单例对象，如果是单例对象依然会被放到内存中。
## 实现一个 Factory Bean 注册服务
```java
public abstract class FactoryBeanRegistrySupport extends DefaultSingletonBeanRegistry {

    /**
     * Cache of singleton objects created by FactoryBeans: FactoryBean name --> object
     */
    private final Map<String, Object> factoryBeanObjectCache = new ConcurrentHashMap<String, Object>();

    protected Object getCachedObjectForFactoryBean(String beanName) {
        Object object = this.factoryBeanObjectCache.get(beanName);
        return (object != NULL_OBJECT ? object : null);
    }

    protected Object getObjectFromFactoryBean(FactoryBean factory, String beanName) {
        if (factory.isSingleton()) {
            Object object = this.factoryBeanObjectCache.get(beanName);
            if (object == null) {
                object = doGetObjectFromFactoryBean(factory, beanName);
                this.factoryBeanObjectCache.put(beanName, (object != null ? object : NULL_OBJECT));
            }
            return (object != NULL_OBJECT ? object : null);
        } else {
            return doGetObjectFromFactoryBean(factory, beanName);
        }
    }

    private Object doGetObjectFromFactoryBean(final FactoryBean factory, final String beanName){
        try {
            return factory.getObject();
        } catch (Exception e) {
            throw new BeansException("FactoryBean threw exception on object[" + beanName + "] creation", e);
        }
    }

}
```
## 扩展 AbstractBeanFactory 创建对象逻辑
```java
public abstract class AbstractBeanFactory extends FactoryBeanRegistrySupport implements ConfigurableBeanFactory {

    protected <T> T doGetBean(final String name, final Object[] args) {
        Object sharedInstance = getSingleton(name);
        if (sharedInstance != null) {
            // 如果是 FactoryBean，则需要调用 FactoryBean#getObject
            return (T) getObjectForBeanInstance(sharedInstance, name);
        }

        BeanDefinition beanDefinition = getBeanDefinition(name);
        Object bean = createBean(name, beanDefinition, args);
        return (T) getObjectForBeanInstance(bean, name);
    }  
   
    private Object getObjectForBeanInstance(Object beanInstance, String beanName) {
        if (!(beanInstance instanceof FactoryBean)) {
            return beanInstance;
        }

        Object object = getCachedObjectForFactoryBean(beanName);

        if (object == null) {
            FactoryBean<?> factoryBean = (FactoryBean<?>) beanInstance;
            object = getObjectFromFactoryBean(factoryBean, beanName);
        }

        return object;
    }
        
    // ...
}
```
- 首先这里把 `AbstractBeanFactory` 原来继承的 `DefaultSingletonBeanRegistry`，修改为继承 `FactoryBeanRegistrySupport`。因为需要扩展出创建 `FactoryBean` 对象的能力，所以这就想一个链条服务上，截出一个段来处理额外的服务，并把链条再链接上。
- 此处新增加的功能主要是在 `doGetBean` 方法中，添加了调用 `(T) getObjectForBeanInstance(sharedInstance, name)` 对获取 FactoryBean 的操作。
- 在 `getObjectForBeanInstance` 方法中做具体的 `instanceof` 判断，另外还会从 `FactoryBean` 的缓存中获取对象，如果不存在则调用 `FactoryBeanRegistrySupport#getObjectFromFactoryBean`，执行具体的操作。
