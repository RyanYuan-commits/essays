# 目标
在前面我们已经实现了 BeanDefination 的注册和 Bean 对象的获取，并且实现了通过 CgLib 来生成代理对象；来看一下 BeanDefinition 的注册是如何实现的：
```java
    @Test
    public void test_BeanFactory() {
        // 1.初始化 BeanFactory
        DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();

        // 2. UserDao 注册
        beanFactory.registerBeanDefinition("userDao", new BeanDefinition(UserDao.clazss));

        // 3. UserService 设置属性[uId、userDao]
        PropertyValues propertyValues = new PropertyValues();
        propertyValues.addPropertyValue(new PropertyValue("uId", "10001"));
        propertyValues.addPropertyValue(new PropertyValue("userDao",new BeanReference("userDao")));

        // 4. UserService 注入bean
        BeanDefinition beanDefinition = new BeanDefinition(UserService.class, propertyValues);
        beanFactory.registerBeanDefinition("userService", beanDefinition);

        // 5. UserService 获取bean
        UserService userService = (UserService) beanFactory.getBean("userService");
        userService.queryUserInfo();
    }
```
这样显然并不优雅，所以本部分就要学习像 Spring 那样，将属性和依赖的配置写在 xml 配置文件中。
# 工程结构
```java
small-spring-step-05
└── src
    ├── main
    │   └── java
    │       └── cn.bugstack.springframework
    │           ├── beans
    │           │   ├── factory
    │           │   │   ├── config
    │           │   │   │   ├── AutowireCapableBeanFactory.java
    │           │   │   │   ├── BeanDefinition.java
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
    │           │   │   ├── xml
    │           │   │   │   └── XmlBeanDefinitionReader.java
    │           │   │   ├── BeanFactory.java
    │           │   │   ├── ConfigurableListableBeanFactory.java
    │           │   │   ├── HierarchicalBeanFactory.java
    │           │   │   └── ListableBeanFactory.java
    │           │   ├── BeansException.java
    │           │   ├── PropertyValue.java
    │           │   └── PropertyValues.java
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
                └── ApiTest.java
```
![[资源加载器解析文件对象架构图.png|900]]
- 本章节为了能把 Bean 的定义、注册和初始化交给 Spring.xml 配置化处理，那么就需要实现两大块内容，分别是：资源加载器、xml资源处理类，实现过程主要以对接口 `Resource`、`ResourceLoader` 的实现，而另外 `BeanDefinitionReader` 接口则是对资源的具体使用，将配置信息注册到 Spring 容器中去。
- 在 Resource 的资源加载器的实现中包括了，ClassPath、系统文件、云配置文件，这三部分与 Spring 源码中的设计和实现保持一致，最终在 DefaultResourceLoader 中做具体的调用。
- 接口：BeanDefinitionReader、抽象类：AbstractBeanDefinitionReader、实现类：XmlBeanDefinitionReader，这三部分内容主要是合理清晰的处理了资源读取后的注册 Bean 容器操作。接口管定义，抽象类处理非接口功能外的注册Bean组件填充，最终实现类即可只关心具体的业务实现
# 代码实现
## 资源加载接口的定义和实现
```java
public interface Resource {

    InputStream getInputStream() throws IOException;

}
```
- 在 Spring 框架下创建 core.io 核心包，在这个包中主要用于处理资源加载流。
- 定义 Resource 接口，提供获取 InputStream 流的方法，接下来再分别实现三种不同的流文件操作：classPath、FileSystem、URL

**ClassPath**：类路径
```java
public class ClassPathResource implements Resource {

    private final String path;

    private ClassLoader classLoader;

    public ClassPathResource(String path) {
        this(path, (ClassLoader) null);
    }

    public ClassPathResource(String path, ClassLoader classLoader) {
        Assert.notNull(path, "Path must not be null");
        this.path = path;
        this.classLoader = (classLoader != null ? classLoader : ClassUtils.getDefaultClassLoader());
    }

    @Override
    public InputStream getInputStream() throws IOException {
        InputStream is = classLoader.getResourceAsStream(path);
        if (is == null) {
            throw new FileNotFoundException(
                    this.path + " cannot be opened because it does not exist");
        }
        return is;
    }
}
```
- 这一部分的实现是用于通过 `ClassLoader` 读取`ClassPath` 下的文件信息，具体的读取过程主要是：`classLoader.getResourceAsStream(path)`

