# 目标
在目前 IOC、AOP 两大核心功能模块的支撑下，完全可以管理 Bean 对象的注册和获取，不过这样的使用方式总感觉像是刀耕火种有点难用。因此在上一章节我们解决需要手动配置 `Bean` 对象到 `spring.xml` 文件中，改为可以自动扫描带有注解 `@Component` 的对象完成自动装配和注册到 `Spring` 容器的操作。

那么在自动扫描包注册 Bean 对象之后，就需要把原来在配置文件中通过 `property name="token"` 配置属性和Bean的操作，也改为可以自动注入。这就像我们使用 Spring 框架中 `@Autowired`、`@Value` 注解一样，完成我们对属性和对象的注入操作。
#  方案
其实从我们在完成 `Bean` 对象的基础功能后，后续陆续添加的功能都是围绕着 `Bean` 的生命周期进行的，比如修改 `Bean` 的定义 `BeanFactoryPostProcessor`，处理 Bean 的属性要用到 `BeanPostProcessor`，完成个性的属性操作则专门继承 `BeanPostProcessor` 提供新的接口，因为这样才能通过 `instanceof` 判断出具有标记性的接口。
所以关于 Bean 等等的操作，以及监听 Aware、获取 BeanFactory，都需要在 Bean 的生命周期中完成。那么我们在设计属性和 Bean 对象的注入时候，也会用到 `BeanPostProcessor` 来完成在设置 Bean 属性之前，允许 `BeanPostProcessor` 修改属性值。整体设计结构如下图：
![[【small-spring】通过注解简化属性配置 架构图.png]]
- 要处理自动扫描注入，包括属性注入、对象注入，则需要在对象属性 `applyPropertyValues` 填充之前 ，把属性信息写入到 PropertyValues 的集合中去。这一步的操作相当于是解决了以前在 spring.xml 配置属性的过程。
- 而在属性的读取中，需要依赖于对 Bean 对象的类中属性的配置了注解的扫描，`field.getAnnotation(Value.class);` 依次拿出符合的属性并填充上相应的配置信息。这里有一点 ，属性的配置信息需要依赖于 `BeanFactoryPostProcessor` 的实现类 `PropertyPlaceholderConfigurer`，把值写入到 `AbstractBeanFactory` 的 `embeddedValueResolvers集合中`，这样才能在属性填充中利用 beanFactory 获取相应的属性值
- 还有一个是关于 @Autowired 对于对象的注入，其实这一个和属性注入的唯一区别是对于对象的获取 `beanFactory.getBean(fieldType)`，其他就没有什么差一点了。
- 当所有的属性被设置到 `PropertyValues` 完成以后，接下来就到了创建对象的下一步，属性填充，而此时就会把我们一一获取到的配置和对象填充到属性上，也就实现了自动注入的功能。
# 实现
![[【small-spring】通过注解简化属性配置 核心类图.png|900]]
- 在整个类图中以围绕实现接口 InstantiationAwareBeanPostProcessor 的类 AutowiredAnnotationBeanPostProcessor 作为入口点，被 AbstractAutowireCapableBeanFactory创建 Bean 对象过程中调用扫描整个类的属性配置中含有自定义注解 `Value`、`Autowired`、`Qualifier`，的属性值。
- 这里稍有变动的是关于属性值信息的获取，在注解配置的属性字段扫描到信息注入时，包括了占位符从配置文件获取信息也包括 Bean 对象，Bean 对象可以直接获取，但配置信息需要在 AbstractBeanFactory 中添加新的属性集合 embeddedValueResolvers，由 PropertyPlaceholderConfigurer#postProcessBeanFactory 进行操作填充到属性集合中。
## 将读取到的属性填充到容器中
```java
public interface StringValueResolver {

    String resolveStringValue(String strVal);

}
```
- 接口 StringValueResolver 是一个解析字符串操作的接口

