# 目标
当我们的类创建的 Bean 对象，交给 Spring 容器管理以后，这个类对象就可以被赋予更多的使用能力。就像我们在上一章节已经给类对象添加了修改注册Bean定义未实例化前的属性信息修改和初始化过程中的前置和后置处理，这些额外能力的实现，都可以让我们对现有工程中的类对象做相应的扩展处理。

那么除此之外我们还希望可以在 Bean 初始化过程或者说 Bean 销毁的时候，执行一些操作。
比如帮我们做一些数据的加载执行，链接注册中心暴露RPC接口以及在Web程序关闭时执行链接断开，内存销毁等操作。如果说没有Spring我们也可以通过构造函数、静态方法以及手动调用的方式实现，但这样的处理方式终究不如把诸如此类的操作都交给 Spring 容器来管理更加合适。因此你会看到到 spring.xml 中有如下操作：

![[spring init-method 配置.png]]

需要满足用户可以在 xml 中配置初始化和销毁的方法，也可以通过实现类的方式处理，比如我们在使用 Spring 时用到的 InitializingBean, DisposableBean 两个接口。 -其实还可以有一种是注解的方式处理初始化操作，不过目前还没有实现到注解的逻辑，后续再完善此类功能。
# 设计
可能面对像 Spring 这样庞大的框架，对外暴露的接口定义使用或者xml配置，完成的一系列扩展性操作，都让 Spring 框架看上去很神秘。其实对于这样在 Bean 容器初始化过程中额外添加的处理操作，无非就是预先执行了一个定义好的接口方法或者是反射调用类中xml中配置的方法，最终你只要按照接口定义实现，就会有 Spring 容器在处理的过程中进行调用而已。整体设计结构如下图：
![[【small-spring】Bean 的初始化和销毁方法 设计图.png]]
- 在 spring.xml 配置中添加 `init-method、destroy-method` 两个注解，在配置文件加载的过程中，把注解配置一并定义到 BeanDefinition 的属性当中。
  这样在 initializeBean 初始化操作的工程中，就可以通过反射的方式来调用配置在 Bean 定义属性当中的方法信息了。另外如果是接口实现的方式，那么直接可以通过 Bean 对象调用对应接口定义的方法即可，`((InitializingBean) bean).afterPropertiesSet()`，两种方式达到的效果是一样的。
- 除了在初始化做的操作外，`destroy-method` 和 `DisposableBean` 接口的定义，都会在 Bean 对象初始化完成阶段，执行注册销毁方法的信息到 DefaultSingletonBeanRegistry 类中的 disposableBeans 属性里，这是为了后续统一进行操作。 
  关于销毁方法需要在虚拟机执行关闭之前进行操作，所以这里需要用到一个注册钩子的操作，如：`Runtime.getRuntime().addShutdownHook(new Thread(() -> System.out.println("close！")));` 另外你可以使用手动调用 ApplicationContext.close 方法关闭容器。
# 实现方案
![[【small-spring】Bean 的初始化和销毁方法 核心类图.png|800]]
- 以上整个类图结构描述出来的就是本次新增 Bean 实例化过程中的初始化方法和销毁方法。
- 因为我们一共实现了两种方式的初始化和销毁方法，xml配置和定义接口，所以这里既有 `InitializingBean`、`DisposableBean` 也有需要 `XmlBeanDefinitionReader` 加载 spring.xml 配置信息到 BeanDefinition 中。
- 另外接口 ConfigurableBeanFactory 定义了 destroySingletons 销毁方法，并由 AbstractBeanFactory 继承的父类 DefaultSingletonBeanRegistry 实现 ConfigurableBeanFactory 接口定义的 destroySingletons 方法。_这种方式的设计可能数程序员是没有用过的，都是用的谁实现接口谁完成实现类，而不是把实现接口的操作又交给继承的父类处理。所以这块还是蛮有意思的，是一种不错的隔离分层服务的设计方式_
- 最后就是关于向虚拟机注册钩子，保证在虚拟机关闭之前，执行销毁操作。`Runtime.getRuntime().addShutdownHook(new Thread(() -> System.out.println("close！")));`
## 定义初始化和销毁方法的接口
```java
public interface InitializingBean {

    /**
     * Bean 处理了属性填充后调用
     * 
     * @throws Exception
     */
    void afterPropertiesSet() throws Exception;

}
```

