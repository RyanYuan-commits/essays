---
created: 2025-09-27 22:34:06
updated: 2025-09-27 22:34:06
---
# 目标
其实到本章节我们已经把关于 IOC 和 AOP 全部核心内容都已经实现完成了，只不过在使用上还有点像早期的 Spring 版本，需要一个一个在 spring.xml 中进行配置。这与实际的目前使用的 Spring 框架还是有蛮大的差别，而这种差别其实都是在核心功能逻辑之上建设的在更少的配置下，做到更简化的使用。

这其中就包括：包的扫描注册、注解配置的使用、占位符属性的填充等等，而我们的目标就是在目前的核心逻辑上填充一些自动化的功能，让大家可以学习到这部分的设计和实现，从中体会到一些关于代码逻辑的实现过程，总结一些编码经验。
# 方案
首先我们要考虑🤔，为了可以简化 Bean 对象的配置，让整个 Bean 对象的注册都是自动扫描的，那么基本需要的元素包括：扫描路径入口、XML解析扫描信息、给需要扫描的Bean对象做注解标记、扫描Class对象摘取Bean注册的基本信息，组装注册信息、注册成Bean对象。
那么在这些条件元素的支撑下，就可以实现出通过自定义注解和配置扫描路径的情况下，完成 Bean 对象的注册。
除此之外再顺带解决一个配置中占位符属性的知识点，比如可以通过 `${token}` 给 Bean 对象注入进去属性信息，那么这个操作需要用到 BeanFactoryPostProcessor，因为它可以处理 **在所有的 BeanDefinition 加载完成后，实例化 Bean 对象之前，提供修改 BeanDefinition 属性的机制** 而实现这部分内容是为了后续把此类内容结合到自动化配置处理中。
![[👣【small-spring】自动扫描 Bean 注册 架构图.png]]
结合bean的生命周期，包扫描只不过是扫描特定注解的类，提取类的相关信息组装成BeanDefinition注册到容器中。
在XmlBeanDefinitionReader中解析`<context:component-scan />`标签，扫描类组装BeanDefinition然后注册到容器中的操作在ClassPathBeanDefinitionScanner#doScan中实现。
- 自动扫描注册主要是扫描添加了自定义注解的类，在xml加载过程中提取类的信息，组装 BeanDefinition 注册到 Spring 容器中。
- 所以我们会用到 `<context:component-scan />` 配置包路径并在 XmlBeanDefinitionReader 解析并做相应的处理。
- 最后还包括了一部分关于 `BeanFactoryPostProcessor` 的使用，因为我们需要完成对占位符配置信息的加载，所以需要使用到 BeanFactoryPostProcessor 在所有的 BeanDefinition 加载完成后，实例化 Bean 对象之前，修改 BeanDefinition 的属性信息。_这一部分的实现也为后续处理关于占位符配置到注解上做准备_
# 实现
![[【small-spring】自动扫描 Bean 注册 核心类图.png]]
- 整个类的关系结构来看，其实涉及的内容并不多，主要包括的就是 xml 解析类 `XmlBeanDefinitionReader` 对 `ClassPathBeanDefinitionScanner#doScan` 的使用。
- 在 `doScan` 方法中处理所有指定路径下添加了注解的类，拆解出类的信息：名称、作用范围等，进行创建 `BeanDefinition` 好用于 Bean 对象的注册操作。
- `PropertyPlaceholderConfigurer` 目前看上去像一块单独的内容，后续会把这块的内容与自动加载 Bean 对象进行整合，也就是可以在注解上使用占位符配置一些在配置文件里的属性信息。
## 处理占位符配置
```java
public class PropertyPlaceholderConfigurer implements BeanFactoryPostProcessor {

    /**
     * Default placeholder prefix: {@value}
     */
    public static final String DEFAULT_PLACEHOLDER_PREFIX = "${";

    /**
     * Default placeholder suffix: {@value}
     */
    public static final String DEFAULT_PLACEHOLDER_SUFFIX = "}";

    private String location;

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        // 加载属性文件
        try {
            DefaultResourceLoader resourceLoader = new DefaultResourceLoader();
            Resource resource = resourceLoader.getResource(location);
            Properties properties = new Properties();
            properties.load(resource.getInputStream());

            String[] beanDefinitionNames = beanFactory.getBeanDefinitionNames();
            for (String beanName : beanDefinitionNames) {
                BeanDefinition beanDefinition = beanFactory.getBeanDefinition(beanName);

                PropertyValues propertyValues = beanDefinition.getPropertyValues();
                for (PropertyValue propertyValue : propertyValues.getPropertyValues()) {
                    Object value = propertyValue.getValue();
                    if (!(value instanceof String)) continue;
                    String strVal = (String) value;
                    StringBuilder buffer = new StringBuilder(strVal);
                    int startIdx = strVal.indexOf(DEFAULT_PLACEHOLDER_PREFIX);
                    int stopIdx = strVal.indexOf(DEFAULT_PLACEHOLDER_SUFFIX);
                    if (startIdx != -1 && stopIdx != -1 && startIdx < stopIdx) {
                        String propKey = strVal.substring(startIdx + 2, stopIdx);
                        String propVal = properties.getProperty(propKey);
                        buffer.replace(startIdx, stopIdx + 1, propVal);
                        propertyValues.addPropertyValue(new PropertyValue(propertyValue.getName(), buffer.toString()));
                    }
                }
            }
        } catch (IOException e) {
            throw new BeansException("Could not load properties", e);
        }
    }

    public void setLocation(String location) {
        this.location = location;
    }

}
```
- 依赖于 `BeanFactoryPostProcessor` 在 Bean 生命周期的属性，可以在 Bean 对象实例化之前，改变属性信息。所以这里通过实现 `BeanFactoryPostProcessor` 接口，完成对配置文件的加载以及摘取占位符中的在属性文件里的配置。
- 这样就可以把提取到的配置信息放置到属性配置中了，`buffer.replace(startIdx, stopIdx + 1, propVal); propertyValues.addPropertyValue`
## 定义注解
```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Scope {

    String value() default "singleton";

}
```
- 用于配置作用域的自定义注解，方便通过配置Bean对象注解的时候，拿到Bean对象的作用域。不过一般都使用默认的 singleton。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Component {

    String value() default "";

}
```
- Component 自定义注解大家都非常熟悉了，用于配置到 Class 类上的。除此之外还有 Service、Controller，不过所有的处理方式基本一致，这里就只展示一个 Component 即可。

## 处理对象扫描装配
```java
public class ClassPathScanningCandidateComponentProvider {

