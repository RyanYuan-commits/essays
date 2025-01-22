# 目标
首先我们回顾下这几章节都完成了什么，包括：实现一个容器、定义和注册 Bean、实例化 Bean、按照是否包含构造函数实现不同的实例化策略，那么在创建对象实例化这我们还缺少什么？其实还缺少一个关于`类中是否有属性的问题`，如果有类中包含属性那么在实例化的时候就需要把属性信息填充上，这样才是一个完整的对象创建。

对于属性的填充不只是 int、Long、String，还包括还没有实例化的对象属性，都需要在 Bean 创建时进行填充操作。不过这里我们暂时不会考虑 Bean 的循环依赖，否则会把整个功能实现撑大，这样新人学习时就把握不住了，待后续陆续先把核心功能实现后，再逐步完善。

循环依赖（Circular Dependency）指的是在一个系统或组件中，两个或多个对象之间相互依赖，形成一个循环的依赖关系。这种依赖关系是相互的，每个对象都依赖于另一个对象，直接或间接地导致它们无法正确地初始化或使用。
# 设计
鉴于属性填充是在 Bean 使用 `newInstance` 或者 `Cglib` 创建后，开始补全属性信息，那么就可以在类 `AbstractAutowireCapableBeanFactory` 的 createBean 方法中添加补全属性方法。

![https://bugstack.cn/assets/images/spring/spring-5-01.png](https://bugstack.cn/assets/images/spring/spring-5-01.png)

- 属性填充要在类实例化创建之后，也就是需要在 `AbstractAutowireCapableBeanFactory` 的 `createBean` 方法中添加 `applyPropertyValues` 操作。
- 由于我们需要在创建Bean时候填充属性操作，那么就需要在 bean 定义 `BeanDefinition` 类中，添加 `PropertyValues` 信息。
- 另外是填充属性信息还包括了 Bean 的对象类型，也就是需要再定义一个 `BeanReference`，里面其实就是一个简单的 Bean 名称，在具体的实例化操作时进行递归创建和填充，与 Spring 源码实现一样。
# 代码实现
`BeanReference` Bean 的引用，指定当前 Bean 引用了名称为 xxx 的 Bean
```java
public class BeanReference {

    private final String beanName;

    public BeanReference(String beanName) {
        this.beanName = beanName;
    }

    public String getBeanName() {
        return beanName;
    }

}
```

`PropertyValue` 、`PropertyValues` 当前 Bean 的属性
```java
public class PropertyValue {

    private final String name;

    private final Object value;

    public PropertyValue(String name, Object value) {
        this.name = name;
        this.value = value;
    }

    public String getName() {
        return name;
    }

    public Object getValue() {
        return value;
    }

}

public class PropertyValues {

    private final List<PropertyValue> propertyValueList = new ArrayList<>();

    public void addPropertyValue(PropertyValue pv) {
        this.propertyValueList.add(pv);
    }

    public PropertyValue[] getPropertyValues() {
        return this.propertyValueList.toArray(new PropertyValue[0]);
    }

    public PropertyValue getPropertyValue(String propertyName) {
        for (PropertyValue pv : this.propertyValueList) {
            if (pv.getName().equals(propertyName)) {
                return pv;
            }
        }
        return null;
    }

}
```

`AbstractAutowireCapableBeanFactory` 中新增 `applyPropertyValues(String beanName, Object bean, BeanDefinition beanDefinition)` 方法来填充 Bean 的依赖数据
```java
   /**
     * Bean 属性填充
     */
    protected void applyPropertyValues(String beanName, Object bean, BeanDefinition beanDefinition) {
        try {
            PropertyValues propertyValues = beanDefinition.getPropertyValues();
            for (PropertyValue propertyValue : propertyValues.getPropertyValues()) {

                String name = propertyValue.getName();
                Object value = propertyValue.getValue();

                if (value instanceof BeanReference) {
                    // A 依赖 B，获取 B 的实例化
                    BeanReference beanReference = (BeanReference) value;
                    value = getBean(beanReference.getBeanName());
                }
                // 属性填充
                BeanUtil.setFieldValue(bean, name, value);
            }
        } catch (Exception e) {
            throw new BeansException("Error setting property values：" + beanName);
        }
    }
    
    @Override
    protected Object createBean(String beanName, BeanDefinition beanDefinition, Object[] args) throws BeansException {
        Object bean = null;
        try {
            bean = createBeanInstance(beanDefinition, beanName, args);
            // 给 Bean 填充属性
            applyPropertyValues(beanName, bean, beanDefinition);
        } catch (Exception e) {
            throw new BeansException("Instantiation of bean failed", e);
        }

        addSingleton(beanName, bean);
        return bean;
    }
```
# 测试案例
```java
@Test  
public void test_BeanFactory() {  
    // 1.初始化 BeanFactory    DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();  
  
    // 2. UserDao 注册  
    beanFactory.registerBeanDefinition("userDao", new BeanDefinition(UserDao.class));  
  
    // 3. UserService 设置属性[uId、userDao]  
    PropertyValues propertyValues = new PropertyValues();  
    propertyValues.addPropertyValue(new PropertyValue("uId", "10001"));  
    propertyValues.addPropertyValue(new PropertyValue("userDao", new BeanReference("userDao")));  
  
    // 4. UserService 注入bean  
    BeanDefinition beanDefinition = new BeanDefinition(UserService.class, propertyValues);  
    beanFactory.registerBeanDefinition("userService", beanDefinition);  
  
    // 5. UserService 获取bean  
    UserService userService = (UserService) beanFactory.getBean("userService");  
    String result = userService.queryUserInfo();  
    System.out.println("测试结果：" + result);  
}
```

测试结果
```
测试结果：ryan
```