```java
public class PropertyPlaceholderConfigurer implements BeanFactoryPostProcessor {

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        try {
            // 加载属性文件
            DefaultResourceLoader resourceLoader = new DefaultResourceLoader();
            Resource resource = resourceLoader.getResource(location);
            
            // ... 占位符替换属性值、设置属性值

            // 向容器中添加字符串解析器，供解析@Value注解使用
            StringValueResolver valueResolver = new PlaceholderResolvingStringValueResolver(properties);
            beanFactory.addEmbeddedValueResolver(valueResolver);
            
        } catch (IOException e) {
            throw new BeansException("Could not load properties", e);
        }
    }

    private class PlaceholderResolvingStringValueResolver implements StringValueResolver {

        private final Properties properties;

        public PlaceholderResolvingStringValueResolver(Properties properties) {
            this.properties = properties;
        }

        @Override
        public String resolveStringValue(String strVal) {
            return PropertyPlaceholderConfigurer.this.resolvePlaceholder(strVal, properties);
        }

    }

}
```
- 在解析属性配置的类 PropertyPlaceholderConfigurer 中，最主要的其实就是这行代码的操作 `beanFactory.addEmbeddedValueResolver(valueResolver)` 这是把属性值写入到了 AbstractBeanFactory 的 embeddedValueResolvers 中。
- 这里说明下，embeddedValueResolvers 是 AbstractBeanFactory 类新增加的集合 `List<StringValueResolver> embeddedValueResolvers` String resolvers to apply e.g. to annotation attribute values
## 自定义属性注入注解
```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.CONSTRUCTOR, ElementType.FIELD, ElementType.METHOD})
public @interface Autowired {
}

@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER, ElementType.TYPE, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Qualifier {

    String value() default "";

}  

@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Value {

    /**
     * The actual value expression: e.g. "#{systemProperties.myProp}".
     */
    String value();

}
```
## 扫描自定义注解
```java
public class AutowiredAnnotationBeanPostProcessor implements InstantiationAwareBeanPostProcessor, BeanFactoryAware {

    private ConfigurableListableBeanFactory beanFactory;

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = (ConfigurableListableBeanFactory) beanFactory;
    }

    @Override
    public PropertyValues postProcessPropertyValues(PropertyValues pvs, Object bean, String beanName) throws BeansException {
        // 1. 处理注解 @Value
        Class<?> clazz = bean.getClass();
        clazz = ClassUtils.isCglibProxyClass(clazz) ? clazz.getSuperclass() : clazz;

        Field[] declaredFields = clazz.getDeclaredFields();

        for (Field field : declaredFields) {
            Value valueAnnotation = field.getAnnotation(Value.class);
            if (null != valueAnnotation) {
                String value = valueAnnotation.value();
                value = beanFactory.resolveEmbeddedValue(value);
                BeanUtil.setFieldValue(bean, field.getName(), value);
            }
        }

        // 2. 处理注解 @Autowired
        for (Field field : declaredFields) {
            Autowired autowiredAnnotation = field.getAnnotation(Autowired.class);
            if (null != autowiredAnnotation) {
                Class<?> fieldType = field.getType();
                String dependentBeanName = null;
                Qualifier qualifierAnnotation = field.getAnnotation(Qualifier.class);
                Object dependentBean = null;
                if (null != qualifierAnnotation) {
                    dependentBeanName = qualifierAnnotation.value();
                    dependentBean = beanFactory.getBean(dependentBeanName, fieldType);
                } else {
                    dependentBean = beanFactory.getBean(fieldType);
                }
                BeanUtil.setFieldValue(bean, field.getName(), dependentBean);
            }
        }

        return pvs;
    }

}
```
- `AutowiredAnnotationBeanPostProcessor` 是实现接口 `InstantiationAwareBeanPostProcessor` 的一个用于在 Bean 对象实例化完成后，设置属性操作前的处理属性信息的类和操作方法。只有实现了 BeanPostProcessor 接口才有机会在 Bean 的生命周期中处理初始化信息
- 核心方法 postProcessPropertyValues，主要用于处理类含有 @Value、@Autowired 注解的属性，进行属性信息的提取和设置。
- 这里需要注意一点因为我们在 `AbstractAutowireCapableBeanFactory` 类中使用的是 `CglibSubclassingInstantiationStrategy` 进行类的创建，所以在 `AutowiredAnnotationBeanPostProcessor#postProcessPropertyValues` 中需要判断是否为 CGlib 创建对象，否则是不能正确拿到类信息的。`ClassUtils.isCglibProxyClass(clazz) ? clazz.getSuperclass() : clazz;`
## 在 Bean 的生命周期中调用属性注入
```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory implements AutowireCapableBeanFactory {

    private InstantiationStrategy instantiationStrategy = new CglibSubclassingInstantiationStrategy();

    @Override
    protected Object createBean(String beanName, BeanDefinition beanDefinition, Object[] args) throws BeansException {
        Object bean = null;
        try {
            // 判断是否返回代理 Bean 对象
            bean = resolveBeforeInstantiation(beanName, beanDefinition);
            if (null != bean) {
                return bean;
            }
            // 实例化 Bean
            bean = createBeanInstance(beanDefinition, beanName, args);
            // 在设置 Bean 属性之前，允许 BeanPostProcessor 修改属性值
            applyBeanPostProcessorsBeforeApplyingPropertyValues(beanName, bean, beanDefinition);
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
            registerSingleton(beanName, bean);
        }
        return bean;
    }

    /**
     * 在设置 Bean 属性之前，允许 BeanPostProcessor 修改属性值
     *
     * @param beanName
     * @param bean
     * @param beanDefinition
     */
    protected void applyBeanPostProcessorsBeforeApplyingPropertyValues(String beanName, Object bean, BeanDefinition beanDefinition) {
        for (BeanPostProcessor beanPostProcessor : getBeanPostProcessors()) {
            if (beanPostProcessor instanceof InstantiationAwareBeanPostProcessor){
                PropertyValues pvs = ((InstantiationAwareBeanPostProcessor) beanPostProcessor).postProcessPropertyValues(beanDefinition.getPropertyValues(), bean, beanName);
                if (null != pvs) {
                    for (PropertyValue propertyValue : pvs.getPropertyValues()) {
                        beanDefinition.getPropertyValues().addPropertyValue(propertyValue);
                    }
                }
            }
        }
    }  

    // ...
}
```
- AbstractAutowireCapableBeanFactory#createBean 方法中有这一条新增加的方法调用，就是在`设置 Bean 属性之前，允许 BeanPostProcessor 修改属性值` 的操作 `applyBeanPostProcessorsBeforeApplyingPropertyValues`
- 那么这个 applyBeanPostProcessorsBeforeApplyingPropertyValues 方法中，首先就是获取已经注入的 BeanPostProcessor 集合并从中筛选出继承接口 InstantiationAwareBeanPostProcessor 的实现类。
- 最后就是调用相应的 postProcessPropertyValues 方法以及循环设置属性值信息，`beanDefinition.getPropertyValues().addPropertyValue(propertyValue);`
