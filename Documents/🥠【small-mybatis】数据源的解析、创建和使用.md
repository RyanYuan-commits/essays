---
created: 2025-09-27 22:34:06
updated: 2025-09-27 22:34:06
---
# 目标
在上一章节我们解析 XML 中的 SQL 配置信息，并在代理对象调用 `DefaultSqlSession` 中进行获取和打印操作，从整个框架结构来看我们解决了对象的代理、Mapper的映射、SQL的初步解析，接下来就应该是连库和执行SQL语句并返回结果了。
那么这部分内容就会涉及到解析 XML 中关于 `dataSource` 数据源信息配置，并简历事务管理和连接池的启动和使用。并将这部分能力在 DefaultSqlSession 执行 SQL 语句时进行调用。但为了不至于在一个章节把整个工程撑大，这里我们会把重点放到解析配置、简历事务框架和引入 `DRUID` 连接池，以及初步完成 SQL 的执行和结果简单包装上。便于读者先熟悉整个框架结构，在后续章节再陆续迭代和完善框架细节。
# 设计
建立数据源连接池和 JDBC 事务工厂操作，并以 xml 配置数据源信息为入口，在 XMLConfigBuilder 中添加数据源解析和构建操作，向配置类configuration添加 JDBC 操作环境信息。以便在 DefaultSqlSession 完成对 JDBC 执行 SQL 的操作。
![[【small-mybatis】数据源的解析、创建和使用 架构图.png|800]]
- 在 parse 中解析 XML DB 链接配置信息，并完成事务工厂和连接池的注册环境到配置类的操作。
- 与上一章节改造 selectOne 方法的处理，不再是打印 SQL 语句，而是把 SQL 语句放到 DB 连接池中进行执行，以及完成简单的结果封装。
- 关于 JDBC 实现增删改查与事务的案例：[[🌴【JDBC】利用 JDBC 实现增删改查与事务]]
# 实现
![[【small-mybatis】数据源的解析、创建和使用 核心类图.png|1000]]
## 事务管理
### 事务接口
```java
public interface Transaction {

    Connection getConnection() throws SQLException;

    void commit() throws SQLException;

    void rollback() throws SQLException;

    void close() throws SQLException;

}
```
- 定义标准的事务接口，链接、提交、回滚、关闭，具体可以由不同的事务方式进行实现，包括：JDBC和托管事务，托管事务是交给 Spring 这样的容器来管理。

