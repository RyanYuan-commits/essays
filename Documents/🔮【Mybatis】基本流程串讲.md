### 1 配置文件
在构建核心对象 `SqlSessionFactory` 的时候，依照的是 Mybatis 配置文件的信息，它常被命名为 `mybatis-config.xml`：
```java
String resource = "mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```
在 DTD 文件中，定义了一些配置节点：
```xml
<configuration>
    <properties/>
    <settings/>
    <typeAliases/>
    <typeHandlers/>
    <objectFactory/>
    <objectWrapperFactory/>
    <plugins>
        <plugin/>
    </plugins>
    <environments>
        <environment/>
    </environments>
    <databaseIdProvider/>
    <mappers/>
</configuration>
```
所谓解析配置文件，实际上就是解析以上各个节点，在 MyBatis 源码中，一个 `XML` 节点会被解析成一个 `XNode` 对象，`XNode` 类中提供了一个 `getChildren()` 方法，用来返回子节点，也就是返回一个`List<XNode>`。
MyBatis 会先解析 `/configuration` 节点得到 `XNode` 对象，进而继续迭代解析子节点，在这个过程中解析出来的内容，最终会被保存在 `Configuration` 对象中。
### 2 关键对象
#### 2.1 MappedStatement 对象
MyBatis 中的一等公民自然就是我们定义的SQL语句了：
```xml
<mapper namespace="com.ryan.common_test.mapper.UserMapper">
	<select id="selectUserById" parameterType="int" resultType="com.ryan.common_test.po.User">  
	    SELECT * FROM users WHERE id = #{id}
	</select>
</mapper>
```
这样的一个 `<select>` 的子节点，最终会被解析并构建成一个 `MappedStatement` 对象，在 `MappedStatement` 对象中有这些关键属性：
- `String id`，表示当前 `MappedStatement` 对象的唯一标识，由接口名 + 方法名组成，比如上面的案例中，id 为`com.ryan.common_test.mapper.UserMapper.selectUserById`
- `StatementType statementType`，默认为 PREPARED，表示利用 PreparedStatement 来执行 SQL
- `SqlSource sqlSource`，表示对应的SQL语句
- `List<ResultMap> resultMaps`，表示对应的ResultMap，一般只有一个，执行存储过程时可以设置多个
- `KeyGenerator keyGenerator`，对于insert语句，如果设置了 `useGeneratedKeys=true`，那么该属性的值为 `Jdbc3KeyGenerator`，表示会在insert执行完后来获取新纪录对应的自增 id；
- `String databaseId`，表示对应的 databaseId。
- `SqlCommandType sqlCommandType`：SQL 语句类型，有 `UNKNOWN, INSERT, UPDATE, DELETE, SELECT, FLUSH`。
##### ResultMap 对象
一个 `mapper` 标签中可以定义多 `resultMap` 标签，`ResultMap` 定义了 `SQL` 返回结果与 `PO` 对象的映射方式：
```java
<resultMap id="userMap" type="UserInfo"> <id property="id" column="c_id"/> 
	<result property="name" column="c_name" />
</resultMap>

<select id="getUserInfo" resultMap="userMap">
  select * from user_info where id = #{id}
</select>
```
当执行查询语句时，需要将查询到的内容转换为 Java 对象，如果配置了 `resultMap` 那这个过程就会根据 `resultMap` 中定义的方式映射。
ResultMap 在 `DefaultResultSetHandler` 的 `handleResultSet` 方法中起作用：
```java
private void handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults,
								ResultMapping parentMapping) throws SQLException {  
  try {  
    if (parentMapping != null) {  
      handleRowValues(rsw, resultMap, null, RowBounds.DEFAULT, parentMapping);  
    } else {  
      if (resultHandler == null) {  
        DefaultResultHandler defaultResultHandler = new DefaultResultHandler(objectFactory);  
        handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);  
        multipleResults.add(defaultResultHandler.getResultList());  
      } else {  
        handleRowValues(rsw, resultMap, resultHandler, rowBounds, null);  
      }  
    }  
  } finally { 
    closeResultSet(rsw.getResultSet());  
  }  
}
```
##### SqlSource 对象
一个SqlSource对象对应的就是我们定义的一个SQL语句，但是我们定义的SQL语句是多种多样的，大概分为以下几类：
```sql
select name from user_info -- 最简单的SQL语句
select name from user_info where id = #{id} -- 用了 #{}
select name from user_info where ${col} = #{id} -- 用了 ${}
select name from user_info <if test="id == 26"> where id = #{id}</if> -- 用了 <if> 标签
select name from user_info where id <![CDATA[=]]> #{id} -- 用了 <![CDATA[=]]> 标签
```
而对于MyBatis而言，底层只分为了两类：**动态 SQL 和静态 SQL**，分别对应 DynamicSqlSource 和 RawSqlSource。
用了 `${}` 和 `<if>` 等标签的就是动态 SQL，这类 SQL 最终生成出来的是 DynamicSqlSource 对象，其他的（就算用了`#{}`和`cdata`）都是静态SQL，这类SQL最终生成出来的是 StaticSqlSource 对象。
- 静态 SQL 的 **结构** 在编译的时候就可以确认，比如 `#{}` 和 `cdata`；
- 动态 SQL 的 结构 在编译过程中会发生变化，比如 `<if>` 标签和 `${}` 等。
#### 2.2 SqlSession 对象
解析完配置文件会得到一个 `SqlSessionFactory` 对象，该对象中就包含了前面提到的 `Configuration` 对象。
这个工厂对象用于构建 `SqlSession` 对象，SqlSession 顾名思义，表示一次数据库的连接，其底层就是 JDBC 的 `Connection` 对象。
具体的路径是这样的：`SqlSession` 中包含了一个 `Executor` 执行器对象，`Executor` 对象中包含了 `Transaction` 对象，`Transaction` 对象中就包含了 `Connection` 对象：
```java
public class JdbcTransaction implements Transaction {  
  // ...
  protected Connection connection;
  // ...
}
```
##### Executor 执行器
Executor 接口，其实现类有 `CachingExecutor`（实现缓存功能的执行器）、`BatchExecutor`（实现批量功能的执行器）等等。
一次 SQL 语句的执行，大概要经过下面的步骤：
1. 获取到数据库的 `Connection` 对象
2. 将语句构建成 `Statement` 对象
3. 执行语句并处理结果
后两个步骤是由 `StatementHandler` 对象执行的，而获取连接依靠的是 `Transaction` 对象。
```java
// org.apache.ibatis.executor.BaseExecutor 的 getConnection 方法
protected Connection getConnection(Log statementLog) throws SQLException {  
  Connection connection = transaction.getConnection();  
  if (statementLog.isDebugEnabled()) {  
    return ConnectionLogger.newInstance(connection, statementLog, queryStack);  
  } else {  
    return connection;  
  }  
}
```
##### Transaction 事务类
对应的实现类是 `JdbcTransaction`，具体获取 `Connection` 的是 `openSession` 方法：
```java
protected void openConnection() throws SQLException {  
  if (log.isDebugEnabled()) {  
    log.debug("Opening JDBC Connection");  
  }  
  connection = dataSource.getConnection();  
  if (level != null) {  
    connection.setTransactionIsolation(level.getLevel());  
  }  
  setDesiredAutoCommit(autoCommit);  
}
```
这个方法从数据源中获取连接，并根据 `level` 配置来设置事务的隔离级别，最终根据 `autoCommit` 配置来决定是否自动提交。
当获取 `SqlSession` 对象的时候，可以使用这个方法来关闭自动提交来自由的控制事务：
```java
SqlSession openSession(boolean autoCommit);
```
##### MapperProxy 代理对象
`SqlSession` 负责连接数据库并执行语句，我们可以选择直接使用 `SqlSession` 来执行配置文件中的语句：
```java
User user = session.selectOne("com.example.mapper.UserMapper.getUserById", 1);
```
为了操作的便捷，SqlSession 提供了 Mapper 对象的代理类，来执行 SQL：
```java
try (SqlSession session = sqlSessionFactory.openSession(false)) { 
	// 使用 Mapper
    UserMapper mapper = session.getMapper(UserMapper.class);  
    mapper.selectAllUsers() .forEach(System.out::println);  
    
    // 使用 id
    List<User> users = session.selectList("com.example.mapper.UserMapper.getUserById", 18);  
}aaaaaaaa
```
即使使用 `Mapper`，其底层也是通过 id，也就是类的全限定名 + 方法名来获取 SQL 的，Mybatis 将这部分逻辑抽离出来，放在了 Mapper 对象的代理方法之中，下面展示的是 Mybatis 使用 JDK 动态代理生成代理对象的步骤：
```java
protected T newInstance(MapperProxy<T> mapperProxy) {  
  return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);  
}  
  
public T newInstance(SqlSession sqlSession) {
  final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
  return newInstance(mapperProxy);  
}
```
`MapperProxy` 实现了 `InvocationHandler` 接口的 `invoke` 方法：
```java
@Override  
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {  
  try {  
    if (Object.class.equals(method.getDeclaringClass())) {  
      return method.invoke(this, args);  
    } else {  
      return cachedInvoker(method).invoke(proxy, method, args, sqlSession);  
    }  
  } catch (Throwable t) {  
    throw ExceptionUtil.unwrapThrowable(t);  
  }  
}
```
如果是 `Object` 类的原生方法，比如 `equals`、`hashCode` 等，会直接通过 `method.invoke` 执行。
否则会通过代理类内置的 `invoker` 执行：
```java
return new PlainMethodInvoker(new MapperMethod(mapperInterface, method, sqlSession.getConfiguration()));

public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {  
  this.command = new SqlCommand(config, mapperInterface, method);  
  this.method = new MethodSignature(config, mapperInterface, method);  
}
```
最终通过 `invoker` 中的 `MapperMethod` 来调用 `SqlSession` 执行语句。
`MapperMethod` 中有两个关键的属性，`command` 用于封装语句 id 和语句类型（select、update 等），`method` 用于封装方法的签名信息，比如方法的返回值、参数等，具体执行语句的方法是这样的：
```java
public Object execute(SqlSession sqlSession, Object[] args) {  
  Object result;  
  switch (command.getType()) {  
	    case INSERT: {  
	      Object param = method.convertArgsToSqlCommandParam(args);  
	      result = rowCountResult(sqlSession.insert(command.getName(), param));  
	      break;  
	    }
    // 其他类型语句
	}
}
```
### 3 流程总结
1. 首先，使用接口名+方法名作为 key，从 `Configuration` 对象的 `mappedStatements` 属性中，获取到 `MappedStatement` 对象；
2. 通过 `MappedStatement` 对象中的 `SqlCommandType`，来判断到底应该调用 SqlSession 的哪个方法（select、update 等）；
3. 再通过 `MappedStatement` 对象中的 `SqlSource` 对象，来获取 `BoundSql` 对象，这个 SQL 中封装了最终的语句，以及参数映射等；动态语句在转化为 `BoundSql` 需要先进行一次解析。
4. 然后利用 `BoundSql`、方法参数等生成 `CacheKey`，用来判断当前执行的 SQL 是否能命中缓存。
5. 如果不能命中缓存，就利用底层的 `Handler` 去执行 SQL，常用的 `Handler` 有：
	1. 利用 `PreparedStatementHandler` 来生成 `PreparedStatement` 对象；
	2. 利用 `ParameterHandler` 来给 `PreparedStatement` 对象的参数赋值；
	3. 利用 `PreparedStatementHandler` 来执行 `PreparedStatement` 对象，也就是真正执行 SQL；
	4. 利用 `ResultSetHandler` 来处理 SQL 执行结果，也就是将结果记录按照 `ResultMap` 转换成相应的 `Java` 对象；
	5. 利用 `TypeHandler` 来实现 `JdbcType` 和 `JavaType` 之间的转换。
最后利用 `SqlSession` 的 `commit()`、`rollback()` 方法来提交或回滚事务。