    public Set<BeanDefinition> findCandidateComponents(String basePackage) {
        Set<BeanDefinition> candidates = new LinkedHashSet<>();
        Set<Class<?>> classes = ClassUtil.scanPackageByAnnotation(basePackage, Component.class);
        for (Class<?> clazz : classes) {
            candidates.add(new BeanDefinition(clazz));
        }
        return candidates;
    }

}
```
- 这里先要提供一个可以通过配置路径 `basePackage=cn.bugstack.springframework.test.bean`，解析出 classes 信息的工具方法 findCandidateComponents，通过这个方法就可以扫描到所有 @Component 注解的 Bean 对象了。

```java
public class ClassPathBeanDefinitionScanner extends ClassPathScanningCandidateComponentProvider {

    private BeanDefinitionRegistry registry;

    public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry) {
        this.registry = registry;
    }

    public void doScan(String... basePackages) {
        for (String basePackage : basePackages) {
            Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
            for (BeanDefinition beanDefinition : candidates) {
                // 解析 Bean 的作用域 singleton、prototype
                String beanScope = resolveBeanScope(beanDefinition);
                if (StrUtil.isNotEmpty(beanScope)) {
                    beanDefinition.setScope(beanScope);
                }
                registry.registerBeanDefinition(determineBeanName(beanDefinition), beanDefinition);
            }
        }
    }

    private String resolveBeanScope(BeanDefinition beanDefinition) {
        Class<?> beanClass = beanDefinition.getBeanClass();
        Scope scope = beanClass.getAnnotation(Scope.class);
        if (null != scope) return scope.value();
        return StrUtil.EMPTY;
    }

    private String determineBeanName(BeanDefinition beanDefinition) {
        Class<?> beanClass = beanDefinition.getBeanClass();
        Component component = beanClass.getAnnotation(Component.class);
        String value = component.value();
        if (StrUtil.isEmpty(value)) {
            value = StrUtil.lowerFirst(beanClass.getSimpleName());
        }
        return value;
    }

}
```
- ClassPathBeanDefinitionScanner 是继承自 ClassPathScanningCandidateComponentProvider 的具体扫描包处理的类，在 doScan 中除了获取到扫描的类信息以后，还需要获取 Bean 的作用域和类名，如果不配置类名基本都是把首字母缩写。
- 这个类提供给外界的接口为：`doScan`，通过将扫描路径传入来实现。
## 解析 XML 中的路径调用扫描
```java
public class XmlBeanDefinitionReader extends AbstractBeanDefinitionReader {

    protected void doLoadBeanDefinitions(InputStream inputStream) throws ClassNotFoundException, DocumentException {
        SAXReader reader = new SAXReader();
        Document document = reader.read(inputStream);
        Element root = document.getRootElement();

        // 解析 context:component-scan 标签，扫描包中的类并提取相关信息，用于组装 BeanDefinition
        Element componentScan = root.element("component-scan");
        if (null != componentScan) {
            String scanPath = componentScan.attributeValue("base-package");
            if (StrUtil.isEmpty(scanPath)) {
                throw new BeansException("The value of base-package attribute can not be empty or null");
            }
            scanPackage(scanPath);
        }
       
        // ... 省略其他
            
        // 注册 BeanDefinition
        getRegistry().registerBeanDefinition(beanName, beanDefinition);
    }

    private void scanPackage(String scanPath) {
        String[] basePackages = StrUtil.splitToArray(scanPath, ',');
        ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(getRegistry());
        scanner.doScan(basePackages);
    }

}
```
- 关于 XmlBeanDefinitionReader 中主要是在加载配置文件后，处理新增的自定义配置属性 `component-scan`，解析后调用 scanPackage 方法，其实也就是我们在 ClassPathBeanDefinitionScanner#doScan 功能。
	- 另外这里需要注意，为了可以方便的加载和解析xml，XmlBeanDefinitionReader 已经全部替换为 dom4j 的方式进行解析处理。