```java
public class JdbcTransaction implements Transaction {

    protected Connection connection;
    protected DataSource dataSource;
    protected TransactionIsolationLevel level = TransactionIsolationLevel.NONE;
    protected boolean autoCommit;

    public JdbcTransaction(DataSource dataSource, TransactionIsolationLevel level, boolean autoCommit) {
        this.dataSource = dataSource;
        this.level = level;
        this.autoCommit = autoCommit;
    }

    @Override
    public Connection getConnection() throws SQLException {
        connection = dataSource.getConnection();
        connection.setTransactionIsolation(level.getLevel());
        connection.setAutoCommit(autoCommit);
        return connection;
    }

    @Override
    public void commit() throws SQLException {
        if (connection != null && !connection.getAutoCommit()) {
            connection.commit();
        }
    }
    
    //...

}
```
- 在 JDBC 事务实现类中，封装了获取链接、提交事务等操作，其实使用的也就是 JDBC 本身提供的能力。
- [[🌴【JDBC】利用 JDBC 实现增删改查与事务#JDBC 实现事务]]
### 事务工厂
```java
public interface TransactionFactory {

    /**
     * 根据 Connection 创建 Transaction
     * @param conn Existing database connection
     * @return Transaction
     */
    Transaction newTransaction(Connection conn);

    /**
     * 根据数据源和事务隔离级别创建 Transaction
     * @param dataSource DataSource to take the connection from
     * @param level Desired isolation level
     * @param autoCommit Desired autocommit
     * @return Transaction
     */
    Transaction newTransaction(DataSource dataSource, TransactionIsolationLevel level, boolean autoCommit);

}
```
- 以工厂方法模式包装 JDBC 事务实现，为每一个事务实现都提供一个对应的工厂。与简单工厂的接口包装不同。
## 别名 Registry
在 Mybatis 框架中我们所需要的基本类型、数组类型以及自己定义的事务实现和事务工厂都需要注册到类型别名的注册器中进行管理，在我们需要使用的时候可以从注册器中获取到具体的对象类型，之后在进行实例化的方式进行使用。
对于一些基本的类型，比如 String、Integer 等，如果每次都让用户书写全类名的话，会比较复杂，此时就可以注册一些别名。
### 基础注册器
```java
public class TypeAliasRegistry {

    private final Map<String, Class<?>> TYPE_ALIASES = new HashMap<>();

    public TypeAliasRegistry() {
        // 构造函数里注册系统内置的类型别名
        registerAlias("string", String.class);

        // 基本包装类型
        registerAlias("byte", Byte.class);
        registerAlias("long", Long.class);
        registerAlias("short", Short.class);
        registerAlias("int", Integer.class);
        registerAlias("integer", Integer.class);
        registerAlias("double", Double.class);
        registerAlias("float", Float.class);
        registerAlias("boolean", Boolean.class);
    }

    public void registerAlias(String alias, Class<?> value) {
        String key = alias.toLowerCase(Locale.ENGLISH);
        TYPE_ALIASES.put(key, value);
    }

    public <T> Class<T> resolveAlias(String string) {
        String key = string.toLowerCase(Locale.ENGLISH);
        return (Class<T>) TYPE_ALIASES.get(key);
    }

}
```
- 在 TypeAliasRegistry 类型别名注册器中先做了一些基本的类型注册，以及提供 registerAlias 注册方法和 resolveAlias 获取方法。
### 在 Configuration 中引入别名注册机
```java
public class Configuration {

    //环境
    protected Environment environment;

    // 映射注册机
    protected MapperRegistry mapperRegistry = new MapperRegistry(this);

    // 映射的语句，存在Map里
    protected final Map<String, MappedStatement> mappedStatements = new HashMap<>();

    // 类型别名注册机
    protected final TypeAliasRegistry typeAliasRegistry = new TypeAliasRegistry();

    public Configuration() {
        typeAliasRegistry.registerAlias("JDBC", JdbcTransactionFactory.class);
        typeAliasRegistry.registerAlias("DRUID", DruidDataSourceFactory.class);
    }
    
    //...
}
```
- 在 Configuration 配置选项类中，添加类型别名注册机，通过构造函数添加 JDBC、DRUID 注册操作。
- 读者应该注意到，整个 Mybatis 的操作都是使用 Configuration 配置项进行串联流程，所以所有内容都会在 Configuration 中进行链接。
## 解析数据源配置
通过在 XML 解析器 XMLConfigBuilder 中，扩展对环境信息的解析，我们这里把数据源、事务类内容称为操作 SQL 的环境。解析后把配置信息写入到 Configuration 配置项中，便于后续使用。
```java
public class XMLConfigBuilder extends BaseBuilder {
         
  public Configuration parse() {
      try {
          // 环境
          environmentsElement(root.element("environments"));
          // 解析映射器
          mapperElement(root.element("mappers"));
      } catch (Exception e) {
          throw new RuntimeException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
      }
      return configuration;
  }
  private void environmentsElement(Element context) throws Exception {
      String environment = context.attributeValue("default");
      List<Element> environmentList = context.elements("environment");
      for (Element e : environmentList) {
          String id = e.attributeValue("id");
          if (environment.equals(id)) {
              // 事务管理器
              TransactionFactory txFactory = (TransactionFactory) typeAliasRegistry.resolveAlias(e.element("transactionManager").attributeValue("type")).newInstance();
              // 数据源
              Element dataSourceElement = e.element("dataSource");
              DataSourceFactory dataSourceFactory = (DataSourceFactory) typeAliasRegistry.resolveAlias(dataSourceElement.attributeValue("type")).newInstance();
              List<Element> propertyList = dataSourceElement.elements("property");
              Properties props = new Properties();
              for (Element property : propertyList) {
                  props.setProperty(property.attributeValue("name"), property.attributeValue("value"));
              }
              dataSourceFactory.setProperties(props);
              DataSource dataSource = dataSourceFactory.getDataSource();
              // 构建环境
              Environment.Builder environmentBuilder = new Environment.Builder(id)
                      .transactionFactory(txFactory)
                      .dataSource(dataSource);
              configuration.setEnvironment(environmentBuilder.build());
          }
      }
  }
}
```
- 以 `XMLConfigBuilder#parse` 解析扩展对数据源解析操作，在 `environmentsElement` 方法中包括事务管理器解析和从类型注册器中读取到事务工程的实现类，同理数据源也是从类型注册器中获取。
- 最后把事务管理器和数据源的处理，通过环境构建 `Environment.Builder` 存放到 `Configuration` 配置项中，也就可以通过 `Configuration` 存在的地方都可以获取到数据源了。
## SQL 执行和结果封装
```java
public class DefaultSqlSession implements SqlSession {

    private Configuration configuration;

    public DefaultSqlSession(Configuration configuration) {
        this.configuration = configuration;
    }

    @Override
    public <T> T selectOne(String statement, Object parameter) {
        try {
            MappedStatement mappedStatement = configuration.getMappedStatement(statement);
            Environment environment = configuration.getEnvironment();

            Connection connection = environment.getDataSource().getConnection();

            BoundSql boundSql = mappedStatement.getBoundSql();
            PreparedStatement preparedStatement = connection.prepareStatement(boundSql.getSql());
            preparedStatement.setLong(1, Long.parseLong(((Object[]) parameter)[0].toString()));
            ResultSet resultSet = preparedStatement.executeQuery();

            List<T> objList = resultSet2Obj(resultSet, Class.forName(boundSql.getResultType()));
            return objList.get(0);
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }
    // ...
}
```
## 测试
## 配置数据源
```xml
<environments default="development">
    <environment id="development">
        <transactionManager type="JDBC"/>
        <dataSource type="DRUID">
            <property name="driver" value="com.mysql.jdbc.Driver"/>
            <property name="url" value="jdbc:mysql://127.0.0.1:3306/mybatis?useUnicode=true"/>
            <property name="username" value="root"/>
            <property name="password" value="123456"/>
        </dataSource>
    </environment>
</environments>
```
## 配置 Mapper
```java
<select id="queryUserInfoById" parameterType="java.lang.Long" resultType="cn.bugstack.mybatis.test.po.User">
    SELECT id, userId, userName, userHead
    FROM user
    where id = #{id}
</select>
```
## 流程验证

```java
@Test
public void test_SqlSessionFactory() throws IOException {
    // 1. 从SqlSessionFactory中获取SqlSession
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(Resources.getResourceAsReader("mybatis-config-datasource.xml"));
    SqlSession sqlSession = sqlSessionFactory.openSession();
    
    // 2. 获取映射器对象
    IUserDao userDao = sqlSession.getMapper(IUserDao.class);
    
    // 3. 测试验证
    User user = userDao.queryUserInfoById(1L);
    logger.info("测试结果：{}", JSON.toJSONString(user));
}
```
- 单元测试没有什么改变，仍是通过 SqlSessionFactory 中获取 SqlSession 并获得映射对象和执行方法调用。
**测试结果**
```
22:34:18.676 [main] INFO  c.alibaba.druid.pool.DruidDataSource - {dataSource-1} inited
22:34:19.286 [main] INFO  cn.bugstack.mybatis.test.ApiTest - 测试结果：{"id":1,"userHead":"1_04","userId":"10001","userName":"小傅哥"}

Process finished with exit code 0
```
- 从现在的测试结果已经可以看出，通过我们对数据源的解析、包装和使用，已经可以对 SQL 语句进行执行和包装返回的结果信息了。
- 读者在学习的过程中可以调试下代码，看看每一步都是如何完成执行步骤的，也在这个过程中进行学习 Mybatis 框架的设计技巧。

## 功能验证
```java
@Test
public void test_selectOne() throws IOException {
    // 解析 XML
    Reader reader = Resources.getResourceAsReader("mybatis-config-datasource.xml");
    XMLConfigBuilder xmlConfigBuilder = new XMLConfigBuilder(reader);
    Configuration configuration = xmlConfigBuilder.parse();

    // 获取 DefaultSqlSession
    SqlSession sqlSession = new DefaultSqlSession(configuration);
    // 执行查询：默认是一个集合参数

    Object[] req = {1L};
    Object res = sqlSession.selectOne("cn.bugstack.mybatis.test.dao.IUserDao.queryUserInfoById", req);
    logger.info("测试结果：{}", JSON.toJSONString(res));
}
```
- 除了整体的功能流程测试外，我们还可以只对本章节新增的内容进行单元测试。由于本章节的主要操作都是在解析内容的添加，处理 XML 配置中的数据源信息，以及解析后可以在 DefaultSqlSession 中调用数据源执行 SQL 语句并返回结果。
- 所以我们这里单独把这部分提取出来进行测试验证，通过基于这样的测试，可以更好的在 Sequence Diagram 中生成对应的 UML 方便读者更加容易理解本章节新增的内容和流程。
