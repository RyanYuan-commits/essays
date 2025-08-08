## 目标
在上一章节我们初步的了解了怎么给一个接口类生成对应的映射器代理，并在代理中完成一些用户对接口方法的调用处理。
虽然我们已经看到了一个核心逻辑的处理方式，但在使用上还是有些刀耕火种的，包括：需要编码告知 `MapperProxyFactory` 要对哪个接口进行代理，以及自己编写一个假的 SqlSession 处理实际调用接口时的返回结果。
那么结合这两块问题点，我们本章节要对映射器的注册提供 `Registry` 处理，满足用户可以在使用的时候提供一个包的路径即可完成扫描和注册。
与此同时需要对 `SqlSession` 进行规范化处理，让它可以把我们的映射器代理和方法调用进行包装，建立一个生命周期模型结构，便于后续的内容的添加。
## 设计
鉴于我们希望把整个工程包下关于数据库操作的 `DAO` 接口与 `Mapper` 映射器关联起来，那么就需要包装一个可以扫描包路径的完成映射的注册器类。
当然我们还要把上一章节中简化的 `SqlSession` 进行完善，由 `SqlSession` 定义数据库处理接口和获取 `Mapper` 对象的操作，并把它交给映射器代理类进行使用。
有了 `SqlSession` 以后，你可以把它理解成一种功能服务，有了功能服务以后还需要给这个功能服务提供一个工厂，来对外统一提供这类服务。比如我们在 `Mybatis` 中非常常见的操作，开启一个 `SqlSession`。
![[【small-mybatis】实现 Mapper 的注册和使用 架构图.png|800]]
- 以包装接口提供映射器代理类为目标，补全映射器注册机 `MapperRegistry`，自动扫描包下接口并把每个接口类映射的代理类全部存入映射器代理的 `HashMap` 缓存中。
- 而 `SqlSession`、`SqlSessionFactory` 是在此注册映射器代理的上层使用标准定义和对外服务提供的封装，便于用户使用。
  我们把使用方当成用户经过这样的封装就就可以更加方便我们后续在框架上功能的继续扩展了，也希望大家可以在学习的过程中对这样的设计结构有一些思考，它可以帮助你解决一些业务功能开发过程中的领域服务包装。
## 实现
### 工程结构
![[【small-mybatis】实现 Mapper 的注册和使用 核心类图.png|800]]
- `MapperRegistry` 提供包路径的扫描和映射器代理类注册机服务，完成接口对象的代理类注册处理。
- `SqlSession`、`DefaultSqlSession` 用于定义执行 SQL 标准、获取映射器以及将来管理事务等方面的操作。基本我们平常使用 Mybatis 的 API 接口也都是从这个接口类定义的方法进行使用的。
- `SqlSessionFactory` 是一个简单工厂模式，用于提供 `SqlSession` 服务，屏蔽创建细节，延迟创建过程。
### Mapper Registry
```java
public class MapperRegistry {

    /**
     * 将已添加的映射器代理加入到 HashMap
     */
    private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<>();

    public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
        if (mapperProxyFactory == null) {
            throw new RuntimeException("Type " + type + " is not known to the MapperRegistry.");
        }
        try {
            return mapperProxyFactory.newInstance(sqlSession);
        } catch (Exception e) {
            throw new RuntimeException("Error getting mapper instance. Cause: " + e, e);
        }
    }

    public <T> void addMapper(Class<T> type) {
        /* Mapper 必须是接口才会注册 */
        if (type.isInterface()) {
            if (hasMapper(type)) {
                // 如果重复添加了，报错
                throw new RuntimeException("Type " + type + " is already known to the MapperRegistry.");
            }
            // 注册映射器代理工厂
            knownMappers.put(type, new MapperProxyFactory<>(type));
        }
    }

    public void addMappers(String packageName) {
        Set<Class<?>> mapperSet = ClassScanner.scanPackage(packageName);
        for (Class<?> mapperClass : mapperSet) {
            addMapper(mapperClass);
        }
    }

}
```
- MapperRegistry 映射器注册类的核心主要在于提供了 `ClassScanner.scanPackage` 扫描包路径，调用 `addMapper` 方法，给接口类创建 `MapperProxyFactory` 映射器代理类，并写入到 knownMappers 的 HashMap 缓存中。
- 另外就是这个类也提供了对应的 `getMapper` 获取映射器代理类的方法，其实这步就包装了我们上一章节手动操作实例化的过程，更加方便在 DefaultSqlSession 中获取 Mapper 时进行使用。
### SqlSession 的定义和实现
```java
public interface SqlSession {

    /**
     * Retrieve a single row mapped from the statement key
     * 根据指定的SqlID获取一条记录的封装对象
     *
     * @param <T>       the returned object type 封装之后的对象类型
     * @param statement sqlID
     * @return Mapped object 封装之后的对象
     */
    <T> T selectOne(String statement);

    /**
     * Retrieve a single row mapped from the statement key and parameter.
     * 根据指定的SqlID获取一条记录的封装对象，只不过这个方法容许我们可以给sql传递一些参数
     * 一般在实际使用中，这个参数传递的是pojo，或者Map或者ImmutableMap
     *
     * @param <T>       the returned object type
     * @param statement Unique identifier matching the statement to use.
     * @param parameter A parameter object to pass to the statement.
     * @return Mapped object
     */
    <T> T selectOne(String statement, Object parameter);

    /**
     * Retrieves a mapper.
     * 得到映射器，这个巧妙的使用了泛型，使得类型安全
     *
     * @param <T>  the mapper type
     * @param type Mapper interface class
     * @return a mapper bound to this SqlSession
     */
    <T> T getMapper(Class<T> type);

}
```
- 在 SqlSession 中定义用来执行 SQL、获取映射器对象以及后续管理事务操作的标准接口。
- 目前这个接口中对于数据库的操作仅仅只提供了 selectOne，后续还会有相应其他方法的定义。

