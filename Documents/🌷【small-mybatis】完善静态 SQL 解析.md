# 目标
实现到本章节前，关于 Mybatis ORM 框架的大部分核心结构已经逐步体现出来了，包括；解析、绑定、映射、事务、执行、数据源等。
但随着更多功能的逐步完善，我们需要对模块内的实现进行细化处理，而不单单只是完成功能逻辑。
这就有点像把 CRUD 使用设计原则进行拆分解耦，满足代码的易维护和可扩展性。而这里我们首先着手要处理的就是关于 XML 解析的问题，把之前粗糙的实现进行细化，满足我们对解析时一些参数的整合和处理。
![[【small-mybatis】完善静态 SQL 解析 目的.png|800]]
- 这一部分的解析，就是在我们本章节之前的 XMLConfigBuilder#mapperElement 方法中的操作。看上去虽然能实现功能，但总会让人感觉它不够规整。就像我们平常开发的 CRUD 罗列到一块的逻辑一样，什么流程都能处理，但什么流程都会越来越混乱。
- 就像我们在 ORM 框架 DefaultSqlSession 中调用具体执行数据库操作的方法，需要进行 PreparedStatementHandler#parameterize 参数时，其实并没有准确的定位到参数的类型，jdbcType和javaType的转换关系，所以后续的属性填充就会显得比较混乱且不易于扩展。_当然，如果你硬写也是写的出来的，不过这种就不是一个好的设计！_
- 所以接下来小傅哥会带着读者，把这部分解析的处理，使用设计原则将流程和职责进行解耦，并结合我们的当前诉求，优先处理静态 SQL 内容。待框架结构逐步完善，再进行一些动态SQL和更多参数类型的处理，满足读者以后在阅读 Mybatis 源码，以及需要开发自己的 X-ORM 框架的时候，有一些经验积累。
# 设计
![[【small-mybatis】完善静态 SQL 解析 核心架构.png|800]]
# 实现
## 工程结构
![[【small-mybatis】完善静态 SQL 解析 核心类图.png|800]]
- 解耦原 XMLConfigBuilder 中对 XML 的解析，扩展映射构建器、语句构建器，处理 SQL 的提取和参数的包装，整个核心流图以 XMLConfigBuilder#mapperElement 为入口进行串联调用。
- 在 XMLStatementBuilder#parseStatementNode 方法中解析 `<select id="queryUserInfoById" parameterType="java.lang.Long" resultType="cn.bugstack.mybatis.test.po.User">...</select>` 配置语句，提取参数类型、结果类型，而这里的语句处理流程稍微较长，因为需要用到脚本语言驱动器，进行解析处理，创建出 SqlSource 语句信息。SqlSource 包含了 BoundSql，同时这里扩展了 ParameterMapping 作为参数包装传递类，而不是仅仅作为 Map 结构包装。因为通过这样的方式，可以封装解析后的 javaType/jdbcType 信息。
## 解耦映射解析
提供单独的 XML 映射构建器 XMLMapperBuilder 类，把关于 Mapper 内的 SQL 进行解析处理。提供了这个类以后，就可以把这个类的操作放到 XML 配置构建器，XMLConfigBuilder#mapperElement 中进行使用了。具体我们看下如下代码。
```java
public class XMLMapperBuilder extends BaseBuilder {

    /**
     * 解析
     */
    public void parse() throws Exception {
        // 如果当前资源没有加载过再加载，防止重复加载
        if (!configuration.isResourceLoaded(resource)) {
            configurationElement(element);
            // 标记一下，已经加载过了
            configuration.addLoadedResource(resource);
            // 绑定映射器到namespace
            configuration.addMapper(Resources.classForName(currentNamespace));
        }
    }

    // 配置mapper元素
    // <mapper namespace="org.mybatis.example.BlogMapper">
    //   <select id="selectBlog" parameterType="int" resultType="Blog">
    //    select * from Blog where id = #{id}
    //   </select>
    // </mapper>
    private void configurationElement(Element element) {
        // 1.配置namespace
        currentNamespace = element.attributeValue("namespace");
        if (currentNamespace.equals("")) {
            throw new RuntimeException("Mapper's namespace cannot be empty");
        }

        // 2.配置select|insert|update|delete
        buildStatementFromContext(element.elements("select"));
    }

    // 配置select|insert|update|delete
    private void buildStatementFromContext(List<Element> list) {
        for (Element element : list) {
            final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, element, currentNamespace);
            statementParser.parseStatementNode();
        }
    }

}
```
在 XMLMapperBuilder#parse 的解析中，主要体现在资源解析判断、Mapper解析和绑定映射器到；
- configuration.isResourceLoaded 资源判断避免重复解析，做了个记录。
- configuration.addMapper 绑定映射器主要是把 namespace `cn.bugstack.mybatis.test.dao.IUserDao` 绑定到 Mapper 上。也就是注册到映射器注册机里。
- configurationElement 方法调用的 buildStatementFromContext，重在处理 XML 语句构建器，下文中单独讲解。
```java
public class XMLConfigBuilder extends BaseBuilder {

    /*
     * <mappers>
     *	 <mapper resource="org/mybatis/builder/AuthorMapper.xml"/>
     *	 <mapper resource="org/mybatis/builder/BlogMapper.xml"/>
     *	 <mapper resource="org/mybatis/builder/PostMapper.xml"/>
     * </mappers>
     */
    private void mapperElement(Element mappers) throws Exception {
        List<Element> mapperList = mappers.elements("mapper");
        for (Element e : mapperList) {
            String resource = e.attributeValue("resource");
            InputStream inputStream = Resources.getResourceAsStream(resource);

            // 在for循环里每个mapper都重新new一个XMLMapperBuilder，来解析
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource);
            mapperParser.parse();
        }
    }

}
```
- 在 XMLConfigBuilder#mapperElement 中，把原来流程化的处理进行解耦，调用 XMLMapperBuilder#parse 方法进行解析处理。
## 语句构建器
XMLStatementBuilder 语句构建器主要解析 XML 中 `select|insert|update|delete` 中的语句，当前我们先以 select 解析为案例，后续再扩展其他的解析流程。
```java
public class XMLStatementBuilder extends BaseBuilder {

    //解析语句(select|insert|update|delete)
    //<select
    //  id="selectPerson"
    //  parameterType="int"
    //  parameterMap="deprecated"
    //  resultType="hashmap"
    //  resultMap="personResultMap"
    //  flushCache="false"
    //  useCache="true"
    //  timeout="10000"
    //  fetchSize="256"
    //  statementType="PREPARED"
    //  resultSetType="FORWARD_ONLY">
    //  SELECT * FROM PERSON WHERE ID = #{id}
    //</select>
    public void parseStatementNode() {
        String id = element.attributeValue("id");
        // 参数类型
        String parameterType = element.attributeValue("parameterType");
        Class<?> parameterTypeClass = resolveAlias(parameterType);
        // 结果类型
        String resultType = element.attributeValue("resultType");
        Class<?> resultTypeClass = resolveAlias(resultType);
        // 获取命令类型(select|insert|update|delete)
        String nodeName = element.getName();
        SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));

        // 获取默认语言驱动器
        Class<?> langClass = configuration.getLanguageRegistry().getDefaultDriverClass();
        LanguageDriver langDriver = configuration.getLanguageRegistry().getDriver(langClass);

        SqlSource sqlSource = langDriver.createSqlSource(configuration, element, parameterTypeClass);

        MappedStatement mappedStatement = new MappedStatement.Builder(configuration, currentNamespace + "." + id, sqlCommandType, sqlSource, resultTypeClass).build();

        // 添加解析 SQL
        configuration.addMappedStatement(mappedStatement);
    }

}
```
- 整个这部分内容的解析，就是从 `XMLConfigBuilder` 拆解出来关于 `Mapper` 语句解析的部分，通过这样这样的解耦设计，会让整个流程更加清晰。
- `XMLStatementBuilder#parseStatementNode` 方法是解析 SQL 语句节点的过程，包括了语句的ID、参数类型、结果类型、命令(`select|insert|update|delete`)，以及使用语言驱动器处理和封装SQL信息，当解析完成后写入到 Configuration 配置文件中的 `Map<String, MappedStatement>` 映射语句存放中。
## 脚本语言驱动
### 定义接口
```java
public interface LanguageDriver {

    SqlSource createSqlSource(Configuration configuration, Element script, Class<?> parameterType);

}
```
- 定义脚本语言驱动接口，提供创建 SQL 信息的方法，入参包括了配置、元素、参数。其实它的实现类一共有3个；`XMLLanguageDriver`、`RawLanguageDriver`、`VelocityLanguageDriver`，这里我们只是实现了默认的第一个即可。
### XML 语言构建器实现
```java
public class XMLLanguageDriver implements LanguageDriver {

    @Override
    public SqlSource createSqlSource(Configuration configuration, Element script, Class<?> parameterType) {
        XMLScriptBuilder builder = new XMLScriptBuilder(configuration, script, parameterType);
        return builder.parseScriptNode();
    }

}

```
### XML 脚本构建器解析
```java
public class XMLScriptBuilder extends BaseBuilder {

    public SqlSource parseScriptNode() {
        List<SqlNode> contents = parseDynamicTags(element);
        MixedSqlNode rootSqlNode = new MixedSqlNode(contents);
        return new RawSqlSource(configuration, rootSqlNode, parameterType);
    }

    List<SqlNode> parseDynamicTags(Element element) {
        List<SqlNode> contents = new ArrayList<>();
        // element.getText 拿到 SQL
        String data = element.getText();
        contents.add(new StaticTextSqlNode(data));
        return contents;
    }

}
```
- XMLScriptBuilder#parseScriptNode 解析SQL节点的处理其实没有太多复杂的内容，主要是对 RawSqlSource 的包装处理。
### SQL 源码构建器
```java
public class SqlSourceBuilder extends BaseBuilder {

    private static final String parameterProperties = "javaType,jdbcType,mode,numericScale,resultMap,typeHandler,jdbcTypeName";

    public SqlSourceBuilder(Configuration configuration) {
        super(configuration);
    }

    public SqlSource parse(String originalSql, Class<?> parameterType, Map<String, Object> additionalParameters) {
        ParameterMappingTokenHandler handler = new ParameterMappingTokenHandler(configuration, parameterType, additionalParameters);
        GenericTokenParser parser = new GenericTokenParser("#{", "}", handler);
        String sql = parser.parse(originalSql);
        // 返回静态 SQL
        return new StaticSqlSource(configuration, sql, handler.getParameterMappings());
    }

    private static class ParameterMappingTokenHandler extends BaseBuilder implements TokenHandler {
       
        @Override
        public String handleToken(String content) {
            parameterMappings.add(buildParameterMapping(content));
            return "?";
        }

        // 构建参数映射
        private ParameterMapping buildParameterMapping(String content) {
            // 先解析参数映射,就是转化成一个 HashMap | #{favouriteSection,jdbcType=VARCHAR}
            Map<String, String> propertiesMap = new ParameterExpression(content);
            String property = propertiesMap.get("property");
            Class<?> propertyType = parameterType;
            ParameterMapping.Builder builder = new ParameterMapping.Builder(configuration, property, propertyType);
            return builder.build();
        }

    }
    
}
```
- 关于以上文中提到的，关于 `BoundSql.parameterMappings` 的参数就是来自于 `ParameterMappingTokenHandler#buildParameterMapping` 方法进行构建处理的。
- 具体的 `javaType`、`jdbcType` 会体现到 `ParameterExpression` 参数表达式中完成解析操作。
## SqlSession 调用调整
```java
public class DefaultSqlSession implements SqlSession {

    private Configuration configuration;
    private Executor executor;

    @Override
    public <T> T selectOne(String statement, Object parameter) {
        MappedStatement ms = configuration.getMappedStatement(statement);
        List<T> list = executor.query(ms, parameter, Executor.NO_RESULT_HANDLER, ms.getSqlSource().getBoundSql(parameter));
        return list.get(0);
    }

}
```