```java
public interface DisposableBean {

    void destroy() throws Exception;

}
```
- initializingBean、DisposableBean，两个接口方法还是比较常用的，在一些需要结合 Spring 实现的组件中，经常会使用这两个方法来做一些参数的初始化和销毁操作。比如接口暴漏、数据库数据读取、配置文件加载等等。
## BeanDfinition 定义初始化和销毁
```java
public class BeanDefinition {

    private Class beanClass;

    private PropertyValues propertyValues;

    private String initMethodName;
    
    private String destroyMethodName;
    
    // ...get/set
}
```
- 在 BeanDefinition 新增加了两个属性：initMethodName、destroyMethodName，这两个属性是为了在 spring.xml 配置的 Bean 对象中，可以配置 `init-method="initDataMethod" destroy-method="destroyDataMethod"` 操作，最终实现接口的效果是一样的。

`XmlBeanDefinitionReader`：加载和读取上面提到的两个属性
```java
public class XmlBeanDefinitionReader extends AbstractBeanDefinitionReader {

    //Constructor

    //其他方法
    
    //doLoadBeanDefinitions中增加对init-method、destroy-method的读取
    protected void doLoadBeanDefinitions(InputStream inputStream) throws ClassNotFoundException{
        Document doc = XmlUtil.readXML(inputStream);
        Element root = doc.getDocumentElement();
        NodeList childNodes = root.getChildNodes();

        for (int i = 0; i < childNodes.getLength(); i++) {
            if (!(childNodes.item(i) instanceof Element)) {
                continue;
            }
            if (!"bean".equals(childNodes.item(i).getNodeName())) {
                continue;
            }

            Element bean = (Element) childNodes.item(i);
            String id = bean.getAttribute("id");
            String name = bean.getAttribute("name");
            String className = bean.getAttribute("class");
            
            //增加对init-method、destroy-method的读取
            String initMethod = bean.getAttribute("init-method");
            String destroyMethodName = bean.getAttribute("destroy-method");

            Class<?> clazz = Class.forName(className);
            String beanName = StrUtil.isNotEmpty(id) ? id : name;
            if (StrUtil.isEmpty(beanName)){
                beanName = StrUtil.lowerFirst(clazz.getSimpleName());
            }

            BeanDefinition beanDefinition = new BeanDefinition(clazz);
            
            //额外设置到beanDefinition中
            beanDefinition.setInitMethodName(initMethod);
            beanDefinition.setDestroyMethodName(destroyMethodName);

            for (int j = 0; j < bean.getChildNodes().getLength(); j++) {
                if (!(bean.getChildNodes().item(j) instanceof Element)) {
                    continue;
                }
                if (!"property".equals(bean.getChildNodes().item(j).getNodeName())) {
                    continue;
                }
                //解析标签：property
                Element property = (Element) bean.getChildNodes().item(j);
                String attrName = property.getAttribute("name");
                String attrValue = property.getAttribute("value");
                String attrRef = property.getAttribute("ref");
                Object value = StrUtil.isNotEmpty(attrRef) ? new BeanReference(attrRef) : attrValue;
                PropertyValue propertyValue = new PropertyValue(attrName, value);
                beanDefinition.getPropertyValues().addPropertyValue(propertyValue);
            }

            if (getRegistry().containsBeanDefinition(beanName)) {
                throw new BeansException("Duplicate beanName["+beanName+"] is not allowed");
            }
            getRegistry().registerBeanDefinition(beanName,beanDefinition);
        }
    }
}
```
## 执行 Bean 对象的初始化方法
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

        // ...

        addSingleton(beanName, bean);
        return bean;
    }

    private Object initializeBean(String beanName, Object bean, BeanDefinition beanDefinition) {
        // 1. 执行 BeanPostProcessor Before 处理
        Object wrappedBean = applyBeanPostProcessorsBeforeInitialization(bean, beanName);

        // 执行 Bean 对象的初始化方法
        try {
            invokeInitMethods(beanName, wrappedBean, beanDefinition);
        } catch (Exception e) {
            throw new BeansException("Invocation of init method of bean[" + beanName + "] failed", e);
        }

        // 2. 执行 BeanPostProcessor After 处理
        wrappedBean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
        return wrappedBean;
    }

    private void invokeInitMethods(String beanName, Object bean, BeanDefinition beanDefinition) throws Exception {
        // 1. 实现接口 InitializingBean
        if (bean instanceof InitializingBean) {
            ((InitializingBean) bean).afterPropertiesSet();
        }

        // 2. 配置信息 init-method {判断是为了避免二次执行销毁}
        String initMethodName = beanDefinition.getInitMethodName();
        if (StrUtil.isNotEmpty(initMethodName)) {
            Method initMethod = beanDefinition.getBeanClass().getMethod(initMethodName);
            if (null == initMethod) {
                throw new BeansException("Could not find an init method named '" + initMethodName + "' on bean with name '" + beanName + "'");
            }
            initMethod.invoke(bean);
        }
    }

}
```
- 抽象类 AbstractAutowireCapableBeanFactory 中的 createBean 是用来创建 Bean 对象的方法，在这个方法中我们之前已经扩展了 BeanFactoryPostProcessor、BeanPostProcessor 操作，这里我们继续完善执行 Bean 对象的初始化方法的处理动作。
- 在方法 invokeInitMethods 中，主要分为两块来执行实现了 InitializingBean 接口的操作，处理 afterPropertiesSet 方法。另外一个是判断配置信息 init-method 是否存在，执行反射调用 initMethod.invoke(bean)。这两种方式都可以在 Bean 对象初始化过程中进行处理加载 Bean 对象中的初始化操作，让使用者可以额外新增加自己想要的动作。
## 定义销毁方法接口适配器
```java
public class DisposableBeanAdapter implements DisposableBean {

