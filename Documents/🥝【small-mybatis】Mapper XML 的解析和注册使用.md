
# 目标
在我们渐进式的逐步实现 Mybatis 框架过程中，首先我们要有一个目标导向的思路，也就是说 Mybatis 的核心逻辑怎么实现。
**其实我们可以把这样一个 ORM 框架的目标，简单的描述成是为了 给一个接口提供代理类**，类中包括了对 Mapper 也就是 xml 文件中的 SQL 信息(`类型`、`入参`、`出参`、`条件`)进行解析和处理，这个处理过程就是对数据库的操作以及返回对应的结果给到接口。
![[【small-mybatis】Mapper XML 的解析和注册使用 目标.png|800]]
那么按照 ORM 核心流程的执行过程，我们本章节就需要在上一章节的基础上，继续扩展对 Mapper 文件的解析以及提取出对应的 SQL 文件。并在当前这个阶段，可以满足我们调用 DAO 接口方法的时候，可以返回 Mapper 中对应的待执行 SQL 语句。
# 设计
结合上一章节我们使用了 `MapperRegistry` 对包路径进行扫描注册映射器，并在 `DefaultSqlSession` 中进行使用。
那么在我们可以把这些命名空间、SQL描述、映射信息统一维护到每一个 DAO 对应的 Mapper XML 的文件以后，其实 XML 就是我们的源头了。通过对 XML 文件的解析和处理就可以完成 Mapper 映射器的注册和 SQL 管理。这样也就更加我们操作和使用了。
![[【small-mybatis】Mapper XML 的解析和注册使用 架构图.png|900]]
- 首先需要定义 `SqlSessionFactoryBuilder` 工厂建造者模式类，通过入口 IO 的方式对 XML 文件进行解析。当前我们主要以解析 SQL 部分为主，并注册映射器，串联出整个核心流程的脉络。
- 文件解析以后会存放到 Configuration 配置类中，接下来你会看到这个配置类会被串联到整个 Mybatis 流程中，所有内容存放和读取都离不开这个类。如我们在 DefaultSqlSession 中获取 Mapper 和执行 selectOne 也同样是需要在 Configuration 配置类中进行读取操作。
我们在 Mapper 的 xml 中可以获得一条具体的 sql 语句，和对其一系列的配置，具体可以看 [[🐂【mybatis】MappedStatement]]
# 实现
![[【small-mybatis】Mapper XML 的解析和注册使用 核心类图.png|800]]
## Configuration 类
```java
public class Configuration {  
  
    /**  
     * 映射注册机  
     */  
    protected MapperRegistry mapperRegistry = new MapperRegistry(this);  
    
	// 向 registry 中注册 mapper
}
```
## SqlSession Factory 添加解析 XML 功能
```java
public class SqlSessionFactoryBuilder {
    public SqlSessionFactory build(Reader reader) {
        XMLConfigBuilder xmlConfigBuilder = new XMLConfigBuilder(reader);
        return build(xmlConfigBuilder.parse());
    }

    public SqlSessionFactory build(Configuration config) {
        return new DefaultSqlSessionFactory(config);
    }
}
```
## XML 解析处理
```java
public class XMLConfigBuilder extends BaseBuilder {

    private Element root;

    public XMLConfigBuilder(Reader reader) {
        // 1. 调用父类初始化Configuration
        super(new Configuration());
        // 2. dom4j 处理 xml
        SAXReader saxReader = new SAXReader();
        try {
            Document document = saxReader.read(new InputSource(reader));
            root = document.getRootElement();
        } catch (DocumentException e) {
            e.printStackTrace();
        }
    }

    public Configuration parse() {
        try {
            // 解析映射器
            mapperElement(root.element("mappers"));
        } catch (Exception e) {
            throw new RuntimeException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
        }
        return configuration;
    }

    private void mapperElement(Element mappers) throws Exception {
        List<Element> mapperList = mappers.elements("mapper");
        for (Element e : mapperList) {
            		// 解析处理，具体参照源码
            		
                // 添加解析 SQL
                configuration.addMappedStatement(mappedStatement);
            }

            // 注册Mapper映射器
            configuration.addMapper(Resources.classForName(namespace));
        }
    }
    
}
```
在配置类中添加映射器注册机和映射语句的存放；
- 映射器注册机是我们上一章节实现的内容，用于注册 Mapper 映射器锁提供的操作类。
- 另外一个 MappedStatement 是本章节新添加的 SQL 信息记录对象，包括记录：SQL类型、SQL语句、入参类型、出参类型等。
## DefaultSqlSession 从 Configuration 类中获取 mapper
```java
public class DefaultSqlSession implements SqlSession {

    private Configuration configuration;

    @Override
    public <T> T selectOne(String statement, Object parameter) {
        MappedStatement mappedStatement = configuration.getMappedStatement(statement);
        return (T) ("你被代理了！" + "\n方法：" + statement + "\n入参：" + parameter + "\n待执行SQL：" + mappedStatement.getSql());
    }

    @Override
    public <T> T getMapper(Class<T> type) {
        return configuration.getMapper(type, this);
    }

}
```
- `DefaultSqlSession` 相对于上一章节，这里把 `MapperRegistry mapperRegistry` 替换为 `Configuration configuration`，这样才能传递更丰富的信息内容，而不只是注册器操作。
- 之后在 `DefaultSqlSession#selectOne`、`DefaultSqlSession#getMapper` 两个方法中都使用 `configuration` 来获取对应的信息。
- 目前 `selectOne` 方法中只是把获取的信息进行打印，后续将引入 SQL 执行器进行结果查询并返回。
## 测试