```java
public class DefaultSqlSession implements SqlSession {

    /**
     * 映射器注册机
     */
    private MapperRegistry mapperRegistry;

    // 省略构造函数

    @Override
    public <T> T selectOne(String statement, Object parameter) {
        return (T) ("你被代理了！" + "方法：" + statement + " 入参：" + parameter);
    }

    @Override
    public <T> T getMapper(Class<T> type) {
        return mapperRegistry.getMapper(type, this);
    }

}
```
- 通过 DefaultSqlSession 实现类对 SqlSession 接口进行实现。
- getMapper 方法中获取映射器对象是通过 MapperRegistry 类进行获取的，后续这部分会被配置类进行替换。
- 在 selectOne 中是一段简单的内容返回，目前还没有与数据库进行关联，这部分在我们渐进式的开发过程中逐步实现。
### SqlSession Factory 的定义和实现
```java
public interface SqlSessionFactory {

    /**
     * 打开一个 session
     * @return SqlSession
     */
   SqlSession openSession();

}
```
- 这其实就是一个简单工厂的定义，在工厂中提供接口实现类的能力，也就是 SqlSessionFactory 工厂中提供的开启 SqlSession 的能力。

```java
public class DefaultSqlSessionFactory implements SqlSessionFactory {

    private final MapperRegistry mapperRegistry;

    public DefaultSqlSessionFactory(MapperRegistry mapperRegistry) {
        this.mapperRegistry = mapperRegistry;
    }

    @Override
    public SqlSession openSession() {
        return new DefaultSqlSession(mapperRegistry);
    }

}
```
### 测试
```java
@Test  
public void test_MapperProxyFactory() {  
    // 1. 注册 Mapper    
    MapperRegistry registry = new MapperRegistry();  
    registry.addMappers("cn.bugstack.mybatis.test.dao");  
  
    // 2. 从 SqlSession 工厂获取 Session    
    SqlSessionFactory sqlSessionFactory = new DefaultSqlSessionFactory(registry);  
    SqlSession sqlSession = sqlSessionFactory.openSession();  
  
    // 3. 获取映射器对象  
    IUserDao userDao = sqlSession.getMapper(IUserDao.class);  
  
    // 4. 测试验证  
    String res = userDao.queryUserName("10001");  
    logger.info("测试结果：{}", res);  
}
```
### 输出结果
```
测试结果：你被代理了！方法：queryUserName 入参：[Ljava.lang.Object;@78dd667e
```
