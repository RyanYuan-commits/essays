## 什么是 Bean 管理？
Spring 包含并管理应用对象的配置和生命周期，在这个意义上它是一种用于承载对象的容器，你可以配置你的每个 Bean 对象是如何被创建的，这些 Bean 可以创建一个单独的实例或者每次需要时都生成一个新的实例，以及它们是如何相互关联构建和使用的。
如果一个 Bean 对象交给 Spring 容器管理，那么这个 Bean 对象就应该以类似零件的方式被拆解后存放到 Bean 的定义中，这样相当于一种把对象解耦的操作，可以由 Spring 更加容易的管理，就像处理循环依赖等操作。
当一个 Bean 对象被定义存放以后，再由 Spring 统一进行装配，这个过程包括 Bean 的初始化、属性填充等，最终我们就可以完整的使用一个 Bean 实例化后的对象了。
而我们本章节的案例目标就是定义一个简单的 Spring 容器，用于定义、存放和获取 Bean 对象。
## 设计
凡是可以存放数据的具体数据结构实现，都可以称之为容器。例如：ArrayList、LinkedList、HashSet等，但在 Spring Bean 容器的场景下，我们需要一种可以用于存放和名称索引式的数据结构，所以选择 HashMap 是最合适不过的。
这里简单介绍一下 HashMap，HashMap 是一种基于扰动函数、负载因子、红黑树转换等技术内容，形成的拉链寻址的数据结构，它能让数据更加散列的分布在哈希桶以及碰撞时形成的链表和红黑树上。它的数据结构会尽可能最大限度的让整个数据读取的复杂度在 O(1) ~ O(Logn) ~O(n)之间，当然在极端情况下也会有 O(n) 链表查找数据较多的情况。不过我们经过10万数据的扰动函数再寻址验证测试，数据会均匀的散列在各个哈希桶索引上，所以 HashMap 非常适合用在 Spring Bean 的容器实现上。
另外一个简单的 Spring Bean 容器实现，还需 Bean 的定义、注册、获取三个基本步骤，简化设计如下；
![[简单 Bean 工厂图例.png]]

- 定义：BeanDefinition，可能这是你在查阅 Spring 源码时经常看到的一个类，例如它会包括 singleton、prototype、BeanClassName 等。但目前我们初步实现会更加简单的处理，只定义一个 Object 类型用于存放对象。
- 注册：这个过程就相当于我们把数据存放到 HashMap 中，只不过现在 HashMap 存放的是定义了的 Bean 的对象信息。
- 获取：最后就是获取对象，Bean 的名字就是key，Spring 容器初始化好 Bean 以后，就可以直接获取了。
接下来我们就按照这个设计，做一个简单的 Spring Bean 容器代码实现。编码的过程往往并不会有多复杂，但知晓设计过程却更加重要！
### 工程结构
```java
small-spring-step-01
└── src
    ├── main
    │   └── java
    │       └── cn.bugstack.springframework
    │           ├── BeanDefinition.java
    │           └── BeanFactory.java
    └── test
        └── java
            └── cn.bugstack.springframework.test  
                ├── bean
                │   └── UserService.java                
                └── ApiTest.java
```
### 代码概览
**1）Bean 工厂**
```java
public class BeanFactory {

    private Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>();

    public Object getBean(String name) {
        return beanDefinitionMap.get(name).getBean();
    }

    public void registerBeanDefinition(String name, BeanDefinition beanDefinition) {
        beanDefinitionMap.put(name, beanDefinition);
    }

}
```
**2）Bean 定义**
```java
public class BeanDefinition {

    private Object bean;

    public BeanDefinition(Object bean) {
        this.bean = bean;
    }

    public Object getBean() {
        return bean;
    }

}
```
**3）测试用例**
```java
@Test
public void test_BeanFactory(){
    // 1.初始化 BeanFactory
    BeanFactory beanFactory = new BeanFactory();
    
    // 2.注册 bean
    BeanDefinition beanDefinition = new BeanDefinition(new UserService());
    beanFactory.registerBeanDefinition("userService", beanDefinition);
    
    // 3.获取 bean
    UserService userService = (UserService) beanFactory.getBean("userService");
    userService.queryUserInfo();
}
```

---

Next -> [[🧩【small-spring】实现 Bean 的注册与获取]]