    private final Object bean;
    private final String beanName;
    private String destroyMethodName;

    public DisposableBeanAdapter(Object bean, String beanName, BeanDefinition beanDefinition) {
        this.bean = bean;
        this.beanName = beanName;
        this.destroyMethodName = beanDefinition.getDestroyMethodName();
    }

    @Override
    public void destroy() throws Exception {
        // 1. 实现接口 DisposableBean
        if (bean instanceof DisposableBean) {
            ((DisposableBean) bean).destroy();
        }

        // 2. 配置信息 destroy-method {判断是为了避免二次执行销毁}
        if (StrUtil.isNotEmpty(destroyMethodName) && !(bean instanceof DisposableBean && "destroy".equals(this.destroyMethodName))) {
            Method destroyMethod = bean.getClass().getMethod(destroyMethodName);
            if (null == destroyMethod) {
                throw new BeansException("Couldn't find a destroy method named '" + destroyMethodName + "' on bean with name '" + beanName + "'");
            }
            destroyMethod.invoke(bean);
        }
        
    }

}
```
- 可能你会想这里怎么有一个适配器的类呢，因为销毁方法有两种甚至多种方式，目前有`实现接口 DisposableBean`、`配置信息 destroy-method`，两种方式。
  而这两种方式的销毁动作是由 AbstractApplicationContext 在注册虚拟机钩子后看，虚拟机关闭前执行的操作动作。
- 那么在销毁执行时不太希望还得关注都销毁那些类型的方法，它的使用上更希望是有一个统一的接口进行销毁，所以这里就新增了适配类，做统一处理。
## 在创建 Bean 的时候注册销毁方法对象
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

        addSingleton(beanName, bean);
        return bean;
    }

    protected void registerDisposableBeanIfNecessary(String beanName, Object bean, BeanDefinition beanDefinition) {
        if (bean instanceof DisposableBean || StrUtil.isNotEmpty(beanDefinition.getDestroyMethodName())) {
            registerDisposableBean(beanName, new DisposableBeanAdapter(bean, beanName, beanDefinition));
        }
    }

}
```
## 虚拟机关闭 Hook 注册调用销毁方法
```java
public interface ConfigurableApplicationContext extends ApplicationContext {

    void refresh() throws BeansException;

    void registerShutdownHook();

    void close();

}
```
- 首先我们需要在 ConfigurableApplicationContext 接口中定义注册虚拟机钩子的方法 `registerShutdownHook` 和手动执行关闭的方法 `close`。
```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader implements ConfigurableApplicationContext {

    // ...

    @Override
    public void registerShutdownHook() {
        Runtime.getRuntime().addShutdownHook(new Thread(this::close));
    }

    @Override
    public void close() {
        getBeanFactory().destroySingletons();
    }

}
```