**FileSystem**
```java
public class FileSystemResource implements Resource {

    private final File file;

    private final String path;

    public FileSystemResource(File file) {
        this.file = file;
        this.path = file.getPath();
    }

    public FileSystemResource(String path) {
        this.file = new File(path);
        this.path = path;
    }

    @Override
    public InputStream getInputStream() throws IOException {
        return new FileInputStream(this.file);
    }

    public final String getPath() {
        return this.path;
    }

}
```
- 通过指定文件路径的方式读取文件信息，这部分大家肯定还是非常熟悉的，经常会读取一些txt、excel文件输出到控制台。

**Url**
```java
public class UrlResource implements Resource{

    private final URL url;

    public UrlResource(URL url) {
        Assert.notNull(url,"URL must not be null");
        this.url = url;
    }

    @Override
    public InputStream getInputStream() throws IOException {
        URLConnection con = this.url.openConnection();
        try {
            return con.getInputStream();
        }
        catch (IOException ex){
            if (con instanceof HttpURLConnection){
                ((HttpURLConnection) con).disconnect();
            }
            throw ex;
        }
    }

}
```
- 通过 HTTP 的方式读取云服务的文件，我们也可以把配置文件放到 GitHub 或者 Gitee 上。

## 包装资源加载器
按照资源加载的不同方式，资源加载器可以把这些方式集中到统一的类服务下进行处理，外部用户只需要传递资源地址即可，简化使用。
```java
public interface ResourceLoader {

    /**
     * Pseudo URL prefix for loading from the class path: "classpath:"
     */
    String CLASSPATH_URL_PREFIX = "classpath:";

    Resource getResource(String location);

}
```
- 定义获取资源接口，里面传递 location 地址即可。

