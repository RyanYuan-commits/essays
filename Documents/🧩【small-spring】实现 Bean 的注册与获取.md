#spring 
# 目标
在上一章节中，初步依照 Spring Bean 容器的概念，实现了一个粗糙版本的代码实现。那么本章节我们需要结合已实现的 Spring Bean 容器进行功能完善，实现 Bean 容器关于 Bean 对象的注册和获取。
这一次我们把 Bean 的创建交给容器，而不是我们在调用时候传递一个实例化好的 Bean 对象，另外还需要考虑**单例对象**，在对象的二次获取时是可以从内存中获取对象的。此外不仅要实现功能还需要完善基础容器框架的类结构体，否则将来就很难扩容进去其他的功能了。
# 设计
我们需要将 Spring Bean 容器完善起来，首先非常重要的一点是 **在 Bean 注册的时候只注册一个类信息**，而不会直接把实例化信息注册到 Spring 容器中；在需要获取 Bean 对象的时候，才对其进行实例化。
那么就需要修改 BeanDefinition 中的属性 Object 为 Class，接下来在需要做的就是在获取 Bean 对象时需要处理 Bean 对象的实例化操作以及判断当前单例对象在容器中是否已经缓存起来了。整体设计如图 3-1。

![https://bugstack.cn/assets/images/spring/spring-3-01.png](https://bugstack.cn/assets/images/spring/spring-3-01.png)

- 首先我们需要定义 **`BeanFactory`** 这样一个 Bean 工厂，提供 Bean 的获取方法 **`getBean(String name)`**，之后这个 Bean 工厂接口由抽象类 **`AbstractBeanFactory`** 实现。这样使用 [**模板模式](https://bugstack.cn/itstack-demo-design/2020/07/07/%E9%87%8D%E5%AD%A6-Java-%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F-%E5%AE%9E%E6%88%98%E6%A8%A1%E6%9D%BF%E6%A8%A1%E5%BC%8F.html)** 的设计方式，可以统一收口通用核心方法的调用逻辑和标准定义，也就很好的控制了后续的实现者不用关心调用逻辑，按照统一方式执行。那么类的继承者只需要关心具体方法的逻辑实现即可。
- 那么在继承抽象类 **`AbstractBeanFactory`** 后的 **`AbstractAutowireCapableBeanFactory`** 就可以实现相应的抽象方法了，因为 **`AbstractAutowireCapableBeanFactory`** 本身也是一个抽象类，所以它只会实现属于自己的抽象方法，其他抽象方法由继承 **`AbstractAutowireCapableBeanFactory`** 的类实现。这里就体现了类实现过程中的各司其职，你只需要关心属于你的内容，不是你的内容，不要参与。
- 另外这里还有块非常重要的知识点，就是关于单例 **`SingletonBeanRegistry`** 的接口定义实现，而 **`DefaultSingletonBeanRegistry`** 对接口实现后，会被抽象类 **`AbstractBeanFactory`** 继承。现在 AbstractBeanFactory 就是一个非常完整且强大的抽象类了，也能非常好的体现出它对模板模式的抽象定义。
# 工程结构
```java
small-spring-step-02
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
    │           │   │   ├── DefaultListableBeanFactory.java
    │           │   │   └── DefaultSingletonBeanRegistry.java
    │           │   └── BeanFactory.java
    │           └── BeansException.java
    └── test
        └── java
            └── cn.bugstack.springframework.test
                ├── bean
                │   └── UserService.java
                └── ApiTest.java
```
![[实现 Bean 的注册与获取：项目结构.png|700]]

# 代码实现
**`SingletonBeanRegistry`** ：单例 Bean 注册机接口
```java
public interface SingletonBeanRegistry {

    Object getSingleton(String beanName);

}
```

**`DefaultSingletonBeanRegistry`** ：单例 Bean 注册机的默认实现
```java
public class DefaultSingletonBeanRegistry implements SingletonBeanRegistry {

    private final Map<String, Object> singletonObjects = new HashMap<>();

    @Override
    public Object getSingleton(String beanName) {
        return singletonObjects.get(beanName);
    }

    protected void addSingleton(String beanName, Object singletonObject) {
        singletonObjects.put(beanName, singletonObject);
    }

}
```

**`BeanFactory`** ：最核心的接口，提供了 getBean 的方法
```java
public interface BeanFactory {

    Object getBean(String name) throws BeansException;

}
```

**`AbstractBeanFactory`** ：抽象 Bean 工厂
- 继承 `DefaultSingletonBeanRegistry` ：获取了**单例 Bean 注册**的能力，这里的 **`Bean`** 和 **`BeanDefination`** 要注意区分
- 实现 `BeanFactory` 接口：实现 Bean 的 **注册和获取**。
```java
public abstract class AbstractBeanFactory extends DefaultSingletonBeanRegistry implements BeanFactory {

    @Override
    public Object getBean(String name) throws BeansException {
        Object bean = getSingleton(name);
        if (bean != null) {
            return bean;
        }

        BeanDefinition beanDefinition = getBeanDefinition(name);
        return createBean(name, beanDefinition);
    }

		// 交给子类去实现的 abstract 方法
    protected abstract BeanDefinition getBeanDefinition(String beanName) throws BeansException;

    protected abstract Object createBean(String beanName, BeanDefinition beanDefinition) throws BeansException;

}
```

**`AbstractAutowireCapableBeanFactory`** ：自动装配 Bean 工厂
- 继承 **`AbstractBeanFactory`**
- 是实现自动注入（Auto Wiring）功能的核心部分之一，其自动注入的能力在本部分没有提现
```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory {
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
}
```
---
**`BeanDefinition`** ：Bean 定义信息
- 定义了 Bean 的元数据（metadata）
- 包括 Bean 的类类型、作用域、属性、构造函数参数等
- 在 Spring 容器 **启动时** 被创建，并用于描述和管理 Bean 的实例化和依赖注入过程
```java
public class BeanDefinition {

    private Class<?> beanClass;

    public BeanDefinition(Class<?>  beanClass) {
        this.beanClass = beanClass;
    }

    public Class<?> getBeanClass() {
        return beanClass;
    }

    public void setBeanClass(Class<?>  beanClass) {
        this.beanClass = beanClass;
    }

}
```

**`BeanDefinitionRegistry`** ：Bean 定义信息注册机
```java
public interface BeanDefinitionRegistry {

    /**
     * 向注册表中注册 BeanDefinition
     */
    void registerBeanDefinition(String beanName, BeanDefinition beanDefinition);

}
```
---
**`DefaultListableBeanFactory`** ：Bean 工厂最核心的实现，提供了「**列出所有 Bean**」功能的默认 Bean 工厂
- 继承了 **`AbstractAutowireCapableBeanFactory`**
- 实现 **`BeanDefinitionRegistry`** 接口：实现了 **`BeanDefinition`** 的注册能力。
```java
public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory implements BeanDefinitionRegistry {

    private final Map<String, BeanDefinition> beanDefinitionMap = new HashMap<>();

    @Override
    public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition) {
        beanDefinitionMap.put(beanName, beanDefinition);
    }

    @Override
    public BeanDefinition getBeanDefinition(String beanName) throws BeansException {
        BeanDefinition beanDefinition = beanDefinitionMap.get(beanName);
        if (beanDefinition == null) throw new BeansException("No bean named '" + beanName + "' is defined");
        return beanDefinition;
    }

}
```