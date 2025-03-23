Mapper 接口如何执行方法
## 目标
比如我们使用 JDBC 的时候，需要手动建立数据库链接、编码 SQL 语句、执行数据库操作、自己封装返回结果等。
但在使用 ORM 框架后，只需要通过简单配置即可对定义的 DAO 接口进行数据库的操作了。
那么本章节我们就来解决 ORM 框架第一个 **关联对象接口和映射类** 的问题，把 DAO 接口使用代理类，包装映射操作。
## 设计
通常如果能找到大家所在事情的共性内容，具有统一的流程处理，那么它就是可以被凝聚和提炼的，做成通用的组件或者服务，被所有人进行使用，减少重复的人力投入。
而参考我们最开始使用 JDBC 的方式，从连接、查询、封装、返回，其实都一个固定的流程，那么这个过程就可以被提炼以及封装和补全大家所需要的功能。
当我们来设计一个 ORM 框架的过程中，首先要考虑 **怎么把用户定义的数据库操作接口、xml配置的SQL语句、数据库三者联系起来**。
其实最适合的操作就是使用代理的方式进行处理，因为代理可以封装一个复杂的流程为接口对象的实现类：
![[创建简单的映射器代理工厂 设计图.png]]


- 首先提供一个映射器的代理实现类 `MapperProxy`，通过代理类包装对数据库的操作，目前我们本章节会先提供一个简单的包装，模拟对数据库的调用。
- 之后对 `MapperProxy` 代理类，提供工厂实例化操作 MapperProxyFactory#newInstance，为每个 IDAO 接口生成代理类。
## 代码实现
![[【small-mybatis】创建简单的映射器代理工厂 核心类图.png]]
- 目前这个 Mybatis 框架的代理操作实现的还只是最核心的功能，相当于是光屁股的娃娃，还没有添加衣服。不过这样渐进式的实现可以让大家先了解到最核心的内容，后续我们在陆续的完善。
- MapperProxy 负责实现 InvocationHandler 接口的 invoke 方法，最终所有的实际调用都会调用到这个方法包装的逻辑。
- MapperProxyFactory 是对 MapperProxy 的包装，对外提供实例化对象的操作。当我们后面开始给每个操作数据库的接口映射器注册代理的时候，就需要使用到这个工厂类了。
###  Mapper 接口代理类
```java
public class MapperProxy<T> implements InvocationHandler, Serializable {  
    private static final long serialVersionUID = -6424540398559729838L;  
  
    private Map<String, String> sqlSession;  
    private final Class<T> mapperInterface;  
  
    public MapperProxy(Map<String, String> sqlSession, Class<T> mapperInterface) {  
        this.sqlSession = sqlSession;  
        this.mapperInterface = mapperInterface;  
    }  
  
    @Override  
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {  
        if (Object.class.equals(method.getDeclaringClass())) {  
            return method.invoke(this, args);  
        } else {  
            return "你的被代理了！" + sqlSession.get(mapperInterface.getName() + "." + method.getName());  
        }  
    }  
}
```
### 代理工厂
```java
public class MapperProxyFactory<T> {  
    private final Class<T> mapperInterface;  
  
    public MapperProxyFactory(Class<T> mapperInterface) {  
        this.mapperInterface = mapperInterface;  
    }  
  
    public T newInstance(Map<String, String> sqlSession) {  
        final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface);  
        return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[]{mapperInterface}, mapperProxy);  
    }  
}
```
### 测试
```java
@Test  
public void test_proxy_class() {  
    IUserDao userDao = (IUserDao) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(),   
            new Class[]{IUserDao.class}, (proxy, method, args) -> "你被代理了！");  
    String result = userDao.queryUserName("10001");  
    System.out.println("测试结果：" + result);  
}
```
输出结果：
```
测试结果：你被代理了！
```

```java
@Test  
public void test_MapperProxyFactory() {  
    MapperProxyFactory<IUserDao> factory = new MapperProxyFactory<>(IUserDao.class);  
  
    Map<String, String> sqlSession = new HashMap<>();  
    sqlSession.put("cn.bugstack.mybatis.test.dao.IUserDao.queryUserName", "模拟执行 Mapper.xml 中 SQL 语句的操作：查询用户姓名");  
    sqlSession.put("cn.bugstack.mybatis.test.dao.IUserDao.queryUserAge", "模拟执行 Mapper.xml 中 SQL 语句的操作：查询用户年龄");  
    IUserDao userDao = factory.newInstance(sqlSession);  
  
    String res = userDao.queryUserName("10001");  
    logger.info("测试结果：{}", res);  
}
```
输出结果：
```
17:22:23.848 [main] INFO  cn.bugstack.mybatis.test.ApiTest - 测试结果：你的被代理了！模拟执行 Mapper.xml 中 SQL 语句的操作：查询用户姓名
```
