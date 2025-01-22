# 目标
上一节的 Bean 的初始化是通过这样的方式实现的：
```java
    @Override
    protected Object createBean(String beanName, BeanDefinition beanDefinition) throws BeansException {
        Object bean;
        try {
            bean = beanDefinition.getBeanClass().newInstance();
        } catch (InstantiationException | IllegalAccessException e) {
            throw new BeansException("Instantiation of bean failed", e);
        }
        addSingleton(beanName, bean);
        return bean;
    }
```
采用的是无参数的构造方法，但是如果类只有一个有参的构造方法，就会抛出异常：
```
java.lang.InstantiationException: cn.bugstack.springframework.test.bean.UserService
	at java.lang.Class.newInstance(Class.java:427)
	at cn.bugstack.springframework.test.ApiTest.test_newInstance(ApiTest.java:51)
	...
```
发生这一现象的主要原因就是因为 beanDefinition.getBeanClass().newInstance(); 实例化方式并没有考虑构造函数的入参，所以就这个坑就在这等着你了！那么我们的目标就很明显了，来把这个坑填平！
# 设计
填平这个坑的技术设计主要考虑两部分，一个是串流程 **从哪合理的把构造函数的入参信息传递到实例化操作里**，另外一个是 **怎么去实例化含有构造函数的对象**。
![[Bean 实例化架构图.png]]
- 参考 Spring Bean 容器源码的实现方式，在 BeanFactory 中添加 `Object getBean(String name, Object... args)` 接口，这样就可以在获取 Bean 时把构造函数的入参信息传递进去了。
- 另外一个核心的内容是使用什么方式来创建含有构造函数的 Bean 对象呢？这里有两种方式可以选择，一个是基于 Java 本身自带的方法 `DeclaredConstructor`，另外一个是使用 Cglib 来动态创建 Bean 对象。
  Cglib 是基于字节码框架 ASM 实现，所以你也可以直接通过 ASM 操作指令码来创建对象。
# 工程结构
```
small-spring-step-03
└── src
    ├── main
    │   └── java
    │       └── cn.bugstack.springframework.beans
    │           ├── factory
    │           │   ├── config
    │           │   │   ├── BeanDefinition.java
    │           │   │   └── SingletonBeanRegistry.java
    │           │   ├── support
    │           │   │   ├── AbstractAutowireCapableBeanFactory.java
    │           │   │   ├── AbstractBeanFactory.java
    │           │   │   ├── BeanDefinitionRegistry.java
    │           │   │   ├── CglibSubclassingInstantiationStrategy.java
    │           │   │   ├── DefaultListableBeanFactory.java
    │           │   │   ├── DefaultSingletonBeanRegistry.java
    │           │   │   ├── InstantiationStrategy.java
    │           │   │   └── SimpleInstantiationStrategy.java
    │           │   └── BeanFactory.java
    │           └── BeansException.java
    └── test
        └── java
            └── cn.bugstack.springframework.test
                ├── bean
                │   └── UserService.java
                └── ApiTest.java
```
![[基于 Cglib 实现含构造函数的类实例化策略架构图.png|900]]
# 代码实现
`BeanFactory` 拓展：增添通过参数获取 Bean 的方法
```java
public interface BeanFactory {

    Object getBean(String name) throws BeansException;

    Object getBean(String name, Object... args) throws BeansException;

}
```

`AbstractBeanFactory` 实现
```java
@SuppressWarnings("unchecked")
public abstract class AbstractBeanFactory extends DefaultSingletonBeanRegistry implements BeanFactory {

    @Override
    public Object getBean(String name) throws BeansException {
        return doGetBean(name, null);
    }

    @Override
    public Object getBean(String name, Object... args) throws BeansException {
        return doGetBean(name, args);
    }

    protected <T> T doGetBean(final String name, final Object[] args) {
        Object bean = getSingleton(name);
        if (bean != null) {
            return (T) bean;
        }

        BeanDefinition beanDefinition = getBeanDefinition(name);
        return (T) createBean(name, beanDefinition, args);
    }

    protected abstract BeanDefinition getBeanDefinition(String beanName) throws BeansException;

    protected abstract Object createBean(String beanName, BeanDefinition beanDefinition, Object[] args) throws BeansException;

}
```

---

**策略模式实现不同方式的 Bean 获取**

