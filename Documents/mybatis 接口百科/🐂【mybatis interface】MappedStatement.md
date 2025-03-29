xml 中配置的一条 SQL 语句，最终就是被解析成一个 `MappedStatement` 类，例如：
```xml
<!-- UserMapper.xml -->
<mapper namespace="com.example.UserMapper">
    <select id="selectUserById" resultType="com.example.User">
        SELECT * FROM users WHERE id = #{id}
    </select>
</mapper>
```

下面的是这个类的属性：

```java
private String resource;  
private Configuration configuration;  
private String id;  
private Integer fetchSize;  
private Integer timeout;  
private StatementType statementType;  
private ResultSetType resultSetType;  
private SqlSource sqlSource;  
private Cache cache;  
private ParameterMap parameterMap;  
private List<ResultMap> resultMaps;  
private boolean flushCacheRequired;  
private boolean useCache;  
private boolean resultOrdered;  
private SqlCommandType sqlCommandType;  
private KeyGenerator keyGenerator;  
private String[] keyProperties;  
private String[] keyColumns;  
private boolean hasNestedResultMaps;  
private String databaseId;  
private Log statementLog;  
private LanguageDriver lang;  
private String[] resultSets;
```
- `id`: 映射语句的唯一标识符，通常与命名空间组合使用以确保唯一性。
- `resource`: 资源文件路径，通常用于指定映射文件的位置。
- `configuration`: MyBatis 配置对象，包含了 MyBatis 的全局配置信息。
- `fetchSize`: 指定每批读取的记录数，用于优化大数据量查询。
- `timeout`: 设置执行 SQL 语句的超时时间，单位为秒。
- `statementType`: 语句类型，如 STATEMENT, PREPARED, CALLABLE 等。
- `resultSetType`: 结果集类型，如 FORWARD_ONLY, SCROLL_INSENSITIVE, SCROLL_SENSITIVE 等。
- `sqlSource`: SQL 源对象，包含动态 SQL 信息。
- `cache`: 缓存对象，用于缓存查询结果。
- `parameterMap`: 参数映射对象，定义了参数如何映射到 SQL 语句中。
- `resultMaps`: 结果映射列表，定义了如何将数据库结果映射到 Java 对象。
- `flushCacheRequired`: 是否在执行前强制刷新缓存。
- `useCache`: 是否使用二级缓存。
- `resultOrdered`: 是否要求结果有序。
- `sqlCommandType`: SQL 命令类型，如 SELECT, INSERT, UPDATE, DELETE 等。
- `keyGenerator`: 主键生成器，用于生成主键值。
- `keyProperties`: 主键属性名称，用于指定哪些属性是主键。
- `keyColumns`: 主键列名称，用于指定数据库中的主键列。
- `hasNestedResultMaps`: 是否包含嵌套的结果映射。
- `databaseId`: 数据库 ID，用于多数据库支持。
- `statementLog`: 日志对象，用于记录 SQL 语句的执行日志。
- `lang`: 语言驱动器，用于处理动态 SQL。
- `resultSets`: 结果集名称，用于指定多个结果集。