实现 `ResourceLoader` 接口：`DefaultResourceLoader`：
```java
public class DefaultResourceLoader implements ResourceLoader {

    @Override
    public Resource getResource(String location) {
        Assert.notNull(location, "Location must not be null");
        if (location.startsWith(CLASSPATH_URL_PREFIX)) {
            return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()));
        }
        else {
            try {
                URL url = new URL(location);
                return new UrlResource(url);
            } catch (MalformedURLException e) {
                return new FileSystemResource(location);
            }
        }
    }

}
```
- 在获取资源的实现中，主要是把三种不同类型的资源处理方式进行了包装，分为：判断是否为ClassPath、URL以及文件。
- 虽然 DefaultResourceLoader 类实现的过程简单，但这也是设计模式约定的具体结果，像是这里不会让外部调用放知道过多的细节，而是仅关心具体调用结果即可。
## BeanDefinition 读取接口
```java
public interface BeanDefinitionReader {
    BeanDefinitionRegistry getRegistry();
    
    ResourceLoader getResourceLoader();
    
    void loadBeanDefinitions(Resource resource) throws BeansException;
    
    void loadBeanDefinitions(Resource... resources) throws BeansException;
    
    void loadBeanDefinitions(String location) throws BeansException;
}
```
- 这是一个 _Simple interface for bean definition readers._ 其实里面无非定义了几个方法，包括：getRegistry()、getResourceLoader()，以及三个加载Bean定义的方法。
- 这里需要注意 getRegistry()、getResourceLoader()，都是用于提供给后面三个方法的工具，加载和注册，这两个方法的实现会包装到抽象类中，以免污染具体的接口实现方法。
## 读取接口的抽象类实现
```java
public abstract class AbstractBeanDefinitionReader implements BeanDefinitionReader {

    private final BeanDefinitionRegistry registry;

    private ResourceLoader resourceLoader;

    protected AbstractBeanDefinitionReader(BeanDefinitionRegistry registry) {
        this(registry, new DefaultResourceLoader());
    }

    public AbstractBeanDefinitionReader(BeanDefinitionRegistry registry, ResourceLoader resourceLoader) {
        this.registry = registry;
        this.resourceLoader = resourceLoader;
    }

    @Override
    public BeanDefinitionRegistry getRegistry() {
        return registry;
    }

    @Override
    public ResourceLoader getResourceLoader() {
        return resourceLoader;
    }

}
```
## 解析 XML 处理 Bean Definition
```java
public class XmlBeanDefinitionReader extends AbstractBeanDefinitionReader {

    public XmlBeanDefinitionReader(BeanDefinitionRegistry registry) {
        super(registry);
    }

    public XmlBeanDefinitionReader(BeanDefinitionRegistry registry, ResourceLoader resourceLoader) {
        super(registry, resourceLoader);
    }

    @Override
    public void loadBeanDefinitions(Resource resource) throws BeansException {
        try {
            try (InputStream inputStream = resource.getInputStream()) {
                doLoadBeanDefinitions(inputStream);
            }
        } catch (IOException | ClassNotFoundException e) {
            throw new BeansException("IOException parsing XML document from " + resource, e);
        }
    }

    @Override
    public void loadBeanDefinitions(Resource... resources) throws BeansException {
        for (Resource resource : resources) {
            loadBeanDefinitions(resource);
        }
    }

    @Override
    public void loadBeanDefinitions(String location) throws BeansException {
        ResourceLoader resourceLoader = getResourceLoader();
        Resource resource = resourceLoader.getResource(location);
        loadBeanDefinitions(resource);
    }

    protected void doLoadBeanDefinitions(InputStream inputStream) throws ClassNotFoundException {
        Document doc = XmlUtil.readXML(inputStream);
        Element root = doc.getDocumentElement();
        NodeList childNodes = root.getChildNodes();

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
            // 获取 Class，方便获取类中的名称
            Class<?> clazz = Class.forName(className);
            // 优先级 id > name
            String beanName = StrUtil.isNotEmpty(id) ? id : name;
            if (StrUtil.isEmpty(beanName)) {
                beanName = StrUtil.lowerFirst(clazz.getSimpleName());
            }

            // 定义Bean
            BeanDefinition beanDefinition = new BeanDefinition(clazz);
            // 读取属性并填充
            for (int j = 0; j < bean.getChildNodes().getLength(); j++) {
                if (!(bean.getChildNodes().item(j) instanceof Element)) continue;
                if (!"property".equals(bean.getChildNodes().item(j).getNodeName())) continue;
                // 解析标签：property
                Element property = (Element) bean.getChildNodes().item(j);
                String attrName = property.getAttribute("name");
                String attrValue = property.getAttribute("value");
                String attrRef = property.getAttribute("ref");
                // 获取属性值：引入对象、值对象
                Object value = StrUtil.isNotEmpty(attrRef) ? new BeanReference(attrRef) : attrValue;
                // 创建属性信息
                PropertyValue propertyValue = new PropertyValue(attrName, value);
                beanDefinition.getPropertyValues().addPropertyValue(propertyValue);
            }
            if (getRegistry().containsBeanDefinition(beanName)) {
                throw new BeansException("Duplicate beanName[" + beanName + "] is not allowed");
            }
            // 注册 BeanDefinition
            getRegistry().registerBeanDefinition(beanName, beanDefinition);
        }
    }

}


```
`XmlBeanDefinitionReader` 类最核心的内容就是对 XML 文件的解析，把我们本来在代码中的操作放到了通过解析 XML 自动注册的方式。
- `loadBeanDefinitions` 方法，处理资源加载，这里新增加了一个内部方法：`doLoadBeanDefinitions`，它主要负责解析 xml
- 在 `doLoadBeanDefinitions` 方法中，主要是对xml的读取 `XmlUtil.readXML(inputStream)` 和元素 Element 解析。在解析的过程中通过循环操作，以此获取 Bean 配置以及配置中的 id、name、class、value、ref 信息。
- 最终把读取出来的配置信息，创建成 `BeanDefinition` 以及 `PropertyValue`，最终把完整的 Bean 定义内容注册到 Bean 容器：`getRegistry().registerBeanDefinition(beanName, beanDefinition)`
# 测试
```java
@Test  
public void test_xml() {  
    // 1.初始化 BeanFactory    DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();  
  
    // 2. 读取配置文件&注册Bean  
    XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(beanFactory);  
    reader.loadBeanDefinitions("classpath:spring.xml");  
  
    // 3. 获取Bean对象调用方法  
    UserService userService = beanFactory.getBean("userService", UserService.class);  
    String result = userService.queryUserInfo();  
    System.out.println("测试结果：" + result);  
}
```

测试结果：
```
测试结果：Ryan
```