`InstantiationStrategy` 总接口
```java
public interface InstantiationStrategy {

    /**
     * Bean 的实例化
     * @param beanDefinition Bean 定义信息，Class 信息
     * @param beanName 实例化的 Bean 名称
     * @param ctor 构造器
     * @param args 构造方法
     * @return 实例化的 Bean
     * @throws BeansException Bean 实例化异常
     */
    Object instantiate(BeanDefinition beanDefinition, String beanName,
                       Constructor<?> ctor, Object[] args) throws BeansException;

}
```

`SimpleInstantiationStrategy` `CglibSubclassingInstantiationStrategy` 两个实现子类
```java
public class CglibSubclassingInstantiationStrategy implements InstantiationStrategy {

    @Override
    public Object instantiate(BeanDefinition beanDefinition, String beanName,
                              Constructor<?> ctor, Object[] args) throws BeansException {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(beanDefinition.getBeanClass());
        enhancer.setCallback(new NoOp() {
            @Override
            public int hashCode() {
                return super.hashCode();
            }
        });
        if (null == ctor) return enhancer.create();
        return enhancer.create(ctor.getParameterTypes(), args);
    }

}

public class SimpleInstantiationStrategy implements InstantiationStrategy{

    @Override
    public Object instantiate(BeanDefinition beanDefinition, String beanName,
                              Constructor<?> ctor, Object[] args) throws BeansException {
        Class<?> clazz = beanDefinition.getBeanClass();
        try {
            if (null != ctor) {
                return clazz.getDeclaredConstructor(ctor.getParameterTypes()).newInstance(args);
            } else {
                return clazz.getDeclaredConstructor().newInstance();
            }
        } catch (NoSuchMethodException | InstantiationException | IllegalAccessException | InvocationTargetException e) {
            throw new BeansException("Failed to instantiate [" + clazz.getName() + "]", e);
        }
    }

}
```

---

`AbstractAutowireCapableBeanFactory` 拓展策略模式选择
```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory {

    private InstantiationStrategy instantiationStrategy = new CglibSubclassingInstantiationStrategy();

    @Override
    protected Object createBean(String beanName, BeanDefinition beanDefinition, Object[] args) throws BeansException {
        Object bean = null;
        try {
            bean = createBeanInstance(beanDefinition, beanName, args);
        } catch (Exception e) {
            throw new BeansException("Instantiation of bean failed", e);
        }

        addSingleton(beanName, bean);
        return bean;
    }

    protected Object createBeanInstance(BeanDefinition beanDefinition, String beanName, Object[] args) {
        Constructor<?> constructorToUse = null;
        Class<?> beanClass = beanDefinition.getBeanClass();
        Constructor<?>[] declaredConstructors = beanClass.getDeclaredConstructors();
        for (Constructor<?> ctor : declaredConstructors) {
            if (null != args && ctor.getParameterTypes().length == args.length) {
                constructorToUse = ctor;
                break;
            }
        }
        return getInstantiationStrategy().instantiate(beanDefinition, beanName, constructorToUse, args);
    }

    public InstantiationStrategy getInstantiationStrategy() {
        return instantiationStrategy;
    }

    public void setInstantiationStrategy(InstantiationStrategy instantiationStrategy) {
        this.instantiationStrategy = instantiationStrategy;
    }

}
```

### 测试
```java
public class test {

    @Test
    public void test_BeanFactory() {
        // 1.初始化 BeanFactory
        DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();

        // 3. 注入bean
        BeanDefinition beanDefinition = new BeanDefinition(UserService.class);
        beanFactory.registerBeanDefinition("userService", beanDefinition);

        // 4.获取bean
        UserService userService = (UserService) beanFactory.getBean("userService", "kkq");
        userService.queryUserInfo();
    }

}

public class UserService {

    private String name;

    public UserService() {}

    public UserService(String name) {
        this.name = name;
    }

    public void queryUserInfo() {
        System.out.println("查询用户信息：" + name);
    }

    @Override
    public String toString() {
        return name;
    }

}
```

测试结果
```java
查询用户信息：kkq

Process finished with exit code 0
```

### CgLib 测试案例
```java
/**
 * @author kq
 * 2024-08-04 21:05
 * CGLIB 测试
 **/
public class CgLibTest {

    @Test
    public void textBeanCreation() {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(UserService.class);
        enhancer.setCallback(new SimpleMethodInterceptor());
        UserService proxy = (UserService)enhancer.create();
        proxy.test();
    }

}
```